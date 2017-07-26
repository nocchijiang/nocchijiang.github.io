---
layout: post
title: ConcurrentHashMap 并发实现分析
tags: [Java]
---
* toc
{:toc}
## 初始化
`ConcurrentHashMap` 的默认构造器是空的，对相关数据结构的初始化会推迟到首次插入操作前。首先看一下相关的两个成员。

```java
transient volatile Node<K,V>[] table;
```
稍有数据结构基础知识的读者知道，散列表的底层数据结构就是数组 + 链表。Java 8 引入了一个当链表过长时将链表红黑树化的性能优化，以期提高散列冲突严重时的查找效率，不过本篇文章的重点不在数据结构，不会过度关注它，**为了简洁和突出重点，下文中的源码也会直接删去与树化相关的分支和语句**。

```java
private transient volatile int sizeCtl;
```
`sizeCtl` 是一个用来标识 `ConcurrentHashMap` 初始化和扩容的状态变量，初始化时它的值为 `-1`，扩容时也是一个负值（具体是多少对理解初始化来说并不重要）。

```java
private final Node<K,V>[] initTable() {
  Node<K,V>[] tab; int sc;
  while ((tab = table) == null || tab.length == 0) {
    if ((sc = sizeCtl) < 0)
      Thread.yield(); // lost initialization race; just spin
    else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
      try {
        if ((tab = table) == null || tab.length == 0) {
          int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
          @SuppressWarnings("unchecked")
          Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
          table = tab = nt;
          sc = n - (n >>> 2);
        }
      } finally {
        sizeCtl = sc;
      }
      break;
    }
  }
  return tab;
}
```
初始化的步骤如上方 `initTable()` 所示。进入对桶数组 `table` 的判空自旋，进一步判断 `sizeCtl` 是否小于 `0`，即 `table` 正被其它线程初始化：如果是，则说明其它线程正在初始化 `table`，自旋到初始化完成为止，返回即可；否则，CAS 设 `sizeCtl` 为 `-1`，标记正在执行初始化 `table` 操作。CAS 成功后，再一次对 `table` 判空，才执行真正的初始化，这与常见的 DCL + `volatile` 保护单例的*反模式*有异曲同工之妙。
## 查询

```java
public V get(Object key) {
  Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
  int h = spread(key.hashCode());
  if ((tab = table) != null && (n = tab.length) > 0 &&
    (e = tabAt(tab, (n - 1) & h)) != null) {
    if ((eh = e.hash) == h) {
      if ((ek = e.key) == key || (ek != null && key.equals(ek)))
        return e.val;
    }
    else if (eh < 0)
      return (p = e.find(h, key)) != null ? p.val : null;
    while ((e = e.next) != null) {
      if (e.hash == h &&
        ((ek = e.key) == key || (ek != null && key.equals(ek))))
        return e.val;
    }
  }
  return null;
}
```
读者已经看到 `table` 被 `volatile` 修饰，然而对数组而言，`volatile` 的语义覆盖不到对其元素的访问。`ConcurrentHashMap` 使用 `Unsafe` 提供的低级原语来访问桶数组，从而实现免锁对数组元素的 `volatile` 语义下的访问（`java.util.concurrent.atomic.AtomicReferenceArray` 也使用了相同的实现）。

```java
@SuppressWarnings("unchecked")
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
  return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```
如此，对桶的读操作实现免锁。

细心的读者应该发现，`get` 中对散列值为负数的桶视为特殊情形处理，这是 `ConcurrentHashMap` 实现在扩容散列表的同时支持并发读写的关键。`ConcurrentHashMap` 在插入操作时使用 `spread()` 将散列值的二进制首位置 0，因此，普通桶的散列值必定是非负数，查询这个桶中的结点时走的是普通散列表的链表查询；散列值为负数时，调用桶结点的 `find()` 代为完成查询。从 OO 的角度来看，`find` 对 `get` 屏蔽了 `ConcurrentHashMap` 实现并发读的细节。

```java
// 笔者注：HASH_BITS = 0x7fffffff
static final int spread(int h) {
  return (h ^ (h >>> 16)) & HASH_BITS;
}
```

## 插入
Java 1.5 新的 `java.util.concurrent.ConcurrentMap` 接口定义了 `putIfAbsent()`、 `getOrDefault()` 等方法，消灭了客户代码手动加锁 *check-then-act* 的必要。本节关注的是插入操作，`ConcurrentHashMap` 的实现中，`put()` 与 `putIfAbsent()` 均走到了同一个包权限方法中，区别仅在布尔入参 `onlyIfAbsent`。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
  if (key == null || value == null) throw new NullPointerException();
  int hash = spread(key.hashCode());
  int binCount = 0;
  for (Node<K,V>[] tab = table;;) {
    Node<K,V> f; int n, i, fh;
    if (tab == null || (n = tab.length) == 0)
      tab = initTable();
    else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // <-- 空桶
      if (casTabAt(tab, i, null,
             new Node<K,V>(hash, key, value, null)))
        break;           // no lock when adding to empty bin
    }
    else if ((fh = f.hash) == MOVED) // <-- 扩容中
      tab = helpTransfer(tab, f);
    else { // <-- 常规情形
      V oldVal = null;
      synchronized (f) {
        if (tabAt(tab, i) == f) {
          if (fh >= 0) {
            binCount = 1;
            for (Node<K,V> e = f;; ++binCount) {
              K ek;
              if (e.hash == hash &&
                ((ek = e.key) == key ||
                 (ek != null && key.equals(ek)))) {
                oldVal = e.val;
                if (!onlyIfAbsent)
                  e.val = value;
                break;
              }
              Node<K,V> pred = e;
              if ((e = e.next) == null) {
                pred.next = new Node<K,V>(hash, key, value, null);
                break;
              }
            }
          }
        }
      }
      if (binCount != 0) {
        if (oldVal != null)
          return oldVal;
        break;
      }
    }
  }
  addCount(1L, binCount);
  return null;
}
```
### 插入前
一次插入操作可分为以下几种情况处理。
#### 桶中没有结点
这种情况最为简单，直接 CAS 设 `table` 数组该位置为一个新构造的结点即可，不需额外加锁；如果 CAS 失败，说明其它线程抢先构造了一个新的桶结点，当前线程只需自旋进入另一处理分支。
#### 桶中有结点，并且散列表不在扩容
这是最普遍的情况。`ConcurrentHashMap` 使用每个桶对应的链表的首个结点作为修改此链表的锁，加锁后，先确保链表表头没有发生变化，然后执行查链表、赋值操作（又有 DCL + `volatile` 的影子，对吧）。`Node.val` 被 `volatile` 修饰，因此 `e.val = value`（修改一个已经存在的结点维护的键值映射关系）的线程安全得以保障。
#### 桶中有结点，并且散列表正在扩容
调用 `helpTransfer()` 来辅助扩容过程，然后自旋。扩容的详细过程笔者将在[下文](#helpTransfer)中分析，读者在这里只需注意进入此分支的条件：桶结点的散列值等于*常量* `MOVED`。
### 插入后
调用 `addCount()` 方法维护散列表中的表项数量和执行可能需要的扩容操作。
## 计数
如果只使用一个变量来维护整个 Map 中的表项数量，则在进行插入操作时，维护 Map 中的表项数量的操作将不可避免地变为同步操作。阿姆达尔定律告诉我们，程序中串行运行的部分会成为性能优化的瓶颈，故 `ConcurrentHashMap` 采取了一个折衷的做法，这套高性能并发累加求和算法在 Java 8 时由一个 `public` 类 `java.util.concurrent.atomic.LongAdder` 向客户代码公开。本文不关心它的详细实现。

```java
static final class CounterCell {
  volatile long value;
  CounterCell(long x) { value = x; }
}
private transient volatile long baseCount;
private transient volatile int cellsBusy;
private transient volatile CounterCell[] counterCells;

```

### 查询

```java
@Override
public int size() {
  long n = sumCount();
  return ((n < 0L) ? 0 :
          (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
          (int)n);
}

final long sumCount() {
  CounterCell[] as = counterCells; CounterCell a;
  long sum = baseCount;
  if (as != null) {
    for (int i = 0; i < as.length; ++i) {
      if ((a = as[i]) != null)
        sum += a.value;
    }
  }
  return sum;
}
```
可见查询到的大小在并发环境下**很可能**是不准确的，在遍历 `counterCells` 时并未加锁，只利用 `volatile` 语义保证在遍历到其中一个 `CounterCell` 时获取的一个 `cell` 中的值是最新的。
### 维护
`ConcurrentHashMap` 的 `put()` 和 `remove()` 操作都使用 `addCount()` 来维护大小信息，区别仅在于入参的 `x` 的正负。

```java
private final void addCount(long x, int check) {
  CounterCell[] as; long b, s;
  if ((as = counterCells) != null ||
    !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
    CounterCell a; long v; int m;
    boolean uncontended = true;
    if (as == null || (m = as.length - 1) < 0 ||
      (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
      !(uncontended =
        U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
      fullAddCount(x, uncontended);
      return;
    }
    if (check <= 1)
      return;
    s = sumCount();
  }
  if (check >= 0) {
    Node<K,V>[] tab, nt; int n, sc;
    while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
         (n = tab.length) < MAXIMUM_CAPACITY) {
      int rs = resizeStamp(n);
      if (sc < 0) {
        if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
          sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
          transferIndex <= 0)
          break;
        if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
          transfer(tab, nt);
      }
      else if (U.compareAndSwapInt(this, SIZECTL, sc,
                     (rs << RESIZE_STAMP_SHIFT) + 2))
        transfer(tab, null);
      s = sumCount();
    }
  }
}
```
可以看出 `addCount()` 方法体由两个 `if` 块组成，它们分别代表 `addCount()` 要完成的两个工作：维护大小计数、检查是否需要变更桶数量。

`addCount()` 首先判 `counterCells` 是否非空，如果为空，则尝试 CAS 更新 `baseCount`。可见，在没有发生竞争的情况下，`ConcurrentHashMap` 仅使用 `baseCount` 维护大小信息（与 `HashMap` 相似）。如果对 `baseCount` 的 CAS 失败，则说明存在竞争，改用 `counterCells` 维护。

## 扩容

### 尝试
`addCount()` 另一入参 `check` 用于指定是否尝试扩容 `table` 数组，如果 `check < 0`，则不尝试；如果 `0 <= check <= 1` 则在 `ConcurrentHashMap` 不被多个线程竞争时尝试；否则总是尝试。通过 `put()` 或 `remove()` 进入 `addCount()` 传入的 `check` 均为非负值。

`sizeCtl` 在 `table` 初始化和变更大小时均为负值；在普通情况下它等于变更 `table` 尺寸的阈值。如果更新计数后，散列表中的映射数量超过了阈值，且 `table` 数组长度小于允许的最大值（`1 << 30`），则判定需要为 `table` 扩容。

首先来看 `sizeCtl >= 0`（普通情况）的情形。`ConcurrentHashMap` 首先计算了局部变量 `rs` 的值：

```java
// 笔者注：RESIZE_STAMP_BIT = 16，入参为当前 table 数组长度
static final int resizeStamp(int n) {
  return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```
然后以 CAS 方式更新 `sizeCtl` 的值为 `(rs << RESIZE_STAMP_SHIFT) + 2`（`RESIZE_STAM_SHIFT = 32 - RESIZE_STAMP_BIT`），一旦成功则开始 `transfer` 过程，否则重新执行扩容条件检查。

`resizeStamp` 生成的值就如其名，它是 `table` 数组长度的『邮戳（stamp）』，将邮戳值左移 `RESIZE_STAMP_SHIFT` 位，意在把邮戳值放到 `sizeCtl` 的高 16 位。读者可以尝试运行如下代码，即可观察到阈值、邮戳值和变更大小时对应的 `sizeCtl` 的特征。可以看到，`table` 数组长度与其邮戳值满足一对一的映射关系。

```java
int i = 16;
do {
  System.out.println(i);
  System.out.println(Integer.toBinaryString(resizeStamp(i)));
  System.out.println(Integer.toBinaryString((resizeStamp(i) << RESIZE_STAMP_SHIFT) + 2));
  System.out.println();
  i <<= 2;
} while (i < (1 << 30)); }

/* 
output:
16
1000000000011011
10000000000110110000000000000010

32
1000000000011010
10000000000110100000000000000010

64
1000000000011001
10000000000110010000000000000010

128
1000000000011000
10000000000110000000000000000010

256
1000000000010111
10000000000101110000000000000010

512
1000000000010110
10000000000101100000000000000010

1024
1000000000010101
10000000000101010000000000000010

2048
1000000000010100
10000000000101000000000000000010

4096
1000000000010011
10000000000100110000000000000010

8192
1000000000010010
10000000000100100000000000000010

16384
1000000000010001
10000000000100010000000000000010

32768
1000000000010000
10000000000100000000000000000010

65536
1000000000001111
10000000000011110000000000000010

131072
1000000000001110
10000000000011100000000000000010

262144
1000000000001101
10000000000011010000000000000010

524288
1000000000001100
10000000000011000000000000000010

1048576
1000000000001011
10000000000010110000000000000010

2097152
1000000000001010
10000000000010100000000000000010

4194304
1000000000001001
10000000000010010000000000000010

8388608
1000000000001000
10000000000010000000000000000010

16777216
1000000000000111
10000000000001110000000000000010

33554432
1000000000000110
10000000000001100000000000000010

67108864
1000000000000101
10000000000001010000000000000010

134217728
1000000000000100
10000000000001000000000000000010

268435456
1000000000000011
10000000000000110000000000000010

536870912
1000000000000010
10000000000000100000000000000010
*/
```
为什么还对左位移后的结果 `+ 2`？阅读 `ConcurrentHashMap` 对 `sizeCtl` 的定义便知，在变更大小时，`sizeCtl` 的高 16 位存放的是变更大小前 `table` 数组长度的邮戳，低 16 位存放的是 `(参与到扩容过程的线程数 + 1)`，而刚刚开始 `transfer` 时自然只有一条线程参与到扩容过程。`ConcurrentHashMap` 限制了同时参加到扩容过程的线程的数量，但生产环境中很难想象要开如此多的线程。

```java
// 笔者注：MAX_RESIZERS = 65535
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
```

再来看 `sizeCtl < 0` （当前有其它线程正在 `transfer`）的情形。同样，`ConcurrentHashMap` 首先计算 `rs`，然后执行一个复杂的短路逻辑：

* `(sc >>> RESIZE_STAMP_SHIFT) != rs`：`sizeCtl` 高 16 位与当前 `table` 数组长度的邮戳不相等，这表明当前有一个与刚刚拿到的 `table` 数组长度不匹配的进行中的扩容过程，无法『帮助』扩容
* `sc == rs + 1` 与 `sc == rs + MAX_RESIZERS`：这两个条件令笔者百思不得其解。笔者认定这两个条件肯定无法满足，理由是：`sc` 是负数，而 `rs` 肯定是正数，`MAX_RESIZER` 等于 `65535`。
* `(nt = nextTable) == null`：新散列表 `nextTable` 尚未被首个发起扩容的线程初始化完成，无法『帮助』扩容
* `transferIndex <= 0`：扩容工作已经全部开始，无法『帮助』扩容。下文中会对 `transferIndex` 的意义作详细解释。

### the dirty work
`ConcurrentHashMap` 扩容的过程发生在两处，一处主要流程在 `transfer` 方法中，这个方法比较长，笔者把它分段粘贴到本文中。另一处位于 `helpTransfer` 方法，读者已经在 [`putVal()`](#插入) 中看到它被调用了一次。

#### `transfer`
`transfer` 方法有两个入参，第一个是待扩容的的 `table` 数组，第二个是扩容使用的 `nextTable` 数组，扩容完成后，直接用后者的引用替换前者。从普通状态发起扩容时，第二个入参为空，由这个线程负责对 `nextTable` 的初始化：

```java
if (nextTab == null) {      // initiating
  try {
    @SuppressWarnings("unchecked")
    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
    nextTab = nt;
  } catch (Throwable ex) {    // try to cope with OOME
    sizeCtl = Integer.MAX_VALUE;
    return;
  }
  nextTable = nextTab;
  transferIndex = n;
}
```
可见，`ConcurrentHashMap` 每次扩容都是当前 `table` 数组的两倍。熟悉 `HashMap` 实现的读者应该知道，这种以 2 的指数扩容的模式的一大优点就是原来位于桶 `n` 的结点，在扩容后要么还位于桶 `n`，要么位于新桶 `(n + 扩容前桶数量)`。不过，`ConcurrentHashMap` 与 `HashMap` 不同的是，前者在转移结点时并非像后者那样，简单地顺序遍历桶数组，`transfer` 会计算 `stride` 来确定两次桶结点转移间的跨距。

```java
// 笔者注：MIN_TRANSFER_STRIDE = 16
int n = tab.length, stride;
if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
  stride = MIN_TRANSFER_STRIDE; // subdivide range
```

静态常量 `NCPU` 的值通过 `Runtime.getRuntime().availableProcessors()` 取得，经笔者验证，对于带超线程的 CPU，它取到的值也包含由超线程带来的『核心』数。可见，在单核环境下，`stride` 始终等于旧 `table` 数组的长度；在多核环境下，`stride` 由旧 `table` 数组长度与核心数确定。

*为什么要设置遍历的跨距？*这个答案会在后文中揭晓。

在确保 `nextTable` 已经初始化后，`transfer` 开始结点的转移过程。转移过程由一个 `for` 循环完成，循环体从功能上可以分为两个部分：

* `while (advance)` 循环：确定外层 `for` 循环该趟转移的桶结点的上下界
* `if` 块构成的剩余部分：结点转移真正发生的地方

进入循环之前，`transfer` 还初始化了这些局部变量：

```java
int nextn = nextTab.length;
ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
boolean advance = true;
boolean finishing = false; // to ensure sweep before committing nextTab
```
`nextn` 是一个*等效常量*，它记录新 `table` 数组的长度；`fwd` 结点是一个不持有实际数据的特殊桶结点，它是 `ConcurrentHashMap` 实现扩容时支持并发查询的重要工具；`advance` 与 `finishing` 均是控制执行流程的布尔标志，其中前者控制的是内部的 `while` 循环，后者的翻转标志着整个转移过程的结束。

```java
for (int i = 0, bound = 0;;) {
  Node<K,V> f; int fh;
  while (advance) {
    int nextIndex, nextBound;
    if (--i >= bound || finishing)
      advance = false;
    else if ((nextIndex = transferIndex) <= 0) {
      i = -1;
      advance = false;
    }
    else if (U.compareAndSwapInt
         (this, TRANSFERINDEX, nextIndex,
          nextBound = (nextIndex > stride ?
                 nextIndex - stride : 0))) {
      bound = nextBound;
      i = nextIndex - 1;
      advance = false;
    }
  }
```
前文已经提到，这个 `while` 循环的意义是确定外层 `for` 循环本趟准备转移的桶结点在旧 `table` 中的下标 `i`（同时还可能需要确定下界 `bound`），一旦确定了 `i`，`advance` 被置为 `false`，`while` 循环退出。循环有三个分支，分别代表三个退出条件：

* `--i >= bound || finishing`：`i` 自减 `1` 后尚未越界，或者 `finishing` 被置为 `true`。初次进入 `while` 循环时，该条件永远失败。
* `(nextIndex = transferIndex) <= 0`：要理解这个条件，需要先弄清 `transferIndex` 的意义。`transferIndex` 记录的是 `(当前桶结点转移过程的整体下界 - 1)`。`ConcurrentHashMap` 被设计为允许多个线程同时转移桶结点，以加速散列表的扩容过程。假设一个线程在判断该 `if` 条件时读到的 `transferIndex` 的值为 `x`，`table` 数组长度为 `n`，则说明 `table` 数组的后 `n - x` 个桶结点正在或已经被其它线程转移到新的 `nextTable` 数组。现在，`transfer()` 一开始计算 `stride` 的原因也得到了解释：在散列表长度足够大、对 `ConcurrentHashMap` 的插入冲突足够严重的情况下，`transfer` 通过降低每一趟 `for` 循环转移的桶结点数量，把结点转移的工作量平摊到多个线程中来提升扩容散列表的性能。
  
  在理解了 `transferIndex` 的意义之后，这个条件的意义就很明确了，`transferIndex <= 0` 说明旧 `table` 表中所有的桶结点都正在或已经被其它线程转移，当前线程没有办法『帮助』到扩容过程，于是 `transfer` 将 `i` 置为 `-1`（这是退出外围 `for` 循环的条件之一）。
* CAS 设 `transferIndex` 为 `transferIndex - stride`：如果成功，则确定了本趟 `for` 循环的起始下标 `i` 和下界 `bound`；失败则说明有其它线程已经抢先『申请成功』承担这一部分桶结点转移工作，当前线程自旋重试即可。

确定了 `i` 与 `bound` 后，真正的结点转移工作开始。

```java
  if (i < 0 || i >= n || i + n >= nextn) {
    int sc;
    if (finishing) {
      nextTable = null;
      table = nextTab;
      sizeCtl = (n << 1) - (n >>> 1);
      return;
    }
    if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
      if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
        return;
      finishing = advance = true;
      i = n; // recheck before commit
    }
  }
```
这个分支是整个 `for` 循环，乃至整个 `transfer` 退出的控制分支。

* `finishing` 被置为 `true`：结点转移工作完全完成，将新表引用赋给 `table`，清空 `nextTable`，更新下一次扩容的阈值（读者可以在这里看到，`ConcurrentHashMap` 的装填因子固定为 `0.75`），退出 `transfer`。
* CAS 更新 `sizeCtl` 的值为 `sizeCtl - 1`：当前线程完成了它分担的结点转移工作，`sizeCtl` 减 `1` 表示减少一个参加到扩容过程的线程（别忘了 `sizeCtl` 在扩容过程中低 16 位的意义）。如果 CAS 成功，检查它是否是最后一个参加扩容的线程，如果不是，直接退出 `transfer`；否则，置 `finishing` 与 `advance` 同时为 `true`，并置 `i` 为旧 `table` 长度 `n`，再一次倒序遍历旧 `table` 数组中的桶，验证它们是否已经全部转移到新的 `nextTable` 中，直至 `i` 再次越界，提交变更。

```java
  else if ((f = tabAt(tab, i)) == null)
    advance = casTabAt(tab, i, null, fwd);
  else if ((fh = f.hash) == MOVED)
    advance = true; // already processed
```
接下来的分支中 `i` 均没有越界。

* `f = tabAt(tab, i)) == null`：旧 `table` 数组中这个桶里没有结点，于是（*check-then-act*）CAS 往这个桶里放一个 `ForwardingNode` 结点。如果 CAS 失败，则说明有其它线程同时在这个桶中放入了结点，`advance` 置为 `false`，无需走上方 `while` 循环更新 `i`，简单自旋外层 `for`，进入另一个处理分支即可。
* 读者可能还不太明白 `ForwardingNode` 的意义，紧接着的这个分支就很好地解释了它的一个用途：`(fh = f.hash) == MOVED` 说明旧 `table` 数组中这个桶存放的是 `ForwardingNode` 结点（`ForwardingNode` 结点的 `hash` 域等于常量 `MOVED`），该桶中的结点已经被转移走（或者本来这个桶就是空桶，比如上面那种分支的情形）。

```java
  else {
    synchronized (f) {
      if (tabAt(tab, i) == f) {
        Node<K,V> ln, hn;
        if (fh >= 0) {
          int runBit = fh & n;
          Node<K,V> lastRun = f;
          for (Node<K,V> p = f.next; p != null; p = p.next) {
            int b = p.hash & n;
              runBit = b;
              lastRun = p;
            }
          }
          if (runBit == 0) {
            ln = lastRun;
            hn = null;
          }
          else {
            hn = lastRun;
            ln = null;
          }
          for (Node<K,V> p = f; p != lastRun; p = p.next) {
            int ph = p.hash; K pk = p.key; V pv = p.val;
            if ((ph & n) == 0)
              ln = new Node<K,V>(ph, pk, pv, ln);
            else
              hn = new Node<K,V>(ph, pk, pv, hn);
          }
          setTabAt(nextTab, i, ln);
          setTabAt(nextTab, i + n, hn);
          setTabAt(tab, i, fwd);
          advance = true;
        }
      }
    }
  }
}
```

这个看起来最冗长的分支，反而是最容易理解的一个：进入该分支则说明旧 `table` 数组的第 `i` 个桶里有待转移的结点。

`transfer` 对链表表头结点加监视器锁，如果同时有其它线程在对同一个桶做 `put` 操作，则 `transfer` 会阻塞到 `put` 结束；反之，如果 `transfer` 请求监视器锁成功，在转移这个桶中结点的过程中，对同一个桶的所有 `put` 操作都将阻塞。*但是*，`get` 操作仍可并发执行。
加完锁之后，`transfer` 执行了两趟链表遍历，目的是为了尽量减少新 `Node` 对象的创建，但整体上结点转移与 `HashMap` 仍高度相似。写操作均使用链表头结点作为互斥锁，故 `transfer` 对数组元素的赋值不需使用 CAS；但即使 `put` 与 `transfer` 加了相同的锁，使用由 `Unsafe` 原语提供的对*数组元素*的 `volatile` 访问来保证可见性仍是必须的。桶中结点全部转移后，`transfer` 将特殊结点 `fwd` 置于旧 `table` 桶中，这对并发的 `get`、`transfer` 乃至 `put` 的实现至关重要。

#### `helpTransfer`
这个方法会在 `put` 操作遇到散列值为 `MOVED` 的桶时被调用，其获取 `helpTransfer` 的返回值替换局部 `tab` 引用后，在（可能是 `nextTable` 的）`tab` 中查找满足条件的桶。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
  Node<K,V>[] nextTab; int sc;
  if (tab != null && (f instanceof ForwardingNode) &&
    (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
    int rs = resizeStamp(tab.length);
    while (nextTab == nextTable && table == tab &&
         (sc = sizeCtl) < 0) {
      if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
        sc == rs + MAX_RESIZERS || transferIndex <= 0)
        break;
      if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
        transfer(tab, nextTab);
        break;
      }
    }
    return nextTab;
  }
  return table;
}

```

只要理解了前文，理解这个方法的逻辑应该不难：在确保 `nextTable` 和 `table` 不变的情况下，CAS 修改 `sizeCtl` 以声明『有一个新的线程要参加到扩容过程中』，然后调用 `transfer`，返回新散列表的引用，供 `put` 在新散列表中做插入操作。

### `ForwardingNode`

看完了 `transfer` 之后，读者心中想必还有个疑问：既然有部分桶已经被转移到 `nextTable`，那 `get` 是如何得知哪些桶还在 `table` 中，哪些桶已经到了 `nextTable` 里？答案就在特殊结点 `ForwadingNode` 的实现中。

`ForwardingNode` 持有 `nextTable` 的引用，`hash` 域的值正是我们在 `get`、`put`、`transfer` 中已经见到数次的常量 `MOVED`。

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
  final Node<K,V>[] nextTable;
  ForwardingNode(Node<K,V>[] tab) {
    super(MOVED, null, null, null);
    this.nextTable = tab;
  }

  Node<K,V> find(int h, Object k) {
    // loop to avoid arbitrarily deep recursion on forwarding nodes
    outer: for (Node<K,V>[] tab = nextTable;;) {
      Node<K,V> e; int n;
      if (k == null || tab == null || (n = tab.length) == 0 ||
        (e = tabAt(tab, (n - 1) & h)) == null)
        return null;
      for (;;) {
        int eh; K ek;
        if ((eh = e.hash) == h &&
          ((ek = e.key) == k || (ek != null && k.equals(ek))))
          return e;
        if (eh < 0) {
          if (e instanceof ForwardingNode) {
            tab = ((ForwardingNode<K,V>)e).nextTable;
            continue outer;
          }
          else
            return e.find(h, k);
        }
        if ((e = e.next) == null)
          return null;
      }
    }
  }
}
```

`ForwardingNode.find` 将在 `nextTable` 中查找已经转移了的结点的过程封装起来，但本质上仍是链表查找。一个极端的情况是，`ConcurrentHashMap` 在该次 `get` 操作中发生了多次扩容，则在 `find` 的内层 `for` 循环中还会读到散列值小于 `0` 的桶结点。`find` 判断如果此结点依然是 `ForwardingNode`，则将其持有的 `nextTable` 引用取出，继续在这张表上查找。`ForwardingNode.nextTable` 被 `final` 关键字修饰，它的内存可见性可以得到保障。

---
layout: post
title: "LeakCanary 实现分析"
tags: [Android]
---

### 构件

在使用 LeakCanary 时，我们通常只会添加 `leakcanary-android` 作为项目依赖。实际上 LeakCanary 被拆分成了三个组成部分：

* `leakcanary-android`：供 Android 平台使用的用户界面、接入入口、配置定义、dump 实现。总体上可理解为 LeakCanary 对 Android 的适配和专有实现。依赖 `leakcanary-analyzer`。
* `leakcanary-analyzer`：针对 Android 平台特性，实现对 heap dump 的分析并输出结果。依赖 `leakcanary-watcher`。
* `leakcanary-watcher`：纯 Java Library，定义与平台无关的顶层接口和通用 Java 实现。

### 组件

`leakcanary-android` 注册了两个 Service 和两个 Activity，均独立被分析 App 主进程，运行于自己的进程。`HeapAnalyzerService` 顾名思义就是堆转储分析的入口了，稍后我们重点关注它；两个 Activity 方面，一个用于展示分析结果，另一个用于申请外存读写权限。这里比较有趣的一点是，默认状况下，这四个组件都是 `enabled=false` 的；在初始化时，它们会根据配置按需启用。

```xml
<manifest
    xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.squareup.leakcanary"
    >

  <!-- To store the heap dumps and leak analysis results. -->
  <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
  <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />

  <application>
    <service
        android:name=".internal.HeapAnalyzerService"
        android:process=":leakcanary"
        android:enabled="false"
        />
    <service
        android:name=".DisplayLeakService"
        android:process=":leakcanary"
        android:enabled="false"
        />
    <activity
        android:theme="@style/leak_canary_LeakCanary.Base"
        android:name=".internal.DisplayLeakActivity"
        android:process=":leakcanary"
        android:enabled="false"
        android:label="@string/leak_canary_display_activity_label"
        android:icon="@drawable/leak_canary_icon"
        android:taskAffinity="com.squareup.leakcanary.${applicationId}"
        >
      <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
      </intent-filter>
    </activity>
    <activity
        android:theme="@style/leak_canary_Theme.Transparent"
        android:name=".internal.RequestStoragePermissionActivity"
        android:process=":leakcanary"
        android:taskAffinity="com.squareup.leakcanary.${applicationId}"
        android:enabled="false"
        android:excludeFromRecents="true"
        android:icon="@drawable/leak_canary_icon"
        android:label="@string/leak_canary_storage_permission_activity_label"
        />

  </application>
</manifest>
```

### 入口与配置

接入过 LeakCanary 的开发者应该还记得，LeakCanary 的接入非常简单，在主进程 `Application#onCreate` 中 `LeakCanary.install(Application)` 即可。

```java
public static RefWatcher install(Application application) {
  return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
    .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
    .buildAndInstall();
}

public static AndroidRefWatcherBuilder refWatcher(Context context) {
  return new AndroidRefWatcherBuilder(context);
}
```

`RefWatcher` 是一个定义在 `leakcanary-watcher` 中的通用实现，它用于监听『应该变成**弱可达**』的引用。当垃圾回收器发现一个对象没有被强引用、软引用时，它就是弱可达的。`RefWatcherBuilder` 使用 Builder 模式定义了 `RefWatcher` 的可配置选项，现在先只关注我们在 `install` 中看到的两个：

* `listenerServiceClass`：这个选项其实是 `AndroidRefWatcherBuilder` 专有的，也就是 Android 平台特有的可配置项，用于监听 heap dump 分析的结果。默认的结果会通过 `Intent` 交给 `DisplayLeakService` 服务。这个服务是一个 `IntentService`，做的事情很简单，从 `Intent` 中解析 `HeapDump` 和 `AnalysisResult`，然后使用结果判断是否有泄露、（如果有泄露的话）发送通知、存储结果。
* `excludedRefs`：这是一个通用可配置选项，用于过滤特定的引用链，当分析发现此引用链上有泄露时，会自行过滤。该选项用于排除已知的泄露，`AndroidExcludedRefs` 中提供了许多已知的 SDK 泄露和特定厂家的系统泄露。

`buildAndInstall` 首先构造 `RefWatcher`，然后在后台线程启用用于展示分析结果的 `DisplayLeakActivity`（via `PackageManager#setComponentEnabledSetting`）；最后，注册一个 `Application.LifecycleCallbacks`，用于监听 Activity 的 `onDestroy`，将 Activity 的引用交给 `RefWatcher` 进行监视。

### 监视

我们知道，在没有发生内存泄露的情况下，Activity 在销毁后就应该变为**弱可达**，LeakCanary 通过分析 heap dump 来确认这一点。

```java
void onActivityDestroyed(Activity activity) {
  refWatcher.watch(activity);
}
```

Activity 被销毁后，其引用被传入 `RefWatcher#watch`。本质上，`RefWatcher` 可用来监听任何对象，而在 Android 平台，我们通常关心的就是 Activity 对象是否泄露。

```java
public void watch(Object watchedReference) {
  watch(watchedReference, "");
}

public void watch(Object watchedReference, String referenceName) {
  if (this == DISABLED) {
    return;
  }
  checkNotNull(watchedReference, "watchedReference");
  checkNotNull(referenceName, "referenceName");
  final long watchStartNanoTime = System.nanoTime();
  String key = UUID.randomUUID().toString();
  retainedKeys.add(key);
  final KeyedWeakReference reference =
      new KeyedWeakReference(watchedReference, key, referenceName, queue);

  ensureGoneAsync(watchStartNanoTime, reference);
}

private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
  watchExecutor.execute(new Retryable() {
    @Override public Retryable.Result run() {
        return ensureGone(reference, watchStartNanoTime);
    }
  });
}
```

`watch` 记录了开始监视的时间，为其生成了一个 UUID，并将其加入到 `retainedKeys` 中（后者是一个 `Set<String>`），然后生成对应的 `KeyedWeakReference`，传入 `ensureGoneAsync` 开始对该对象的异步分析。异步分析进行在一个 `WatchExecutor` 上，`leakcanary-android` 中的默认实现是 `AndroidWatchExecutor`，新开了一个 `HandlerThread`（也就是说，默认的分析行为是在后台串行执行的）。

```java
@SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
  long gcStartNanoTime = System.nanoTime();
  long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

  removeWeaklyReachableReferences();

  if (debuggerControl.isDebuggerAttached()) {
    // The debugger can create false leaks.
    return RETRY;
  }
  if (gone(reference)) {
    return DONE;
  }
  gcTrigger.runGc();
  removeWeaklyReachableReferences();
  if (!gone(reference)) {
    long startDumpHeap = System.nanoTime();
    long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

    File heapDumpFile = heapDumper.dumpHeap();
    if (heapDumpFile == RETRY_LATER) {
      // Could not dump the heap.
      return RETRY;
    }
    long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
    heapdumpListener.analyze(
        new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
            gcDurationMs, heapDumpDurationMs));
  }
  return DONE;
}
  
private boolean gone(KeyedWeakReference reference) {
  return !retainedKeys.contains(reference.key);
}

private void removeWeaklyReachableReferences() {
  // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
  // reachable. This is before finalization or garbage collection has actually happened.
  KeyedWeakReference ref;
  while ((ref = (KeyedWeakReference) queue.poll()) != null) {
    retainedKeys.remove(ref.key);
  }
}
```

`RefWatcher` 准备了一个 `ReferenceQueue`，用来接收 `KeyedWeakReference` 引用的对象被回收的通知。`removeWeaklyReachableReferences` 将进入到 `queue` 中的 `KWR` 对应 `key` 从 `retainedKeys` 中移除，以标记对应对象确实被回收。

首次 `rWRR` 调用显然是一个『短路作弊』行为——因为谁也不知道，在这段时间内，被监视的对象是否会被 GC 回收，如果真的回收了（也就是 `KeyedWeakReference` 入队 `RefWatcher#queue`），那当然是件好事；但这么美好的事情显然不会经常发生，因此，在短路逻辑失败后，LeakCanary 会发起一次 GC 过程。

```java
GcTrigger DEFAULT = new GcTrigger() {
  @Override public void runGc() {
    // Code taken from AOSP FinalizationTest:
    // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
    // java/lang/ref/FinalizationTester.java
    // System.gc() does not garbage collect every time. Runtime.gc() is
    // more likely to perfom a gc.
    Runtime.getRuntime().gc();
    enqueueReferences();
    System.runFinalization();
  }

  private void enqueueReferences() {
    // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
    // references to the appropriate queues.
    try {
      Thread.sleep(100);
    } catch (InterruptedException e) {
      throw new AssertionError();
    }
  }
};
```

触发 GC 后，LeakCanary 强制等待 100ms，并触发执行 finalizer（为什么？），完毕后再次过滤进入 `queue` 的弱可达对象，如果此时经过 `rWWR` 过滤后，被监视对象对应的 key 仍存在于 `retainedKeys` 中，这个对象则**有可能**泄露（我们知道，`System#gc` 没有任何保证）。考虑最坏的情况（也就是真的发生泄露），LeakCanary 随即发起一次 heap dump 及分析过程。

### 分析

在 Android 平台上，`leakcanary-android` 使用了 `android.os.Debug#dumpHprofData` 来生成 heap dump（此外还包含一些申请外存权限、弹 Toast 的代码，此处不再赘述），结果以 `File` 的形式传给 `HeapDump.Listener` 的实现，在 Android 平台上，`leakcanary-android` 中的实现是 `ServiceHeapDumpListener`，其又间接将相关信息通过 `Intent` 传递给 `HeapAnalyserService`。

```java
@Override protected void onHandleIntent(Intent intent) {
  if (intent == null) {
    CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
    return;
  }
  String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
  HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

  HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);

  AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);
  AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
}
```

可见，实际执行分析的逻辑被 `HeapAnalyzer` 封装，此类位于 `leakcanary-analyzer` 插件中。`HeapAnalyzer` 本质上封装了 square 另一个 Android heap 分析 library [haha](https://github.com/square/haha) ，而 haha 本质上又是对另一些内存分析工具的二次封装。

```java
public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
  long analysisStartNanoTime = System.nanoTime();

  if (!heapDumpFile.exists()) {
    Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
    return failure(exception, since(analysisStartNanoTime));
  }

  try {
    HprofBuffer buffer = new MemoryMappedFileBuffer(heapDumpFile);
    HprofParser parser = new HprofParser(buffer);
    Snapshot snapshot = parser.parse();
    deduplicateGcRoots(snapshot);

    Instance leakingRef = findLeakingReference(referenceKey, snapshot);

    // False alarm, weak reference was cleared in between key check and heap dump.
    if (leakingRef == null) {
      return noLeak(since(analysisStartNanoTime));
    }

    return findLeakTrace(analysisStartNanoTime, snapshot, leakingRef);
  } catch (Throwable e) {
    return failure(e, since(analysisStartNanoTime));
  }
}
```

`checkForLeak` 大致可分为三步：

* 解析 hprof，生成 `Snapshot`
* 使用[监视时](#监视)生成的 UUID 寻找对应 `KeyedWeakReference` 对象，其 `referent` 即为泄露的对象
* 如果 `referent` 非空，生成泄露引用链，否则为误报

本文不关心解析过程，而第二步非常简单，因此我们跳到最后一步。

```java
private AnalysisResult findLeakTrace(long analysisStartNanoTime, Snapshot snapshot,
    Instance leakingRef) {

  ShortestPathFinder pathFinder = new ShortestPathFinder(excludedRefs);
  ShortestPathFinder.Result result = pathFinder.findPath(snapshot, leakingRef);

  // False alarm, no strong reference path to GC Roots.
  if (result.leakingNode == null) {
    return noLeak(since(analysisStartNanoTime));
  }

  LeakTrace leakTrace = buildLeakTrace(result.leakingNode);

  String className = leakingRef.getClassObj().getClassName();

  // Side effect: computes retained size.
  snapshot.computeDominators();

  Instance leakingInstance = result.leakingNode.instance;

  long retainedSize = leakingInstance.getTotalRetainedSize();

  // TODO: check O sources and see what happened to android.graphics.Bitmap.mBuffer
  if (SDK_INT <= N_MR1) {
    retainedSize += computeIgnoredBitmapRetainedSize(snapshot, leakingInstance);
  }

  return leakDetected(result.excludingKnownLeaks, className, leakTrace, retainedSize,
      since(analysisStartNanoTime));
}
```

`ShortestPathFinder` 从 GC Root 出发，遍历对象图，搜索 `referent` 对应的 `Instance`，以确定引用路径。这部分逻辑涉及到较多琐碎的基础数据结构设计，笔者会一笔带过，读者若感兴趣，可自行查阅源码。

```java
Result findPath(Snapshot snapshot, Instance leakingRef) {
  clearState();
  canIgnoreStrings = !isString(leakingRef);

  enqueueGcRoots(snapshot);

  boolean excludingKnownLeaks = false;
  LeakNode leakingNode = null;
  while (!toVisitQueue.isEmpty() || !toVisitIfNoPathQueue.isEmpty()) {
    LeakNode node;
    if (!toVisitQueue.isEmpty()) {
      node = toVisitQueue.poll();
    } else {
      node = toVisitIfNoPathQueue.poll();
      if (node.exclusion == null) {
        throw new IllegalStateException("Expected node to have an exclusion " + node);
      }
      excludingKnownLeaks = true;
    }

    // Termination
    if (node.instance == leakingRef) {
      leakingNode = node;
      break;
    }

    if (checkSeen(node)) {
      continue;
    }

    if (node.instance instanceof RootObj) {
      visitRootObj(node);
    } else if (node.instance instanceof ClassObj) {
      visitClassObj(node);
    } else if (node.instance instanceof ClassInstance) {
      visitClassInstance(node);
    } else if (node.instance instanceof ArrayInstance) {
      visitArrayInstance(node);
    } else {
      throw new IllegalStateException("Unexpected type for " + node.instance);
    }
  }
  return new Result(leakingNode, excludingKnownLeaks);
}
```

`ShortestPathFinder` 准备了两个 `Deque<LeakNode>`：`toVisitQueue` 与 `toVIsitIfNoPathQueue`，同时有两个 `Set<LeakNode>` `visitedSet` 和 `visitedIfNoPathSet`，用于滤掉已经访问过的结点。首先，`enqueueGcRoots` 将 GC ROOT 加入双向队列，然后执行对对象图的 BFS。`LeakNode` 持有结点图中入边的源头（即引用该对象的对象的 `LeakNode`），因此任一 `LeakNode` 均可追溯到 GC ROOT，获得了 `LeakNode` 即可得之其引用链。在遍历过程中，命中 `excludedRefs` 的路径会被忽略，而默认排除规则中就包含排除各类 `java.lang.ref.Reference` 的子类的规则，因此最终找出的泄露引用链势必是一条强引用链。

* 如果找到了 `referent` 对应的 `Instance`（`node.instance == leakingRef`），则在对象图中找到了泄露的对象，退出；
* 如果已经访问过此结点（`checkSeen(node)`），直接看下一个队列中的结点；
* 否则，根据结点的实例类型，执行对应的结点访问操作（`visit*`）。
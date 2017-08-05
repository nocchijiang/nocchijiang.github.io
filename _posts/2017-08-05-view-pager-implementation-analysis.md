---
layout: post
title: ViewPager 实现分析
tags: [Android, View]
---
* toc
{:toc}

本文中的代码，除特别说明外，均**顺序**截取自小节对应标题的方法。

本文中的代码截取自 Support Library 25.4.0 与 SDK Version 25。

## 测量及布局

### `onMeasure`

作为一个 `ViewGroup`，其应该测量自己的所有子视图，并确定自身的大小；而 `ViewPager` 的测量过程与常见的 `ViewGroup` 不太一样，其子视图的大小不会对 `ViewPager` 自身的大小产生任何影响。

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
  setMeasuredDimension(getDefaultSize(0, widthMeasureSpec),
      getDefaultSize(0, heightMeasureSpec));
```

`getDefaultSize` 是定义在 `View` 类中的便利方法，它的实现很直白，即如果父视图指定的测量规格为 `UNSPECIFIED`，则返回传入的 `size`；否则使用父视图在 `measureSpec` 中指定的 `specSize`。`ViewPager` 传入的 `size` 均为 `0`，某种意义上说这个值传多少都没有意义，`ViewPager` 的目的是『父视图要我多大，我就多大』。

```java
public static int getDefaultSize(int size, int measureSpec) {
  int result = size;
  int specMode = MeasureSpec.getMode(measureSpec);
  int specSize = MeasureSpec.getSize(measureSpec);

  switch (specMode) {
  case MeasureSpec.UNSPECIFIED:
    result = size;
    break;
  case MeasureSpec.AT_MOST:
  case MeasureSpec.EXACTLY:
    result = specSize;
    break;
  }
  return result;
}
```

`ViewPager` 把自身区域的左右各 `mGutterSize` 像素的区域定义为 Gutter，这个间距的大小取自身宽度结果除以 10 与缺省值中更小的那一个。在[拦截触摸手势](#`onInterceptTouchEvent`)时，Gutter 区总是被视为 `ViewPager` 的可拖拽区域，从该区域触发的横向拖拽无法传递到子视图。

```java
// 笔者备注：mDefaultGutterSize 的值为 16dp，根据设备分辨率已换算为 px 
final int measuredWidth = getMeasuredWidth();
final int maxGutterSize = measuredWidth / 10;
mGutterSize = Math.min(maxGutterSize, mDefaultGutterSize);
```

接下来 `ViewPager` 会测量 `DecorView`。`ViewPager` 把它的子视图分为了两类，一类即为刚刚提到的 `DecorView`，另一类则是普通子视图，这个信息由 `ViewPager.LayoutParams.isDecor` 携带。`DecorView` 是不由 `PagerAdapter` 提供的子视图，并且它们占据的空间无法被普通子视图使用。任何位于布局 XML 文件 `ViewPager` 结点中的子结点所代表的视图均被视为**可能的** `DecorView`（这个视图的 Java class 还需要被 `@ViewPager.DecorView` 标注）。

```java
int childWidthSize = measuredWidth - getPaddingLeft() - getPaddingRight();
int childHeightSize = getMeasuredHeight() - getPaddingTop() - getPaddingBottom();

int size = getChildCount();
for (int i = 0; i < size; ++i) {
  final View child = getChildAt(i);
  if (child.getVisibility() != GONE) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    if (lp != null && lp.isDecor) {
      final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
      final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;
      int widthMode = MeasureSpec.AT_MOST;
      int heightMode = MeasureSpec.AT_MOST;
      boolean consumeVertical = vgrav == Gravity.TOP || vgrav == Gravity.BOTTOM;
      boolean consumeHorizontal = hgrav == Gravity.LEFT || hgrav == Gravity.RIGHT;

      if (consumeVertical) {
        widthMode = MeasureSpec.EXACTLY;
      } else if (consumeHorizontal) {
        heightMode = MeasureSpec.EXACTLY;
      }

      int widthSize = childWidthSize;
      int heightSize = childHeightSize;
      if (lp.width != LayoutParams.WRAP_CONTENT) {
        widthMode = MeasureSpec.EXACTLY;
        if (lp.width != LayoutParams.MATCH_PARENT) {
          widthSize = lp.width;
        }
      }
      if (lp.height != LayoutParams.WRAP_CONTENT) {
        heightMode = MeasureSpec.EXACTLY;
        if (lp.height != LayoutParams.MATCH_PARENT) {
          heightSize = lp.height;
        }
      }
      final int widthSpec = MeasureSpec.makeMeasureSpec(widthSize, widthMode);
      final int heightSpec = MeasureSpec.makeMeasureSpec(heightSize, heightMode);
      child.measure(widthSpec, heightSpec);

      if (consumeVertical) {
        childHeightSize -= child.getMeasuredHeight();
      } else if (consumeHorizontal) {
        childWidthSize -= child.getMeasuredWidth();
      }
    }
  }
}
```

`onMeasure` 根据配置在 `LayoutParams` 中的 `gravity` 确定 `DecorView` 在横、纵向上的测量规格，完成测量后，从 `childHeightSize` 和 `childWidthSize` 中减去 `DecorView` 消费了的宽度和高度。

`onMeasure` 首先测量 `DecorView` 的意图也是非常明确的：`DecorView` 占用的空间无法被普通子视图利用，因此要先测量 `DecorView`，才能确定普通子视图可用的空间，如下所示，所有普通子视图的测量规格均使用 `mChild*MeasureSpec`（宽度稍有不同，因为 `ViewPager` 允许不同的 `Page` 拥有不同的宽度系数）。

```java
mChildWidthMeasureSpec = MeasureSpec.makeMeasureSpec(childWidthSize, MeasureSpec.EXACTLY);
mChildHeightMeasureSpec = MeasureSpec.makeMeasureSpec(childHeightSize, MeasureSpec.EXACTLY);

mInLayout = true;
populate();
mInLayout = false;

size = getChildCount();
for (int i = 0; i < size; ++i) {
  final View child = getChildAt(i);
  if (child.getVisibility() != GONE) {
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    if (lp == null || !lp.isDecor) {
      final int widthSpec = MeasureSpec.makeMeasureSpec(
          (int) (childWidthSize * lp.widthFactor), MeasureSpec.EXACTLY);
      child.measure(widthSpec, mChildHeightMeasureSpec);
    }
  }
}
```

想必读者的注意力都被这三行代码吸引住了。[`populate`](#管理子视图) 做了什么？`mInLayout` 标记的用途是什么？这里我们暂时忽略它们的存在，读者暂时只需知道，`populate` 从 `PagerAdapter` 中请求了待展示的子视图，并添加到了 `ViewPager` 中。

```java
mInLayout = true;
populate();
mInLayout = false;
```

### `onLayout`

`ViewPager` 的布局过程同样是分为两趟，第一趟布局 `DecorView`，从而确定普通子视图的偏移量。两个 `switch` 块，在横、纵向分别根据配置的 `gravity` 属性确定该 `DecorView` 相对于 `ViewPager` 的 `left` 和 `top`。

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
  final int count = getChildCount();
  int width = r - l;
  int height = b - t;
  int paddingLeft = getPaddingLeft();
  int paddingTop = getPaddingTop();
  int paddingRight = getPaddingRight();
  int paddingBottom = getPaddingBottom();
  final int scrollX = getScrollX();

  int decorCount = 0;

  for (int i = 0; i < count; i++) {
    final View child = getChildAt(i);
    if (child.getVisibility() != GONE) {
      final LayoutParams lp = (LayoutParams) child.getLayoutParams();
      int childLeft = 0;
      int childTop = 0;
      if (lp.isDecor) {
        final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
        final int vgrav = lp.gravity & Gravity.VERTICAL_GRAVITY_MASK;
        switch (hgrav) {
          default:
            childLeft = paddingLeft;
            break;
          case Gravity.LEFT:
            childLeft = paddingLeft;
            paddingLeft += child.getMeasuredWidth();
            break;
          case Gravity.CENTER_HORIZONTAL:
            childLeft = Math.max((width - child.getMeasuredWidth()) / 2,
                paddingLeft);
            break;
          case Gravity.RIGHT:
            childLeft = width - paddingRight - child.getMeasuredWidth();
            paddingRight += child.getMeasuredWidth();
            break;
        }
        switch (vgrav) {
          default:
            childTop = paddingTop;
            break;
          case Gravity.TOP:
            childTop = paddingTop;
            paddingTop += child.getMeasuredHeight();
            break;
          case Gravity.CENTER_VERTICAL:
            childTop = Math.max((height - child.getMeasuredHeight()) / 2,
                paddingTop);
            break;
          case Gravity.BOTTOM:
            childTop = height - paddingBottom - child.getMeasuredHeight();
            paddingBottom += child.getMeasuredHeight();
            break;
        }
        childLeft += scrollX;
        child.layout(childLeft, childTop,
            childLeft + child.getMeasuredWidth(),
            childTop + child.getMeasuredHeight());
        decorCount++;
      }
    }
  }
```

需要注意的一点是，`ViewPager` 在每一个 `childLeft` 上都加上了当前 `ViewPager` 自己的 `scrollX`。这说明，`DecorView` 会跟随 `ViewPager` 滚动，并且读者可以据此想像，`ViewPager`，乃至 `ListView` 等类似功能的 `ViewGroup` 的布局原理都是相通的：在横、纵向上展开一张巨大的白纸，子视图沿着可滚动方向铺开，设备屏幕只展示了这张白纸上的一小块区域。

```java
  final int childWidth = width - paddingLeft - paddingRight;
  for (int i = 0; i < count; i++) {
    final View child = getChildAt(i);
    if (child.getVisibility() != GONE) {
      final LayoutParams lp = (LayoutParams) child.getLayoutParams();
      ItemInfo ii;
      if (!lp.isDecor && (ii = infoForChild(child)) != null) {
        int loff = (int) (childWidth * ii.offset);
        int childLeft = paddingLeft + loff;
        int childTop = paddingTop;
        if (lp.needsMeasure) {
          lp.needsMeasure = false;
          final int widthSpec = MeasureSpec.makeMeasureSpec(
              (int) (childWidth * lp.widthFactor),
              MeasureSpec.EXACTLY);
          final int heightSpec = MeasureSpec.makeMeasureSpec(
              (int) (height - paddingTop - paddingBottom),
              MeasureSpec.EXACTLY);
          child.measure(widthSpec, heightSpec);
        }
        child.layout(childLeft, childTop,
            childLeft + child.getMeasuredWidth(),
            childTop + child.getMeasuredHeight());
      }
    }
  }
  mTopPageBounds = paddingTop;
  mBottomPageBounds = height - paddingBottom;
  mDecorChildCount = decorCount;

  if (mFirstLayout) {
    scrollToItem(mCurItem, false, 0, false);
  }
  mFirstLayout = false;
}
```

[`ItemInfo`](#ItemInfo：维护子视图信息) 是 `ViewPager` 用于管理非 `Decor` 的子视图的展示信息的内部嵌套类，现在读者只需知道，`ItemInfo.offset` 的值等于`(一个子视图相对于 ViewPager 供子视图布局区域的最左部的偏移 / ViewPager 供子视图布局区域的宽度)`。布局过程中，`ViewPager` 使用它来确定子视图的 `left`。

当 `ViewPager.mInLayout` 为 `true` 时，通过 `addView` 添加到 `ViewPager` 的非 `Decor` 子视图的 `ViewPager.LayoutParams.needsMeasure` 被置为 `true`，这些子视图在布局过程中会被重新测量。稍后读者就会看到 `ViewPager.addView` 的逻辑。

## 管理子视图

### PagerAdapter：创建、添加与移除子视图

使用过 `ViewPager` 的读者知道，`ViewPager` 与 `ListView` 相似，使用了 Adapter 模式，由 `PagerAdapter` 提供子视图信息。`PagerAdapter` 必须要覆盖的方法和其意义如下表所示。

方法签名|返回值类型|意义
---|---|---
`instantiateItem(ViewGroup container, int position)`|`Object`|为指定的 `position` 创建 `Page`，并由 `Adapter` 自己负责将 `View` 添加到 `container` 中；返回的 `Object` 是一个与添加到 `ViewPager` 中的 `View` 构成一一对应关系的任意对象，通常可直接返回创建的 `View`
`destroyItem(ViewGroup container, int position, Object object)`|`void`|移除指定 `position` 的页面，由 `Adapter` 自己负责将添加到 `container` 中的 `View` 移除，`object` 即为 `instaintiateItem` 对应 `position` 的返回值
`isViewFromObject(View view, Object object)`|`boolean`|判定传入的 `View` 与 `Object` 是否满足对应关系
`getCount`|`int`|返回 `ViewPager` 要容纳的 `Page` 数量

`PagerAdapter` 与 `AdapterView` 使用的 `Adapter` 在设计上存在异同。

首先谈谈主要的不同之处。`PagerAdapter` 需要客户代码手动将子视图添加到 `ViewPager` / 从 `ViewPager` 中移除；并且 `ViewPager` 没有*复用子视图*的设计。从另一种角度看，这种宽松的设计，给 `ViewPager` 与 `Fragment` 整合（`FragmentPagerAdapter` 和 `FragmentStatePagerAdapter`）提供了空间。

两者的相同之处，除了均使用 Adapter 模式外，还均使用了*观察者模式*：当 Adapter 的后备数据发生变化时，客户代码需要主动将变更通知到观察者，而`ViewPager`、`AdapterView` 自身必然是观察者之一，客户代码在调用 `ViewPager.setAdapter()` 时，观察者的注册就悄然发生了：

```java
public void setAdapter(PagerAdapter adapter) {
  if (mAdapter != null) {
    mAdapter.setViewPagerObserver(null);
    mAdapter.startUpdate(this);
    for (int i = 0; i < mItems.size(); i++) {
      final ItemInfo ii = mItems.get(i);
      mAdapter.destroyItem(this, ii.position, ii.object);
    }
    mAdapter.finishUpdate(this);
    mItems.clear();
    removeNonDecorViews();
    mCurItem = 0;
    scrollTo(0, 0);
  }

  final PagerAdapter oldAdapter = mAdapter;
  mAdapter = adapter;
  mExpectedAdapterCount = 0;

  if (mAdapter != null) {
    if (mObserver == null) {
      mObserver = new PagerObserver();
    }
    mAdapter.setViewPagerObserver(mObserver);
    mPopulatePending = false;
    final boolean wasFirstLayout = mFirstLayout;
    mFirstLayout = true;
    mExpectedAdapterCount = mAdapter.getCount();
    if (mRestoredCurItem >= 0) {
      mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
      setCurrentItemInternal(mRestoredCurItem, false, true);
      mRestoredCurItem = -1;
      mRestoredAdapterState = null;
      mRestoredClassLoader = null;
    } else if (!wasFirstLayout) {
      populate();
    } else {
      requestLayout();
    }
  }

  if (mAdapterChangeListeners != null && !mAdapterChangeListeners.isEmpty()) {
    for (int i = 0, count = mAdapterChangeListeners.size(); i < count; i++) {
      mAdapterChangeListeners.get(i).onAdapterChanged(this, oldAdapter, adapter);
    }
  }
}
```

`PagerAdapter.startUpdate() 与 PagerAdapter.fisishUpdate()` 在 `PagerAdapter` 中的默认实现是空的，这两个带有明显事务风格的接口，更加印证了 `ViewPager` 先天与 `Fragment` 的亲近。

### ItemInfo：子视图信息

`ItemInfo` 是一个简单的 `struct` 风格的内部嵌套类。

```java
static class ItemInfo {
  Object object;      // 即 PagerAdapter.instantiateItem 的返回值
  int position;       // 页面下标
  boolean scrolling;  // 此页面正在被滚动与否
  float widthFactor;  // 即 PagerAdapter.getPageWidth 的返回值
  float offset;       // 前文已经提到它了
}
```

`ViewPager` 通过 `addNewItem()` 为每一个 `Page` 生成 `ItemInfo` 对象来维护相关信息，这些对象由成员 `mItems` 持有。

```java
private final ArrayList<ItemInfo> mItems = new ArrayList<ItemInfo>();

ItemInfo addNewItem(int position, int index) {
  ItemInfo ii = new ItemInfo();
  ii.position = position;
  ii.object = mAdapter.instantiateItem(this, position);
  ii.widthFactor = mAdapter.getPageWidth(position);
  if (index < 0 || index >= mItems.size()) {
    mItems.add(ii);
  } else {
    mItems.add(index, ii);
  }
  return ii;
}
```

`mItems` 并非持有着 `PagerAdapter.getCount()` 个 `ItemInfo`。在 `ViewPager` 处于闲置状态下，`mItems` 只持有着当前**的确**位于 `ViewPager` 中的子视图的信息。读者马上就会看到一些与 `ItemInfo` 相关的方法，其中的 getters 初看上去令人感觉非常困惑：为什么需要遍历 `mItems` 去获取某一个页面的 `ItemInfo`？看了 [`ViewPager.populate()`](#`populate`：切换与维护子视图信息) 的实现后，读者就会明白，只要 `mOffscreenPageLimit` 的值没有被设置得不合理地大，这些 getters 执行的循环次数都是常数次（具体来说，`1 <= 循环次数 <= 当前 ViewPager 的非 Decor 子视图个数`）。

```java
ItemInfo infoForChild(View child) {
  for (int i = 0; i < mItems.size(); i++) {
    ItemInfo ii = mItems.get(i);
    if (mAdapter.isViewFromObject(child, ii.object)) {
      return ii;
    }
  }
  return null;
}
```

### `populate`：切换与维护子视图信息

`ViewPager.populate()` 是 `ViewPager` 的核心逻辑所在。我们在 [`onMeasure`](#`onMeasure`) 和 [`setAdapter`](#PagerAdapter：创建、添加与移除子视图) 中已经看到它的身影。这个方法有点长，笔者删去了一些容错逻辑，以便读者阅读。

```java
void populate() {
  populate(mCurItem);
}

void populate(int newCurrentItem) {
  ItemInfo oldCurInfo = null;
  if (mCurItem != newCurrentItem) {
    oldCurInfo = infoForPosition(mCurItem);
    mCurItem = newCurrentItem;
  }
```

首先 `populate` 判断待切换的页面下标是否与当前下标 `mCurItem` 相等，如果不是的话，取出当前页面的 `ItemInfo` 并更新 `mCurItem`。

```java
  mAdapter.startUpdate(this);

  final int pageLimit = mOffscreenPageLimit;
  final int startPos = Math.max(0, mCurItem - pageLimit);
  final int N = mAdapter.getCount();
  final int endPos = Math.min(N - 1, mCurItem + pageLimit);
```

接下来，`populate` 向 `PagerAdapter` 声明页面变更事务的开始，并计算屏外页面范围 `startPos` 与 `endPos`。`mOffscreenPageLimit` 的默认值和最小值都是 `1`，使用者可以将它改为一个大于等于 `1` 的任意整数。

```java
  int curIndex = -1;
  ItemInfo curItem = null;
  for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
    final ItemInfo ii = mItems.get(curIndex);
    if (ii.position >= mCurItem) {
      if (ii.position == mCurItem) curItem = ii;
      break;
    }
  }

  if (curItem == null && N > 0) {
    curItem = addNewItem(mCurItem, curIndex);
  }
```

`populate` 取出待切换到的页面的 `ItemInfo`，如果不存在，则创建一个。

```java
  if (curItem != null) {
    float extraWidthLeft = 0.f;
    int itemIndex = curIndex - 1;
    ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
    final int clientWidth = getClientWidth();
    final float leftWidthNeeded = clientWidth <= 0 ? 0 :
        2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;
    for (int pos = mCurItem - 1; pos >= 0; pos--) {
      if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
        if (ii == null) {
          break;
        }
        if (pos == ii.position && !ii.scrolling) {
          mItems.remove(itemIndex);
          mAdapter.destroyItem(this, pos, ii.object);
          itemIndex--;
          curIndex--;
          ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
        }
      } else if (ii != null && pos == ii.position) {
        extraWidthLeft += ii.widthFactor;
        itemIndex--;
        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
      } else {
        ii = addNewItem(pos, itemIndex + 1);
        extraWidthLeft += ii.widthFactor;
        curIndex++;
        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
      }
    }

    float extraWidthRight = curItem.widthFactor;
    itemIndex = curIndex + 1;
    if (extraWidthRight < 2.f) {
      ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
      final float rightWidthNeeded = clientWidth <= 0 ? 0 :
          (float) getPaddingRight() / (float) clientWidth + 2.f;
      for (int pos = mCurItem + 1; pos < N; pos++) {
        if (extraWidthRight >= rightWidthNeeded && pos > endPos) {
          if (ii == null) {
            break;
          }
          if (pos == ii.position && !ii.scrolling) {
            mItems.remove(itemIndex);
            mAdapter.destroyItem(this, pos, ii.object);
            ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
          }
        } else if (ii != null && pos == ii.position) {
          extraWidthRight += ii.widthFactor;
          itemIndex++;
          ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
        } else {
          ii = addNewItem(pos, itemIndex);
          itemIndex++;
          extraWidthRight += ii.widthFactor;
          ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
        }
      }
    }

    calculatePageOffsets(curItem, curIndex, oldCurInfo);
  }
```

紧接着的这个巨大的 `if` 块，本质上只做了一件事：通知 `PagerAdapter` 创建足够多的离屏页面视图 / 移除超出当前离屏范围的页面视图，生成 / 丢弃对应的 `ItemInfo`。`ViewPager` 在决定离屏页面的范围时，会取先前计算的 `startPos`、`endPos` 与 `ViewPager` 子视图可用宽度的 `3` 倍中更大的那一个，作为待生成的离屏页面范围的边界。两个 `for` 循环的逻辑除了遍历方向相反外，逻辑完全一致，倒序遍历当前页面之前的 `ItemInfo` / 正序遍历当前页面之后的 `ItemInfo`：

* 如果额外的左 / 右侧页面总宽度系数（`extraWidth*`）超过了 `2.f`（具体的指标是 `*WidthNeeded`，需要将 `padding` 计入），且当前遍历到的页面下标小于 `startPos` / 大于 `endPos`：左 / 右侧的离屏页面总宽度超出了 `ViewPager` 所需的离屏页面范围。如果该页面不在滚动（`ItemInfo.scrolling`），通知 `PagerAdapter` 将该页面从 `ViewPager` 中移除，同时删除其 `ItemInfo`；
* 否则，如果当前遍历到的页面 `ItemInfo` 不为空，将其宽度系数计入 `extraWidth*`；否则，先通知 `PagerAdapter` 创建该页面，再计入其宽度系数。

在填充了足够的离屏页面之后，`populate` 调用 `caculatePageOffsets` 来计算所有 `ItemInfo` 的 `offset`。此方法的三个入参的意义分别是：

* `curItem`：即将作为 `ViewPager` 的 primary item 的页面信息
* `curIndex`：`curItem` 在 `mItems` 中的下标
* `oldCurInfo`：如果执行跨页跳转，为跳转前的 primary item 的页面信息；否则为 `null`

```java
private void calculatePageOffsets(ItemInfo curItem, int curIndex, ItemInfo oldCurInfo) {
  final int N = mAdapter.getCount();
  final int width = getClientWidth();
  final float marginOffset = width > 0 ? (float) mPageMargin / width : 0;
```

`mPageMargin` 是 `ViewPager` 的一个可配置项，允许客户代码在页面间插入任意大小的横向间距，并且还可以指定间距内绘制的 `Drawable`。

```java
  if (oldCurInfo != null) {
```

首先，`calculatePageOffsets` 要确定当前页面（也就是入参 `curItem`）的偏移，因为接下来 `curItem` 会用来确定离屏页面的 `ItemInfo.offset`。实际上，`curItem` 的偏移不确定**只有一种可能**，那就是 `ViewPager` 在执行跨页跳转，`curItem` 是在上层 `populate` 中刚刚创建出来的。因此，对 `oldCurInfo` 判空，即可得之 `curItem.offset` 是否确定。 

```java
    final int oldCurPosition = oldCurInfo.position;
    if (oldCurPosition < curItem.position) {
      int itemIndex = 0;
      ItemInfo ii = null;
      float offset = oldCurInfo.offset + oldCurInfo.widthFactor + marginOffset;
      for (int pos = oldCurPosition + 1;
          pos <= curItem.position && itemIndex < mItems.size(); pos++) {
        ii = mItems.get(itemIndex);
        while (pos > ii.position && itemIndex < mItems.size() - 1) {
          itemIndex++;
          ii = mItems.get(itemIndex);
        }
        while (pos < ii.position) {
          offset += mAdapter.getPageWidth(pos) + marginOffset;
          pos++;
        }
        ii.offset = offset;
        offset += ii.widthFactor + marginOffset;
      }
    } else if (oldCurPosition > curItem.position) {
      int itemIndex = mItems.size() - 1;
      ItemInfo ii = null;
      float offset = oldCurInfo.offset;
      for (int pos = oldCurPosition - 1;
          pos >= curItem.position && itemIndex >= 0; pos--) {
        ii = mItems.get(itemIndex);
        while (pos < ii.position && itemIndex > 0) {
          itemIndex--;
          ii = mItems.get(itemIndex);
        }
        while (pos > ii.position) {
          offset -= mAdapter.getPageWidth(pos) + marginOffset;
          pos--;
        }
        offset -= ii.widthFactor + marginOffset;
        ii.offset = offset;
      }
    }
  }
```

`oldCurInfo` 里包含了有效的 `offset` 信息，因此 `ViewPager` 使用它作为基准来设置 `curItem` 的偏移。两个 `if` 分支的逻辑几乎一样，均是从 `oldCurInfo.position` 出发，顺序 / 倒序遍历到 `curItem.position` 为止，途中更新经过的页面的偏移，循环退出前的那一趟刚好更新到 `curItem`。

```java
  final int itemCount = mItems.size();
  float offset = curItem.offset;
  int pos = curItem.position - 1;
  mFirstOffset = curItem.position == 0 ? curItem.offset : -Float.MAX_VALUE;
  mLastOffset = curItem.position == N - 1
      ? curItem.offset + curItem.widthFactor - 1 : Float.MAX_VALUE;

  for (int i = curIndex - 1; i >= 0; i--, pos--) {
    final ItemInfo ii = mItems.get(i);
    while (pos > ii.position) {
      offset -= mAdapter.getPageWidth(pos--) + marginOffset;
    }
    offset -= ii.widthFactor + marginOffset;
    ii.offset = offset;
    if (ii.position == 0) mFirstOffset = offset;
  }
  offset = curItem.offset + curItem.widthFactor + marginOffset;
  pos = curItem.position + 1;

  for (int i = curIndex + 1; i < itemCount; i++, pos++) {
    final ItemInfo ii = mItems.get(i);
    while (pos < ii.position) {
      offset += mAdapter.getPageWidth(pos++) + marginOffset;
    }
    if (ii.position == N - 1) {
      mLastOffset = offset + ii.widthFactor - 1;
    }
    ii.offset = offset;
    offset += ii.widthFactor + marginOffset;
  }

  mNeedCalculatePageOffsets = false;
}

```

接下来基于确定了偏移的 `curItem` 与 `marginOffset` 来设置剩余 `ItemInfo` 的偏移。与填充离屏页面的逻辑相似：两个 `for` 循环，先倒序计算当前页面之前页面的偏移，再顺序计算之后页面的偏移。此外，`calculatePageOffsets` 还更新了 `mFirstOffset` 和 `mLastOffset` 的值，这两个值用于确保 `ViewPager` 在正常滚动中不会超出页面范围。

回到 `populate` 中来。

```java
  mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);
  mAdapter.finishUpdate(this);
```

`populate` 向 `PagerAdapter` 通知当前页面的更新，与页面变更事务的结束。

```java
  final int childCount = getChildCount();
  for (int i = 0; i < childCount; i++) {
    final View child = getChildAt(i);
    final LayoutParams lp = (LayoutParams) child.getLayoutParams();
    lp.childIndex = i;
    if (!lp.isDecor && lp.widthFactor == 0.f) {
      final ItemInfo ii = infoForChild(child);
      if (ii != null) {
        lp.widthFactor = ii.widthFactor;
        lp.position = ii.position;
      }
    }
  }
  sortChildDrawingOrder();
```

`populate` 更新所有非 `Decor` 子视图的 `LayoutParams`，记录它们作为一个 `ViewGroup` 的 `child` 的 `childIndex` 和从 Adapter 中获取的 `widthFactor`。

```java
  if (hasFocus()) {
    View currentFocused = findFocus();
    ItemInfo ii = currentFocused != null ? infoForAnyChild(currentFocused) : null;
    if (ii == null || ii.position != mCurItem) {
      for (int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        ii = infoForChild(child);
        if (ii != null && ii.position == mCurItem) {
          if (child.requestFocus(View.FOCUS_FORWARD)) {
            break;
          }
        }
      }
    }
  }
}
```

最后，`populate` 判断如果当前 `ViewPager` 持有焦点，找出焦点所在的 `Page`，如果其 `ItemInfo` 为 `null`（即持有焦点的视图已经不在 `ViewPager` 当前的子视图中），将焦点转移到当前页面。

## 手势处理

### `onInterceptTouchEvent`

首先简单扫一眼短路逻辑。

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {

  final int action = ev.getAction() & MotionEventCompat.ACTION_MASK;

  if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
    resetTouch();
    return false;
  }

  // Nothing more to do here if we have decided whether or not we
  // are dragging.
  if (action != MotionEvent.ACTION_DOWN) {
    if (mIsBeingDragged) {
      return true;
    }
    if (mIsUnableToDrag) {
      return false;
    }
  }
```

* 将 `CANCEL` 与 `UP` 事件均视为拖拽结束，不拦截此事件
* 如果不是 `DOWN` 事件：
  * 如果正在被拖拽，拦截此事件
  * 如果不能被拖拽，不拦截此事件

```java
  switch (action) {
    case MotionEvent.ACTION_MOVE: {
      final int activePointerId = mActivePointerId;
      if (activePointerId == INVALID_POINTER) {
        break;
      }

      final int pointerIndex = ev.findPointerIndex(activePointerId);
      final float x = ev.getX(pointerIndex);
      final float dx = x - mLastMotionX;
      final float xDiff = Math.abs(dx);
      final float y = ev.getY(pointerIndex);
      final float yDiff = Math.abs(y - mInitialMotionY);

      if (dx != 0 && !isGutterDrag(mLastMotionX, dx)
          && canScroll(this, false, (int) dx, (int) x, (int) y)) {
        mLastMotionX = x;
        mLastMotionY = y;
        mIsUnableToDrag = true;
        return false;
      }
      if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
        mIsBeingDragged = true;
        requestParentDisallowInterceptTouchEvent(true);
        setScrollState(SCROLL_STATE_DRAGGING);
        mLastMotionX = dx > 0
            ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
        mLastMotionY = y;
        setScrollingCacheEnabled(true);
      } else if (yDiff > mTouchSlop) {
        mIsUnableToDrag = true;
      }
      if (mIsBeingDragged) {
        // Scroll to follow the motion event
        if (performDrag(x)) {
          ViewCompat.postInvalidateOnAnimation(this);
        }
      }
      break;
    }
```

对 `MOVE` 事件的处理逻辑可总结如下：

* 如果当前 `mActivePointerId` 不是一个合法的 ID，则说明 `ViewPager` 没有记录到这个 `MOVE` 事件对应的 `DOWN` 事件，不拦截后续事件。这是一些支持触摸手势的 `ViewPager` （比如 `ScrollView`）通行的处理策略。
* 如果该事件存在横向位移，且不起始于 Gutter 区，且子视图在横向可以滚动，则让子视图处理此事件。
* 如果该事件横向位移超过拖拽阈值 `mTouchSlop`，且横向位移距离超过纵向位移距离的两倍，将其视为拖拽手势；否则，如果纵向位移超过拖拽阈值 `mTouchSlop`，临时禁止 `ViewPager` 被拖拽，让子视图（有机会）拦截并处理所有非 `DOWN` 事件。

经过如上逻辑判断后，如果 `ViewPager` 认定自己正在被拖拽，则跟随该 `MOVE` 事件执行拖拽动画，并拦截后续事件。

```java
    case MotionEvent.ACTION_DOWN: {
      mLastMotionX = mInitialMotionX = ev.getX();
      mLastMotionY = mInitialMotionY = ev.getY();
      mActivePointerId = ev.getPointerId(0);
      mIsUnableToDrag = false;

      mIsScrollStarted = true;
      mScroller.computeScrollOffset();
      if (mScrollState == SCROLL_STATE_SETTLING
          && Math.abs(mScroller.getFinalX() - mScroller.getCurrX()) > mCloseEnough) {
        mScroller.abortAnimation();
        mPopulatePending = false;
        populate();
        mIsBeingDragged = true;
        requestParentDisallowInterceptTouchEvent(true);
        setScrollState(SCROLL_STATE_DRAGGING);
      } else {
        completeScroll(false);
        mIsBeingDragged = false;
      }
      break;
    }
```

`ViewPager` 接收到 `DOWN` 事件有两种情形：

* `ViewPager` 自己正在执行『归位』动画（`SCROLL_STATE_SETTLING`，后文中笔者将介绍 `ViewPager` 中定义的三种滚动状态）并且距离动画结束尚有一段距离（见 `mCloseEnough`），此时若接收到 `DOWN` 事件，则立即转入拖拽态（`SCROLL_STATE_DRAGGING`）、拦截后续事件，并立即终止归位动画。
* `ViewPager` 处于其它状态，终止滚动（如果正在执行的话），不拦截后续事件。

无论哪种情形，`ViewPager` 都会记下该事件的坐标和 Pointer ID，将其视为可能的拖拽手势的起点。

```java
    case MotionEventCompat.ACTION_POINTER_UP:
      onSecondaryPointerUp(ev);
      break;
  }

  if (mVelocityTracker == null) {
    mVelocityTracker = VelocityTracker.obtain();
  }
  mVelocityTracker.addMovement(ev);

  return mIsBeingDragged;
}
```

`ViewPager` 自身不支持多点手势，因此如果接收到 `POINTER_UP` 事件，检查是否需要更新 `mActivePointerId`。读者如果熟悉 `ScrollView` 的手势处理逻辑，就知道两者对这一类事件的处理逻辑完全一致。

### `onTouchEvent`

```java
@Override
public boolean onTouchEvent(MotionEvent ev) {
  if (mFakeDragging) {
    return true;
  }

  if (ev.getAction() == MotionEvent.ACTION_DOWN && ev.getEdgeFlags() != 0) {
    return false;
  }

  if (mAdapter == null || mAdapter.getCount() == 0) {
    return false;
  }
  
  if (mVelocityTracker == null) {
    mVelocityTracker = VelocityTracker.obtain();
  }
  mVelocityTracker.addMovement(ev);
```

首先我们依然关注短路逻辑。

`ViewPager` 提供了一个名为『fake drag』的功能，在此功能打开时，`ViewPager` 会忽略所有的手势并消费对应事件。

`ViewPager` 不消费发生在屏幕边缘的 `DOWN` 事件；如果客户代码没有绑定 `PagerAdapter`，或者 `PagerAdapter.getCount` 返回 `0`，也不消费任何事件。

除短路逻辑外，`ViewPager` 会消费所有传入 `onTouchEvent` 的事件。

```java
  final int action = ev.getAction();
  boolean needsInvalidate = false;
  
  switch (action & MotionEventCompat.ACTION_MASK) {
    case MotionEvent.ACTION_DOWN: {
      mScroller.abortAnimation();
      mPopulatePending = false;
      populate();

      mLastMotionX = mInitialMotionX = ev.getX();
      mLastMotionY = mInitialMotionY = ev.getY();
      mActivePointerId = ev.getPointerId(0);
      break;
    }
```

在这里接收到 `DOWN` 事件只有一种可能：子视图没有消费此事件，事件传递递归返回回到了 `ViewPager` 中。

```java
    case MotionEvent.ACTION_MOVE:
      if (!mIsBeingDragged) {
        final int pointerIndex = ev.findPointerIndex(mActivePointerId);
        if (pointerIndex == -1) {
          needsInvalidate = resetTouch();
          break;
        }
        final float x = ev.getX(pointerIndex);
        final float xDiff = Math.abs(x - mLastMotionX);
        final float y = ev.getY(pointerIndex);
        final float yDiff = Math.abs(y - mLastMotionY);
        if (xDiff > mTouchSlop && xDiff > yDiff) {
          mIsBeingDragged = true;
          requestParentDisallowInterceptTouchEvent(true);
          mLastMotionX = x - mInitialMotionX > 0 ? mInitialMotionX + mTouchSlop :
              mInitialMotionX - mTouchSlop;
          mLastMotionY = y;
          setScrollState(SCROLL_STATE_DRAGGING);
          setScrollingCacheEnabled(true);

          ViewParent parent = getParent();
          if (parent != null) {
            parent.requestDisallowInterceptTouchEvent(true);
          }
        }
      }
```

处理 `MOVE` 手势分为两种情况考虑。一种是 `ViewPager` 没有处于被拖拽状态，如果该事件 Pointer ID 与 `mActivePointerId` 吻合，并且满足横向位移超过阈值并且大于纵向位移的条件，则 `ViewPager` 视该事件为拖拽手势，逻辑与 `onInterceptTouchEvent` 中处理 `MOVE` 事件完全一致。

```java
      if (mIsBeingDragged) {
        final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
        final float x = ev.getX(activePointerIndex);
        needsInvalidate |= performDrag(x);
      }
      break;
```

另一种情况是 `ViewPager` 已经处于被拖拽状态，则执行拖拽动画。注意，在上一个 `if (!mIsBeingDragged)` 块中，`mIsBeingDragged` 可能被置为 `true` 并继续执行 `if (mIsBeingDragged)`，读者将这一块逻辑与 `onInterceptTouchEvent` 中处理 `MOVE` 事件的逻辑相比较即知，这里 `ViewPager` 只是简单复用了一下执行拖拽动画的代码。

```java
    case MotionEvent.ACTION_UP:
      if (mIsBeingDragged) {
        final VelocityTracker velocityTracker = mVelocityTracker;
        velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
        int initialVelocity = (int) VelocityTrackerCompat.getXVelocity(
            velocityTracker, mActivePointerId);
        mPopulatePending = true;
        final int width = getClientWidth();
        final int scrollX = getScrollX();
        final ItemInfo ii = infoForCurrentScrollPosition();
        final float marginOffset = (float) mPageMargin / width;
        final int currentPage = ii.position;
        final float pageOffset = (((float) scrollX / width) - ii.offset)
            / (ii.widthFactor + marginOffset);
        final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
        final float x = ev.getX(activePointerIndex);
        final int totalDelta = (int) (x - mInitialMotionX);
        int nextPage = determineTargetPage(currentPage, pageOffset, initialVelocity,
            totalDelta);
        setCurrentItemInternal(nextPage, true, true, initialVelocity);

        needsInvalidate = resetTouch();
      }
      break;
```

接收到 `UP` 手势时，如果当前 `ViewPager` 正处于被拖拽状态，则说明用户松开了拖拽所用的手指。使用 `VelocityTracker` 计算当前手势的横向速度，作为扫动动画的初始速度；并置 `mPopulatePending` 为 `true`，将扫动期间的任何 `populate` 请求推迟到动画结束。

紧接着，`onTouchEvent` 调用 `infoForCurrentScrollPosition()` 获取当前滚动位置对应页面的 `ItemInfo`。为节省篇幅，笔者直接摆出结论：它会返回当前 `ViewPager` 可见的首个页面的 `ItemInfo`。

`ViewPager` 对拖拽手势是否满足页面间跳转设置了条件，条件的判断逻辑被封装在 `determineTargetPage()` 中。它的入参分别为：

* `velocity`：`UP` 事件发生时手势的横向速度（带符号）
* `currentPage`：当前 `ViewPager` 首个可见页面的 `ItemInfo.position`（**只露出部分**也视为可见）
* `pageOffset`：当前 `ViewPager` 横向滚动偏移减去当前 `ViewPager` 首个可见页面的 `ItemInfo.offset`。为了便于读者理解，笔者该参数的意义分两种情形解释：
  * 手势是右滑手势（即 `velocity > 0`），则该参数的值是当前页面的左侧页面**尚未**被 `ViewPager` 展示出来的部分的宽度与 `ViewPager` 非 `Decor` 子视图可用宽度之比
  * 手势是左滑手势（即 `velocity < 0`），则该参数的值是当前页面的右侧页面**已经**被 `ViewPager` 展示出来的部分的宽度与 `ViewPager` 非 `Decor` 子视图可用宽度之比
* `deltaX`：手势的横向偏移量

```java
private int determineTargetPage(int currentPage, float pageOffset, int velocity, int deltaX) {
  int targetPage;
  if (Math.abs(deltaX) > mFlingDistance && Math.abs(velocity) > mMinimumVelocity) {
    targetPage = velocity > 0 ? currentPage : currentPage + 1;
  } else {
    final float truncator = currentPage >= mCurItem ? 0.4f : 0.6f;
    targetPage = currentPage + (int) (pageOffset + truncator);
  }
```

`mFlingDistance` 是 `ViewPager` 将『`DOWN` - `MOVE` - `UP`』事件判断为『切换到相邻页』手势的距离限制，大小为 `25dp`。`mMinimunVelocity` 是将『`DOWN` - `MOVE` - `UP`』事件判断为『扫动』手势的最小速度限制，`ViewPager` 从 `ViewConfiguration` 中取得该值，以确保 `ViewPager` 获得一致的触摸体验。

`determineTargetPage` 判断手势的偏移和速度如果都满足要求，则通过 `velocity` 的符号判断手势的方向，如果为正，则说明此手势为右滑（回退一页），此时前一页已经在 `ViewPager` 上可见，`currentPage` 就是回退的那一页的 `position`；为负，则说明此手势为左滑（前进一页），`currentPage + 1` 才是下一页的 `position`。

如果手势在速度或偏移条件上没有满足要求，`determineTargetPage` 退而判断相邻页面已经处于可见区域的比例（见笔者对 `pageOffset` 的意义解释）。`ViewPager` 的实现者在这里巧妙地使用了 `float` 强转 `int` 丢失精度的机制，实现了『如果手势速度、偏移不达标，则需相邻页面已经有 60% 区域可见，才能滚动到相邻页面』的逻辑。

```java
  if (mItems.size() > 0) {
    final ItemInfo firstItem = mItems.get(0);
    final ItemInfo lastItem = mItems.get(mItems.size() - 1);

    targetPage = Math.max(firstItem.position, Math.min(targetPage, lastItem.position));
  }

  return targetPage;
}
```

在返回 `targetPage` 之前，`determineTargetPage` 还确保它没有越过当前 `mItems` 的首尾页面。

```java
    case MotionEvent.ACTION_CANCEL:
      if (mIsBeingDragged) {
        scrollToItem(mCurItem, true, 0, false);
        needsInvalidate = resetTouch();
      }
      break;
```

回到 `onTouchEvent` 中来。接收到 `CANCEL` 手势，滚回当前页面。

```java
    case MotionEventCompat.ACTION_POINTER_DOWN: {
      final int index = MotionEventCompat.getActionIndex(ev);
      final float x = ev.getX(index);
      mLastMotionX = x;
      mActivePointerId = ev.getPointerId(index);
      break;
    }
    case MotionEventCompat.ACTION_POINTER_UP:
      onSecondaryPointerUp(ev);
      mLastMotionX = ev.getX(ev.findPointerIndex(mActivePointerId));
      break;
  }
  if (needsInvalidate) {
    ViewCompat.postInvalidateOnAnimation(this);
  }
  return true;
}
```

`ViewPager` 对多点触摸的处理较为简单（同样，与 `ScollView` 完全相同），可总结为两点：

* 识别到一个多点 `DOWN` 事件，将其作为新的 active pointer。
* 识别到一个多点 `UP` 事件，切换 active pointer（如果抬起的正好是当前的 active pointer 的话）。

## 滚动处理

### 状态定义

`ViewPager` 定义了三种状态：

* `SCROLL_STATE_IDLE`：闲置态，primary item 完全可见，没有任何动画处于执行状态，可以转移到拖拽态或归位态；
* `SCROLL_STATE_DRAGGING`：拖拽态：`ViewPager` 正在被拖动，可以转移到归位态；
* `SCROLL_STATE_SETTLING`：归位态：`ViewPager` 正在自动滚动到目标页面，可以转移到闲置态或拖拽态。

### `performDrag`

`ViewPager.performDrag()` 用来响应拖拽手势，滚动 `ViewPager` 自身，入参 `x` 是当前拖拽手势的横向坐标。

```java
private boolean performDrag(float x) {
  boolean needsInvalidate = false;

  final float deltaX = mLastMotionX - x;
  mLastMotionX = x;

  float oldScrollX = getScrollX();
  float scrollX = oldScrollX + deltaX;
  final int width = getClientWidth();

  float leftBound = width * mFirstOffset;
  float rightBound = width * mLastOffset;
  boolean leftAbsolute = true;
  boolean rightAbsolute = true;

  final ItemInfo firstItem = mItems.get(0);
  final ItemInfo lastItem = mItems.get(mItems.size() - 1);
  if (firstItem.position != 0) {
    leftAbsolute = false;
    leftBound = firstItem.offset * width;
  }
  if (lastItem.position != mAdapter.getCount() - 1) {
    rightAbsolute = false;
    rightBound = lastItem.offset * width;
  }

  if (scrollX < leftBound) {
    if (leftAbsolute) {
      float over = leftBound - scrollX;
      needsInvalidate = mLeftEdge.onPull(Math.abs(over) / width);
    }
    scrollX = leftBound;
  } else if (scrollX > rightBound) {
    if (rightAbsolute) {
      float over = scrollX - rightBound;
      needsInvalidate = mRightEdge.onPull(Math.abs(over) / width);
    }
    scrollX = rightBound;
  }
  // Don't lose the rounded component
  mLastMotionX += scrollX - (int) scrollX;
  scrollTo((int) scrollX, getScrollY());
  pageScrolled((int) scrollX);

  return needsInvalidate;
}
```

可见，`performDrag` 的逻辑较为简单，主要是防止滚动目标位置超过当前离屏页面的边界，触发 `EdgeEffect`（如果已经滚动到数据源的两端），然后调用 `View.scrollTo()` 滚动自身。

#### `pageScrolled` 及 `onPageScrolled`

```java
private boolean pageScrolled(int xpos) {
  if (mItems.size() == 0) {
    if (mFirstLayout) {
      return false;
    }
    mCalledSuper = false;
    onPageScrolled(0, 0, 0);
    if (!mCalledSuper) {
      throw new IllegalStateException(
          "onPageScrolled did not call superclass implementation");
    }
    return false;
  }
  final ItemInfo ii = infoForCurrentScrollPosition();
  final int width = getClientWidth();
  final int widthWithMargin = width + mPageMargin;
  final float marginOffset = (float) mPageMargin / width;
  final int currentPage = ii.position;
  final float pageOffset = (((float) xpos / width) - ii.offset)
      / (ii.widthFactor + marginOffset);
  final int offsetPixels = (int) (pageOffset * widthWithMargin);

  mCalledSuper = false;
  onPageScrolled(currentPage, pageOffset, offsetPixels);
  if (!mCalledSuper) {
    throw new IllegalStateException(
        "onPageScrolled did not call superclass implementation");
  }
  return true;
}
```

`pageScrolled` 自己并没有做什么事，它完成了一些计算，把结果传给了 `onPageScrolled`，并且确保子类实现调用了 `super.onPageScrolled()`。`onPageScrolled` 做了三件事：

```java
@CallSuper
protected void onPageScrolled(int position, float offset, int offsetPixels) {
  if (mDecorChildCount > 0) {
    final int scrollX = getScrollX();
    int paddingLeft = getPaddingLeft();
    int paddingRight = getPaddingRight();
    final int width = getWidth();
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
      final View child = getChildAt(i);
      final LayoutParams lp = (LayoutParams) child.getLayoutParams();
      if (!lp.isDecor) continue;

      final int hgrav = lp.gravity & Gravity.HORIZONTAL_GRAVITY_MASK;
      int childLeft = 0;
      switch (hgrav) {
        default:
          childLeft = paddingLeft;
          break;
        case Gravity.LEFT:
          childLeft = paddingLeft;
          paddingLeft += child.getWidth();
          break;
        case Gravity.CENTER_HORIZONTAL:
          childLeft = Math.max((width - child.getMeasuredWidth()) / 2,
              paddingLeft);
          break;
        case Gravity.RIGHT:
          childLeft = width - paddingRight - child.getMeasuredWidth();
          paddingRight += child.getMeasuredWidth();
          break;
      }
      childLeft += scrollX;

      final int childOffset = childLeft - child.getLeft();
      if (childOffset != 0) {
        child.offsetLeftAndRight(childOffset);
      }
    }
  }
```

第一件事是修正 `Decor` 子视图的横向偏移。`ViewPager` 发生了横向滚动，因此 `ViewPager` 需要及时地修改它们的位置，以让它们*看起来*『悬浮』在 `ViewPager` 上。

```java
  dispatchOnPageScrolled(position, offset, offsetPixels);
```

第二件事，是向订阅了 `ViewPager` 滚动事件的订阅者发布最新的滚动事件。

```java
  if (mPageTransformer != null) {
    final int scrollX = getScrollX();
    final int childCount = getChildCount();
    for (int i = 0; i < childCount; i++) {
      final View child = getChildAt(i);
      final LayoutParams lp = (LayoutParams) child.getLayoutParams();

      if (lp.isDecor) continue;
      final float transformPos = (float) (child.getLeft() - scrollX) / getClientWidth();
      mPageTransformer.transformPage(child, transformPos);
    }
  }

  mCalledSuper = true;
}
```

第三件事，是更新页面切换动画效果。`ViewPager` 支持自定义页面切换动画，我们在一些启动器中看到的花哨的页面切换效果就是使用 `PageTransformer` 实现的。

### `computeScroll`

`computeScroll` 是凡是支持『扫动』、『过度滚动』效果的 `View` 都会覆盖的一个方法，它在绘制过程中被自动调用。通常，`View` 子类在这里向 `Scroller` 或者 `OverScroller` 请求当前的滚动偏移，然后调用 `View.scrollTo()`。

```java
@Override
public void computeScroll() {
  mIsScrollStarted = true;
  if (!mScroller.isFinished() && mScroller.computeScrollOffset()) {
    int oldX = getScrollX();
    int oldY = getScrollY();
    int x = mScroller.getCurrX();
    int y = mScroller.getCurrY();

    if (oldX != x || oldY != y) {
      scrollTo(x, y);
      if (!pageScrolled(x)) {
        mScroller.abortAnimation();
        scrollTo(0, y);
      }
    }

    ViewCompat.postInvalidateOnAnimation(this);
    return;
  }

  completeScroll(true);
}
```

如果 `Scroller` 告知滚动尚未结束，`ViewPager` 取出其计算的 `x` 与 `y`，更新滚动位置。`pageScrolled` 只在 `ViewPager` 尚未完成首次布局或是 `PagerAdapter` 后备数据源为空时返回 `false`，在这两种情形提前终止滚动是合理的。

如果滚动已经结束，`ViewPager` 调用 `completeScroll` 执行清理过程。

### `completeScroll`

```java
private void completeScroll(boolean postEvents) {
  boolean needPopulate = mScrollState == SCROLL_STATE_SETTLING;
  if (needPopulate) {
    setScrollingCacheEnabled(false);
    boolean wasScrolling = !mScroller.isFinished();
    if (wasScrolling) {
      mScroller.abortAnimation();
      int oldX = getScrollX();
      int oldY = getScrollY();
      int x = mScroller.getCurrX();
      int y = mScroller.getCurrY();
      if (oldX != x || oldY != y) {
        scrollTo(x, y);
        if (x != oldX) {
          pageScrolled(x);
        }
      }
    }
  }
```

`completeScroll` 判断当前 `ViewPager` 的状态，如果处于*归位态*，则置 `needPopulate` 为 `true`，表示归位后立即执行一次 `populate` 过程，以确保归位后 primary item 的相邻页面有可用的离屏视图 / 更新 `ViewPager` 的边界，防止接下来的滚动超出边界。如果此时 `Scroller` 滚动仍为完成，则最后执行一次滚动步进，终止滚动动画。

```java
  mPopulatePending = false;
  for (int i = 0; i < mItems.size(); i++) {
    ItemInfo ii = mItems.get(i);
    if (ii.scrolling) {
      needPopulate = true;
      ii.scrolling = false;
    }
  }
  if (needPopulate) {
    if (postEvents) {
      ViewCompat.postOnAnimation(this, mEndScrollRunnable);
    } else {
      mEndScrollRunnable.run();
    }
  }
}
```

接下来 `completeScroll` 将所有 `ItemInfo` 的 `scrolling` 成员全部置为 `false`（为接下来的 `populate` 正确执行做准备），然后根据入参 `postEvents` 决定是就地执行 `populate` 还是推迟到下一个 animation time step。

### 页面间滚动处理

`ViewPager` 与一般的可滚动 `ViewGroup` 有一个明显的区别：使用触摸手势滚动 `ViewPager` 时，无论手势的速度有多快，`ViewPager` 最多只能滚动到相邻的那个页面。因此，`ViewPager` 不能*放任*扫动手势，需要限制其距离。接下来，笔者从 [`onTouchEvent`](#`onTouchEvent`) 处理 `UP` 手势出发，分析 `ViewPager` 处理页面间滚动的设计。

#### `setCurrentItemInternal`

这个方法有两个重载，笔者粘贴于文中的是带 4 个形参的版本，[`onTouchEvent`](#`onTouchEvent`) 传入的实参分别为 `nextPage`（目标页面的 `position`）、`true`、`true` 和由 `VelocityTracker` 计算的扫动手势初始速度。

```java
void setCurrentItemInternal(int item, boolean smoothScroll, boolean always, int velocity) {
  if (mAdapter == null || mAdapter.getCount() <= 0) {
    setScrollingCacheEnabled(false);
    return;
  }
  if (!always && mCurItem == item && mItems.size() != 0) {
    setScrollingCacheEnabled(false);
    return;
  }

  if (item < 0) {
    item = 0;
  } else if (item >= mAdapter.getCount()) {
    item = mAdapter.getCount() - 1;
  }
```

首先 `setCurrentItemInternal` 执行了一些保护逻辑。可见，如果传入无效的 `item`，`ViewPager` 会自动将其修改到合法的边界值。

```java
  final int pageLimit = mOffscreenPageLimit;
  if (item > (mCurItem + pageLimit) || item < (mCurItem - pageLimit)) {
    for (int i = 0; i < mItems.size(); i++) {
      mItems.get(i).scrolling = true;
    }
  }
```

如果在执行跨越离屏页面缓冲的滚动，`ViewPager` 会将所有 `ItemInfo.scrolling` 置为 `true`，以防止归位前的 `populate` 过程提前将途中页面的视图移出 `ViewPager`，造成视觉上的异常。
  
```java
  final boolean dispatchSelected = mCurItem != item;

  if (mFirstLayout) {
    mCurItem = item;
    if (dispatchSelected) {
      dispatchOnPageSelected(item);
    }
    requestLayout();
  } else {
    populate(item);
    scrollToItem(item, smoothScroll, velocity, dispatchSelected);
  }
}
```

接下来，`setCurrentItemInternal` 判断 `ViewPager` 是否完成了初次布局，如果没有，则简单更新 `mCurItem` 即可，由随后在 `onMeasure` 中执行的 `populate` 过程来处理剩下的事；否则就地 `populate`，确保目标页面周围已经填充好了足够的离屏页面，再执行滚动。

#### `scrollToItem`

```java
private void scrollToItem(int item, boolean smoothScroll, int velocity,
    boolean dispatchSelected) {
  final ItemInfo curInfo = infoForPosition(item);
  int destX = 0;
  if (curInfo != null) {
    final int width = getClientWidth();
    destX = (int) (width * Math.max(mFirstOffset,
        Math.min(curInfo.offset, mLastOffset)));
  }
```

这个方法做了两件事，第一件事就是取出当前 primary item 来计算滚动偏移。速度不会对偏移产生任何影响，这与 `ScrollView`、`ListView` 等的*扫动*动画行为不同。

```java
  if (smoothScroll) {
    smoothScrollTo(destX, 0, velocity);
    if (dispatchSelected) {
      dispatchOnPageSelected(item);
    }
  } else {
    if (dispatchSelected) {
      dispatchOnPageSelected(item);
    }
    completeScroll(false);
    scrollTo(destX, 0);
    pageScrolled(destX);
  }
}
```

第二件事，就是发起真正的滚动过程：如果执行*流畅滚动*（触摸手势发起的滚动均为流畅滚动），交给 `smoothScrollTo` 处理；否则，就地调用 `View.scrollTo`。页面切换事件的分发均在该方法中完成——也就是说，对于流畅滚动而言，动画刚刚开始执行时，事件就已分发给订阅者。

#### `smoothScrollTo`

```java
void smoothScrollTo(int x, int y, int velocity) {
  if (getChildCount() == 0) {
    setScrollingCacheEnabled(false);
    return;
  }

  int sx;
  boolean wasScrolling = (mScroller != null) && !mScroller.isFinished();
  if (wasScrolling) {
    sx = mIsScrollStarted ? mScroller.getCurrX() : mScroller.getStartX();
    mScroller.abortAnimation();
    setScrollingCacheEnabled(false);
  } else {
    sx = getScrollX();
  }
  int sy = getScrollY();
```

`smoothScrollTo` 首先获取当前 `ViewPager` 的 `scrollX` 与 `scrollY`。如果当前有另一个滚动动画正在进行，`smoothScrollTo` 取 `mScroller.getCurrX()` 或 `mScroller.getStartX()`（如果上次滚动动画尚未开始，即 `mIsScrollStarted == false`）作为当前 `scrollX`，并取消该动画。
  
```java
  int dx = x - sx;
  int dy = y - sy;
  if (dx == 0 && dy == 0) {
    completeScroll(false);
    populate();
    setScrollState(SCROLL_STATE_IDLE);
    return;
  }
  
  setScrollingCacheEnabled(true);
  setScrollState(SCROLL_STATE_SETTLING);
```

如果横向、纵向均没有滚动偏移，不执行动画，提前结束滚动过程。

```java
  final int width = getClientWidth();
  final int halfWidth = width / 2;
  final float distanceRatio = Math.min(1f, 1.0f * Math.abs(dx) / width);
  final float distance = halfWidth + halfWidth
      * distanceInfluenceForSnapDuration(distanceRatio);

  int duration;
  velocity = Math.abs(velocity);
  if (velocity > 0) {
    duration = 4 * Math.round(1000 * Math.abs(distance / velocity));
  } else {
    final float pageWidth = width * mAdapter.getPageWidth(mCurItem);
    final float pageDelta = (float) Math.abs(dx) / (pageWidth + mPageMargin);
    duration = (int) ((pageDelta + 1) * 100);
  }
  duration = Math.min(duration, MAX_SETTLE_DURATION);
```

接下来，`smoothScrollTo` 计算扫动动画的时长 `duration`。如果手势带有速度，则时长由速度与滚动距离共同决定；否则，时长仅由滚动距离决定。无论如何，时长 `duration` 不会超过常量 `MAX_SETTLE_DURATION = 600` 毫秒。

```java
  mIsScrollStarted = false;
  mScroller.startScroll(sx, sy, dx, dy, duration);
  ViewCompat.postInvalidateOnAnimation(this);
}
```

最后，`smoothScrollTo` 调用 `mScroller.startScroll()`，并 `invalidate` 自己；后续的滚动偏移更新由 `computeScroll()` 处理。
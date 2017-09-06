---
layout: post
title: "ListView 实现分析"
tags: [Android, View]
---
## 概述

`ListView` 的类继承树上方两层分别是 `AbsListView` 和 `AdapterView`，其兄弟是另一个 App 开发中常用的 `ViewGroup`——`GridView`。大体上，这三层继承结构实现了对各自关心的问题的分工（封装得并不完美，比如读者会看到 `ListView` 中仍频繁使用 `AbsListView` 的包权限成员）。

简单类名|功能
------|---
`AdapterView`|监听 `Adapter` 变化；分发 Item 选中、点击、长按等事件
`AbsListView`|处理触摸事件；管理及复用所有子视图；管理对 Item 的选择
`ListView`、`GridView`|管理 `Header` 与 `Footer` 信息；测量及布局子视图

## 复用子视图

我们无法接受为每一份数据构造独立的视图。如果数据量不大，情况似乎还不糟；但如果有上千、上万条数据需要展示呢？*复用*是 `AdapterView` 的核心议题（尽管实现逻辑位于 `AbsListView` 中），读者熟悉的 `Adapter.getView()` 方法签名中就留下了深深的烙印：形参 `convertView` 即为可复用的视图（如果不为空）。

`AbsListView.RecycleBin` 是待复用的子视图的容器。`ReclcyeBin` 将子视图分为两类：

```java
private int mFirstActivePosition;
private View[] mActiveViews = new View[0];
```

一类是 **Active** 视图。如果数据源没有发生变化（即一趟布局不是由 `Adapter.notifyDataSetChanged()` 引发的），在每一趟布局开始前，`AbsListView` 将所有子视图（不包括位于 `RecycleBin` 中的待复用视图和 `ListView` / `GridView` 的 `Header` / `Footer`）加入 `mActiveViews`，布局过程中，这些视图将会被优先复用，如果没有可以复用的 **Active** 视图，则尝试复用 `RecycleBin` 持有的另一类视图；一趟布局结束后，`mActiveViews` 中没有被复用的所有视图被转移到 `mScrapViews` 中。

```java
private ArrayList<View>[] mScrapViews;
private int mViewTypeCount;
private ArrayList<View> mCurrentScrap;
private ArrayList<View> mSkippedScrap;
```

另一类即为 **Scrap** 视图。如果数据源发生了变化，则在一趟布局开始前，`AbsListView` 将所有子视图加入 `mScrapViews`，布局过程中，在最终向 `Adapter` 请求新视图之前，`AbsListView` 始终会尝试获取一个 **Scrap** 视图，作为 `Adapter.getView()` 的实参 `convertView` 传入。**Scrap** 视图会一直被 `RecycleBin` 持有，直到下一个 `setAdapter` 调用或是 `AbsListView.onDetachedFromWindow` 时被释放。

一个子视图在被加入 `mScrapViews` 时，`RecycleBin` 会向它分发 `start temporary detach` 消息；当 **Scrap** 被复用时，如果它 `isTemporarilyDetached`，`AbsListView` 会向它分发 `finish temporary detach` 消息。一趟布局完成后，没有被复用的临时 `detach` 视图会被 `remove`（`ViewGroup` 语义上的 `remove`，其引用仍被 `RecycleBin` 持有）。 

`AdapterView` 支持使用者通过 `Adapter.getItemViewType()` 定义多种类型的子视图，但有两个值被视为特殊值，在复用过程中这种特殊类型的视图也会被特别对待。

```java
public static final int ITEM_VIEW_TYPE_IGNORE = -1;
public static final int ITEM_VIEW_TYPE_HEADER_OR_FOOTER = -2;
```

类型为 `ITEM_VIEW_TYPE_IGNORE` 的子视图，它们会被加入到 `RecycleBin.mSkippedScrap` 中，在每趟布局开始前、`AbsListView` 处理每一步视图滚动时会被清空。`ITEM_VIEW_TYPE_HEADER_OR_FOOTER` 这一类型，顾名思义就是为 `ListView` / `GridView` 的 `Header` / `Footer` 预留的类型，它们永远不会被 `RecycleBin` 持有、复用。

```java
private SparseArray<View> mTransientStateViews;
private LongSparseArray<View> mTransientStateViewsById;
```

`RecycleBin` 还持有一类非常特殊的视图的引用，它用来配合 API 16 中 `View` 新增加的一个属性：[`transientState`](https://developer.android.com/reference/android/view/View.html#setHasTransientState(boolean))。如果你想为 `AbsListView` 中的一个 item 增加动画效果，你应该保证在开始动画的同时将该 item 所在 `View` 置于 `transient state`（并且在动画结束时恢复到普通状态），这样 `RecycleBin` 就不会复用执行动画过程中的 `View`。感兴趣的读者可以观看 Andorid Developers YouTube 官方频道的[这段视频](https://www.youtube.com/watch?v=8MIfSxgsHIs)。

从实现角度来说，`AbsListView` 在将尝试子视图加入 `mScrapView` 之前，会判断该视图是否 `hasTransientState`；在尝试复用 **Scrap** 视图前，会优先复用*相同位置上*的 `transient` 视图。

如果不考虑 `transient state`，一个普通 `viewType` 子视图在 `AbsListView` 中的生命周期，可以大致总结如下图。

![子视图在 `AbsListView` 中的生命周期]({{ site.url }}/assets/Lifecycles_of_Views_in_AbsListView.png)

## 布局过程

### 测量

`ListView` 的测量过程没有做太多复杂工作，因此笔者将其放在布局过程中的一个子小节简要概括。

正常情况下，`AbsListView` 的宽高不应受其子视图影响。但如果使用者在 `AbsListView` 的可伸展方向（即纵向）加上了一个 `AT_MOST` 或 `UNSPECIFIED` 的测量模式，那么 `AbsListView` 必须依赖部分子视图对自身的尺寸做出*估计*。

如果宽高中任一侧的测量模式被设为 `UNSPECIFIED`（这意味着 `AbsListView` 极可能被放入 `ScrollView` / `HorizontalScrollView` 中），`ListView` 采用的策略是使用其自身的测量规格去测量由 `Adapter` 提供的首个子视图，将测量结果（加上 padding 等自身固有空间）作为自身尺寸——尝试过将 `ListView` 与 `ScrollView` 嵌套的读者势必遇到过此问题，`ListView` 的高度等于首个 item 的高度。

如果 `ListView` 的高度测量模式被设为 `AT_MOST`，`ListView` 会尝试向 `Adapter` 请求多个子视图，并使用一个特殊的测量规格（参见 `ListView.measureHeightOfChildren()`）来测量它们，若已经测量过子视图的累计高度之和超过 `AT_MOST` 模式附带指定的最大高度，则直接使用 `ListView` 的父布局指定的最大高度作为自身高度；亦或是此时 `Adapter` 提供的所有子视图高度之和尚未超过最大高度，则使用子视图高度之和作为自身高度。

### layout mode

`AbsListView` 定义了多种布局模式，它们适用于不同的场景。

模式|功能
---|---
`LAYOUT_NORMAL`|普通模式，以填满 `AbsListView` 的视区为准
`LAYOUT_FORCE_TOP`|将 `Adapter` 提供的第一个 `Item` 置于 `AbsListView` 的顶部
`LAYOUT_SET_SELECTION`|将选中的 item 置于视区的任意位置
`LAYOUT_FORCE_BOTTOM`|将 `Adapter` 提供的最后一个 `Item` 至于 `AbsListView` 的底部
`LAYOUT_SPECIFIC`|将选中的 item 置于 `mSpecificTop`，并基于此 item 布局其它子视图
`LAYOUT_SYNC`|保持数据变更前后子视图位置不改变，只在 `Adapter` 提供 `stable id` 的数据时使用此模式

### layoutChildren

`AbsListView ` 定义了一个名为 `layoutChildren` 的保护方法，供子类布局子视图使用，子类不应覆盖 `onLayout()` 来布局。下面，笔者根据简化了的 `ListView.layoutChildren` 源码来介绍 `ListView` 的一趟完整布局过程。

```java
@Override
protected void layoutChildren() {
    final boolean blockLayoutRequests = mBlockLayoutRequests;
    if (blockLayoutRequests) {
        return;
    }

    mBlockLayoutRequests = true;

    try {
        super.layoutChildren();

        invalidate();

        if (mAdapter == null) {
            resetList();
            invokeOnItemScrollListener();
            return;
        }
```

如果 `mBlockLayoutRequests` 为 `true`，则跳过本次布局过程。这个标记在 `AbsListView` 的另一个重要方法 `trackMotionScroll()` 中也会被置为 `true`。

如果 `mAdapter` 为空，同样跳过本次布局过程—— `AdapterView` 没有 `Adapter`，显然无法工作。

```java
        final int childrenTop = mListPadding.top;
        final int childrenBottom = mBottom - mTop - mListPadding.bottom;
        final int childCount = getChildCount();

        // ...

        boolean dataChanged = mDataChanged;
        if (dataChanged) {
            handleDataChanged();
        }

        if (mItemCount == 0) {
            resetList();
            invokeOnItemScrollListener();
            return;
        } else if (mItemCount != mAdapter.getCount()) {
            throw new IllegalStateException("/* ... */");
        }
        
        // ...
```

调用 `BaseAdapter.notifyDataSetChanged()` 时，`mDataChanged` 被置为 `true`，同时会发起新一趟的布局。如果 `mDataChange` 为 `true`，`AbsListView.handleDataChanged()` 将使用 `Adapter` 提供的信息更新 `mItemCount`、执行一些维护当前选中 Item 的数据结构的逻辑，并根据客户代码的配置确定 `mLayoutMode`。

如果 `Adapter` 提供的数据数量为 `0`，则提前结束布局过程；`ListView` 使用 `mItemCount` 与 `Adapter.getCount()` 的返回值是否一致来判断客户代码是否调用了 `BaseAdapter.notifyDataSetChanged()` 来告知 `AbsListView` 数据变更，这显然是考虑性能前提下的折衷行为。

```java
        final int firstPosition = mFirstPosition;
        final RecycleBin recycleBin = mRecycler;
        if (dataChanged) {
            for (int i = 0; i < childCount; i++) {
                recycleBin.addScrapView(getChildAt(i), firstPosition+i);
            }
        } else {
            recycleBin.fillActiveViews(childCount, firstPosition);
        }
        
        detachAllViewsFromParent();
        recycleBin.removeSkippedScrap();
```

`AdapterView.mFirstPosition` 维护的是当前 `AdapterView` 可见的首个子视图对应 `Adapter` 后备数据源的 `position`。

如[前文](#复用子视图)介绍 `RecycleBin` 时所言，`ListView` 判断：如果接受到客户代码的数据变更通知，则将目前位于 `ListView` 中的所有子视图加入到 Scrap 中，否则将它们归为 Active。两种做法有何区别？事实上，`layoutChildren` 在 `onLayout` 之外也会被调用（最典型的情形，可能是 `AbsListView` 处理『点击』手势的 `UP` 事件时）；而且，数据源没有发生改变，对于 `AbsListView` 是一个强烈的暗示：大部分的子视图理应都可以被**直接**复用。很快读者就会看到，**Active** 视图被复用时，`Adapter.getView()` 完全不会被调用，被复用的子视图直接被『摆回原位』，而 **Scrap** 视图则不然，`Adapter` 需要对其重新执行数据绑定。

无论子视图被归为 Active 还是 Scrap，它们都会被临时 `detach`：简单地修改子视图的 `parent` 引用为 `null` 和清理 `ViewGroup` 自身的 `mChildren` 引用，没有 re-layout，没有 re-draw，一个非常轻量的数据结构操作。在布局完成后，被复用的视图会被 `attach`，剩下的视图要么被归为 Scrap，要么被彻底丢弃（skipped scrap）。

做完上述准备工作之后，布局便正式开始。

```java
        switch (mLayoutMode) {
        case LAYOUT_SET_SELECTION:
            // ...
            break;
        case LAYOUT_SYNC:
            // ...
            break;
        case LAYOUT_FORCE_BOTTOM:
            // ...
            break;
        case LAYOUT_FORCE_TOP:
            // ...
            break;
        case LAYOUT_SPECIFIC:
            // ...
            break;
        case LAYOUT_MOVE_SELECTION:
            // ...
            break;
        default:
            if (childCount == 0) {
                if (!mStackFromBottom) {
                    final int position = lookForSelectablePosition(0, true);
                    setSelectedPositionInt(position);
                    sel = fillFromTop(childrenTop);
                } else {
                    final int position = lookForSelectablePosition(mItemCount - 1, false);
                    setSelectedPositionInt(position);
                    sel = fillUp(mItemCount - 1, childrenBottom);
                }
            } else {
                // ...
            }
            break;
        }
```

这个庞大的 `switch` 块处理了各种 layout mode 的情形，我们暂只关注最普遍的那一种（即 `mLayoutMode == LAYOUT_NORMAL`，落入 `default` 分支）。这一分支又分为两种情形：这一趟布局之前 `ListView` 已经有子视图 / 没有子视图。为了尽可能简化讨论，我们进一步缩小关注的范围，关注 `ListView` 还没有子视图的情形。

`mStackFromBottom` 代表 `ListView` 的一个可配置属性，通常只有通信应用的对话界面会使用它，在这种模式下，`ListView` 会优先把底部的视区空间填满。`ListView.lookForSelectablePosition()` 在只有触摸屏作为输入设备（home 键、电源键、音量键除外）时永远返回 `INVALID_POSITION`，即 `-1`，因此，在这个分支中传给 `ListView.fillFromTop()` 的实参几乎都是 `-1`。

毫无疑问 `fillFromTop` 是一个非常重要的方法，不过它和 `ListView` 中一系列 `fill*` 方法相似，最终的 *fill* 工作会交给一个更加通用的方法完成。

```java
        recycleBin.scrapActiveViews();

        removeUnusedFixedViews(mHeaderViewInfos);
        removeUnusedFixedViews(mFooterViewInfos);

        // ...
        
        mLayoutMode = LAYOUT_NORMAL;
        mDataChanged = false;
        
        // ...
    } finally {
        if (mFocusSelector != null) {
            mFocusSelector.onLayoutComplete();
        }
        if (!blockLayoutRequests) {
            mBlockLayoutRequests = false;
        }
    }
}
```

布局的收尾工作，`ListView` 将没有得以复用的 Active 视图转为 Scrap，并且 `remove` 布局开始时可能被 `detach` 了的 `Header` / `Footer` 视图。

### fill*

`ListView` 中定义了一系列 `fill*` 方法，用来搭配不同的 layout mode。我们从上节的 `fillFromTop` 出发：

```java
private View fillFromTop(int nextTop) {
    mFirstPosition = Math.min(mFirstPosition, mSelectedPosition);
    mFirstPosition = Math.min(mFirstPosition, mItemCount - 1);
    if (mFirstPosition < 0) {
        mFirstPosition = 0;
    }
    return fillDown(mFirstPosition, nextTop);
}
```

确定了 `mFirstPosition` 的值（且保证不越过 `Adapter` 后备数据边界）后，执行流程转入 `fillDown()`。`fillDown` 是一个相当通用的方法，许多其它 `fill*` 方法最终会委托 `fillDown` / `fillUp` 执行它们需要的语义。顾名思义，`fill` 是『填满』，`fillDown` 即为『向下填满』。`fillDown` 的两个 `int` 形参 `pos` 与 `nextTop` 分别表示『从 `Adapter` 后备数据的第 `pos` 个位置开始』和『将生成的子视图上边界放置在 `ListView` 的 `nextTop` 位置』。

```java
private View fillDown(int pos, int nextTop) {
    View selectedView = null;

    int end = (mBottom - mTop);
    if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
        end -= mListPadding.bottom;
    }

    while (nextTop < end && pos < mItemCount) {
        // is this the selected item?
        boolean selected = pos == mSelectedPosition;
        View child = makeAndAddView(pos, nextTop, true, mListPadding.left, selected);

        nextTop = child.getBottom() + mDividerHeight;
        if (selected) {
            selectedView = child;
        }
        pos++;
    }

    setVisibleRangeHint(mFirstPosition, mFirstPosition + getChildCount() - 1);
    return selectedView;
}
```

`fillDown` 首先确定填充到哪个位置为止（即确定 `end` 的值），然后执行 `while` 循环，将 `nextTop` 与 `end` 间（视数据和子视图高度而定，可能没有填到 `end`，数据就用完了；也可能有一个子视图的 `bottom` 超过了 `end`，这分别是 `while` 的两个退出条件）用 `Adapter` 提供的子视图填满。

### makeAndAddView 与 obtainView

读者的目光想必停留在了 `View child = makeAndAddView()` 这行上，没错，我们离向 `Adapter` 请求子视图、复用子视图、添加子视图等逻辑只差一步了。

```java
private View makeAndAddView(int position, int y, boolean flow, int childrenLeft,
        boolean selected) {
    if (!mDataChanged) {
        final View activeView = mRecycler.getActiveView(position);
        if (activeView != null) {
            setupChild(activeView, position, y, flow, childrenLeft, selected, true);
            return activeView;
        }
    }

    final View child = obtainView(position, mIsScrap);

    setupChild(child, position, y, flow, childrenLeft, selected, mIsScrap[0]);

    return child;
}
```

`makeAndAddView` 分为两种情形，即是否接收到客户代码的数据变化通知：如果没有，直接尝试重用同一位置的 Active 视图，否则通过 `obtainView` 尝试复用 Scrap 视图并通知 `Adapter.getView()`，与[布局前](#layoutChildren)的子视图回收逻辑保持一致。

```java
View obtainView(int position, boolean[] outMetadata) {
    outMetadata[0] = false;

    final View transientView = mRecycler.getTransientStateView(position);
    if (transientView != null) {
        final LayoutParams params = (LayoutParams) transientView.getLayoutParams();

        if (params.viewType == mAdapter.getItemViewType(position)) {
            final View updatedView = mAdapter.getView(position, transientView, this);

            if (updatedView != transientView) {
                setItemViewLayoutParams(updatedView, position);
                mRecycler.addScrapView(updatedView, position);
            }
        }

        outMetadata[0] = true;

        transientView.dispatchFinishTemporaryDetach();
        return transientView;
    }
```

`obtainView` 方法有一个*出参* `outMetadata`，该数组的首个元素表示 `obtainView` 返回的视图是否已经 `attach` 到 `Window` 上，这个信息将交给 `setupChild` 来进一步处理子视图。

`obtainView` 对待 `transientState` 的子视图与 Active 视图既有相似，又有不同：相同的是，它们都只能就地复用；不同的是，`transientState` 的子视图必须满足 `itemType` 与该位置上数据对应的 `itemType` 相同才会尝试复用。

```java
    final View scrapView = mRecycler.getScrapView(position);
    final View child = mAdapter.getView(position, scrapView, this);
    if (scrapView != null) {
        if (child != scrapView) {
            mRecycler.addScrapView(scrapView, position);
        } else if (child.isTemporarilyDetached()) {
            outMetadata[0] = true;

            child.dispatchFinishTemporaryDetach();
        }
    }
    
    // ...

    setItemViewLayoutParams(child, position);
    
    // ...

    return child;
}
```

没有 `transientState` 子视图供复用是更普遍的情形：取一个 Scrap 视图（此视图的 `viewType` 由 `RecycleBin` 保证与待使用的类型一致）作为 `Adapter.getView()` 的实参传入，如果客户代码复用了此 Scrap，判断它在布局开始前是否被 `detach` 过，修正出参 `outMetadata` 的值；否则，将 Scrap 重新放入 `RecycleBin`。

私有方法 `AbsListView.setItemViewLayoutParams()` 用于配置 `AbsListView` 子视图的布局参数，它会向 `Adapter` 请求当前位置的 `itemId`、`viewType` 并存放于布局参数中。

### setupChild

到目前为止，`makeAndAddView` 已经『make』出子视图，后面的『add』工作则交给 `ListView.setupChild()` 完成。

```java
private void setupChild(View child, int position, int y, boolean flowDown, int childrenLeft,
        boolean selected, boolean isAttachedToWindow) {
    // ...
    final boolean needToMeasure = !isAttachedToWindow || updateChildSelected
            || child.isLayoutRequested();

    // Respect layout params that are already in the view. Otherwise make
    // some up...
    AbsListView.LayoutParams p = (AbsListView.LayoutParams) child.getLayoutParams();
    if (p == null) {
        p = (AbsListView.LayoutParams) generateDefaultLayoutParams();
    }
    p.viewType = mAdapter.getItemViewType(position);
    p.isEnabled = mAdapter.isEnabled(position);

    // ...
```

添加子视图之后，如果该视图之前没有被 `attach` 到 `Window` 上（这就是先前提到的 `obtainView` 的出参之用途），或是子视图的 `View.forceLayout()` 被调用过，它还需要被重新 / 首次测量。在 `AbsListView.onLayout()` 中，如果实参 `changed` 为 `true`，则在调用 `layoutChildren` 前，`AbsListView` 会对当前其所有子视图和 `RecycleBin` 中的待复用视图调用 `forceLayout`，以备它们在当前这一趟布局被复用时，及时对它们重新测量、布局。

```java 
    if ((isAttachedToWindow && !p.forceAdd) || (p.recycledHeaderFooter
            && p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER)) {
        attachViewToParent(child, flowDown ? -1 : 0, p);

        if (isAttachedToWindow
                && (((AbsListView.LayoutParams) child.getLayoutParams()).scrappedFromPosition)
                        != position) {
            child.jumpDrawablesToCurrentState();
        }
    } else {
        p.forceAdd = false;
        if (p.viewType == AdapterView.ITEM_VIEW_TYPE_HEADER_OR_FOOTER) {
            p.recycledHeaderFooter = true;
        }
        addViewInLayout(child, flowDown ? -1 : 0, p, true);
        child.resolveRtlPropertiesIfNeeded();
    }
```

添加子视图分为两种情况：一种是添加复用的子视图（或是 `ListView` 的 `Header` / `Footer`），它们先前被 `detach`，所以只需要简单 `attach` 回来即可；另一种是添加全新的视图，则调用 `ViewGroup.addViewInLayout()` 来添加。关于为什么不要在布局过程中调用 `requestLayout()`（为什么这里没有用 `ViewGroup.addView()` 的原因），读者若有兴趣可观看[这个](#https://www.youtube.com/watch?v=HbAeTGoKG6k)视频。

`AbsListView.LayoutParams.forceAdd` 这个标记，在 `ListView` 在自身测量模式不为 `EXACTLY` 时，为*估算*自身尺寸，临时向 `Adapter` 请求子视图时被置为 `true`。在测量完成后，这些子视图被归为 Scrap 视图，但是它们从未被添加到 `ListView` 中（也就是说，它们从未被 `attach` 到 `Window` 上）。设置这个标记就是为了在 `setupChild` 中妥善地处理这种情形。

```java
    if (needToMeasure) {
        final int childWidthSpec = ViewGroup.getChildMeasureSpec(mWidthMeasureSpec,
                mListPadding.left + mListPadding.right, p.width);
        final int lpHeight = p.height;
        final int childHeightSpec;
        if (lpHeight > 0) {
            childHeightSpec = MeasureSpec.makeMeasureSpec(lpHeight, MeasureSpec.EXACTLY);
        } else {
            childHeightSpec = MeasureSpec.makeSafeMeasureSpec(getMeasuredHeight(),
                    MeasureSpec.UNSPECIFIED);
        }
        child.measure(childWidthSpec, childHeightSpec);
    } else {
        cleanupLayoutState(child);
    }

    final int w = child.getMeasuredWidth();
    final int h = child.getMeasuredHeight();
    final int childTop = flowDown ? y : y - h;

    if (needToMeasure) {
        final int childRight = childrenLeft + w;
        final int childBottom = childTop + h;
        child.layout(childrenLeft, childTop, childRight, childBottom);
    } else {
        child.offsetLeftAndRight(childrenLeft - child.getLeft());
        child.offsetTopAndBottom(childTop - child.getTop());
    }
    
    // ...
}
```

子视图添加 /  `attach` 完成，最后需要对它们做测量（如果需要的话）和布局。横向上，宽度测量规格沿用 `ListView` 的父布局指定给自己的那个；纵向上，如果子视图的布局参数高度指定了一个精确的值（即大于 `0`，而不是 `WRAP_CONTENT` / `MATCH_PARENT`），则使用 `EXACTLY` 模式采纳这个值，否则用 `UNSPECIFIED` 模式 + `ListView` 自身的测量高度。

布局就很直白了，根据入参 `flowDown` 决定将该子视图摆放在 `y` 的上方或是下方；如果是首次测量 / 布局，则需调用 `View.layout()`，否则直接用 `View.offset*And*()` 来平移子视图在 `ListView` 中的位置即可。

## 触摸事件处理

`AbsListView` 使用继承向子类提供多种布局的可能性，同样通过继承，对子类屏蔽了触摸事件处理的实现细节，这一层封装相较布局来说是非常干净的，`ListView` / `GridView` 中没有任何附加的处理手势的逻辑。

### touch mode

`AbsListView` 是一个可滚动的 `ViewGroup`，定义了如下所列的触摸模式。

模式|说明
---|---
`TOUCH_MODE_REST`|无触摸手势，默认模式
`TOUCH_MODE_DOWN`|收到一个 `DOWN` 事件，等待后续事件以转到其它模式
`TOUCH_MODE_TAP`|确定先前收到的 `DOWN` 事件是一次点按，等待后续事件以确定是普通点击还是长按
`TOUCH_MODE_DONE_WAITING`|确定先前收到的 `DOWN` 事件是一次长按，但仍未收到后续的其它事件
`TOUCH_MODE_SCROLL`|滚动手势
`TOUCH_MODE_FLING`|已收到 `UP` 事件，`AbsListView` 仍在执行扫动动画
`TOUCH_MODE_OVERSCROLL`|过度滚动手势（滚动开始时位于 `AbsListView` 的边缘）
`TOUCH_MODE_OVERFLING`|过度扫动手势（扫动速度在到达 `AbsListView` 边缘时仍未消耗完）

### 辅助数据结构

`AbsListView` 使用了数个成员来维护触摸事件信息，本节介绍其中关键的几个。注意 `mMotionY` 与 `mLastY` 的区别。

```java
// 触摸事件位于哪一个子视图之上（Adapter Position）
int mMotionPosition;

// 触摸模式
int mTouchMode = TOUCH_MODE_REST;

/**
 * `DOWN` 事件位于的那个子视图的 `top`
 */
int mMotionViewOriginalTop;

/**
 * The desired offset to the top of the mMotionPosition view after a scroll
 */
int mMotionViewNewTop;

// 当前接收到的 DOWN 事件的横坐标
int mMotionX;

// 当前接收到的 DOWN 事件的纵坐标
int mMotionY;

// 上一次接收到的事件的纵坐标
int mLastY;
```

### onTouchEvent

笔者刻意跳过了 `onInterceptTouchEvent`，其逻辑与普通的可滚动的视图相比没有值得一提的明显区别。

```java
@Override
public boolean onTouchEvent(MotionEvent ev) {
    // ...

    initVelocityTrackerIfNotExists();
    final MotionEvent vtev = MotionEvent.obtain(ev);

    final int actionMasked = ev.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        mNestedYOffset = 0;
    }
    vtev.offsetLocation(0, mNestedYOffset);
    switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: {
            onTouchDown(ev);
            break;
        }

        case MotionEvent.ACTION_MOVE: {
            onTouchMove(ev, vtev);
            break;
        }

        case MotionEvent.ACTION_UP: {
            onTouchUp(ev);
            break;
        }

        case MotionEvent.ACTION_CANCEL: {
            onTouchCancel();
            break;
        }

        case MotionEvent.ACTION_POINTER_UP: {
            // ...
            break;
        }

        case MotionEvent.ACTION_POINTER_DOWN: {
            // ...
            break;
        }
    }

    if (mVelocityTracker != null) {
        mVelocityTracker.addMovement(vtev);
    }
    vtev.recycle();
    return true;
}
```

可见，不同事件的处理封装到了不同的方法中。接下来，我们重点关注 `onTouchDown`、`onTouchMove` 与 `onTouchUp`。

#### onTouchDown

```java
private void onTouchDown(MotionEvent ev) {
    mHasPerformedLongPress = false;
    mActivePointerId = ev.getPointerId(0);

    if (mTouchMode == TOUCH_MODE_OVERFLING) {
        mFlingRunnable.endFling();
        if (mPositionScroller != null) {
            mPositionScroller.stop();
        }
        mTouchMode = TOUCH_MODE_OVERSCROLL;
        mMotionX = (int) ev.getX();
        mMotionY = (int) ev.getY();
        mLastY = mMotionY;
        mMotionCorrection = 0;
        mDirection = 0;
    }
```

如果正在执行『过度扫动』，在接收到 `DOWN` 事件时转入『过度滚动』态。

```java
    else {
        final int x = (int) ev.getX();
        final int y = (int) ev.getY();
        int motionPosition = pointToPosition(x, y);

        if (!mDataChanged) {
            if (mTouchMode == TOUCH_MODE_FLING) {
                // Stopped a fling. It is a scroll.
                createScrollingCache();
                mTouchMode = TOUCH_MODE_SCROLL;
                mMotionCorrection = 0;
                motionPosition = findMotionRow(y);
                mFlingRunnable.flywheelTouch();
            }
```

使用事件坐标找出对应的子视图的 Adapter Position。

如果正在执行『扫动』，转入『滚动』态。

```java
            else if ((motionPosition >= 0) && getAdapter().isEnabled(motionPosition)) {
                mTouchMode = TOUCH_MODE_DOWN;

                if (mPendingCheckForTap == null) {
                    mPendingCheckForTap = new CheckForTap();
                }

                mPendingCheckForTap.x = ev.getX();
                mPendingCheckForTap.y = ev.getY();
                postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
            }
        }
```

否则，进入 `TOUCH_MODE_DOWN` 态，并延迟 `ViewConfiguration.getTapTime()` 后在主线程上执行 `Runnable` `CheckForTap`。这个 `Runnable` 用于判断，在它执行时，`mTouchMode` 是否仍然是 `TOUCH_MODE_DOWN`，是则转入 `TOUCH_MODE_TAP` 态，并且再延迟执行另一个 `Runnable` `CheckForLongPress` 来判断长按手势并触发 `AdapterView.OnItemLongClickListener` 监听，并转入 `TOUCH_MODE_DONE_WAITING` 态。不过，对于普通的点按，`CheckForTap` 不一定得到机会执行，当 `UP` 事件早于其执行时，该 `Runnable` 就会从主线程队列上撤销，读者可以阅读[后文](#onTouchUp)加以确认。

```java
        if (motionPosition >= 0) {
            final View v = getChildAt(motionPosition - mFirstPosition);
            mMotionViewOriginalTop = v.getTop();
        }

        mMotionX = x;
        mMotionY = y;
        mMotionPosition = motionPosition;
        mLastY = Integer.MIN_VALUE;
    }

    // ...
}
```

最后是对触摸事件相关数据结构的更新。仍然提醒读者注意 `mMotionY` 与 `mLastY` 的区别。

#### onTouchUp

```java
private void onTouchUp(MotionEvent ev) {
    switch (mTouchMode) {
    case TOUCH_MODE_DOWN:
    case TOUCH_MODE_TAP:
    case TOUCH_MODE_DONE_WAITING:
        final int motionPosition = mMotionPosition;
        final View child = getChildAt(motionPosition - mFirstPosition);
        if (child != null) {
            if (mTouchMode != TOUCH_MODE_DOWN) {
                child.setPressed(false);
            }

            final float x = ev.getX();
            final boolean inList = x > mListPadding.left && x < getWidth() - mListPadding.right;
            if (inList && !child.hasFocusable()) {
                if (mPerformClick == null) {
                    mPerformClick = new PerformClick();
                }

                final AbsListView.PerformClick performClick = mPerformClick;
                performClick.mClickMotionPosition = motionPosition;
                performClick.rememberWindowAttachCount();

                mResurrectToPosition = motionPosition;

                if (mTouchMode == TOUCH_MODE_DOWN || mTouchMode == TOUCH_MODE_TAP) {
                    removeCallbacks(mTouchMode == TOUCH_MODE_DOWN ?
                            mPendingCheckForTap : mPendingCheckForLongPress);
                    mLayoutMode = LAYOUT_NORMAL;
                    if (!mDataChanged && mAdapter.isEnabled(motionPosition)) {
                        mTouchMode = TOUCH_MODE_TAP;
                        setSelectedPositionInt(mMotionPosition);
                        layoutChildren();
                        child.setPressed(true);
                        positionSelector(mMotionPosition, child);
                        setPressed(true);
                        if (mSelector != null) {
                            Drawable d = mSelector.getCurrent();
                            if (d != null && d instanceof TransitionDrawable) {
                                ((TransitionDrawable) d).resetTransition();
                            }
                            mSelector.setHotspot(x, ev.getY());
                        }
                        if (mTouchModeReset != null) {
                            removeCallbacks(mTouchModeReset);
                        }
                        mTouchModeReset = new Runnable() {
                            @Override
                            public void run() {
                                mTouchModeReset = null;
                                mTouchMode = TOUCH_MODE_REST;
                                child.setPressed(false);
                                setPressed(false);
                                if (!mDataChanged && !mIsDetaching && isAttachedToWindow()) {
                                    performClick.run();
                                }
                            }
                        };
                        postDelayed(mTouchModeReset,
                                ViewConfiguration.getPressedStateDuration());
                    } else {
                        mTouchMode = TOUCH_MODE_REST;
                        updateSelectorState();
                    }
                    return;
                } else if (!mDataChanged && mAdapter.isEnabled(motionPosition)) {
                    performClick.run();
                }
            }
        }
        mTouchMode = TOUCH_MODE_REST;
        updateSelectorState();
        break;
        
```

首先是对 `TOUCH_MODE_DOWN`、`TOUCH_MODE_TAP` 和 `TOUCH_MODE_DONE_WAITING` 三种情况的处理。与 `View` 的点按处理相似，`AbsListView` 的 Item 点按也是 `post` 一个 `Runnable` 到主线程上延迟触发点击监听——这是为了让点击的视觉效果得以在点击事件触发前得到执行。一个例外是 `TOUCH_MODE_DONE_WAITING`，在这种模式下，`PerformClick` 会就地执行。

需要特别说明的一点是，`PerformClick` 是一个 `WindowRunnable`，它有一个记录 `AbsListView` `windowAttachCount` 的成员，`AbsListView` 在执行 `WindowRunnable` 前会判断当前的 `windowAttachCount` 与 `WindowRunnable` 中缓存的计数是否相等，相等则执行，否则直接返回。

```java
    case TOUCH_MODE_SCROLL:
        final int childCount = getChildCount();
        if (childCount > 0) {
            final int firstChildTop = getChildAt(0).getTop();
            final int lastChildBottom = getChildAt(childCount - 1).getBottom();
            final int contentTop = mListPadding.top;
            final int contentBottom = getHeight() - mListPadding.bottom;
            if (mFirstPosition == 0 && firstChildTop >= contentTop &&
                    mFirstPosition + childCount < mItemCount &&
                    lastChildBottom <= getHeight() - contentBottom) {
                mTouchMode = TOUCH_MODE_REST;
                reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
            } else {
                final VelocityTracker velocityTracker = mVelocityTracker;
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);

                final int initialVelocity = (int)
                        (velocityTracker.getYVelocity(mActivePointerId) * mVelocityScale);
                boolean flingVelocity = Math.abs(initialVelocity) > mMinimumVelocity;
                if (flingVelocity &&
                        !((mFirstPosition == 0 &&
                                firstChildTop == contentTop - mOverscrollDistance) ||
                          (mFirstPosition + childCount == mItemCount &&
                                lastChildBottom == contentBottom + mOverscrollDistance))) {
                    if (!dispatchNestedPreFling(0, -initialVelocity)) {
                        if (mFlingRunnable == null) {
                            mFlingRunnable = new FlingRunnable();
                        }
                        reportScrollStateChange(OnScrollListener.SCROLL_STATE_FLING);
                        mFlingRunnable.start(-initialVelocity);
                        dispatchNestedFling(0, -initialVelocity, true);
                    } else {
                        mTouchMode = TOUCH_MODE_REST;
                        reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
                    }
                } else {
                    mTouchMode = TOUCH_MODE_REST;
                    reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
                    if (mFlingRunnable != null) {
                        mFlingRunnable.endFling();
                    }
                    if (mPositionScroller != null) {
                        mPositionScroller.stop();
                    }
                    if (flingVelocity && !dispatchNestedPreFling(0, -initialVelocity)) {
                        dispatchNestedFling(0, -initialVelocity, false);
                    }
                }
            }
        } else {
            mTouchMode = TOUCH_MODE_REST;
            reportScrollStateChange(OnScrollListener.SCROLL_STATE_IDLE);
        }
        break;
```

然后是对 `TOUCH_MODE_SCROLL` 的处理。对于一个可滚动视图，在滚动态接收到 `UP` 事件则意味着可能需要执行『扫动』动画，对 `AbsListView` 来说，还要更新 `mTouchMode` 为 `TOUCH_MODE_FLING`。如果已经到达 `AbsListView` 的边界，则 `mTouchMode` 直接转入 `TOUCH_MODE_REST` 态，不执行扫动；否则，如果手势的速度超过扫动阈值，使用 `AbsListView.FlingRunnale` 来执行每帧的扫动（这与 `ScrollView` 重载 `View.computeScroll`、在扫动结束前每一帧都调用 `View.postInvalidateOnAnimation` 的做法不同）。

`FlingRunnable` 使用 `VelocityTracker` 计算出扫动偏移后，调用 [`AbsListView.trackMotionScroll()`](#trackMotionScroll) 来滚动子视图。这个方法既用于跟随 `MOVE` 事件滚动子视图，又用于执行『扫动』效果。

```java
    case TOUCH_MODE_OVERSCROLL:
        if (mFlingRunnable == null) {
            mFlingRunnable = new FlingRunnable();
        }
        final VelocityTracker velocityTracker = mVelocityTracker;
        velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
        final int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);

        reportScrollStateChange(OnScrollListener.SCROLL_STATE_FLING);
        if (Math.abs(initialVelocity) > mMinimumVelocity) {
            mFlingRunnable.startOverfling(-initialVelocity);
        } else {
            mFlingRunnable.startSpringback();
        }

        break;
    }
    
    setPressed(false);
    
    // ...
    
    invalidate();
    removeCallbacks(mPendingCheckForLongPress);
    recycleVelocityTracker();

    mActivePointerId = INVALID_POINTER;
    
    // ...
}
```

最后是对 `TOUCH_MODE_OVERSCROLL` 的处理。`AbsListView` 将这种过度滚动后回弹的逻辑一并封装到了 `FlingRunnable` 中。

#### onTouchMove

相较于 `DOWN` 和 `UP` 事件，笔者自己在研究 `AbsListView` 时更在意 `MOVE` 事件的处理逻辑，因为它不仅涉及到对子视图的位移处理，还需及时通知子类，将因滚动产生的『空隙』用新的子视图填上。`AbsListView` 定义了一个抽象方法 `fillGap()` 来通知子类完成对空隙的填充。

```java
private void onTouchMove(MotionEvent ev, MotionEvent vtev) {
    if (mHasPerformedLongPress) {
        return;
    }

    int pointerIndex = ev.findPointerIndex(mActivePointerId);
    if (pointerIndex == -1) {
        pointerIndex = 0;
        mActivePointerId = ev.getPointerId(pointerIndex);
    }

    if (mDataChanged) {
        layoutChildren();
    }
```

`mHasPerformedLongPress` 为 `true` 表明 `AbsListView` 已经触发了长按监听，并且监听方法返回 `true`。此时 `onTouchMove` 静默消费所有 `MOVE` 事件。

如果接收到数据变化通知，在处理触摸事件前重布局。

```java
    final int y = (int) ev.getY(pointerIndex);

    switch (mTouchMode) {
        case TOUCH_MODE_DOWN:
        case TOUCH_MODE_TAP:
        case TOUCH_MODE_DONE_WAITING:
            if (startScrollIfNeeded((int) ev.getX(pointerIndex), y, vtev)) {
                break;
            }
            // ...
            break;
```

`MOVE` 事件的处理分两种情形，一种是对 `TOUCH_MODE_DOWN`、`TOUCH_MODE_TAP` 和 `TOUCH_MODE_DONE_WAITING` 三种情况的处理，使用 `AbsListView.startScrollIfNeeded()` 来判断 `MOVE` 事件是否满足开始滚动的条件（并执行滚动，如果条件满足的话）。

```java
        case TOUCH_MODE_SCROLL:
        case TOUCH_MODE_OVERSCROLL:
            scrollIfNeeded((int) ev.getX(pointerIndex), y, vtev);
            break;
    }
}
```

另一种情形是目前的触摸模式已经处于 `TOUCH_MODE_SCROLL` 或 `TOUCH_MOVE_OVERSCROLL`，则调用 `AbsListView.scrollIfNeeded()` 跟随 `MOVE` 事件滚动子视图即可。

### scrollIfNeeded

`AbsListView` 将对 `MOVE` 事件的处理分两步封装在了 `startScrollIfNeeded` 和 `scrollIfNeeded` 中：对 `MOVE` 事件的位移判断、对 `mTouchMode` 的更新、向父 `ViewParent` `requestDisallowInterceptTouchEvent(true)` 等相关操作封装在前者，真正跟随 `MOVE` 事件滚动内容的逻辑则封装在后者。本节主要关注后者的逻辑。

```java
private void scrollIfNeeded(int x, int y, MotionEvent vtev) {
    int rawDeltaY = y - mMotionY;
    int scrollOffsetCorrection = 0;
    int scrollConsumedCorrection = 0;
    if (mLastY == Integer.MIN_VALUE) {
        rawDeltaY -= mMotionCorrection;
    }
    if (dispatchNestedPreScroll(0, mLastY != Integer.MIN_VALUE ? mLastY - y : -rawDeltaY,
            mScrollConsumed, mScrollOffset)) {
        rawDeltaY += mScrollConsumed[1];
        scrollOffsetCorrection = -mScrollOffset[1];
        scrollConsumedCorrection = mScrollConsumed[1];
        if (vtev != null) {
            vtev.offsetLocation(0, mScrollOffset[1]);
            mNestedYOffset += mScrollOffset[1];
        }
    }
    final int deltaY = rawDeltaY;
    int incrementalDeltaY =
            mLastY != Integer.MIN_VALUE ? y - mLastY + scrollConsumedCorrection : deltaY;
    int lastYCorrection = 0;
```

首先确定滚动偏移 `deltaY` 与 `incrementalDeltaY`，这两个 `delta` 的意义不同，前者指『自该 `MOVE` 事件对应的 `DOWN` 事件以来，事件的累积纵向偏移』，而后者指『当前处理的 `MOVE` 事件距上一个 `MOVE` 事件的纵向偏移』。这部分代码中还掺杂有嵌套滚动相关的逻辑，读者如对相关话题感兴趣，可参阅笔者的[另一篇博文]({{ site.url }}/2017/08/28/handling-nested-scrolling.html)。

`scrollIfNeeded` 分为两种情形，一种是触摸模式为 `TOUCH_MODE_SCROLL`，另一种是 `TOUCH_MODE_OVERSCROLL`。首先看对前者的处理。

```java
    if (mTouchMode == TOUCH_MODE_SCROLL) {
        // ...

        if (y != mLastY) {
            // ...

            final int motionIndex;
            if (mMotionPosition >= 0) {
                motionIndex = mMotionPosition - mFirstPosition;
            } else {
                motionIndex = getChildCount() / 2;
            }

            int motionViewPrevTop = 0;
            View motionView = this.getChildAt(motionIndex);
            if (motionView != null) {
                motionViewPrevTop = motionView.getTop();
            }

            boolean atEdge = false;
            if (incrementalDeltaY != 0) {
                atEdge = trackMotionScroll(deltaY, incrementalDeltaY);
            }
```

根据 [`DOWN` 事件处理](#onTouchDown)中确定的 `mMotionPosition` 确定本趟滚动前触摸事件始于的子视图（后文中简称它为 **motion view**）的 `top`。如果 `mMotionPosition` 是一个无效值，则 `scrollIfNeeded` 使用 `getChildCount() / 2` 作为 `motionIndex` 一个估计值。稍后 `motionViewPrevTop` 会用于计算过度滚动的偏移。

如果本次 `MOVE` 事件距上次的偏移不为 `0`，则滚动 `AbsListView` 自身。[`AbsListView.trackMotionScroll()`](#trackMotionScroll) 是一个非常重要的方法，读者在 `FlingRunnable` 中已经看到了它一次。目前，读者只需要知道，如果 `AbsListView` 已经滚动到可滚动区域的尽头，`trackMotionScroll` 返回 `true`。
 
```java
            motionView = this.getChildAt(motionIndex);
            if (motionView != null) {
                final int motionViewRealTop = motionView.getTop();
                if (atEdge) {
                    int overscroll = -incrementalDeltaY -
                            (motionViewRealTop - motionViewPrevTop);
                    if (dispatchNestedScroll(0, overscroll - incrementalDeltaY, 0, overscroll,
                            mScrollOffset)) {
                        lastYCorrection -= mScrollOffset[1];
                        if (vtev != null) {
                            vtev.offsetLocation(0, mScrollOffset[1]);
                            mNestedYOffset += mScrollOffset[1];
                        }
                    } else {
                        final boolean atOverscrollEdge = overScrollBy(0, overscroll,
                                0, mScrollY, 0, 0, 0, mOverscrollDistance, true);

                        // ...
                    }
                }
                mMotionY = y + lastYCorrection + scrollOffsetCorrection;
            }
            mLastY = y + lastYCorrection + scrollOffsetCorrection;
        }
    }
```

针对 `TOUCH_MOVE_SCROLL` 情形的剩余逻辑均是对过度滚动的处理。`trackMotionScroll` 完成滚动后，`scrollIfNeeded` 使用滚动前确定的 `motionIndex` 再取一次 motion view 的 `top`，取前后差值加上 `incrementalDeltaY` 作为过度滚动的偏移量。笔者无意详细展开分析过度滚动效果的实现，这里只提醒读者注意：`AbsListView` 的过度滚动**的确**滚动了自身（调用了 `View.overScrollBy()`）。

```java
    else if (mTouchMode == TOUCH_MODE_OVERSCROLL) {
        if (y != mLastY) {
            final int oldScroll = mScrollY;
            final int newScroll = oldScroll - incrementalDeltaY;
            int newDirection = y > mLastY ? 1 : -1;

            if (mDirection == 0) {
                mDirection = newDirection;
            }

            int overScrollDistance = -incrementalDeltaY;
            if ((newScroll < 0 && oldScroll >= 0) || (newScroll > 0 && oldScroll <= 0)) {
                overScrollDistance = -oldScroll;
                incrementalDeltaY += overScrollDistance;
            } else {
                incrementalDeltaY = 0;
            }

            if (overScrollDistance != 0) {
                overScrollBy(0, overScrollDistance, 0, mScrollY, 0, 0,
                        0, mOverscrollDistance, true);
                // ...
            }

            if (incrementalDeltaY != 0) {
                if (mScrollY != 0) {
                    mScrollY = 0;
                    invalidateParentIfNeeded();
                }

                trackMotionScroll(incrementalDeltaY, incrementalDeltaY);

                mTouchMode = TOUCH_MODE_SCROLL;

                final int motionPosition = findClosestMotionRow(y);

                mMotionCorrection = 0;
                View motionView = getChildAt(motionPosition - mFirstPosition);
                mMotionViewOriginalTop = motionView != null ? motionView.getTop() : 0;
                mMotionY =  y + scrollOffsetCorrection;
                mMotionPosition = motionPosition;
            }
            mLastY = y + lastYCorrection + scrollOffsetCorrection;
            mDirection = newDirection;
        }
    }
}
```

`AbsListView` 位于 `TOUCH_MODE_OVERSCROLL` 模式时，接收到的 `MOVE` 事件有两种可能：一种是向边界方向滚动，这会触发过度滚动；另一种与前一种的滚动方向相反，过度滚动会减轻、消失，甚至可能会有少量滚动偏移剩余，足以滚动 `AbsListView` 的内容。因此，过度滚动和内容滚动可能会同时发生。

`scrollIfNeeded` 首先计算 `newScroll`——即如果本次 `MOVE` 的偏移量 `incrementalDeltaY` 加在 `AbsListView` 自身的 `scrollY` 值。如果 `oldScroll`（即当前 `AbsListView` 的 `scrollY`）与 `newScroll` 异号，则执行过度滚动的『回滚』，并将回滚的距离累加到 `incrementalDeltaY` 上，更新内容滚动的偏移；如果不异号，则把 `incrementalDeltaY` 作为过度滚动的偏移，不滚动 `AbsListView` 的内容（子视图）。

最后提一处有意思的『修正』：如果 `incrementalDeltaY` 不为 `0`（即需要滚动 `AbsListView` 内容），强行把 `AbsListView` 的 `mScrollY` 置为 `0`，退出过度滚动模式。加上目前我们已经看到的种种过度滚动的处理，想必给予了读者一个强烈的暗示：`AbsListView` 并不依赖自身的 `scrollY` 滚动其子视图（这与 `ScrollView` 和 `ViewPager` 两个常见的可滚动 `ViewGroup` 的实现不同）。

### trackMotionScroll

现在是时候向读者揭秘 `AbsListView` 是如何分别处理普通的内容滚动和过度滚动了。不过，`trackMotionScroll` 本质上是一个既管布局又管滚动的方法：当 `AbsListView` 因内容滚动而产生『空隙』时，它要调用 `fillGap()` 及时将空隙用新的子视图补上，这是布局过程，但此过程不需要完整的一趟布局(走 measure - layout 流程)。 

```java
boolean trackMotionScroll(int deltaY, int incrementalDeltaY) {
    final int childCount = getChildCount();
    if (childCount == 0) {
        return true;
    }

    final int firstTop = getChildAt(0).getTop();
    final int lastBottom = getChildAt(childCount - 1).getBottom();

    final Rect listPadding = mListPadding;

    int effectivePaddingTop = 0;
    int effectivePaddingBottom = 0;
    if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
        effectivePaddingTop = listPadding.top;
        effectivePaddingBottom = listPadding.bottom;
    }
    
    final int spaceAbove = effectivePaddingTop - firstTop;
    final int end = getHeight() - effectivePaddingBottom;
    final int spaceBelow = lastBottom - end;
```

计算首个子视图距顶部的空间和最后一个子视图距底部的空间，为 `fillGap` 做准备。

```java
    final int height = getHeight() - mPaddingBottom - mPaddingTop;
    if (deltaY < 0) {
        deltaY = Math.max(-(height - 1), deltaY);
    } else {
        deltaY = Math.min(height - 1, deltaY);
    }

    if (incrementalDeltaY < 0) {
        incrementalDeltaY = Math.max(-(height - 1), incrementalDeltaY);
    } else {
        incrementalDeltaY = Math.min(height - 1, incrementalDeltaY);
    }
```

保证偏移量不要大过 `AbsListView` 可用的高度。

```java
    final int firstPosition = mFirstPosition;

    // Update our guesses for where the first and last views are
    if (firstPosition == 0) {
        mFirstPositionDistanceGuess = firstTop - listPadding.top;
    } else {
        mFirstPositionDistanceGuess += incrementalDeltaY;
    }
    if (firstPosition + childCount == mItemCount) {
        mLastPositionDistanceGuess = lastBottom + listPadding.bottom;
    } else {
        mLastPositionDistanceGuess += incrementalDeltaY;
    }

    final boolean cannotScrollDown = (firstPosition == 0 &&
            firstTop >= listPadding.top && incrementalDeltaY >= 0);
    final boolean cannotScrollUp = (firstPosition + childCount == mItemCount &&
            lastBottom <= getHeight() - listPadding.bottom && incrementalDeltaY <= 0);

    if (cannotScrollDown || cannotScrollUp) {
        return incrementalDeltaY != 0;
    }
```

如果已经到达 `AbsListView` 边界，提前返回：`trackMotionScroll` 的返回值的意义即为是否已经到达 `AbsListView` 边界。

```java
    final boolean down = incrementalDeltaY < 0;

    final boolean inTouchMode = isInTouchMode();
    if (inTouchMode) {
        hideSelector();
    }

    final int headerViewsCount = getHeaderViewsCount();
    final int footerViewsStart = mItemCount - getFooterViewsCount();

    int start = 0;
    int count = 0;

    if (down) {
        int top = -incrementalDeltaY;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            top += listPadding.top;
        }
        for (int i = 0; i < childCount; i++) {
            final View child = getChildAt(i);
            if (child.getBottom() >= top) {
                break;
            } else {
                count++;
                int position = firstPosition + i;
                if (position >= headerViewsCount && position < footerViewsStart) {
                    child.clearAccessibilityFocus();
                    mRecycler.addScrapView(child, position);
                }
            }
        }
    } else {
        int bottom = getHeight() - incrementalDeltaY;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            bottom -= listPadding.bottom;
        }
        for (int i = childCount - 1; i >= 0; i--) {
            final View child = getChildAt(i);
            if (child.getTop() <= bottom) {
                break;
            } else {
                start = i;
                count++;
                int position = firstPosition + i;
                if (position >= headerViewsCount && position < footerViewsStart) {
                    child.clearAccessibilityFocus();
                    mRecycler.addScrapView(child, position);
                }
            }
        }
    }
```

使用 `incrementalDeltaY` 的符号来确定滚动的方向 `down`。

清理即将因滚动被移出 `AbsListView` 视区的子视图（也就是将这些子视图列为 Scrap 视图）。`if` 语句的两个分支几乎一模一样，区别是由滚动方向相反带来的：下滚时，最顶部的子视图最有可能被滚出视区，因此顺序遍历所有子视图，将滚动后 `bottom` 超过 `AbsListView` 上边界、并且不是 `Header` / `Footer` 的子视图加入 Scrap，直到该条件不满足，提前退出循环；上滚时同理，但遍历方向相反。在遍历的同时，更新 `start`、`count`，它们表示自 `start` 起连续的 `count` 个子视图需要被 `detach`。

```java
    mMotionViewNewTop = mMotionViewOriginalTop + deltaY;

    mBlockLayoutRequests = true;

    if (count > 0) {
        detachViewsFromParent(start, count);
        mRecycler.removeSkippedScrap();
    }

    if (!awakenScrollBars()) {
       invalidate();
    }

    offsetChildrenTopAndBottom(incrementalDeltaY);
```

置 `mBlockLayoutRequests` 为 `true`。`ListView.layoutChildren` 中也会置其为 `true`（如果进入 `layoutChildren` 时已经为 `true` 则直接返回）。然后，将先前标记为 Scrap 的子视图全部 `detach`。

唤起 `ScrollBar`，如果动画没有执行的话，则自行 `invalidate`。

然后就是读者*期待已久的谜题*的答案：`AbsListView` 通过直接修改所有子视图的 `top` 和 `bottom` 来模拟『滚动子视图』的效果。

```java
    if (down) {
        mFirstPosition += count;
    }

    final int absIncrementalDeltaY = Math.abs(incrementalDeltaY);
    if (spaceAbove < absIncrementalDeltaY || spaceBelow < absIncrementalDeltaY) {
        fillGap(down);
    }

    mRecycler.fullyDetachScrapViews();
    
    // ...

    mBlockLayoutRequests = false;

    invokeOnItemScrollListener();

    return false;
}
```

如果是下滚，则可以直接更新 `mFirstPosition`；上滚则无法确定（因为 `AbsListView` 无法确定填入上方空隙的子视图的数量），待 `fillGap` 修正。

`AbsListView` 的任一端空隙如果小于当前 `MOVE` 事件的滚动偏移，则需要用新的子视图将空隙填上。下一子小节中，笔者将分析 `ListView.fillGap()` 的实现。

空隙填充完毕后，通知 `RecycleBin` 将没有被复用的 Scrap `remove` 掉（`ViewGroup` 的语义，视图引用仍然被 `RecycleBin` 持有），然后复位 `mBlockLayoutRequests`。

#### ListView.fillGap

```java
@Override
void fillGap(boolean down) {
    final int count = getChildCount();
    if (down) {
        int paddingTop = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            paddingTop = getListPaddingTop();
        }
        final int startOffset = count > 0 ? getChildAt(count - 1).getBottom() + mDividerHeight :
                paddingTop;
        fillDown(mFirstPosition + count, startOffset);
        correctTooHigh(getChildCount());
    } else {
        int paddingBottom = 0;
        if ((mGroupFlags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK) {
            paddingBottom = getListPaddingBottom();
        }
        final int startOffset = count > 0 ? getChildAt(0).getTop() - mDividerHeight :
                getHeight() - paddingBottom;
        fillUp(mFirstPosition - 1, startOffset);
        correctTooLow(getChildCount());
    }
}
```

`ListView.fillGap` 通过入参 `down` 来确定填充 `ListView` 上端 / 下端的空隙；通过首个 / 最后一个子视图的 `top` / `bottom` 确定填充的起点，然后将这两个参数交给 `fillDown` / `fillUp`（笔者在[前文](#fill*)分析了前者的实现）。
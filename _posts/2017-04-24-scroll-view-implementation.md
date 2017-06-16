---
layout: post
title: ScrollView 实现分析
tags: Android View
---
* toc
{:toc}
`ScrollView` 继承自 `FrameLayout`。它与后者的主要不同之处是只能容纳一个孩子，并且*可能*可以被滚动。
## 我可以滚动吗？
作为一个*人类*，我们很容易想象 `ScrollView` 在什么条件下可以被滚动，那就是它唯一的那个子视图的高度大于它自己的高度时。那么 `ScrollView` 如何判定自己可以滚动？在 `View` 类中有一个 `canScrollVertically(int)` 方法，其中涉及到三个 `protected` 方法： `computeVerticalScrollOffset()`, `computeVerticalScrollRange()` 和 `computeVerticalScrollExtent()`， `ScrollView` 中覆盖了这三个方法中的前两个。首先来看一下这三个方法在 `View` 类中的定义与初始实现。

```java
/**
 * 计算垂直方向可滚动区域的起点距离当前可视的可滚动区域顶部的距离。
 */
protected int computeVerticalScrollOffset() {
    return mScrollY;
}

/**
 * 计算垂直方向的可滚动区域的长度。
 */
protected int computeVerticalScrollRange() {
    return getHeight();
}

/**
 * 计算垂直方向的可视区域长度。
 */
protected int computeVerticalScrollExtent() {
    return getHeight();
}
```

这三个方法的意义显得略为晦涩。接下来结合 `View.canScrollVertically(int)` 的实现帮助理解。

```java
public boolean canScrollVertically(int direction) {
    final int offset = computeVerticalScrollOffset();
    // 可滚动区域总长度减去可视区域长度
    final int range = computeVerticalScrollRange() - computeVerticalScrollExtent();
    // 如果没有剩余，则说明无法滚动。
    if (range == 0) return false;
    // 否则，根据传入的方向判断（大于等于 0 为向下滚，小于 0 则反之）：
    if (direction < 0) {
        // 向上滚，则只要当前位置不是顶部，则可以上滚。
        return offset > 0;
    } else {
        // 向下滚，则只要当前位置不是底部，则可以下滚。
        return offset < range - 1;
    }
}
```


看完了 `View.canScrollVertically()` 的实现之后，应该对这些 `protected` 方法的意义有非常清晰的认识了。现在，来看 `ScrollView` 中的覆盖实现。

```java
@Override
protected int computeVerticalScrollOffset() {
    // 没有什么特别的，只是对过度滚动做了一个保护（过度滚动时， scrollY 会短时变为负值）。
    return Math.max(0, super.computeVerticalScrollOffset());
}

@Override
protected int computeVerticalScrollRange() {
    final int count = getChildCount();
    final int contentHeight = getHeight() - mPaddingBottom - mPaddingTop;
    // 如果没有子视图，那就将除去了 padding 的自身高度作为可滚动区域大小
    if (count == 0) {
        return contentHeight;
    }

    // 取唯一的那个子视图的 bottom 
    int scrollRange = getChildAt(0).getBottom();
 
    // 本来取子视图的 bottom 作为可滚动区域大小已经足够了，但是因为 ScrollView 
    // 支持过度滚动，所以还要针对过度滚动修正一下这个值
    final int scrollY = mScrollY;
    // 计算没有发生过度下滚时的最大 scrollY（这个局部变量命名实在是太诡异了！）
    final int overscrollBottom = Math.max(0, scrollRange - contentHeight);
 
    if (scrollY < 0) {
        // 如果当前发生了过度上滚，要加上过度上滚的距离
        scrollRange -= scrollY;
    } else if (scrollY > overscrollBottom) {
        // 如果发生了过度下滚，加上过度下滚的距离
        scrollRange += scrollY - overscrollBottom;
    }

    return scrollRange;
}
```

## 我怎么测量和布局我的子视图？
乍一看上去， `ScrollView` 的测量过程似乎直接复用了 `FrameLayout` 的测量过程。不过它*悄悄地*覆盖了 `FrameLayout.onMeasure()` 中用到的一个定义在 `ViewGroup` 中的 `protected` 方法。

```java
@Override
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int usedTotal = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin +
            heightUsed;
    final int childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(
            Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - usedTotal),
            MeasureSpec.UNSPECIFIED); // <- 指定 UNSPECIFIED 测量规格模式。

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
可以看到，这个覆盖版本与 `ViewGroup` 中的实现的唯一区别就是对子视图的高度测量参数装配过程；布局过程就是完完全全对 `FrameLayout` 的复用了，滚动位置调整的逻辑稍后再关注。

## 我怎么处理触摸手势？
熟知事件分发机制的读者应该知道，对于一个 `ViewGroup` ，处理手势的逻辑分为两部分：拦截和拦截后的处理。**注意**：以下代码中省略了对**嵌套滚动**的处理。

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    final int action = ev.getAction();
    if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
        // 如果是单指 Move 手势并且正在被拖拽，拦截此事件
        return true;
    }

    if (super.onInterceptTouchEvent(ev)) {
        return true;
    }

    if (getScrollY() == 0 && !canScrollVertically(1)) {
        // 没有被滚动，并且不能被滚动，不拦截
        return false;
    }

    // ScrollView 支持多点手势（虽然在笔者看来挺鸡肋的），
    // 继续执行下面的判断前，抹去 MotionEvent 中的 Pointer 信息，关心 Action
    switch (action & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_MOVE: {
            
            final int activePointerId = mActivePointerId;
            if (activePointerId == INVALID_POINTER) {
                // ScrollView 没有缓存到 Down 手势 ID ，则该 Move 手势的
                // 起始位置不在 ScrollView 内部，不执行任何逻辑
                break;
            }

            // 找出 ScrollView 缓存的 Down 手势 ID 在该 MotionEvent 中对应的 index
            final int pointerIndex = ev.findPointerIndex(activePointerId);
            if (pointerIndex == -1) {
                // 失配，不执行任何逻辑
                Log.e(TAG, "Invalid pointerId=" + activePointerId
                        + " in onInterceptTouchEvent");
                break;
            }

            final int y = (int) ev.getY(pointerIndex);
            // 稍后关注一下 mLastMotionY 是如何被维护的。
            final int yDiff = Math.abs(y - mLastMotionY);
            // 此 Move 手势的纵轴坐标与 mLastMotionY 的差值超过滚动阈值，
            // 并且 ScrollView 的（直接或间接）子视图在纵轴方向没有发生嵌套滚动
            if (yDiff > mTouchSlop && (getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0) {
                // 可以判定 ScrollView 正在被拖拽了。
                mIsBeingDragged = true;
                // 更新上一次事件的纵轴坐标
                mLastMotionY = y;
                // 尝试初始化手势速度追踪器。速度追踪器用来计算滚动手势抬手并发起『扫动』
                // 时的扫动速度。
                // VelocityTracker 使用了对象池设计。
                // 这个方法就是从池中尝试取一个对象出来。
                initVelocityTrackerIfNotExists();
                // 把该手势添加到追踪器。
                mVelocityTracker.addMovement(ev);
                // 如果 ScrollView 自身有父视图，要求父视图不要再拦截此事件
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }
            break;
        }

        case MotionEvent.ACTION_DOWN: {
            final int y = (int) ev.getY();
            // Down 事件没有落在 ScrollView 的子视图中，不拦截
            if (!inChild((int) ev.getX(), (int) y)) {
                mIsBeingDragged = false;
                recycleVelocityTracker();
                break;
            }

            // 否则，记下此事件
            mLastMotionY = y;
            // ACTION_DOWN 的 pointer index 永远是 0
            mActivePointerId = ev.getPointerId(0);
            // 如果已经有速度追踪器的引用，复位它，否则从对象池取新的
            initOrResetVelocityTracker();
            // 将该手势事件添加到追踪器
            mVelocityTracker.addMovement(ev);
 
            // 计算新的滚动偏移，用于判断是否正在执行『扫动』动画
            mScroller.computeScrollOffset();
            // 如果正在执行扫动，isFinished 应该返回 false ，此时用户触摸屏幕，
            // 视为准备重新发起拖拽。
            // 注意，必须先调用 computeScrollOffset, isFinished 才会返回正确的结果
            // （扫动是否结束）
            mIsBeingDragged = !mScroller.isFinished();
            break;
        }

        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP:
            // Up / Cancel 手势，复位相关成员的值
            mIsBeingDragged = false;
            mActivePointerId = INVALID_POINTER;
            // 回收速度追踪器，放入对象池
            recycleVelocityTracker();
            // 
            if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange())) {
                postInvalidateOnAnimation();
            }
            break;
        case MotionEvent.ACTION_POINTER_UP:
            // 多点触摸时其中一只手指抬起，如果是正在跟踪的那一个 pointer，
            // 就把另一个 pointer 更新为正在跟踪的那个，否则什么都不做即可
            onSecondaryPointerUp(ev);
            break;
    }

    // 只在判断被拖拽时拦截手势。    
    return mIsBeingDragged;
}

@Override
public boolean onTouchEvent(MotionEvent ev) {
    initVelocityTrackerIfNotExists();

    // 要对该事件的纵轴坐标做一些修改，因此复制一份
    MotionEvent vtev = MotionEvent.obtain(ev);

    final int actionMasked = ev.getActionMasked();

    if (actionMasked == MotionEvent.ACTION_DOWN) {
        mNestedYOffset = 0;
    }
    vtev.offsetLocation(0, mNestedYOffset);

    switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: 
            // 进到这个 case 说明 ScrollView 的子视图（如果有的话）
            // 没有消费 Down 事件，递归返回中被传递回来了，处理逻辑与拦截
            // （ onInterceptTouchEvent ）基本一致
 
            if (getChildCount() == 0) {
                // Down 事件，但是没有子视图，放弃处理该手势
                return false;
            }
            if ((mIsBeingDragged = !mScroller.isFinished())) {
                // 如果正在执行『扫动』动画，判定 ScrollView 开始被拖拽，与拦截的逻辑一致
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }

            if (!mScroller.isFinished()) {
                // 如果正在执行『扫动』动画，停止动画
                mScroller.abortAnimation();
            }

            // 记下此事件
            mLastMotionY = (int) ev.getY();
            mActivePointerId = ev.getPointerId(0);
            break;
        }
        case MotionEvent.ACTION_MOVE:
            final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
            if (activePointerIndex == -1) {
                // 收到的 Move 事件不包含 ScrollView 确定被拖拽基于的那个 pointer，
                // 不处理
                Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent");
                break;
            }

            // 获取与上次手势的纵向的偏移
            final int y = (int) ev.getY(activePointerIndex);
            int deltaY = mLastMotionY - y;
            if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
                // 没有在被拖拽，但是偏移超过了滚动阈值
                // 让父视图不要截断后续的触摸事件
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
                // 判定 ScrollView 正在被拖拽，还将执行后面的逻辑
                mIsBeingDragged = true;
                // 将滚动阈值消费掉
                if (deltaY > 0) {
                    deltaY -= mTouchSlop;
                } else {
                    deltaY += mTouchSlop;
                }
            }
            if (mIsBeingDragged) {
                // 正在被拖拽，跟随手势滚动视图
                // 为何要修正 mLastMotionY？
                mLastMotionY = y - mScrollOffset[1];

                final int oldY = mScrollY;
                // getScrollRange 这个方法返回的是 ScrollView 的可滚动范围长度，
                // 即为子视图高度减去 ScrollView 可视范围的高度
                final int range = getScrollRange();
                /** 
                 *  View 中定义了三种过度滚动模式：
                 *  1. OVER_SCROLL_ALWAYS（默认值）：只要是一个可滚动的视图，那就永远允许过度滚动
                 *  2. OVER_SCROLL_IF_CONTENT_SCROLLS（ ScrollView 使用的值 ）：
                 *                                   如果可滚动内容的长度足以滚动，允许过度滚动
                 *  3. OVER_SCROLL_NEVER：不允许过度滚动
                 */
                final int overscrollMode = getOverScrollMode();
                // 根据过度滚动模式以及可滚动范围判定 ScrollView 是否可以过度滚动
                boolean canOverscroll = overscrollMode == OVER_SCROLL_ALWAYS ||
                        (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

                // overScrollBy 是一个定义在 View 类中的 protected 便利方法，
                // 调用这个方法的视图需要覆盖 onOverScrolled() 方法
                // （overScrollBy 在通过简单的计算后会将计算结果通过调用
                // onOverScrolled 传递给调用者）。这个方法如果返回 true 则说明
                // 过度滚动发生。ScrollView 的 onOverScrolled() 中就是滚动
                // （调用 scrollTo ）实际发生的地方。
                if (overScrollBy(0, deltaY, 0, mScrollY, 0, range, 0, mOverscrollDistance, true)
                        && !hasNestedScrollingParent()) {
                    // 如果发生过度滚动，复位速度追踪器
                    mVelocityTracker.clear();
                }

                if (canOverscroll) {
                    // 过度滚动边缘效果处理逻辑，省略。
                }
            }
            break;
        case MotionEvent.ACTION_UP:
            if (mIsBeingDragged) {
                // 正在被拖拽时收到 Up 手势，使用速度追踪器计算扫动速度，
                // 准备发起『扫动』动画。
                final VelocityTracker velocityTracker = mVelocityTracker;
                // 计算当前的速度（按像素 / 秒计）。
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                // 取到所求的初始速度。
                int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);

 
                if ((Math.abs(initialVelocity) > mMinimumVelocity)) {
                    // 如果超过了允许的最小扫动发起速度，则发起扫动
                    flingWithNestedDispatch(-initialVelocity);
                } else if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0,
                        getScrollRange())) {
                    // OverScroller.springBack() 方法用来发起过度滚动后的弹回效果，
                    // 如果返回 true 说明弹回效果成功发起，需要 invalidate
                    postInvalidateOnAnimation();
                }

 
                // 一次拖拽手势处理完毕，复位 pointer id
                mActivePointerId = INVALID_POINTER;
                // endDrag 也是一些成员的复位操作
                endDrag();
            }
            break;
        case MotionEvent.ACTION_CANCEL:
            if (mIsBeingDragged && getChildCount() > 0) {
                // Cancel 手势只尝试回弹而不发起『扫动』。
                if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange())) {
                    postInvalidateOnAnimation();
                }
                mActivePointerId = INVALID_POINTER;
                endDrag();
            }
            break;
        case MotionEvent.ACTION_POINTER_DOWN: {
            // ScrollView 的多点触摸处理。如果有另一个 pointer 按下，则覆盖掉之前追踪的 pointer 。
            final int index = ev.getActionIndex();
            mLastMotionY = (int) ev.getY(index);
            mActivePointerId = ev.getPointerId(index);
            break;
        }
        case MotionEvent.ACTION_POINTER_UP:
            // 如果有一个 pointer 按下，并且它是当前追踪的 pointer，
            // 则用另外一个 pointer 替换成为当前追踪的 pointer 。
            onSecondaryPointerUp(ev);
            mLastMotionY = (int) ev.getY(ev.findPointerIndex(mActivePointerId));
            break;
    }

    if (mVelocityTracker != null) {
        // 将该手势加入速度追踪器
        mVelocityTracker.addMovement(vtev);
    }
    // 回收该事件（它是一个拷贝！）
    vtev.recycle();
    return true;
}
```

## 我怎样滚动自己？我怎么实现『扫动』的效果？

我们已经在手势处理部分看到一些 `ScrollView` 如何随触摸事件滚动、扫动自己的逻辑了：因为支持过度滚动，所以调用 `overScrollBy` 并覆盖 `onOverScrolled`，在重载中最终调用 `View.scrollTo`；但是扫动呢？
`ScrollView` 使用 `OverScroller` 来实现过度滚动，本质上 `OverScroller` 只是封装了滚动坐标的计算过程，它没有直接控制滚动。使用 `Scroller` 或者 `OverScroller` 的视图，需要覆盖 `View.computeScroll` 方法，在该方法中更新 `mScrollX` 或 `mScrollY`。下面来逐个看一下这些方法的实现。

```java
protected boolean overScrollBy(int deltaX, int deltaY,
        int scrollX, int scrollY,
        int scrollRangeX, int scrollRangeY,
        int maxOverScrollX, int maxOverScrollY,
        boolean isTouchEvent) {
    final int overScrollMode = mOverScrollMode;
    // 这里可以看到 View 也是通过先前介绍的几个 compute* 方法来判断是否可滚动的。
    final boolean canScrollHorizontal =
            computeHorizontalScrollRange() > computeHorizontalScrollExtent();
    final boolean canScrollVertical =
            computeVerticalScrollRange() > computeVerticalScrollExtent();
    final boolean overScrollHorizontal = overScrollMode == OVER_SCROLL_ALWAYS ||
            (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollHorizontal);
    final boolean overScrollVertical = overScrollMode == OVER_SCROLL_ALWAYS ||
            (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollVertical);

    // 计算滚动后的新 scroll*
    int newScrollX = scrollX + deltaX;
    int newScrollY = scrollY + deltaY;
 
    // 如果某个方向上不支持过度滚动，直接置 maxOverScroll* 为 0
    if (!overScrollHorizontal) {
        maxOverScrollX = 0;
    }
    if (!overScrollVertical) {
        maxOverScrollY = 0;
    }

    // 计算过度滚动的边界。
    final int left = -maxOverScrollX;
    final int right = maxOverScrollX + scrollRangeX;
    final int top = -maxOverScrollY;
    final int bottom = maxOverScrollY + scrollRangeY;

    // clamped* 表示在某个坐标轴方向超过了边界，
    // 下面分别在横轴、纵轴上判断是否超限，更新标记并修正新的 scroll* 。
    boolean clampedX = false;
    if (newScrollX > right) {
        newScrollX = right;
        clampedX = true;
    } else if (newScrollX < left) {
        newScrollX = left;
        clampedX = true;
    }

    boolean clampedY = false;
    if (newScrollY > bottom) {
        newScrollY = bottom;
        clampedY = true;
    } else if (newScrollY < top) {
        newScrollY = top;
        clampedY = true;
    }

    // 将结果传给 onOverScrolled ，可以看到这个方法并未做实际滚动操作，
    // 只是完成了一些计算工作。
    onOverScrolled(newScrollX, newScrollY, clampedX, clampedY);

    return clampedX || clampedY;
}

@Override
protected void onOverScrolled(int scrollX, int scrollY,
        boolean clampedX, boolean clampedY) {
    if (!mScroller.isFinished()) {
        // 如果当前滚动没有完成，模拟 View.scrollTo，直接修改 mScroll*
        final int oldX = mScrollX;
        final int oldY = mScrollY;
        mScrollX = scrollX;
        mScrollY = scrollY;
        invalidateParentIfNeeded();
        // onScrollChanged 是定义在 View 中的 protected 方法，
        // View.scrollTo 或 View.scrollBy 的调用均会触发对它的调用，
        // 这里 ScrollView 自己改了 mScroll* ，因此自己调了 onScrollChanged
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        // 如果纵轴过度滚动了，通知 OverScroller 发起回弹
        if (clampedY) {
            mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange());
        }
    } else {
        // 没有正在进行中的滚动，调 View 的 scrollTo，因为 scrollTo 会把滚动条绘制出来，
        // 而这不是 ScrollView 的预期行为
        super.scrollTo(scrollX, scrollY);
    }
}

@Override
public void computeScroll() {
    // 让 Scroller 计算当前的滚动偏移，如果返回 true 则说明滚动尚未完成，需要取计算结果
    // 来更新 scrollY
    if (mScroller.computeScrollOffset()) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        int x = mScroller.getCurrX();
        int y = mScroller.getCurrY();

        // 如果任一新值与旧值不等，则滚动
        if (oldX != x || oldY != y) {
            final int range = getScrollRange();
            final int overscrollMode = getOverScrollMode();
            final boolean canOverscroll = overscrollMode == OVER_SCROLL_ALWAYS ||
                    (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

            overScrollBy(x - oldX, y - oldY, oldX, oldY, 0, range,
                    0, mOverflingDistance, false);
            // 这里 ScrollView 又调用了一次 onScrollChanged，不解
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        }
    }
}
```
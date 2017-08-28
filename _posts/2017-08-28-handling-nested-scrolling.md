---
layout: post
title: 处理嵌套滚动
tags: [Android, View]
---

Android 对嵌套滚动在 framework 中的支持是从 API 21 开始引入的，在此之前，任何对相互嵌套可滚动视图的尝试势必会得到令人失望的体验。但即便 V4 包加入了对低版本 API 的嵌套滚动的向下兼容，获得的体验仍可能与需求有差距（比如很多开发者试过的 `ScrollView` 嵌套 `ListView`）。

## 接入方式

嵌套滚动的本质是子视图处理手势时与父视图通过一套特殊协议完成的交互。由于这套协议是从 API 21 开始才进入 framework，所以如果要实现一个既支持嵌套滚动，又可以在 API 4 和以上的 OS 中使用的视图，需要使用 V4 包提供的如下工具：

* `android.support.v4.view.NestedScrollingChild`：定义了与支持嵌套滚动的父 `ViewGroup` 通信的接口。视图不仅要实现该接口，还要把 `android.support.v4.view.NestedScrollingChildHelper` 中存在的与前者签名一致的方法委托给后者执行（本质上，这个 `Helper` 就是把支持嵌套滚动所需的数据结构和逻辑从 framework 中拷贝出来了）。API 21 后此接口中的方法及 `Helper` 实现已进入 `View` 中。
* `android.support.v4.view.NestedScrollingParent`：定义了与支持嵌套滚动的子视图通信的接口。`ViewGroup` 不仅要实现该接口，还要把 `android.support.v4.view.NestedScrollingParentHelper` 中存在的与前者签名一致的方法委托给后者执行（与 `NestedScrollingChild` 相同的原因）。API 21 后此接口中的方法及 `Helper` 实现已进入 `ViewGroup` 中。

### 实现额外接口

除了把与各自 `Helper` 签名相同的方法委托给 `Helper` 执行外，剩下的方法（主要来自于 `NestedScrollingParent`）需开发者根据需求自行实现。[后文](#通信原理)中笔者将讲解其中关键方法的意义。

### 手势处理过程中插入附加逻辑

#### 子视图的 `onTouchEvent` 中处理 `ACTION_DOWN` 事件

当 `DOWN` 事件传递到一个 `ViewGroup` 的 `onTouchEvent` 中时，这表明它的直接 / 间接子视图没有消费此事件，一个可滚动的视图此时可假定该事件是一个拖拽手势的起点，在此处调用 `NestedScrollingChild.startNestedScroll(int)`。此方法接收一个 `int` 参数，可为 `ViewCompat.SCROLL_AXIS_HORIZONTAL` 或 `ViewCompat.SCROLL_AXIS_VERTICAL` 或这两个值的按位或（表示横向、纵向均可滚动）。下面是 `android.support.v4.widget.NestedScrollView` 的 `onTouchEvent` 中对 `DOWN` 事件的处理代码。

```java
case MotionEvent.ACTION_DOWN: {
    if (getChildCount() == 0) {
        return false;
    }
    if ((mIsBeingDragged = !mScroller.isFinished())) {
        final ViewParent parent = getParent();
        if (parent != null) {
            parent.requestDisallowInterceptTouchEvent(true);
        }
    }

    if (!mScroller.isFinished()) {
        mScroller.abortAnimation();
    }

    mLastMotionY = (int) ev.getY();
    mActivePointerId = ev.getPointerId(0);
    startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
    break;
}
```

#### 子视图处理 `ACTION_MOVE` 事件

当视图接收到 `MOVE` 事件，并且此事件距缓存的上一次事件的偏移超过了滚动手势的阈值，则视图视该手势为拖拽手势。视图在确定滚动偏移之前，需要与支持嵌套滚动的父 `ViewParent`（这个 `ViewParent` 并不一定是子视图的直接父亲）交互一次，即调用 `dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow)`；在执行实际滚动后（也可以在滚动前或是没有执行滚动，见 API 21 及更新的 `AbsListView`），再与父 `ViewParent` 通过 `dispatchNestedScroll(int dxConsumed, int dyConsumed, int dxUnconsumed, int dyUnconsumed, int[] offsetInWindow)` 交互一次。下面是精简过（但刻意暴露出交互逻辑）的 `NestedScrollView` 在 `onTouchEvent` 中处理 `MOVE` 事件的代码，这两步交互的意义，笔者将在[后文](#执行嵌套滚动)中分析。

```java
case MotionEvent.ACTION_MOVE:
    // ...

    final int y = (int) ev.getY(activePointerIndex);
    int deltaY = mLastMotionY - y;
    if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) {
        deltaY -= mScrollConsumed[1];
        vtev.offsetLocation(0, mScrollOffset[1]);
        mNestedYOffset += mScrollOffset[1];
    }

    // ...

    if (mIsBeingDragged) {
        // ...

        final int scrolledDeltaY = getScrollY() - oldY;
        final int unconsumedY = deltaY - scrolledDeltaY;
        if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
            mLastMotionY -= mScrollOffset[1];
            vtev.offsetLocation(0, mScrollOffset[1]);
            mNestedYOffset += mScrollOffset[1];
        } else if (canOverscroll) { /* ... */ }
    }
    break;
```

#### 子视图处理 `ACTION_UP` 事件

当视图接收到 `UP` 事件时，如果它正在被拖拽，则可根据手势的速度决定是否执行『扫动』动画，并退出拖拽状态。子视图如判定手势满足执行扫动动画的条件，应先调用 `boolean dispatchNestedPreFling(int, int)` 告知父 `ViewGroup`，由该方法返回值判断是否执行扫动动画；执行扫动动画前，应调用 `dispatchNestedFling(int, int, boolean)`；无论是否执行了扫动动画，在退出拖拽态时，都要调用 `stopNestedScroll()` 通知父 `ViewGroup`。下面是 `NestedScrollView` 处理扫动、停止拖拽的代码。

```java
private void flingWithNestedDispatch(int velocityY) {
    final int scrollY = getScrollY();
    final boolean canFling = (scrollY > 0 || velocityY > 0)
            && (scrollY < getScrollRange() || velocityY < 0);
    if (!dispatchNestedPreFling(0, velocityY)) {
        dispatchNestedFling(0, velocityY, canFling);
        if (canFling) {
            fling(velocityY);
        }
    }
}

private void endDrag() {
    mIsBeingDragged = false;

    recycleVelocityTracker();
    stopNestedScroll();

    if (mEdgeGlowTop != null) {
        mEdgeGlowTop.onRelease();
        mEdgeGlowBottom.onRelease();
    }
}
```

#### 父布局处理 `ACTION_MOVE` 事件

支持嵌套滚动的父 `ViewGroup` 主要需要实现 `NestedScrollingParent` 中定义的数个方法以完成和子视图的交互。此外，父 `ViewGroup` 自己处理触摸事件时，需要调用 `getNestedScrollAxes()` 判断当前是否有一个相同方向的嵌套滚动正在执行。下面是 `NestedScrollView` 在 `onInterceptTouchEvent` 中处理 `MOVE` 事件的代码。

```java
case MotionEvent.ACTION_MOVE: {
    final int activePointerId = mActivePointerId;
    if (activePointerId == INVALID_POINTER) {
        break;
    }

    final int pointerIndex = ev.findPointerIndex(activePointerId);
    if (pointerIndex == -1) {
        Log.e(TAG, "Invalid pointerId=" + activePointerId
                + " in onInterceptTouchEvent");
        break;
    }

    final int y = (int) ev.getY(pointerIndex);
    final int yDiff = Math.abs(y - mLastMotionY);
    if (yDiff > mTouchSlop
            && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
        mIsBeingDragged = true;
        mLastMotionY = y;
        initVelocityTrackerIfNotExists();
        mVelocityTracker.addMovement(ev);
        mNestedYOffset = 0;
        final ViewParent parent = getParent();
        if (parent != null) {
            parent.requestDisallowInterceptTouchEvent(true);
        }
    }
    break;
}
```

## 通信原理

### 开始嵌套滚动

支持嵌套滚动的子视图调用 `NestedScrollingChild.startNestedScroll()` 来通知一个支持嵌套滚动的直接 / 间接父 `ViewGroup` 开始一次新的嵌套滚动。这部分逻辑通用性极高，由 `NestedScrollingChildHelper` 实现，客户代码只需将调用委托给它即可。

```java
public boolean startNestedScroll(int axes) {
    if (hasNestedScrollingParent()) {
        // Already in progress
        return true;
    }
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        while (p != null) {
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes)) {
                mNestedScrollingParent = p;
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}
```

该实现在 `View` 树中逐层向上，寻找一个满足条件的父 `ViewParent`：对其调用 `NestedScrollingParent.onStartNestedScroll()`。如果该 `ViewParent` 接受此嵌套滚动，即 `onStartNestedScroll` 返回 `true`，则对其调用 `NestedScrollingParent.onNestedScrollAccepted()`。

对于不同的 `ViewGroup`，`onStartNestedScroll` 显然可根据其自身特性提供不同的实现。这里，请读者看一个最简单的 `NestedScrollView` 的版本：只要这个嵌套滚动的方向是纵向，就接受它。

```java
@Override
public boolean onStartNestedScroll(View child, View target, int nestedScrollAxes) {
    return (nestedScrollAxes & ViewCompat.SCROLL_AXIS_VERTICAL) != 0;
}
```

虽然 `NestedScrollingParentHelper` 提供了 `onNestedScrollAccepted` 的默认实现，不同的 `ViewGroup` 依然可以根据自身特性执行一些其它的逻辑。仍然看 `NestedScrollView` 的实现：首先执行 `Helper` 的实现，然后将该嵌套滚动事件传递到上层——`NestedScrollView` 既是一个 `NestedScrollingParent`，也是 `NestedScrollingChild`。

```java
@Override
public void onNestedScrollAccepted(View child, View target, int nestedScrollAxes) {
    mParentHelper.onNestedScrollAccepted(child, target, nestedScrollAxes);
    startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL);
}
```

`NestedScrollingParentHelper` 的对应实现很简陋，仅更新了当前嵌套滚动的方向标记。

```java
public void onNestedScrollAccepted(View child, View target, int axes) {
    mNestedScrollAxes = axes;
}
```

还有一点值得关注的是：如果一个支持嵌套滚动的视图同时是一个支持嵌套滚动的 `ViewParent`，则其自己开始滚动前，需要调用 `NestedScrollingParent.getNestedScrollAxes()` 判断该方向上是否有一个正在进行中的嵌套滚动。依然使用 `NestedScrollView` 作为例子：

```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    // ...

    switch (action & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_MOVE: {
            // ...

            if (yDiff > mTouchSlop && (getNestedScrollAxes() & SCROLL_AXIS_VERTICAL) == 0) {
                mIsBeingDragged = true;

                // ...
            }
            break;
        }

        // ...
    }

    return mIsBeingDragged;
}
```

### 结束嵌套滚动

当子视图判断滚动结束时，调用 `stopNestedScroll()`。`NestedScrollingChildHelper` 提供的默认实现将嵌套滚动结束事件通知给 `start` 时确定的支持嵌套滚动的直接 / 间接 `ViewParent`；`NestedScrollingParentHelper` 的对应实现同样简陋，只重置了先前维护的嵌套滚动方向。 

```java
public void stopNestedScroll() {
    if (mNestedScrollingParent != null) {
        ViewParentCompat.onStopNestedScroll(mNestedScrollingParent, mView);
        mNestedScrollingParent = null;
    }
}

public void onStopNestedScroll(View target) {
    mNestedScrollAxes = 0;
}
```

`NestedScrollView` 的实现中，还将此事件传递给上层，原因在上一节已经阐述。

```java
@Override
public void onStopNestedScroll(View target) {
    mParentHelper.onStopNestedScroll(target);
    stopNestedScroll();
}
```

### 执行嵌套滚动

我们熟悉的触摸事件传递机制存在一个问题：如果子视图决定拦截一个事件，则直到 `UP` 事件发生前，该视图的直接 / 间接 `ViewParent` 都没有机会处理这个事件的后续事件——这将造成父 `ViewParent` 无法跟随**始于可滚动的子视图区域**的触摸事件滚动。

新的嵌套滚动框架为父 `ViewGroup` 处理、消费被子视图拦截了的 `MOVE` 事件的机会。笔者在[前面](#手势处理过程中插入附加逻辑)已经展示了子视图执行滚动时需要与父 `ViewParent` 进行的两步交互。

#### 子视图滚动前

##### `child.dispatchNestedPreScroll`

子视图在确定了手势的偏移后，不再直接将其记作自身的滚动偏移，支持嵌套滚动的父 `ViewParent` **有资格**先消费一部分该手势的偏移。

```java
public boolean dispatchNestedPreScroll(int dx, int dy, int[] consumed, int[] offsetInWindow) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        if (dx != 0 || dy != 0) {
            int startX = 0;
            int startY = 0;
            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }

            if (consumed == null) {
                if (mTempNestedScrollConsumed == null) {
                    mTempNestedScrollConsumed = new int[2];
                }
                consumed = mTempNestedScrollConsumed;
            }
            consumed[0] = 0;
            consumed[1] = 0;
            ViewParentCompat.onNestedPreScroll(mNestedScrollingParent, mView, dx, dy, consumed);

            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            return consumed[0] != 0 || consumed[1] != 0;
        } else if (offsetInWindow != null) {
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}
```

`dispatchNestedPreScroll` 返回一个布尔值，如果是 `true`，则说明父 `ViewParent` 消费了一部分该手势的滚动偏移。众所周知，Java 方法参数是只传值的，这也是为何该方法的形参中定义了两个整型数组。上方是 `NestedScrollingChildHelper` 对此方法的实现，子视图只需将调用委托给 `Helper` 即可。

`dispatchNestedPreScroll` 的 4 个参数中，前三个都容易理解：子视图确定的手势的横向偏移、纵向偏移和父 `ViewParent` 消费了的偏移；最后一个 `offsetInWindow` 的作用则需要花些篇幅来解释。

设想这样一个场景：一个 `child` 处理 `MOVE` 事件，计算得到该事件相对其缓存的上一次事件的偏移为 `20px`，它首先将这一段偏移通过 `dispatchNestedPreScroll` 交给 `parent` 处理，`parent` 消费其中的 `15px`。然而，`parent` 其自身可能也是一个支持嵌套滚动的 `child`，它会将这 `15px` 的偏移再传给它的 `parent` 优先消费……因此，从 `consumed` 数组中传出的 `parent` 消费了的偏移不一定与 `child` 随 `parent` 抢先消费滚动偏移而走过的位移相等。`NestedScrollingChildHelper.dispatchNestedPreScroll` 的实现中，在将滚动偏移传给 `parent` 前，先通过 `View.getLocationInWindow` 取得 `child` 当前在其贴付到的 `Window` 中坐标；传递之后，再取一次 `Window` 中的坐标，前后两次取差值，得到 `child` 在滚动偏移向上传递过程中发生的实际位移，这就是*出参* `offsetInWindow` 的意义。

通常，一个可滚动的视图需要维护**始于**该视图上的滚动手势的相关触摸事件数据。下面列出了 `ScrollView` 在支持嵌套滚动前与滚动控制相关的成员。`mLastMotionY` 记录的是滚动过程中触摸事件相对于子视图原点的纵向坐标。

```java
private int mLastMotionY;
private boolean mIsBeingDragged = false;
private int mActivePointerId = INVALID_POINTER;
```

引入嵌套滚动后，当支持嵌套滚动的父 `ViewParent` 消费了一部分滚动偏移后，子视图自己不仅要少滚动前者消费了的那一部分距离，还需要更新：

* `mLastMotion*`：对 `mLastMotion*` 的更新并不会影响这一步滚动的效果（已经使用过它计算出当前处理的触摸事件的纵向位移了），但是它会影响下一步滚动的偏移量。如果父 `ViewParent` 抢先消费了当前步滚动的一部分偏移，`mLastMotion*` 为这个子视图实际贡献的滚动偏移就会减少（减少了多少？`offsetInWindow[1]`），则在执行下一步滚动时，通过 `mLastMotion* - current*` 获取的滚动手势的偏移就不准确了，这会导致下一步子视图滚动的距离超过触摸事件的实际偏移。因此，子视图在 `dispatchNestedPreScroll`（`dispatchNestedScroll` 同理）之后，如果通过其返回值，得之有一部分滚动偏移被消费，则需要修正 `mLastMotion*`。

* 当前正在处理的触摸事件的坐标：原理同 `mLastMotion*`，不过修正它是为了维护正确的『扫动』初速度，不修正的话会导致扫动初速度超过触摸事件的实际速度。

下面展示的是 `NestedScrollView` 调用 `dispatchNestedPreScroll` 的上下文。`deltaY` 是触摸事件的纵向位移，`NestedScrollView` 在其基础上减去 `parent` 消费的纵向偏移（即 `mScrollConsumed[1]`），并使用自身在 `dispatchNestedPreScroll` 中发生的纵向位移（`mScrollOffset[1]`）修正当前事件的纵坐标（在 `onTouchEvent` 的最后它才会被传入速度追踪器）以及 `mLastMotionY`。

```java
case MotionEvent.ACTION_MOVE:
    // ...
    
    final int y = (int) ev.getY(activePointerIndex);
    int deltaY = mLastMotionY - y;
    if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset)) {
        deltaY -= mScrollConsumed[1];
        vtev.offsetLocation(0, mScrollOffset[1]);
        mNestedYOffset += mScrollOffset[1];
    }

    // ...
    if (mIsBeingDragged) {
        mLastMotionY = y - mScrollOffset[1];
       
        // ...
    }
    
    // ...
```

除此之外，读者应该注意到，`NestedScrollView` 还维护了一个成员 `mNestedYOffset`，它用于统计单次拖拽手势中累计发生的嵌套滚动偏移，在 `onTouchEvent` 处理任意非 `DOWN` 事件前，使用 `mNestedYOffset` 修正事件的坐标。这个成员的用途是用来维护正确的扫动手势初速度，[后文](#维护正确的速度)中再讲解其原理。

##### `parent.onNestedPreScroll`

支持嵌套滚动的父 `ViewParent` ，在此处接收到直接 / 间接子视图的 `dispatchNestedPreScroll` 消息。父 `ViewParent` 可自行决定消费不超过入参 `dx`、`dy` 的滚动偏移量，并将实际消费的滚动偏移通过 `consumed` 数组传出。支持嵌套滚动的 `ViewParent` 同时也可能是一个支持嵌套滚动的子视图，因此，`parent` 还要把剩余的偏移通过 `dispatchNestedPreScroll` 传给它自己的 `parent`。一个经典的案例是 `support` 包中的 `android.support.v4.widget.SwipeRefreshLayout`，它实现了 Material Design 中的下拉刷新效果（其自身逻辑并不重要，笔者意在展示 `ViewParent` 可以根据自身需要灵活地消费滚动偏移）。

```java
@Override
public void onNestedPreScroll(View target, int dx, int dy, int[] consumed) {
    // If we are in the middle of consuming, a scroll, then we want to move the spinner back up
    // before allowing the list to scroll
    if (dy > 0 && mTotalUnconsumed > 0) {
        if (dy > mTotalUnconsumed) {
            consumed[1] = dy - (int) mTotalUnconsumed;
            mTotalUnconsumed = 0;
        } else {
            mTotalUnconsumed -= dy;
            consumed[1] = dy;
        }
        moveSpinner(mTotalUnconsumed);
    }

    // If a client layout is using a custom start position for the circle
    // view, they mean to hide it again before scrolling the child view
    // If we get back to mTotalUnconsumed == 0 and there is more to go, hide
    // the circle so it isn't exposed if its blocking content is moved
    if (mUsingCustomStart && dy > 0 && mTotalUnconsumed == 0
            && Math.abs(dy - consumed[1]) > 0) {
        mCircleView.setVisibility(View.GONE);
    }

    // Now let our nested parent consume the leftovers
    final int[] parentConsumed = mParentScrollConsumed;
    if (dispatchNestedPreScroll(dx - consumed[0], dy - consumed[1], parentConsumed, null)) {
        consumed[0] += parentConsumed[0];
        consumed[1] += parentConsumed[1];
    }
}
```

#### 子视图滚动后

##### `child.dispatchNestedScroll`

子视图通过 `dispatchNestedPreScroll` 得知滚动偏移被消费了多少之后，尝试消费剩余的偏移，然后通过 `dispatchNestedScroll` 告知 `ViewParent` 已消费的偏移和剩余的偏移，后者在此得以有机会消费子视图没有消费的偏移。限于篇幅，笔者不再将 `NestedScrollingChildHelper.dispatchNestedScroll` 的实现贴出——它与 `dispatchNestedPreScroll` 的实现几乎一模一样。

`dispatchNestedScroll` 的参数中同样有*出参* `offsetInWindow`，它的意义与 `dispatchNestedPreScroll` 中完全相同——子视图也同样需要用它来修正 `mLastMotion*`、当前正在处理的触摸事件坐标以及当前滚动手势的累计嵌套滚动偏移。

```java
final int scrolledDeltaY = getScrollY() - oldY;
final int unconsumedY = deltaY - scrolledDeltaY;
if (dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset)) {
    mLastMotionY -= mScrollOffset[1];
    vtev.offsetLocation(0, mScrollOffset[1]);
    mNestedYOffset += mScrollOffset[1];
} else // ...
```

##### `parent.onNestedScroll`

支持嵌套滚动的父 `ViewParent` ，在此处接收到直接 / 间接子视图的 `dispatchNestedScroll` 消息。父 `ViewParent` 可自行决定消费不超过入参 `dxUnconsumed`、`dyUnconsumed` 的滚动偏移量。别忘了，如果 `parent` 自己也是一个支持嵌套滚动的 `child`，还需要将该事件继续通过 `dispatchNestedScroll` 向上层传递。下面是 `NestedScrollView` 的 `onNestedScroll` 实现。

```java
@Override
public void onNestedScroll(View target, int dxConsumed, int dyConsumed, int dxUnconsumed,
        int dyUnconsumed) {
    final int oldScrollY = getScrollY();
    scrollBy(0, dyUnconsumed);
    final int myConsumed = getScrollY() - oldScrollY;
    final int myUnconsumed = dyUnconsumed - myConsumed;
    dispatchNestedScroll(0, myConsumed, 0, myUnconsumed, null);
}
```

### 执行嵌套扫动

嵌套扫动与嵌套滚动的执行流程一眼看上去较为相似——在执行扫动前、扫动后各通知 `ViewParent` 一次，事件仍然逐层向上传递。但是，相较嵌套滚动而言，目前这套框架的嵌套扫动功能显得相对鸡肋：只要父 `ViewParent` 决定抢先执行扫动，子视图就再无机会消费它了；当子视图扫动到可滚动区域的末端时，即使当前扫动速度仍超过扫动阈值，剩余的速度也无法传递到上层。

#### 维护正确的速度

`mNested*Offset` 维护着当前一次拖拽过程中子视图发生的位移，在 `onTouchEvent` 处理任何一个非 `DOWN` 事件之前，子视图都使用它来修正事件的坐标。如下是 `NestedScrollView` 的 `onTouchEvent` 的前几行代码。同时，你可能需要回顾一下 `mNestedYOffset` 是[如何在滚动过程中被维护](#`child.dispatchNestedPreScroll`)的。

```java
@Override
public boolean onTouchEvent(MotionEvent ev) {
    initVelocityTrackerIfNotExists();

    MotionEvent vtev = MotionEvent.obtain(ev);

    final int actionMasked = MotionEventCompat.getActionMasked(ev);

    if (actionMasked == MotionEvent.ACTION_DOWN) {
        mNestedYOffset = 0;
    }
    vtev.offsetLocation(0, mNestedYOffset);
    
    // ...
}
```

试想这样一个场景：一个始于子视图的滚动手势，在位移一段距离后，将子视图滚动到了可滚动区域的边界。此时，嵌套滚动框架将发挥作用，将这之后的滚动偏移传递给 `ViewParent`，从而使得子视图在 `Window` 中发生位移，间接导致 `mNestedYOffset` 被更新。请注意，此时该手势仍然在被子视图处理，子视图收到的触摸事件（原始）坐标始终相对其自身原点，如果不使用 `mNestedYOffset` 更新它，那么，如果此时释放触摸手势，子视图会计算出多大的初始速度？这个值会*偏小*，甚至为 `0`，因为，当嵌套滚动发生时，触摸事件仍在由子视图处理，并且其相对于子视图的坐标变化并未反映出其相对于整个 `Window` 的真实变化。

#### 子视图扫动前

##### `child.dispatchNestedPreFling`

子视图确定手势初速度后，首先将扫动事件通过 `dispatchNestedPreFling` 传递给支持嵌套滚动的直接 / 间接 `ViewParent`，如果后者决定消费该扫动事件，则子视图什么都不用做（如果它遵守这套框架约定的协议的话）。子视图将该调用直接委托给 `Helper` 即可，后者的实现非常简单。

```java
public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
    if (isNestedScrollingEnabled() && mNestedScrollingParent != null) {
        return ViewParentCompat.onNestedPreFling(mNestedScrollingParent, mView, velocityX,
                velocityY);
    }
    return false;
}
```

需要注意一点，无论子视图是否可以扫动，只要触摸事件的初速度超过了阈值，它都应该将此事件传递到上层，如下 `NestedScrollView.flingWithNestedDispatch` 所示。

```java
private void flingWithNestedDispatch(int velocityY) {
    final int scrollY = getScrollY();
    final boolean canFling = (scrollY > 0 || velocityY > 0)
            && (scrollY < getScrollRange() || velocityY < 0);
    if (!dispatchNestedPreFling(0, velocityY)) {
        dispatchNestedFling(0, velocityY, canFling);
        if (canFling) {
            fling(velocityY);
        }
    }
}
```

##### `parent.onNestedPreFling`

`ViewParent` 可根据自身需求判断是否要消费该扫动手势，并返回 `true` / `false`。同样，它也可以选择将此消息传递到上层。

#### 子视图扫动后

##### `child.dispatchNestedFling`

笔者希望上节中 `NestedScrollView` 的逻辑不要给读者造成误导：子视图无论是否消费了扫动手势，都可以通过 `dispatchNestedFling` 将该事件传递给 `ViewParent`（读者可参见 `AbsListView` 的 `onTouchUp()`）。`dispatchNestedFling` 的入参 `consumed` 表示子视图是否消费了该扫动事件。

##### `parent.onNestedFling`

`ViewParent` 可根据自身需求判断是否要消费该扫动手势（如果入参 `consumed` 为 `false` 的话），并返回 `true` / `false`。同样，它也可以选择将此消息传递到上层。
---
layout: post
title: 处理复杂手势
tags: Android View
---
处理触摸事件是一件繁琐的工作，当单点变为多点时更是如此。除了一些内置于 `View` 内的最基本的单点手势（点击、长按）外， Android SDK 中的 `GestureDetector` 等类提供了能力更强的手势检测工具。
## 单点手势
`android.view.GestureDetector` 提供了检测单点手势的能力。如下是 `GestureDetector` 的监听方法。

```java
// 这个类提供了 GestureDetector 所有可用的监听方法的 Stub 实现，使用者
// 如果只想选择性地检测某些手势，继承这个 Stub 类并覆盖其中的方法即可。
public static class SimpleOnGestureListener implements OnGestureListener, OnDoubleTapListener,
        OnContextClickListener {

    // 单点手势的 Up 事件，参数 e 即为该 Up 事件
    public boolean onSingleTapUp(MotionEvent e) {
        return false;
    }

    // 长按，参数 e 是发起该长按的 Down 事件
    public void onLongPress(MotionEvent e) {
    }

    // 滚动，参数 e1 是发起滚动的 Down 事件， e2 是触发滚动的 Move 事件，
    // distance* 是此次滚动在横、纵坐标轴上的距离，注意，并非 e1 和 e2 的距离！
    public boolean onScroll(MotionEvent e1, MotionEvent e2,
            float distanceX, float distanceY) {
        return false;
    }

    // 扫动，参数 e1 是发起滚动的 Down 事件， e2 是触发扫动的 Move 事件（为何不是 Up ？）
    // velocity* 是此次扫动在横、纵坐标轴上的初始速度
    public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX,
            float velocityY) {
        return false;
    }

    // 这个监听用来向用户界面展示『按下』效果（已经确定接受了 onDown 的事件）
    public void onShowPress(MotionEvent e) {
    }

    // 按下，参数 e 即为 Down 事件
    public boolean onDown(MotionEvent e) {
        return false;
    }

    // 双击，参数 e 是首次点按的 Down 事件
    public boolean onDoubleTap(MotionEvent e) {
        return false;
    }

    // 双击过程中的所有事件，参数 e 是双击过程中的所有事件（Down, Move, Up）
    public boolean onDoubleTapEvent(MotionEvent e) {
        return false;
    }

    // 单击，与 onSingleTapUp 不同的是，手势检测器在确定该事件不是一次双击
    // 事件的第一次点击后才会调用此监听
    public boolean onSingleTapConfirmed(MotionEvent e) {
        return false;
    }

    // 鼠标右键、手写笔按键按下，对于触摸手势来说此监听无用
    public boolean onContextClick(MotionEvent e) {
        return false;
    }
}
```

## 追踪手势的速度
`android.view.VelocityTracker` 提供了追踪触摸事件的速度的能力，它经常被用来测量『扫动』动画的初始速度，在可滚动的视图中可以看到它被频繁使用。下面的样例中展示了 `VelocityTracker` 的关键方法的使用。

```java
public class MainActivity extends Activity {
    private static final String DEBUG_TAG = "Velocity";
        ...
    private VelocityTracker mVelocityTracker = null;
    @Override
    public boolean onTouchEvent(MotionEvent event) {
        int index = event.getActionIndex();
        int action = event.getActionMasked();
        int pointerId = event.getPointerId(index);

        switch(action) {
            case MotionEvent.ACTION_DOWN:
                if(mVelocityTracker == null) {
                    // VelocityTracker 使用对象池设计。
                    mVelocityTracker = VelocityTracker.obtain();
                }
                else {
                    // 一个 Down 事件通常意味着手势的开始，复位以准备好
                    // 记录新的手势。
                    mVelocityTracker.clear();
                }
                // 立即将事件加入到追踪器中。
                mVelocityTracker.addMovement(event);
                break;
            case MotionEvent.ACTION_MOVE:
                // 每一个 Move 事件都加入到追踪器中。
                mVelocityTracker.addMovement(event);
                // 在想要计算速度时，调用 computeCurrentVelocity 让追踪器
                // 计算速度，传入的 1000 表示 1000 毫秒，即所得速度为像素 / 秒
                mVelocityTracker.computeCurrentVelocity(1000);
                // 计算后，调用 get*Velocity 取计算结果。在尽可能的地方都使用
                // VelocityTrackerCompat 以获得更好的向后兼容性。
                Log.d("", "X velocity: " +
                        VelocityTrackerCompat.getXVelocity(mVelocityTracker,
                        pointerId));
                Log.d("", "Y velocity: " +
                        VelocityTrackerCompat.getYVelocity(mVelocityTracker,
                        pointerId));
                break;
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                // Up 事件说明一个手势的结束，暂时不再需要使用追踪器，将它还给对象池
                // 注意要在 Up 事件前计算速度，否则速度将变为 0
                mVelocityTracker.recycle();
                break;
        }
        return true;
    }
}
```

## 多点手势
### 缩放手势
缩放手势是非常经典的多点手势。 Android SDK 提供了 `ScaleGestureDetector` 来处理缩放手势。下方是 `ScaleGestureDetector` 提供的监听接口。

```java
public interface OnScaleGestureListener {
    // 缩放手势进行中，返回 true 表示处理了该事件，返回 false 表示暂不处理，但
    // Detector 还会继续累积此次事件的移动量到下一次
    public boolean onScale(ScaleGestureDetector detector);

    // 缩放手势开始，返回 true 表示继续接收该手势随后的事件
    public boolean onScaleBegin(ScaleGestureDetector detector);

    // 缩放手势结束
    public void onScaleEnd(ScaleGestureDetector detector);
}
```

方法的参数均为 `ScaleGestureDetector` 自身，具体的事件数据需要通过它的公有方法获得，下面是 `PhotoView` 使用 `ScaleGestureDetector` 的代码。

```java
ScaleGestureDetector.OnScaleGestureListener mScaleListener = new ScaleGestureDetector.OnScaleGestureListener() {

    @Override
    public boolean onScale(ScaleGestureDetector detector) {
        // 获取缩放系数
        float scaleFactor = detector.getScaleFactor();

        if (Float.isNaN(scaleFactor) || Float.isInfinite(scaleFactor))
            return false;

        // focus* 是当前缩放手势的『焦点』，焦点将作为缩放的基准点。
        // 只需要焦点和缩放系数即可实现缩放效果，ScaleGestureDetector 将
        // 手势处理、焦点计算、缩放系数计算的所有细节都隐藏了。
        mListener.onScale(scaleFactor,
                detector.getFocusX(), detector.getFocusY());
        return true;
    }

    @Override
    public boolean onScaleBegin(ScaleGestureDetector detector) {
        return true;
    }

    @Override
    public void onScaleEnd(ScaleGestureDetector detector) {
        // NO-OP
    }
};
```
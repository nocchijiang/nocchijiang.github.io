---
layout: post
title: MotionEvent
tags: [Android, View]
---
`MotionEvent` 用于表示鼠标、笔、手指等输入设备的移动事件。
## Action
Action 描述事件的种类。使用 `getAction()` 或者 `getActionMasked()` 来获取 Action 信息。两种方法的差别是，前者会直接返回 `MotionEvent` 的后备 `int` ，后者则会把前者返回的信息用位运算洗掉无关的位。 `MotionEvent` 的实现也是用一个 `int` 加上位运算来负载多种信息，其中 Action 被放在低 8 位。下面列出了与触摸相关（也与其它输入设备有关，但是触摸只产生这些事件）的事件种类。

| Action 常量名 | 多点触摸对应名称 | 意义 | 附加信息|
|------------ | ------------- | ---- | ------|
|`ACTION_DOWN` | `ACTION_POINTER_DOWN` | 一个『按下』动作的开始 | 动作的位置坐标|
|`ACTION_UP` | `ACTION_POINTER_UP`	 | 一个『按下』动作的结束 | 动作的位置坐标，以及自上一个『按下』或『拖动』事件以来的所有中间坐标|
|`ACTION_MOVE`	 | N/A | `ACTION_DOWN` 与 `ACTION_UP` 之间『按下』手势发生变化 |动作的位置坐标，以及自上一个『按下』或『拖动』事件以来的所有中间坐标|
|`ACTION_CANCEL` | N/A | 当前手势被取消，可视为一个 `ACTION_UP` 但不要执行 Up 的逻辑|
|`ACTION_OUTSIDE` | N/A | 一个超过了 UI 边界的动作 | 动作的位置坐标|

## Pointer 
一些输入设备支持多点输入，每一个输入点即为一个 Pointer 。每个 Pointer 都有一个唯一的 `ID`，在 `ACTION_DOWN` （`ACTION_POINTER_DOWN`）时给定，直到对应的 Up 事件或 `ACTION_CANCEL` 发生前，这个 `ID` 都有效。同时，在一次多点输入事件中，Pointer 还会被分配 `index`。Pointer 的 `index` 是不定的，所以如果需要持续追踪一个 Pointer 的后续事件，**不要缓存 `index`，应该缓存它的 `ID`。**可以使用 `findPointerIndex(int)` 取得 `ID -> index` 的映射，使用 `getPointerId(int)` 取得 `index -> ID` 的映射。
 
`index` 信息被放在 `getAction()` 返回的 `int` 低 9 位至低 16 位。直接调用 `getActionIndex()` 即可获取一个 `ACTION_POINTER_DOWN` 或 `ACTION_POINTER_UP` 的 Pointer 的 `index` 。
 
## 附加信息
使用下列的方法来获取动作的附加信息（虽然返回值为 `float` ，但是单位均为像素）。若没有说明，传入的参数均为 Pointer 的 `index` 。

|方法名|意义|
|----|----|
|`getX(int)`|获取动作的横轴坐标|
|`getY(int)`|获取动作的纵轴坐标|
|`getAxisValue(int)`|获取 `index` 为 `0` 的 pointer 的某一方向坐标轴的坐标，传入`AXIS_X` 或 `AXIS_Y` 以表示横轴或纵轴|

### `ACTION_MOVE`
为了效率考虑， `ACTION_MOVE` 的事件可能会携带 Move 动作中的多个采样。调用上述的 `getX()` 和 `getY()` 获取到的是当前的 Pointer 坐标；其它的采样可以使用 `getHistoricalX(int, int)` 和 `getHistoricalY(int, int)` 来获取，第一个参数是 Pointer 的 index ，第二个参数是『历史』的下标，必须小于 `getHistorySize()` 返回的值。所有的『历史』已经按时间序排列。 
**『历史』是相对于 `getX()` 和 `getY()` 而言的**。这一次事件中收到的所有位置坐标都是以前的事件中没有的。下面是官方 API 文档中的一个使用『历史』的例子。

```java
void printSamples(MotionEvent ev) {
  final int historySize = ev.getHistorySize();
  final int pointerCount = ev.getPointerCount();
  for (int h = 0; h < historySize; h++) {
    System.out.printf("At time %d:", ev.getHistoricalEventTime(h));
    for (int p = 0; p < pointerCount; p++) {
      System.out.printf("  pointer %d: (%f,%f)",
      ev.getPointerId(p), ev.getHistoricalX(p, h), ev.getHistoricalY(p, h));
    }
  }
  System.out.printf("At time %d:", ev.getEventTime());
  for (int p = 0; p < pointerCount; p++) {
    System.out.printf("  pointer %d: (%f,%f)",
    ev.getPointerId(p), ev.getX(p), ev.getY(p));
  }
}
```
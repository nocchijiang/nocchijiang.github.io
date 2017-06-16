---
layout: post
title: ViewConfiguration
tags: Android View
---
`ViewConfiguration` 用来获取在 UI 中会使用到的一系列常量。

方法名|返回值类型|意义
-----|--------|---
`getScaledTouchSlop()`|`int`|将一次触摸事件视为滚动操作的距离阈值，单位为像素
`getScaledMinimumFlingVelocity()`|`int`|触摸事件发起一次扫动（ fling ）手势所需的最小速度，单位为像素 / 秒
`getScaledMaximumFlingVelocity()`|`int`|触摸事件发起一次扫动（ fling ）手势所需的最大速度，单位为像素 / 秒
`getScaledOverscrollDistance()`|`int`|视图过度滚动（ scroll ）的最大距离，单位为像素
`getScaledOverflingDistance()`|`int`|视图过度扫动（ fling ）的最大距离，单位为像素
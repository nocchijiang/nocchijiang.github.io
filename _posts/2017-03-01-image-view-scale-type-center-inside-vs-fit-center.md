---
layout: post
title: "ImageView.ScaleType: CENTER_INSIDE Vs. FIT_CENTER"
tags: [Android, View]
---
两种 `ScaleType` 均是把 `ImageView` 中的图片居中显示，唯一的区别：

* `CENTER_INSIDE` 不会将过小的图片放大。如果 `ImageView` 承载的图片比其自己的尺寸还小，那么 `ImageView` 只会简单将此图片居中，而不作 `Scale` 变换。

* `FIT_CENTER` 无论承载的图片尺寸如何都会把它放大 / 缩小到 `ImageView` 能完整地显示该图片后，再居中显示。

可见从效果上看 `CENTER_INSIDE` 就是不能做 Scale Up 的 `FIT_CENTER` 。因此在做图片完整居中时，通常都 favor 后者，这样可以保证图片填满可利用的 `ImageView` 的尺寸。
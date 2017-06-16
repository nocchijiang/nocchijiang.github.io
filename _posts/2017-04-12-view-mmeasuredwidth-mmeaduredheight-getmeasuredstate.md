---
layout: post
title : "View.mMeasuredWidth, View.mMeasuredHeight & View.getMeasuredState()"
tags: Android View
---
```java
/**
 * Width as measured during measure pass.
 * {@hide}
 */
int mMeasuredWidth;
 
/**
 * Height as measured during measure pass.
 * {@hide}
 */
int mMeasuredHeight;
```

光看定义，看起来是非常有迷惑性的，会让读者以为这两个成员直接存储着此视图经过测量后测得的宽高。

```java
/**
 * Bits of {@link #getMeasuredWidthAndState()} and
 * {@link #getMeasuredWidthAndState()} that provide the actual measured size.
 */
public static final int MEASURED_SIZE_MASK = 0x00ffffff;
 
/**
 * Bits of {@link #getMeasuredWidthAndState()} and
 * {@link #getMeasuredWidthAndState()} that provide the additional state bits.
 */
public static final int MEASURED_STATE_MASK = 0xff000000;
 
/**
 * Bit shift of {@link #MEASURED_STATE_MASK} to get to the height bits
 * for functions that combine both width and height into a single int,
 * such as {@link #getMeasuredState()} and the childState argument of
 * {@link #resolveSizeAndState(int, int, int)}.
 */
public static final int MEASURED_HEIGHT_STATE_SHIFT = 16;
  
/**
 * Bit of {@link #getMeasuredWidthAndState()} and
 * {@link #getMeasuredWidthAndState()} that indicates the measured size
 * is smaller that the space the view would like to have.
 */
public static final int MEASURED_STATE_TOO_SMALL = 0x01000000;
```

看过上面的位掩码定义后，才明白，`mMeasure*` 是由两个值拼装得到的，高 8 位用来存储测量状态信息，低 24 位存储的是测量出的宽高。
这足足占用了 8 位的状态信息具体又是何物？其实这 8 位仅仅用来表示了两种状态，一种便是上方掩码定义的最后一个 `MEASURED_STATE_TOO_SMALL` ，它表示测得的尺寸小于视图想要拥有的尺寸；另一种则是全 0 ，表示与前者相反的状态。注释中还特别提到，这个状态应该仅在测量与布局过程中被使用。
下面是一个 `View` 中定义的静态方法使用到此状态的实例。这个方法在许多 `ViewGroup` 实现的 `onMeasure()` 方法中都被使用到了，返回的值会直接用在 `setMeasuredDimension()` 的参数中。

```java
/**
 * @param size ViewGroup 在当前维度剩余的可用尺寸，通常是测量了部分子视图后加以计算
 *                       得到的值
 * @param measureSpec 父 ViewGroup 加在这个 View 上的 measure spec
 * @param childMeasuredState 子视图的状态标记，通常是取所有子视图 measure 完后的状态
 *                           求或，即只要有一个子视图测量状态为 MEASURED_STATE_TOO_SMALL ，
 *                           这个值就是 MEASURED_STATE_TOO_SMALL
 */
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        // 父 ViewGroup 限定了最大尺寸
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                // 但是此 View 想要占用更大的空间，叠加 MEASURED_STATE_TOO_SMALL 状态码
                result = specSize | MEASURED_STATE_TOO_SMALL; // <--
            } else {
                // 没有超过限制，则 View 想要多大就多大
                result = size;
            }
            break;
        // 已经显式指定 View 的尺寸了，无视所有参数，直接取父 ViewGroup 指定的值
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        // 没有限制尺寸，则 View 想要多大就多大
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    // 按位或叠加子视图的测量状态信息
    return result | (childMeasuredState & MEASURED_STATE_MASK /* 洗掉潜在存在的右位移过的状态位 */);
}
```

可以看到，子视图的状态信息会强行带到其父 `ViewGroup` 中。  
吐槽一下， `View` 的实现里眼花缭乱的位运算太多了。
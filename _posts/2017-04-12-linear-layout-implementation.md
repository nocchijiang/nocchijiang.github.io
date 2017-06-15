---
layout: post
title: LinearLayout 实现分析
tags: [Android, View]
---
**注意**：为便于阅读与分析，代码中抹去了[这个提交](https://github.com/android/platform_frameworks_base/commit/40e1df34a9549a25020a325d0abc4d43c5a6b6e2?diff=split)涉及到的兼容性修改的存在（令其始终等效于运行在最新的环境下），并删去了与 `baseline` 和 `useLargestChild` 相关的所有逻辑。

## `measureVertical()`

```java
// 纵向模式下测量 LinearLayout 和其所有子视图的尺寸。
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
    // 将长度复位，并初始化各局部变量。
    mTotalLength = 0;
    // 最宽的子视图的宽度。
    int maxWidth = 0;
    // 子视图 measured state 的按位或。
    int childState = 0;
    // 这里又维护了两个最大宽度，它们和 maxWidth 的区别是， maxWidth 始终在跟踪着测量后的最大子视图宽度，而
    // alternativeMaxWidth 与 weightedMaxWidth 在更新自己前会检查当前子视图的宽度是否被设为 MATCH_PARENT，
    // 如果是的话，则只将该视图的 margin 计入，否则与 maxWidth 无异。
    int alternativeMaxWidth = 0;
    int weightedMaxWidth = 0;
    // 所有子视图的宽度是否都是 MATCH_PARENT 。
    boolean allFillParent = true;
    // 所有子视图的权重之和（用于没有设置 weightSum 的情形）。
    float totalWeight = 0;

    final int count = getVirtualChildCount();
    
    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

    // 标记是否存在子视图的宽度被设为 MATCH_PARENT 。这些视图会在测量过程的最后再一次被重新测量。
    boolean matchWidth = false;
    // 标记是否在第一轮子视图测量过程中跳过了某些视图（它们必须在第二轮测量过程中被重测）。
    boolean skippedMeasure = false;

    // 记录在非 EXACTLY 模式下，那些需要占据尽可能多的高度的子视图在第一轮测量过程中消费了多少高度。
    int consumedExcessSpace = 0;

    // 第一轮子视图测量开始。
    for (int i = 0; i < count; ++i) {
        final View child = getVirtualChildAt(i);
        // 跳过空视图。
        if (child == null) {
            mTotalLength += measureNullChild(i);
            continue;
        }

        // 跳过 GONE 掉的视图。
        if (child.getVisibility() == View.GONE) {
           i += getChildrenSkipCount(child, i);
           continue;
        }

        // 如果需要在这个孩子上方加分隔线，对总长度累加分隔线的高度。
        if (hasDividerBeforeChildAt(i)) {
            mTotalLength += mDividerHeight;
        }

        // 取这个孩子的布局参数。
        final LayoutParams lp = (LayoutParams) child.getLayoutParams();

        // 更新总权重。
        totalWeight += lp.weight;

        // 如果此孩子的高度设为 0 且权重大于 0 ，则说明它想要根据权重获得 LinearLayout 分配给它的纵向空间，
        // 并且没有固有高度（ Intrinsic Height ）——换句话说，它『不在乎』自己多高。
        // 在第二轮测量中， LinearLayout 会保证带权的孩子获得的纵向空间至少大于它的固有高度。
        // 这里， useExcessSpace 这个变量的命名传递的意义为： *只* 使用第一轮测量后 LinearLayout 剩余的纵向空间。
        final boolean useExcessSpace = lp.height == 0 && lp.weight > 0;
 
        // 如果 LinearLayout 的高度已经被明确指定（即高度测量规格模式为 EXACTLY），并且此孩子既要占用尽可能多的高度，又没有固有高度，
        // 则在第一轮测量中直接跳过对它的测量，因为这个孩子只使用 LinearLayout 的剩余纵向空间，第二轮测量时，如果有剩余空间的话，
        // 再将（有也可能没有的）剩余空间经过加权后分配给这些带权的孩子。如此，可以省去对这个孩子的一次测量过程，以优化性能。
        if (heightMode == MeasureSpec.EXACTLY && useExcessSpace) {
            final int totalLength = mTotalLength;
            // 因为负 margin 是被允许的，这里需要做一个保护。同时也可以看出 margin 是被视作固有高度来强制执行的。
            mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
            // 标记跳过了某个子视图的测量过程。
            skippedMeasure = true;
        } else {
 
            // 如果这个孩子只使用 LinearLayout 剩余的纵向空间，将其布局参数的高度『篡改』成 WRAP_CONTENT ，看看它需要多大的高度。
            // 在完成对这个孩子的测量后，会立即把参数改回 0 。
            if (useExcessSpace) {
                lp.height = LayoutParams.WRAP_CONTENT;
            }

            // 如果当前总权重为 0 ，则当前的总长度就是 LinearLayout 已经使用了的高度；
            // 否则只能传 0 （因为现在根本不知道那些权重大于 0 的孩子有多高）。
            // 权重为 0 的孩子将可以保证获得不大于 LinearLayout 自身高度的纵向空间。
            final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
 
            // 测量这个子视图。
            measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
                    heightMeasureSpec, usedHeight);
          
            final int childHeight = child.getMeasuredHeight(); 
            // 将之前『篡改』了的布局参数改回来，记录这种子视图一共占用了多少高度，会在第二轮测量中被用到。
            if (useExcessSpace) {
                lp.height = 0;
                consumedExcessSpace += childHeight;
            }

            // 更新总长度。同样，对负数 margin 做了保护。
            final int totalLength = mTotalLength;
            mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                   lp.bottomMargin + getNextLocationOffset(child));
        }

        // 高度在第一轮测量管到这里就差不多了（实际上也只能做到这么多），现在需要看宽度。
        
        // 标记当前遍历到的这个孩子的宽度是否设为了 MATCH_PARENT 。
        boolean matchWidthLocally = false;
        // 该 LinearLayout 的宽度测量规格没有被限制死，则现在我们无从得知 LinearLayout 的真实宽度（因为还有孩子可能没被测量）。
        // 更新标记，等到第二轮测量过程中再来处理。
        if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
            matchWidth = true;
            matchWidthLocally = true;
        }

        final int margin = lp.leftMargin + lp.rightMargin;
        final int measuredWidth = child.getMeasuredWidth() + margin;
        // 对负 margin 做保护的前提下更新最大子视图宽度。
        maxWidth = Math.max(maxWidth, measuredWidth);
        // 按位或，更新 measured state 。
        childState = combineMeasuredStates(childState, child.getMeasuredState());
        // 基于该子视图的宽度是否设为 MATCH_PARENT 更新标记。
        allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
 
        // 如果此孩子的权重大于 0 ，那么它可能会被重新测量，宽度则可能会发生变化。
        // 所以对于权重大于 0 和不大于 0 的两种孩子的最大宽度需分别维护。第二轮测量中，会补测第一轮中没有测量的子视图。
        if (lp.weight > 0) {
            // 这个孩子的宽度如果设为 MATCH_PARENT ，那么只计它的纵向 margin 为其宽度，
            // 因为它具体多宽完全取决于 LinearLayout 的宽度，但此时 LinearLayout 有多宽并不确定
            // （别忘了， matchWidthLocally 这个标记是在 LinearLayout 的宽度测量规格模式不为 EXACTLY 时才置为 true 的）；
            // 否则，直接计测量所得值。
            weightedMaxWidth = Math.max(weightedMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);
        } else {
            // 判断 matchWidthLocally 的原因同上。
            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);
        }

        i += getChildrenSkipCount(child, i);
    }
    // 第一轮子视图测量结束。

    // 在底部补上分隔线的高度，如果需要的话。
    if (mTotalLength > 0 && hasDividerBeforeChildAt(count)) {
        mTotalLength += mDividerHeight;
    }

    // 将 LinearLayout 自己的 padding 补上。
    mTotalLength += mPaddingTop + mPaddingBottom;

    // 到此为止，对 LinearLayout 高度的测量就完成了……
    int heightSize = mTotalLength;

    // 但不能比设定的最小高度还小……
    heightSize = Math.max(heightSize, getSuggestedMinimumHeight());
    
    // 根据 LinearLayout 自身被施加的测量规格更新 LinearLayout 的高度。
    // 现在 heightSize 即为 LinearLayout 最终的高度。
    int heightSizeAndState = resolveSizeAndState(heightSize, heightMeasureSpec, 0);
    heightSize = heightSizeAndState & MEASURED_SIZE_MASK;
    
    // 第二轮子视图测量开始。
    // 我们已经知道，在第一轮测量中，当 LinearLayout 的测量规格模式为 EXACTLY 时，那些权重大于 0 的子视图都没有被测量，
    // 原因是我们不知道这些孩子后面的子视图会占用多少高度，也就无法确定它们的测量规格。现在，除了权重大于 0 的子视图外，
    // 所有的子视图都已测量过了，因此现在既应该，也已有充分的信息补测这些权重大于 0 的子视图了。
 
    // 首先计算有多少高度能拿给这些权重大于 0 的子视图按权重分配的 （ remainingExcess ）。为什么要这样计算？
 
    // heightSize 是 LinearLayout 的最终高度， mTotalHeight 是第一轮子视图测量后 LinearLayout 的理想高度。
    // consumedExcessSpace 是在 LinearLayout 自身的高度测量规格模式不为 EXACTLY 时，通过『篡改』子视图高度为
    // WRAP_CONTENT 测量得到的权重大于 0 的子视图的测量高度之和。
 
    // 首先让我们考虑最常见的一种情形，即为 LinearLayout 自身高度测量规格就是 EXACTLY 的情形。
    // 此时 consumedExcessSpace 为 0 ，
    // 即 remainingExcess 是 <LinearLayout 实际高度> 与 <所有子视图的高度（只计入了权重大于 0 的子视图的的纵向 margin ）的总和> 
    // 的差值，它可能是一个正值（纵向上有富余的空间分配），也可能是一个负值（纵向空间不足）。
    
    // 另一种情形则是 LinearLayout 自身高度测量规格不为 EXACTLY 的情形。此时 consumedExcessSpace 是以 WRAP_CONTENT 为高度测出的
    // 权重大于 0 的子视图的高度总和，因为接下来我们马上要根据权重重新分配空闲纵向空间了，而先前我们已经把其中部分空间计入了 mTotalLength ，
    // 所以在这里要把这部分空间重新加回来，否则将会白白损失一部分可利用的纵向空间。
    int remainingExcess = heightSize - mTotalLength + consumedExcessSpace;
 
    // skippedMeasure 为 true 则说明在第一轮测量过程中刻意跳过了对某些视图的测量；
    // 或者，是我们需要根据权重和剩余纵向空间对带权视图的高度重分配，这两种情形均需要对权重大于 0 的视图补测、重测。
    if (skippedMeasure || remainingExcess != 0 && totalWeight > 0.0f) {
        // 如果用户设置了权值之和，那么使用用户的设定；否则使用第一轮测量中计算出来的和。
        float remainingWeightSum = mWeightSum > 0.0f ? mWeightSum : totalWeight;

        // 重置总长度，准备开始第二轮测量。
        mTotalLength = 0;

        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);
            // 跳过空视图和 GONE 掉的视图。
            if (child == null || child.getVisibility() == View.GONE) {
                continue;
            }

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final float childWeight = lp.weight;
            // 这个子视图权重大于 0 ，说明它需要被补测 / 重测。
            if (childWeight > 0) {
                // 根据这个孩子的权重计算它分到的高度大小，更新剩余空间与权值之和。
                final int share = (int) (childWeight * remainingExcess / remainingWeightSum);
                remainingExcess -= share;
                remainingWeightSum -= childWeight;

 
                final int childHeight;
                // 这个条件判断与第一轮测量过程中的 useExcessSpace 语义相同，即：
                if (lp.height == 0 && heightMode == MeasureSpec.EXACTLY) {
                    // 只使用根据权重分得的 LinearLayout 剩余的纵向空间。
                    childHeight = share;
                } else {
                    // 否则，还需加上这个子视图的固有高度。
                    // 进入这个分支则说明这个子视图在第一轮测量过程中一定被测量过，因此 getMeasuredHeight() 才能获取到正确的值。
                    childHeight = child.getMeasuredHeight() + share;
                }

                // 装配高度、宽度测量规格并测量子视图。
                // 这里唯一值得注意的一点是，对于高度 LinearLayout 给定模式为 EXACTLY ，这与根据权值精确分配
                // 空间的行为保持了一致；对于高度则走的是一个常规流程。
                final int childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        Math.max(0, childHeight), MeasureSpec.EXACTLY);
                final int childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin,
                        lp.width);
                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

                // 将测量后的 measured state 按位或合入局部变量。注意高度状态的位
                // 位于 getMeasuredState() 返回的的高 16 位，因此这里做了一次位移。
                childState = combineMeasuredStates(childState, child.getMeasuredState()
                        & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
            } // 否则，这个视图不需要重测。

            // 确保这个孩子被测量之后，需要更新最大宽度信息（先前要么没测这个孩子，要么测量结果可能因为测量规格的变化而发生改变）。
            final int margin =  lp.leftMargin + lp.rightMargin;
            final int measuredWidth = child.getMeasuredWidth() + margin;
            maxWidth = Math.max(maxWidth, measuredWidth);
 
            // 如果 LinearLayout 的宽度不定，并且这个子视图又想占满 LinearLayout 的全部宽度，
            // 则只能将这个子视图的横向 margin 计为其宽度作为最宽子视图的候选，至于为什么，在前面已经做了说明。
            boolean matchWidthLocally = widthMode != MeasureSpec.EXACTLY &&
                    lp.width == LayoutParams.MATCH_PARENT;
 
            // 与第一轮测量时不同，这里没有再维护 weightedMaxWidth ，因为第二轮测量时已经确定了子视图的宽度。
            alternativeMaxWidth = Math.max(alternativeMaxWidth,
                    matchWidthLocally ? margin : measuredWidth);

            // 这个，有必要再来一次吗？笔者笔者觉得好像是多余的。
            allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;

            final int totalLength = mTotalLength;
            // 在负 margin 保护的前提下更新 LinearLayout 总长度。
            mTotalLength = Math.max(totalLength, totalLength + child.getMeasuredHeight() +
                    lp.topMargin + lp.bottomMargin + getNextLocationOffset(child));
        }

        // 子视图遍历完之后别忘了加上自己的纵向 padding
        mTotalLength += mPaddingTop + mPaddingBottom;
        // 至此，对于因子视图带权需要的重测、补测完成。
    } else {
        // 进入这个分支，说明 LinearLayout 不需要重新测量任何子视图， weightedMaxWidth 的值是准确的。
        // 因此，取更大的那一个作为 LinearLayout 的理想宽度。 
        alternativeMaxWidth = Math.max(alternativeMaxWidth, weightedMaxWidth);
    } // 第二轮子视图测量结束。

    // 如果不是所有的子视图的宽度都设为 MATCH_PARENT 并且宽度测量规格模式不为 EXACTLY ，则用 alternativeWidth 替换 maxWidth
    // （为什么？）
    if (!allFillParent && widthMode != MeasureSpec.EXACTLY) {
        maxWidth = alternativeMaxWidth;
    }
    
    // 将 LinearLayout 自己的 padding 补上。
    maxWidth += mPaddingLeft + mPaddingRight;

    // 确保宽度大于设定的最小宽度。
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
    
    // 完成对自己的测量。
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            heightSizeAndState);

    // 有一些子视图的宽度设为了 MATCH_PARENT ，但在测量完自己之前， 
    // LinearLayout 无法为这些视图构造宽度测量规格（ EXACTLY 模式是可以确定，但尺寸是多少则未知）。
    // 因此在测量完了自己之后， LinearLayout 还需要做最后一件事：用正确的宽度测量规格重测量宽度设为 MATCH_PARENT 的子视图。
    if (matchWidth) {
        forceUniformWidth(count, heightMeasureSpec);
    }
}
```

Q: 为什么使用 alternativeWidth 替换 maxWidth ？

## `measureHorizontal()`

除了涉及到 `baseline` 的部分，和 `measureVertical()` 是**一模一样**的。因为中文开发几乎不会使用到 `baseline` ，我们暂不关注与它相关的内容。

## `layoutVertical()`
```java
// 纵向模式下的 LinearLayout 布局过程。
void layoutVertical(int left, int top, int right, int bottom) {
    final int paddingLeft = mPaddingLeft;

    // 下一个子视图的顶部和左部相对该 LinearLayout 的坐标。
    int childTop;
    int childLeft;
    
    final int width = right - left;
    int childRight = width - mPaddingRight;
    
    int childSpace = width - paddingLeft - mPaddingRight;
    
    final int count = getVirtualChildCount();

    // 这里的 major 与 minor 是相对于此 LinearLayout 的方向而言的，对于纵向 LinearLayout 而言，
    // 它的『 major 』 即为纵向，『 minor 』即为横向。
    // 通过位运算分别将纵向对齐、横向对齐方式从一个 int 变量中取出。
    final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
    final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

    // 首先根据纵向对齐方式确定第一个 childTop 。
    // bottom - top 可以直接视为此 LinearLayout 的父布局为它分配的高度。
    switch (majorGravity) {
    
       // 底部对齐，则起始位置距离 LinearLayout 底部的距离应刚刚好等于 LinearLayout 子视图高度之和 + LinearLayout 的 paddingBottom 。
       case Gravity.BOTTOM:
           // mTotalLength 中已经包含了 LinearLayout 的 mPaddingTop 了。没有想清这一点前，加一个 mPaddingTop 实在是太混淆读者视听了……
           childTop = mPaddingTop + bottom - top - mTotalLength;
           break;

       // 纵向居中对齐，则起始位置应位于 LinearLayout 垂直中部往上走子视图高度之和的一半距离。
       case Gravity.CENTER_VERTICAL:
           childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
           break;

       // 对于顶部对齐以及其它模式（虽然 LinearLayout 不支持，但也视为 TOP），
       // 最简单，只计算自身 paddingTop 即可作为顶部起始位置。
       case Gravity.TOP:
       default:
           childTop = mPaddingTop;
           break;
    }

    // 遍历子视图，为每一个子视图布局。主要任务是要确定每一个视图在水平方向的起始位置（即 childLeft）。
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        // 跳过空子视图和不可见的视图。
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
            
            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();
            
            // 检查子视图是否设置了横向对齐方式。如果没有（即 gravity < 0），采用 LinearLayout 的全局配置。
            int gravity = lp.gravity;
            if (gravity < 0) {
                gravity = minorGravity;
            }
 
            // RTL 语言兼容（LEFT / RIGHT 兼容为 START / END），作为 LTR 语言的使用者我们无视即可。
            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                // 横向居中对齐，起始位置应位于 LinearLayout 水平中部向左平移子视图宽度的一半。同时我们还看到了对 margin 的处理。
                // mTotalLength 里已经包含纵向的 margin 了，因此在确定首个 childTop 时我们没有看到相关计算。
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                            + lp.leftMargin - lp.rightMargin;
                    break;

                // 右对齐，起始位置应位于 LinearLayout 右边界 - 子视图宽度 - 其右 margin 。
                case Gravity.RIGHT:
                    childLeft = childRight - childWidth - lp.rightMargin;
                    break;

                // 最简单的左对齐，起始位置位于 LinearLayout 右边界 + 其左 margin 。
                case Gravity.LEFT:
                default:
                    childLeft = paddingLeft + lp.leftMargin;
                    break;
            }

            // 如果有分隔线，要为它预留位置。
            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }

            // 在测量子视图的时候，要提前考虑它的 margin 。
            // 归根到底， margin 是由 ViewGroup （的某个派生类）引入的视图属性，而非视图自身的属性。
            childTop += lp.topMargin;
            // 这个辅助方法其实就是简单地调用了 child.measure() 。
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            // 更新 childTop 准备布局下一个子视图。
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}
```

## 总结

对 `LinearLayout` 实现的分析到此为止。下面对它的实现作一简单的总结。

* 是一个可绘制的 `ViewGroup`，可以在子视图间插入分隔线
* 如果没有带权子视图，只需跑一轮测量，否则若在该 `LinearLayout` 的方向设置了不为 `EXACTLY` 模式的测量参数的情况下必须为带权子视图跑两轮测量
* 如果子视图在设置的方向的对向（即若设置的方向是垂直，则对向为水平）尺寸设为了 `MATCH_PARENT` 则还需单独为这些子视图做第三轮测量
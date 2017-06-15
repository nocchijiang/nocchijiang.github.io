---
layout: post
title: RelativeLayout 实现分析
tags: [Android, View]
---
## 布局参数
`RelativeLayout` 的布局参数嵌套类与 `LinearLayout` 的类似，继承自 `ViewGroup.MarginLayoutParams` 。它承载了子视图间或子视图与此 `RelativeLayout` 的关系信息，这些关系信息在 `RelativeLayout.LayoutParams` 中被抽象为『规则（ Rule ）』。

```java
public static class LayoutParams extends ViewGroup.MarginLayoutParams {

    // 使用 int 数组承载所有可用的 RelativeLayout 规则信息。 VERB_COUNT 是一个定义在 RelativeLayout 中的常量，
    // 在 SDK 25 中它的值是 22 。
    private int[] mRules = new int[VERB_COUNT];
 
    // 存放了一份冗余的规则，它的用途需要在被使用到的场合中进行分析。
    private int[] mInitialRules = new int[VERB_COUNT];

    // 标记使用此布局参数的视图在新一轮布局时是否需要被重新分析。
    private boolean mNeedsLayoutResolution;

    // 标记规则是否发生变化。
    private boolean mRulesChanged = false;

    // 标记此规则是否对齐父 RelativeLayout 或是依赖视图可见性为 GONE 。
    public boolean alignWithParent;
 
    // 使用此布局参数的视图在 RelativeLayout 的测量过程中被确定的布局位置。
    // 如果其值为 VALUE_NOT_SET （定义在 RelativeLayout 中的常量，值为 int 能表示的最小值），则说明具体的值应该通过关系推断得出。
    // 很快我们就会看到， RelativeLayout 的测量与布局事实上是同时完成的。
    private int mLeft, mTop, mRight, mBottom;
 
    public LayoutParams(Context c, AttributeSet attrs) {
        super(c, attrs);

        TypedArray a = c.obtainStyledAttributes(attrs,
                com.android.internal.R.styleable.RelativeLayout_Layout);

 
        final int[] rules = mRules;
        final int[] initialRules = mInitialRules;

        final int N = a.getIndexCount();
        for (int i = 0; i < N; i++) {
            int attr = a.getIndex(i);
 
            // 在这个构造器中可以看出，不同的规则通过固定的数组偏移定义存放到 mRules 中。这些偏移的定义位于 RelativeLayout 类中。
            switch (attr) {
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignWithParentIfMissing:
                    alignWithParent = a.getBoolean(attr, false);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_toLeftOf:
                    rules[LEFT_OF] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_toRightOf:
                    rules[RIGHT_OF] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_above:
                    rules[ABOVE] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_below:
                    rules[BELOW] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignBaseline:
                    rules[ALIGN_BASELINE] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignLeft:
                    rules[ALIGN_LEFT] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignTop:
                    rules[ALIGN_TOP] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignRight:
                    rules[ALIGN_RIGHT] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignBottom:
                    rules[ALIGN_BOTTOM] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignParentLeft:
                    // 对于值为布尔值的规则，在规则数组中用 TRUE 常量（值为 -1）标记它们。
                    rules[ALIGN_PARENT_LEFT] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignParentTop:
                    rules[ALIGN_PARENT_TOP] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignParentRight:
                    rules[ALIGN_PARENT_RIGHT] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignParentBottom:
                    rules[ALIGN_PARENT_BOTTOM] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_centerInParent:
                    rules[CENTER_IN_PARENT] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_centerHorizontal:
                    rules[CENTER_HORIZONTAL] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_centerVertical:
                    rules[CENTER_VERTICAL] = a.getBoolean(attr, false) ? TRUE : 0;
                   break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_toStartOf:
                    rules[START_OF] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_toEndOf:
                    rules[END_OF] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignStart:
                    rules[ALIGN_START] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignEnd:
                    rules[ALIGN_END] = a.getResourceId(attr, 0);
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignParentStart:
                    rules[ALIGN_PARENT_START] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
                case com.android.internal.R.styleable.RelativeLayout_Layout_layout_alignParentEnd:
                    rules[ALIGN_PARENT_END] = a.getBoolean(attr, false) ? TRUE : 0;
                    break;
            }
        }
        mRulesChanged = true;
 
        // mRules 初始化完毕后，内容拷贝到 mInitialRules 数组中。
        System.arraycopy(rules, LEFT_OF, initialRules, LEFT_OF, VERB_COUNT);

        a.recycle();
    }

    /**
     * 添加一条规则。这个类中还定义了不少其它类似签名的便利方法，最终均指向了这个方法。
     */
    public void addRule(int verb, int subject) {
        // 如果在删除一条相对规则（即 subject 为 0 ，很快就能看到 removeRule() 的定义），
        // 则标记在下一轮布局中使用此布局参数的视图需要被重新分析。
        // 相对规则是为了适配 RTL 文字而做的兼容，具体而言它指的那些用 START, END 替代 LEFT, RIGHT 的规则。
        // 因为中文是一种 LTR 文字，我们略过对它的介绍，专注于我们关心的重点。
        if (!mNeedsLayoutResolution && isRelativeRule(verb)
                && mInitialRules[verb] != 0 && subject == 0) {
            mNeedsLayoutResolution = true;
        }

        mRules[verb] = subject;
        mInitialRules[verb] = subject;
        mRulesChanged = true;
    }

    /**
     * 删除一条规则。可见就是简单的复位对应规则位的数组元素。
     */
    public void removeRule(int verb) {
        addRule(verb, 0);
    }

}
```

## 依赖图
`RelativeLayout` 使用子视图之间的『关系』来测量、布局子视图，这些关系被 `RelativeLayout` 的开发者抽象成为一个有向无环图（ DAG ）。学过数据结构课程的我们可以做一大胆猜测，在 `RelativeLayout` 中，没有『关系』的视图首先被测量和布局，随后是与它存在关系的视图，……，直至 `RelativeLayout` 中所有的视图都已完成测量与布局。可见这是一个典型的拓扑排序，非常说得通，对吧？接下来，我们来了解一下这个 DAG 的数据结构与相关操作。

```java
private static class DependencyGraph {
 
    /**
     * 图中的结点。它持有一个子视图、它依赖的结点和依赖它的结点的引用。
     * 如果这个结点不依赖任何其它结点，它就被视为图中的『根结点』。
     */    
    static class Node { 
        /**
         * 此结点持有的视图引用。
         */
        View view;

        /**
         * 依赖它的结点。这里虽然用 ArrayMap ，但值并不重要（下面就可以看到每次 put 操作传入的值都是 this），
         * 实现者只是想利用它在空间上比 HashMap 更有效率的特性。
         */
        final ArrayMap<Node, DependencyGraph> dependents =
                new ArrayMap<Node, DependencyGraph>();

        /**
         * 它依赖的结点。
         */
        final SparseArray<Node> dependencies = new SparseArray<Node>();
    }

    // 图中的所有结点引用。
    private ArrayList<Node> mNodes = new ArrayList<Node>();

    // 按结点持有的视图 ID 为索引的结点引用稀疏数组。
    private SparseArray<Node> mKeyNodes = new SparseArray<Node>();

    // 一个临时双向队列，用来维护拓扑排序算法每一趟结束后的临时『根结点』。
    // 注意！它持有的并非图中真正的根结点，它只是拓扑排序中使用的临时数据结构。
    private ArrayDeque<Node> mRoots = new ArrayDeque<Node>();

    // 向图中添加一个视图。
    void add(View view) {
        final int id = view.getId();
 
        // Node 类使用了一个简单的对象池设计，对象池的大小为 100 。
        final Node node = Node.acquire(view);

        // 如果为这个视图设置了 ID ，将其按 ID 为索引加到稀疏数组中。
        if (id != View.NO_ID) {
            mKeyNodes.put(id, node);
        }

        mNodes.add(node);
    }

    /**
     * 拓扑排序发生之处。按拓扑序排列图中的视图，结果传入第一个参数中。 sorted 数组的大小必须为 RelativeLayout 中子视图的个数。
     */
    void getSortedViews(View[] sorted, int... rules) {
        // 首先寻找到图中的根结点。回忆拓扑排序算法，一个可行的算法就是从任一没有入边的顶点开始，将该顶点
        // 以及该顶点的所有出边从图中删除，直至图中没有顶点或没有满足条件的顶点为止。可见 getSortedViews 
        // 恰好就使用了这个简单易行的算法。
        final ArrayDeque<Node> roots = findRoots(rules);
        int index = 0;

        Node node;
        // 从『根结点』出发：
        while ((node = roots.pollLast()) != null) {
            final View view = node.view;
            final int key = view.getId();

            // 将其视图引用加入拓扑有序数组尾部。
            sorted[index++] = view;

            // 找出依赖此结点的所有结点，即 DAG 中此结点的『出边』连接的所有结点。
            final ArrayMap<Node, DependencyGraph> dependents = node.dependents;
            final int count = dependents.size();
            // 遍历这些结点。
            for (int i = 0; i < count; i++) {
                final Node dependent = dependents.keyAt(i);
                final SparseArray<Node> dependencies = dependent.dependencies;

                // 将当前的『根结点』从此结点的依赖中删除，即 DAG 中删除『出边』。
                dependencies.remove(key);
 
                // 如果此结点已经没有依赖了，它就成为新的临时『根结点』。将其加入到临时根结点双向队列中。
                if (dependencies.size() == 0) {
                    roots.add(dependent);
                }
            }
        }

        // 如果图中已经没有了『根结点』，而此时拓扑序列还没有装满所有的结点，则说明图中存在环，这违背了 RelativeLayout 的使用规则，直接抛出异常。
        if (index < sorted.length) {
            throw new IllegalStateException("Circular dependencies cannot exist"
                    + " in RelativeLayout");
        }
    }

    /**
     * 这个方法的名字具有误导性，如果要笔者来取，应该取名为 buildGraphAndFindRoots 。没错，这个方法就是先构造 DAG ，后寻找根结点。
     * 参数 rulesFilter 表示本次构造 DAG 只考察哪些规则（将这些规则作为 DAG 的『边』）。没有出现在 rulesFilter 中的规则在本次构造过程中被忽略。
     */
    private ArrayDeque<Node> findRoots(int[] rulesFilter) {
        final SparseArray<Node> keyNodes = mKeyNodes;
        final ArrayList<Node> nodes = mNodes;
        final int count = nodes.size();
       
        // 清理图中的依赖关系引用，也就是删去 DAG 中的『边』。这个方法会根据传入的不同的规则被调用多次，
        // 会构造出不同的 DAG 来，因此这些『边』需要被重新构造。
        for (int i = 0; i < count; i++) {
            final Node node = nodes.get(i);
            node.dependents.clear();
            node.dependencies.clear();
        }

 
        // 遍历图中的所有结点，即 RelativeLayout 中的所有子视图；构造 DAG 。
        for (int i = 0; i < count; i++) {
            final Node node = nodes.get(i);

            // 读取子视图布局参数中的规则。
            final LayoutParams layoutParams = (LayoutParams) node.view.getLayoutParams();
            final int[] rules = layoutParams.mRules;
            final int rulesCount = rulesFilter.length;
 
            // 只读取本次构造感兴趣的的规则。
            for (int j = 0; j < rulesCount; j++) {
                final int rule = rules[rulesFilter[j]];
 
                // int 数组元素默认为 0 ，大于 0 则说明这个子视图配置了相应的规则，值就是此规则对应的依赖视图的 ID ；
                // 小于 0 则说明这个规则是与对齐 RelativeLayout 相关的 alignParent* 或 center* 规则，
                // 不是描述子视图之间的关系，故可以安全跳过。
                if (rule > 0) {
 
                    // 从图中按依赖 ID 为索引找出此结点的依赖引用。
                    final Node dependency = keyNodes.get(rule);
                    // 如果为空，或者依赖自己，则简单跳过。
                    if (dependency == null || dependency == node) {
                        continue;
                    }
 
                    // 维护 DAG 的边关系。虽然这里看起来建立了一个依赖者与被依赖者的『双向』连接，
                    // 但注意 dependents 和 dependencies 的语义是不同的，本质上它们仍然是单向边。
                    // 维护反向关系是为了更便捷地完成后续的拓扑排序。
                    dependency.dependents.put(node, this);
                    node.dependencies.put(rule, dependency);
                }
            }
        } // DAG 构造完毕。

        final ArrayDeque<Node> roots = mRoots;
        roots.clear();

        // 寻找没有依赖的结点，它们即为此图的『根结点』。
        for (int i = 0; i < count; i++) {
            final Node node = nodes.get(i);
            if (node.dependencies.size() == 0) roots.addLast(node);
        }

        return roots;
    }
    
}
```

## 测量与布局
`RelativeLayout` 的测量过程和布局过程是同时进行的，这是由 `RelativeLayout` 的特性决定的：子视图之间描述相对位置关系，则我们需要用位置关系来确定尺寸。从这个角度来理解的话， `RelativeLayout` 的测量与布局过程*似乎*是反过来的（先布局，后测量），但我们看过 `onMeasure()` 的实现后，就会发现，测量与布局过程是穿插进行的。

### `onMeasure()`
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // mDirtyHierarchy 标记在 requestLayout() 中被置为 true ，它表示因为某种原因，需要重新对子视图做拓扑排序。
    // 首次测量时， mDirtyHierarchy 已经是 true ，因为 RelativeLayout 中有子视图，
    // 而 ViewGroup.addView() 中已经调用过 requestLayout() 了。
    if (mDirtyHierarchy) {
        mDirtyHierarchy = false;
        // 此方法会构造依赖图 DependencyGraph 并分别针对横向、纵向规则完成拓扑排序。
        // 排序的结果保存在成员 mSortedVerticalChildren 与 mSortedHorizontalChildren 中。
        sortChildren();
    }

    // myWidth 与 myHeight 存储的是 RelativeLayout 的测量规格尺寸（如果规格不是 UNSPECIFIED 的话）。
    int myWidth = -1;
    int myHeight = -1;

    // width 与 height 存储的是 RelativeLayout 的精确尺寸。
    int width = 0;
    int height = 0;

    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    final int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    // 基于测量规格更新 myWidth, myHeight, width, height 。
    if (widthMode != MeasureSpec.UNSPECIFIED) {
        myWidth = widthSize;
    }

    if (heightMode != MeasureSpec.UNSPECIFIED) {
        myHeight = heightSize;
    }

    // 如果 RelativeLayout 的父亲已经把它的尺寸确定，那么现在就可以直接更新 width 与 height 。
    if (widthMode == MeasureSpec.EXACTLY) {
        width = myWidth;
    }

    if (heightMode == MeasureSpec.EXACTLY) {
        height = myHeight;
    }
 
    // RelativeLayout 与 LinearLayout 相同，也支持全局对齐方式。
    // 通过位运算分别获取横向、纵向的对齐方式。
    int gravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
    final boolean horizontalGravity = gravity != Gravity.START && gravity != 0;
    gravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
    final boolean verticalGravity = gravity != Gravity.TOP && gravity != 0;

    int left = Integer.MAX_VALUE;
    int top = Integer.MAX_VALUE;
    int right = Integer.MIN_VALUE;
    int bottom = Integer.MIN_VALUE;

    // 分别标记在水平、垂直方向是否有子视图需要居中，或是右 / 底部对齐。
    boolean offsetHorizontalAxis = false;
    boolean offsetVerticalAxis = false;

    // 只要测量规格模式不是 EXACTLY ，那么这个视图（RelativeLayout）的宽（高）就肯定被设为 WRAP_CONTENT。
    // 具体模式是 AT_MOST ，还是 UNSPECIFIED ，则取决于 RelativeLayout 的父亲是什么。
    final boolean isWrapContentWidth = widthMode != MeasureSpec.EXACTLY;
    final boolean isWrapContentHeight = heightMode != MeasureSpec.EXACTLY;

    // RTL 文字兼容。
    final int layoutDirection = getLayoutDirection();

    // 准备子视图第一轮测量与布局。这一轮遍历的目的是完成子视图横向的测量与布局。
    // 马上，就可以看到为何说 RelativeLayout 的测量与布局过程是同时进行的。
    View[] views = mSortedHorizontalChildren;
    int count = views.length;

    for (int i = 0; i < count; i++) {
        View child = views[i];
        // 对于可见性为 GONE 的视图，跳过其测量与布局。
        if (child.getVisibility() != GONE) {
            LayoutParams params = (LayoutParams) child.getLayoutParams();
            // RTL 文字的兼容通过这个方法完成，所有 START / END 规则会基于当前系统文字
            // 和 API 版本是否支持 RTL 适当地修正为对应的 LEFT / RIGHT 规则。
            int[] rules = params.getRules(layoutDirection);

            // 根据子视图的布局参数、 RelativeLayout 的测量宽度和经过 RTL 修正后的规则
            // 完成该视图的 *初步* 横向布局（这里的『布局』并非指真正调用 child.layout() ，只是
            // 提前为它们确定了要传入的 left, right 参数）。
            // 笔者建议读者现在就在下方找到此方法的实现阅读一下，非常简单、直白。
            applyHorizontalSizeRules(params, myWidth, rules);
            // 在水平方向测量这个孩子。它装配子视图布局参数的过程值得学习借鉴。
            measureChildHorizontal(child, params, myWidth, myHeight);

            // 为什么笔者刚刚要强调 applyHorizontalSizeRules 是完成初步布局呢？
            // 看过了那个方法实现的读者应该明白，对于不存在规则的边，其 left / right 被设为了
            // VALUE_NOT_SET ，而这个常量的值为 -1 ，它只是一个临时标记值，显然不能直接拿来
            // 作为真正的布局参数。下面这个方法就会在测量了子视图的宽度后，消灭 VALUE_NOT_SET，
            // 并且根据子视图是否配置水平居中属性，将其布局到 RelativeLayout 水平方向的中部。
 
            // 如果这个子视图设为居中对齐或是右对齐，更新标记，表明如果 RelativeLayout 的宽度不是确定的，
            // 在通过测量所有子视图确定 RelativeLayout 宽度后需要对这些子视图重新布局。
            if (positionChildHorizontal(child, params, myWidth, isWrapContentWidth)) {
                offsetHorizontalAxis = true;
            }
        }
    } // 水平方向的子视图测量与布局完成。

    // 现在，准备垂直方向的测量与布局。
    views = mSortedVerticalChildren;
    count = views.length;
    final int targetSdkVersion = getContext().getApplicationInfo().targetSdkVersion;

    for (int i = 0; i < count; i++) {
        final View child = views[i];
        // 同样跳过可见性为 GONE 的子视图。基本上是和水平方向一致的过程，只是方向不同。
        if (child.getVisibility() != GONE) {
            final LayoutParams params = (LayoutParams) child.getLayoutParams();

            // 根据垂直方向的规则，完成垂直方向上的初步布局。
            applyVerticalSizeRules(params, myHeight, child.getBaseline());
            // 这一次测量子视图就是最终的结果了，因为它的横、纵向的规则已经完全确定了。
            measureChild(child, params, myWidth, myHeight);
 
            // 消灭 VALUE_NOT_SET ，尝试垂直居中，更新标记。与水平方向的 positionChildHorizontal 
            // 意义是完全一致的。
            if (positionChildVertical(child, params, myHeight, isWrapContentHeight)) {
                offsetVerticalAxis = true;
            }

            // 到这里，当前遍历到的这个子视图的测量过程完全结束，是时候根据它的尺寸来测量 RelativeLayout 自己了。
            // 如果 RelativeLayout 的测量规格模式不是 EXACTLY ，则需要根据子视图大小来更新自己的大小。
            // 首先的宽度。
            if (isWrapContentWidth) {
                // 判断目标 SDK 版本是为了对以前 RelativeLayout 的 bug 做兼容。可以看到，这个 bug 就是
                // 没有考虑子视图的 margin 。读者无视这个兼容吧。下面也有。
                if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                    width = Math.max(width, params.mRight);
                } else {
                    // 测量规格模式不是 EXACTLY ，更新为当前值与子视图 right 布局参数 + 其右 margin 的更大者。
                    width = Math.max(width, params.mRight + params.rightMargin);
                }
            }

            // 然后是高度。
            if (isWrapContentHeight) {
                if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                    height = Math.max(height, params.mBottom);
                } else {
                    height = Math.max(height, params.mBottom + params.bottomMargin);
                }
            }

            // 如果需要使用 RelativeLayout 的全局对齐，则更新所有子视图的最左、最右、最上、最下边界。
            if (verticalGravity) {
                left = Math.min(left, params.mLeft - params.leftMargin);
                top = Math.min(top, params.mTop - params.topMargin);
            }

            if (horizontalGravity) {
                right = Math.max(right, params.mRight + params.rightMargin);
                bottom = Math.max(bottom, params.mBottom + params.bottomMargin);
            }
        }
    } // 垂直方向的子视图测量与布局完成。

    // 接下来的工作，是在 RelativeLayout 的宽、高被设为 WRAP_CONTENT 时，
    // 对 RelativeLayout 的宽、高做最后的修正，然后为水平、垂直居中的子视图布局。
    // 首先是宽度。
    if (isWrapContentWidth) {
        // 这里只加了右 padding ，因为之前更新宽度时使用的是子视图的右边界，而计算右边界已经考虑了
        // RelativeLayout 的左 padding 了。
        width += mPaddingRight;

        if (mLayoutParams != null && mLayoutParams.width >= 0) {
            // 如果为 RelativeLayout 设定了一个固定的宽度，那么取这个设定值与测量值中更大的那一个。
            width = Math.max(width, mLayoutParams.width);
        }

        // 检查最小值设定，不能小过最小值。
        width = Math.max(width, getSuggestedMinimumWidth());
        // 与自己的宽度测量规格进行比较，就像它测量自己的孩子那样， 
        // RelativeLayout 自己也是某一个 ViewGroup 的子视图，它不能把父布局给撑爆。
        width = resolveSize(width, widthMeasureSpec);
        // 如此下来， RelativeLayout 的宽度算是完全确定了。

        // 确定了宽度，来处理需要水平居中或右对齐的视图的布局问题。
        if (offsetHorizontalAxis) {
            for (int i = 0; i < count; i++) {
                final View child = views[i];
                if (child.getVisibility() != GONE) {
                    final LayoutParams params = (LayoutParams) child.getLayoutParams();
                    final int[] rules = params.getRules(layoutDirection);
                    // 只需重新布局需要水平居中和右对齐的视图。算法很直白了。
                    if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
                        centerHorizontal(child, params, width);
                    } else if (rules[ALIGN_PARENT_RIGHT] != 0) {
                        final int childWidth = child.getMeasuredWidth();
                        params.mLeft = width - mPaddingRight - childWidth;
                        params.mRight = params.mLeft + childWidth;
                    }
                }
            }
        }
    }

    // 然后是高度。步骤与宽度完全一致，仅仅是方向发生变化。理解有困难的读者请对照着上面的注释看。
    if (isWrapContentHeight) {
        // Height already has top padding in it since it was calculated by looking at
        // the bottom of each child view
        height += mPaddingBottom;

        if (mLayoutParams != null && mLayoutParams.height >= 0) {
            height = Math.max(height, mLayoutParams.height);
        }

        height = Math.max(height, getSuggestedMinimumHeight());
        height = resolveSize(height, heightMeasureSpec);

        if (offsetVerticalAxis) {
            for (int i = 0; i < count; i++) {
                final View child = views[i];
                if (child.getVisibility() != GONE) {
                    final LayoutParams params = (LayoutParams) child.getLayoutParams();
                    final int[] rules = params.getRules(layoutDirection);
                    if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_VERTICAL] != 0) {
                        centerVertical(child, params, height);
                    } else if (rules[ALIGN_PARENT_BOTTOM] != 0) {
                        final int childHeight = child.getMeasuredHeight();
                        params.mTop = height - mPaddingBottom - childHeight;
                        params.mBottom = params.mTop + childHeight;
                    }
                }
            }
        }
    }

    // 测量过程最后的一步：处理 RelativeLayout 的全局对齐属性。
    if (horizontalGravity || verticalGravity) {
        // 如果设置了全局的对齐方式：
 
        // 使用两个 Rect 。 mSelfBounds 与 mContentBounds 这两个成员均被 final 修饰。
        final Rect selfBounds = mSelfBounds;
        // 从 set() 传的参数可看出 mSelfBounds 表示的是 RelativeLayout 所有可用的空间。
        selfBounds.set(mPaddingLeft, mPaddingTop, width - mPaddingRight,
                height - mPaddingBottom);

        final Rect contentBounds = mContentBounds;
        // Gravity.apply() 方法会根据传入的对齐属性、传入的宽高、代表可用空间的 Rect 对象
        // （即 contentBounds），生成对应的计算结果到 contentBounds 中。
        // 这里，传入的宽高是第二轮子视图测量中维护并更新的所有子视图的最广边界，也即 mContentBounds
        // 表示的是尺寸刚刚能容纳下所有子视图的一个矩形，它根据对齐属性和尺寸，被妥善地放到了 mSelfBounds 
        // 的内部。接下来就要使用它的位置来修正所有视图的位置。
        Gravity.apply(mGravity, right - left, bottom - top, selfBounds, contentBounds,
                layoutDirection);

        // 计算 contentBounds 相对原最左端、最上端的偏移量。
        final int horizontalOffset = contentBounds.left - left;
        final int verticalOffset = contentBounds.top - top;
        if (horizontalOffset != 0 || verticalOffset != 0) {
            for (int i = 0; i < count; i++) {
                final View child = views[i];
                if (child.getVisibility() != GONE) {
                    final LayoutParams params = (LayoutParams) child.getLayoutParams();
                    // 将偏移加到即将使用的布局方法参数上。
                    if (horizontalGravity) {
                        params.mLeft += horizontalOffset;
                        params.mRight += horizontalOffset;
                    }
                    if (verticalGravity) {
                        params.mTop += verticalOffset;
                        params.mBottom += verticalOffset;
                    }
                }
            }
        }
    }

    setMeasuredDimension(width, height);
}
```

### 测量过程中的辅助方法
#### `applyHorizontalSizeRules()`
```java
// 在水平方向基于设定的规则完成对某一子视图的初步布局。
// 可以看到， RelativeLayout 把布局结果临时写到了子视图的布局参数中。
private void applyHorizontalSizeRules(LayoutParams childParams, int myWidth, int[] rules) {
    RelativeLayout.LayoutParams anchorParams;

    // 默认值 VALUE_NOT_SET 的意义是，子视图的这条边界可以『想在哪就在哪』。
    childParams.mLeft = VALUE_NOT_SET;
    childParams.mRight = VALUE_NOT_SET;

    // 通过依赖图维护的按 ID 索引的子视图引用稀疏数组，获取该子视图右边界想要
    // （直接或间接，如果它们之间存在可见性为 GONE 的视图的话）对齐目标视图左边界的布局参数
    // （即 LEFT_OF 的语义）。
    anchorParams = getRelatedViewParams(rules, LEFT_OF);
    // 如果非空，则执行该语义。
    if (anchorParams != null) {
        childParams.mRight = anchorParams.mLeft - (anchorParams.leftMargin +
                childParams.rightMargin);
    } else if (childParams.alignWithParent && rules[LEFT_OF] != 0) {
        // alignWithParent 是 RelativeLayout 为子视图提供的一个可选属性，它表示如果在某一条边界配置了
        // 想要对齐目标视图，但却没有找到满足条件的视图（即要么通过直接或间接依赖找到的视图要么可视性均为 
        // GONE ，要么指定的 ID 根本不存在），那么就令其对齐父 RelativeLayout 。
        // 这个属性在实际开发中使用得较少。
        if (myWidth >= 0) {
            childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
        }
    }

    // 可以看到，接下来的几步都在根据不同的规则重复上面的步骤，现在是尝试将左边界对齐目标视图的右边界
    // （即 RIGHT_OF 的语义）。
    anchorParams = getRelatedViewParams(rules, RIGHT_OF);
    if (anchorParams != null) {
        childParams.mLeft = anchorParams.mRight + (anchorParams.rightMargin +
                childParams.leftMargin);
    } else if (childParams.alignWithParent && rules[RIGHT_OF] != 0) {
        childParams.mLeft = mPaddingLeft + childParams.leftMargin;
    }

    // 尝试将左边界对齐目标视图的左边界（即 ALIGN_LEFT 的语义）。
    // 可以看到， ALIGN_LEFT 会静默地覆盖掉 RIGHT_OF 的结果。
    anchorParams = getRelatedViewParams(rules, ALIGN_LEFT);
    if (anchorParams != null) {
        childParams.mLeft = anchorParams.mLeft + childParams.leftMargin;
    } else if (childParams.alignWithParent && rules[ALIGN_LEFT] != 0) {
        childParams.mLeft = mPaddingLeft + childParams.leftMargin;
    }

    // 尝试将右边界对齐目标视图的右边界（即 ALIGN_RIGHT 的语义）。
    // 可以看到， ALIGN_RIGHT 会静默地覆盖掉 LEFT_OF 的结果。
    anchorParams = getRelatedViewParams(rules, ALIGN_RIGHT);
    if (anchorParams != null) {
        childParams.mRight = anchorParams.mRight - childParams.rightMargin;
    } else if (childParams.alignWithParent && rules[ALIGN_RIGHT] != 0) {
        if (myWidth >= 0) {
            childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
        }
    }

    // 如果该子视图配置了对齐父 RelativeLayout 左边界，那么执行语义。
    // 可以看到， ALIGN_PARENT_LEFT 又静默地覆盖了 RIGHT_OF, ALIGN_LEFT 的结果。
    if (0 != rules[ALIGN_PARENT_LEFT]) {
        childParams.mLeft = mPaddingLeft + childParams.leftMargin;
    }

    // 如果该子视图配置了对齐父 RelativeLayout 右边界，那么执行语义。
    // 可以看到， ALIGN_PARENT_RIGHT 又静默地覆盖了 LEFT_OF, ALIGN_RIGHT 的结果。
    if (0 != rules[ALIGN_PARENT_RIGHT]) {
        if (myWidth >= 0) {
            childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
        }
    }
}
```
#### `measureChildHorizontal()`
```java
// 在水平方向上测量某一个子视图，在第一轮子视图测量中，完成子视图的布局后调用。
private void measureChildHorizontal(
        View child, LayoutParams params, int myWidth, int myHeight) {
    // 根据布局结果，装配宽度测量参数。调用的这个方法是水平、垂直方向通用的，我们单独分析它的实现。
    final int childWidthMeasureSpec = getChildMeasureSpec(params.mLeft, params.mRight,
            params.width, params.leftMargin, params.rightMargin, mPaddingLeft, mPaddingRight,
            myWidth);

    // 接下来装配高度测量参数。在调用这个方法时， RelativeLayout 还没有做这个子视图在垂直方向上的布局。
    // 因此生成的高度测量参数很有可能是不正确的，但是这并不影响第一轮测量过程。
    final int childHeightMeasureSpec;
 
    // myHeight 如果小于 0 （即为默认值 -1 ），则说明 RelativeLayout 的高度不定。
    if (myHeight < 0) {
        // 那么，如果这个子视图设置的高度大于 0 ，那就『暂时』让它照设置的那么高。
        if (params.height >= 0) {
            childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                    params.height, MeasureSpec.EXACTLY);
        } else {
            // 否则，子视图也可以『暂时』想多高就多高。
            childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
        }
    } else {
        // RelativeLayout 要么有一个确定的高度，要么有一个最大高度限制。
        // 『暂时』不要让这个视图的高度超过 RelativeLayout 的最大限制即可。
        final int maxHeight = Math.max(0, myHeight - mPaddingTop - mPaddingBottom
                    - params.topMargin - params.bottomMargin);

        // 根据子视图的布局参数中设定的高度确定测量规格模式。
        final int heightMode;
        if (params.height == LayoutParams.MATCH_PARENT) {
            heightMode = MeasureSpec.EXACTLY;
        } else {
            heightMode = MeasureSpec.AT_MOST;
        }
        childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(maxHeight, heightMode);
    }

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
### `positionChildHorizontal()`
```java
// 完成子视图水平方向的测量后，确定它的 layout() 方法水平方向参数（实际上就是提前完成布局）。
// 这个方法的返回值表示该子视图要么水平居中于 RelativeLayout ，要么右对齐于 RelativeLayout 。
private boolean positionChildHorizontal(View child, LayoutParams params, int myWidth,
        boolean wrapContent) {

    // RTL 参数修正。
    final int layoutDirection = getLayoutDirection();
    int[] rules = params.getRules(layoutDirection);

    // 根据先前的初步布局结果以及测量后的 measured width 分情况确定 left 和 right 。
    if (params.mLeft == VALUE_NOT_SET && params.mRight != VALUE_NOT_SET) {
        // 右边界已由规则确定，左边界暂不定：将左边界设为右边界位置 - 宽度
        params.mLeft = params.mRight - child.getMeasuredWidth();
    } else if (params.mLeft != VALUE_NOT_SET && params.mRight == VALUE_NOT_SET) {
        // 左边界已由规则确定，右边界暂不定：将右边界设为左边界位置 + 宽度
        params.mRight = params.mLeft + child.getMeasuredWidth();
    } else if (params.mLeft == VALUE_NOT_SET && params.mRight == VALUE_NOT_SET) {
        // 两头都还没有被规则确定。只在这种情况尝试执行居中。
        if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
            // 如果配置了居中属性：
            if (!wrapContent) {
                // 如果 RelativeLayout 的宽度测量规格模式是 EXACTLY ，即 RelativeLayout 宽度确定，
                // 则现在就可以根据测得的宽度确定这个视图的位置了。
                centerHorizontal(child, params, myWidth);
            } else {
                // 否则，现在暂时无法确定居中位置（ RelativeLayout 的宽度尚不确定 ）。
                // 暂时将这个视图左对齐。在确定了 RelativeLayout 的宽度后，会为它们重新布局。
                params.mLeft = mPaddingLeft + params.leftMargin;
                params.mRight = params.mLeft + child.getMeasuredWidth();
            }
            return true;
        } else {
            // 没有配置居中属性，说明在水平方向，该子视图不受任何规则的约束。执行默认的左对齐。
            params.mLeft = mPaddingLeft + params.leftMargin;
            params.mRight = params.mLeft + child.getMeasuredWidth();
        }
    }
    return rules[ALIGN_PARENT_END] != 0;
}
```
#### `applyVerticalSizeRules()`
```java
// 在垂直方向基于设定的规则完成对某一子视图的初步布局。除了被笔者刻意省略的基线规则处理，和没有 RTL 兼容外，
// 整体流程和水平方向的 applyHorizontal 是高度一致的。注释内容请对照着 applyHorizontalSizeRules 看。
private void applyVerticalSizeRules(LayoutParams childParams, int myHeight, int myBaseline) {
    final int[] rules = childParams.getRules();

    RelativeLayout.LayoutParams anchorParams;

    childParams.mTop = VALUE_NOT_SET;
    childParams.mBottom = VALUE_NOT_SET;

    anchorParams = getRelatedViewParams(rules, ABOVE);
    if (anchorParams != null) {
        childParams.mBottom = anchorParams.mTop - (anchorParams.topMargin +
                childParams.bottomMargin);
    } else if (childParams.alignWithParent && rules[ABOVE] != 0) {
        if (myHeight >= 0) {
            childParams.mBottom = myHeight - mPaddingBottom - childParams.bottomMargin;
        }
    }

    anchorParams = getRelatedViewParams(rules, BELOW);
    if (anchorParams != null) {
        childParams.mTop = anchorParams.mBottom + (anchorParams.bottomMargin +
                childParams.topMargin);
    } else if (childParams.alignWithParent && rules[BELOW] != 0) {
        childParams.mTop = mPaddingTop + childParams.topMargin;
    }

    anchorParams = getRelatedViewParams(rules, ALIGN_TOP);
    if (anchorParams != null) {
        childParams.mTop = anchorParams.mTop + childParams.topMargin;
    } else if (childParams.alignWithParent && rules[ALIGN_TOP] != 0) {
        childParams.mTop = mPaddingTop + childParams.topMargin;
    }

    anchorParams = getRelatedViewParams(rules, ALIGN_BOTTOM);
    if (anchorParams != null) {
        childParams.mBottom = anchorParams.mBottom - childParams.bottomMargin;
    } else if (childParams.alignWithParent && rules[ALIGN_BOTTOM] != 0) {
        if (myHeight >= 0) {
            childParams.mBottom = myHeight - mPaddingBottom - childParams.bottomMargin;
        }
    }

    if (0 != rules[ALIGN_PARENT_TOP]) {
        childParams.mTop = mPaddingTop + childParams.topMargin;
    }

    if (0 != rules[ALIGN_PARENT_BOTTOM]) {
        if (myHeight >= 0) {
            childParams.mBottom = myHeight - mPaddingBottom - childParams.bottomMargin;
        }
    }
}
```
#### `measureChild()`
```java
// 这个方法的调用时机是一个子视图水平、垂直方向上的规则均已 apply 之后，此时可以根据规则完全确定
// 这个子视图的尺寸了，因此对它做第二次测量，修正第一次测量中 *可能* 不正确的高度测量规格。
private void measureChild(View child, LayoutParams params, int myWidth, int myHeight) {
    int childWidthMeasureSpec = getChildMeasureSpec(params.mLeft,
            params.mRight, params.width,
            params.leftMargin, params.rightMargin,
            mPaddingLeft, mPaddingRight,
            myWidth);
    // 对比 measureChildHorizontal() 即可发现，这个方法就是直接调用两次 getChildMeasureSpec() ，
    // 原因已经很清楚了（调用前者时，没有分析水平方向的规则，调用这个方法时，则已分析过了）。
    int childHeightMeasureSpec = getChildMeasureSpec(params.mTop,
            params.mBottom, params.height,
            params.topMargin, params.bottomMargin,
            mPaddingTop, mPaddingBottom,
            myHeight);
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
#### `getChildMeasureSpec()`
```java
// 这个方法就是 measureChild*() 中调用的装配子视图测量参数的便利方法。它被设计成了水平、垂直方向通用的。
// 注意这个方法的调用时机，此时每个方向上的『二次』布局还没有发生， VALUE_NOT_SET 这个标志
// 还参与到了子视图的测量过程中。笔者认为，这是一个非常巧妙的设计。
private int getChildMeasureSpec(int childStart, int childEnd,
        int childSize, int startMargin, int endMargin, int startPadding,
        int endPadding, int mySize) {
    int childSpecMode = 0;
    int childSpecSize = 0;

    // mySize 小于 0 即 RelativeLayout 在这个方向上的测量参数规格为 UNSPECIFIED ，大小不受限。
    final boolean isUnspecified = mySize < 0;
    if (isUnspecified) {
        // 如果 RelativeLayout 在这个方向上大小不受限，那就很简单了，子视图在这个方向也可以想要多大就有多大。
        if (childStart != VALUE_NOT_SET && childEnd != VALUE_NOT_SET) {
            // 子视图在该方向起始、终止位置都由规则限定，那就遵照规则限定来。
            childSpecSize = Math.max(0, childEnd - childStart);
            childSpecMode = MeasureSpec.EXACTLY;
        } else if (childSize >= 0) {
            // 子视图没有完全被规则限定（或是完全没有），那就让它『想要多大有多大』。
            childSpecSize = childSize;
            childSpecMode = MeasureSpec.EXACTLY;
        } else {
            // 子视图在这个方向的尺寸被设为 WRAP_CONTENT 或者 MATCH_PARENT 。
            childSpecSize = 0;
            childSpecMode = MeasureSpec.UNSPECIFIED;
        } 
        // 从这一连串判断条件可以看出，为 RelativeLayout 的子视图设定大小时， RelativeLayout
        // 参照的优先级是：加在这一方向上的规则 > 设定的尺寸。如果规则确定了子视图在这一方向上的位置，
        // 那么尺寸的设置会在测量过程中忽略。

        return MeasureSpec.makeMeasureSpec(childSpecSize, childSpecMode);
    }
 
    // 到这里则说明 RelativeLayout 自身在此方向有大小限制。

    // 需要通过这个方向上的规则、子视图想要的大小以及 RelativeLayout 的大小限制三方综合考虑，计算
    // 子视图的大小。
    int tempStart = childStart;
    int tempEnd = childEnd;

    // 子视图的起始边界没有受规则限制，那么它最远也不能远过 RelativeLayout 的起始位置。
    if (tempStart == VALUE_NOT_SET) {
        // 同时，还要考虑 RelativeLayout 自己的 margin 和 padding 
        tempStart = startPadding + startMargin;
    }
    // 子视图的终止边界没有受规则限制，那么它最远也不能远过 RelativeLayout 的终止位置。
    if (tempEnd == VALUE_NOT_SET) {
        tempEnd = mySize - endPadding - endMargin;
    }

    // 现在暂时确定了子视图的边界，计算它在这样的边界规则下可用多少空间。
    final int maxAvailable = tempEnd - tempStart;

    if (childStart != VALUE_NOT_SET && childEnd != VALUE_NOT_SET) {
        // 如果子视图在该方向起始、终止位置都已经被规则限定，那就遵照规则限定来。
        childSpecMode = MeasureSpec.EXACTLY;
        // 不允许负尺寸（可能会和 WRAP_CONTENT / MATCH_PARENT 产生歧义）。
        childSpecSize = Math.max(0, maxAvailable);
    } else {
        // 否则，需要视其想要的尺寸与可用空间的大小关系。
        if (childSize >= 0) {
            // 想要一个大于 0 的尺寸，则测量规格模式可确定。
            childSpecMode = MeasureSpec.EXACTLY;
            // 取可用空间与子视图想要的尺寸中更小的那一个。
            childSpecSize = Math.min(maxAvailable, childSize);
        } else if (childSize == LayoutParams.MATCH_PARENT) {
            // 子视图想要所有可用的尺寸，则把可用空间全部给它。
            childSpecMode = MeasureSpec.EXACTLY;
            childSpecSize = Math.max(0, maxAvailable);
        } else if (childSize == LayoutParams.WRAP_CONTENT) {
            // 子视图想要一个『刚刚好』的尺寸。
            // 则 RelativeLayout 指定最大值即可。
            childSpecMode = MeasureSpec.AT_MOST;
            childSpecSize = maxAvailable;
        }
    }

    return MeasureSpec.makeMeasureSpec(childSpecSize, childSpecMode);
}
```

### `onLayout()`
最后，看一下 `onLayout()` 的实现。毫无意外，它就是对每个可见性不为 `GONE` 的子视图，把在 `onMeasure()` 过程中已经在 `RelativeLayout.LayoutParams` 中维护好的四个布局方法参数取出，作为 `child.layout()` 的实参。

```java

@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    final int count = getChildCount();

    for (int i = 0; i < count; i++) {
        View child = getChildAt(i);
        if (child.getVisibility() != GONE) {
            RelativeLayout.LayoutParams st =
                    (RelativeLayout.LayoutParams) child.getLayoutParams();
            child.layout(st.mLeft, st.mTop, st.mRight, st.mBottom);
        }
    }
}
```

## 总结
对 `RelativeLayout` 的实现分析到这里就结束了。现在就 `RelativeLayout` 的整体实现作一简要的总结。

*  使用有向无环图（ DAG ）描述视图间的依赖关系
*  子视图所有可用的依赖关系和配置项使用一个 `int` 数组存放在 `LayoutParams` 中
*  测量过程中，垂直、水平方向的规则分别作为图中的边，两次构造依赖图并拓扑排序
*  两轮子视图测量，分别 apply 加在子视图上的水平、垂直规则，同时穿插布局过程
*  布局方法参数在测量过程中缓存在每一个子视图的 `LayoutParams` 里，在布局过程中直接取出来用
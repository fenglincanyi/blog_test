---
title: 滑动相关 ScrollView & ListView
date: 2017-07-30 11:13
categories: Android
tags: [滑动, ScrollView, ListView]
---

## ScrollView 嵌套 ListView 问题分析
**滑动冲突的问题**
ScrollView 中嵌套 ListView，ListView 在测量时，高度不论写活还是写死，Listiew 的测量模式都是 不确定模式：unspecified

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView1.png)

ScrollView 换成 LinearLayout后：

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView2.png)

经过上面的实验，什么原因造成的 heightMeasureSpec的值是 680 呢？


**追踪过程**

在ScrollView 测量时：

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView3.png)

Listview 测量时：heightMeasureSpec = 680

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView4.png)

再继续向前追溯：
ScrollView的 onMeasure() 调用了父类的  onMeasure() 即 FrameLayout 的：

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView5.png)

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView6.png)

再仔细观察：

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView7.png)


可以再查看ScrollView 的 measureChild() 方法，也是通过 设置 高度的测量模式：unspecified


我们对比 未覆盖的  ViewGroup的代码：
```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```


到最后，传到listview的 onMeasure 中后，我们来分析：
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // Sets up mListPadding
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);// heightMeasureSpec：680  计算结果： heightMode：0
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    int childWidth = 0;
    int childHeight = 0;
    int childState = 0;

    mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
    if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED
            || heightMode == MeasureSpec.UNSPECIFIED)) {
        final View child = obtainView(0, mIsScrap);

        // Lay out child directly against the parent measure spec so that
        // we can obtain exected minimum width and height.
        measureScrapChild(child, 0, widthMeasureSpec, heightSize);

        childWidth = child.getMeasuredWidth();
        childHeight = child.getMeasuredHeight();// listview 的 item 高度
        childState = combineMeasuredStates(childState, child.getMeasuredState());

        if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
                ((LayoutParams) child.getLayoutParams()).viewType)) {
            mRecycler.addScrapView(child, 0);
        }
    }

    if (widthMode == MeasureSpec.UNSPECIFIED) {
        widthSize = mListPadding.left + mListPadding.right + childWidth +
                getVerticalScrollbarWidth();
    } else {
        widthSize |= (childState & MEASURED_STATE_MASK);
    }

    if (heightMode == MeasureSpec.UNSPECIFIED) {// 符合条件
        heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                getVerticalFadingEdgeLength() * 2;// 所以 listview 的高度为 item 的高度
    }
    // 此种布局下，不管如何调整高度属性， listview 的 高度 测量规格始终都是 unspecified，

    if (heightMode == MeasureSpec.AT_MOST) {
        // TODO: after first layout we should maybe start at the first visible position, not 0
        heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
    }

    setMeasuredDimension(widthSize, heightSize);

    mWidthMeasureSpec = widthMeasureSpec;
}
```

**解决高度显示的问题**

方法挺多的，这里给出一种：重写listview的onMesasure
https://stackoverflow.com/questions/18367522/android-list-view-inside-a-scroll-view
<br>
对于 ScrollView 的子View ，也不能配置高度属性为 match_parent, 即使设置了，也是按照 wrap_content 的效果走的
而且 Android Studio 在lint 检查的时候，已经建议你这样做了

![](http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView8.png)



## ScrollView 的滑动相关

onTouchEvent 中，move 时：
```java
if (mIsBeingDragged) {
    // Scroll to follow the motion event
    mLastMotionY = y - mScrollOffset[1];

    final int oldY = mScrollY;
    final int range = getScrollRange();
    final int overscrollMode = getOverScrollMode();
    boolean canOverscroll = overscrollMode == OVER_SCROLL_ALWAYS ||
            (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

    // Calling overScrollBy will call onOverScrolled, which
    // calls onScrollChanged if applicable.
    if (overScrollBy(0, deltaY, 0, mScrollY, 0, range, 0, mOverscrollDistance, true)
            && !hasNestedScrollingParent()) {
        // Break our velocity if we hit a scroll barrier.
        mVelocityTracker.clear();
    }
    ... ...
}


private int getScrollRange() {
    int scrollRange = 0;
    if (getChildCount() > 0) {
        View child = getChildAt(0);
        scrollRange = Math.max(0,
                child.getHeight() - (getHeight() - mPaddingBottom - mPaddingTop));
    // 可滑动的范围： 子view的高度 - ScrollView 的高度（去掉上下内边距的）
    }
    return scrollRange;
}
```

这里，曾经遇到过的问题：

想在 ScrollView 中滑到底部后，多滑动一部分距离来显示上面的距离，因为上层盖有view, 想做出透明的效果，如主页 tab + 列表的呈现方式；
这时候可以这样处理：重写 addView() , 并把唯一的子View 添加一个 bottomPadding，就是增大 child.Hegiht.  所以增大了 ScrollRange；

类比：listview ，可以增加 一个有高度 的footer ,实现这种功能

**滑动冲突的问题**

解决办法
```java
listView.setOnTouchListener(object : View.OnTouchListener {
    override fun onTouch(v: View, event: MotionEvent): Boolean {
        v.parent.parent.requestDisallowInterceptTouchEvent(true)
        return false;
    }
})
// 请求 父view 不允许拦截，则交给 listview 来处理 move 事件
```
<img src="http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView9.gif" width="320" height="560"/>

另外，ScrollView嵌套RecyclerView也会存在显示不全的问题

---

**翻开历史新篇章**：

V4包 中提供了 可支持嵌套滑动的 ScrollView：
NestedScrollView


我们再试试 NestedScrollView + ListView

1. Listview 高度：wrap_content  ，和上面的情况一样，listview的高度就是一条 item的gaoud
2. Listview 高度：200dp , 则能全部显示出来

如图：

<img src="http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView10.png" width="320" height="560"/>  <img src="http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView11.png" width="320" height="560"/>

我们再换用
NestedScrollView嵌套RecyclerView不会存在显示不全的问题; google 已经帮我们处理好了

继续测试：
1. recyclerView  高度：wrap_content
2. recyclerView 高度：200dp

分别如图：
wrap_content 时，RecycleView 会全部显示出来

<img src="http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView12.png" width="320" height="560"/>  <img src="http://7xr1vo.com1.z0.glb.clouddn.com/ScrollViewListView13.png" width="320" height="560"/>


实现细节，可以看看 NestedScrollView 和 RecyclerView 的 measure 的处理方式

## 总结

ScrollView 嵌套 Listview 存在以下问题：

1. Listview 高度显示不全，就是因为 ScrollView 测量子View 的高度时，是 不确定 模式，导致Listview 绘制时候，不能确定高度，所以显示 item 的高度
2. 滑动冲突，Listview滑动并不能响应，会被ScrollView拦截响应。
3. Listview 复用失效，由于Listview的高度不能确定，则会使屏幕内能够显示的 view 数量不能确定，然后就会不停的 getView()，导致所有的 item 都被创建出来了； RecyclerView 也会出现这类问题

**ListView 在使用 父布局的时候，尽量避免使用RelatvieLayout，因为它的测量、布局过程比 LinearLayout 要多进行几次，就是因为，依赖关系的复杂过程的处理**

NestedScrollView 的特性：

1. 改变了事件传递的性质；原来的事件传递，都是由单个 View 去处理、消费的，而 增加了 嵌套滑动的事件处理机制后，父View和子View就可以同时处理一个事件
2. 借助于 NestedScrollingParentHelper，NestedScrollingChildHelper 来进行事件的传递调用、信息分发和接收，使得父子view 的滑动信息都能获取到，并做相应的节点处理，控制更加自如


----
参考

https://github.com/android-cn/android-open-project-analysis/tree/master/tech/viewdrawflow
http://blog.csdn.net/hanhailong726188/article/details/46136569
https://stackoverflow.com/questions/18367522/android-list-view-inside-a-scroll-view
https://www.zhihu.com/question/34015543
http://www.chenglong.ren/2016/11/14/android%E4%B8%ADnestedscrollview%E7%9A%84%E4%BD%BF%E7%94%A8/



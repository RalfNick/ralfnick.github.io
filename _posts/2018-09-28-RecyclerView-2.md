---
layout: post
title: "RecyclerView-2"
date: 2018-09-28
description: "Android 常用 RecyclerView"
tag: Android 常用
---

[**上一篇**](https://www.jianshu.com/p/553aeaebfa1c) 介绍了 RecyclerView 的基本使用，这篇主要整理下在使用过程中记录的一些小细节，不是很重要的一些知识点，但是这些小细节，如果遇到的话，也是需要上网查资料的，也会有时间成本，所以还是记录下来，以防以后再遇到。

### 1. 分割线

上一篇中已经介绍了分割线，但是悉心的童鞋发现分割线，在 LinearLayoutManager 中使用是正常的，但是在 StaggeredGridLayoutManager 和 GridLayoutManager 中使用是有问题的，在 GridLayoutManager 中，如果是一行两个格子，应该每个格子是自己一条分割线，但是却是公用一条，此外，在使用 StaggeredGridLayoutManager 时显示不出来，针对这种情况，就需要自己实现分割线，继承 RecyclerView.ItemDecoration。

![onedivider](http://pfk1gcktm.bkt.clouddn.com/RecyclerView_fade_edge.png)

还有，对于分割线，一般情况下，使用 ItemDecoration，最底部条目的分割线是不需要的，需要去掉，也是需要在分割线绘制时做一下判断。

下面的是 RecyclerView.ItemDecoration 的三个方法，实现时是要完成这几个方法即可，onDraw 和 onDrawOver 完成其中一个方法即可。

```java

1. 绘制网格线，绘制时机是在每个 item 之前
public void onDraw(Canvas c, RecyclerView parent, State state) {
onDraw(c, parent);
}

2. 绘制网格线，绘制时机是在每个 item 之后
public void onDrawOver(Canvas c, RecyclerView parent, State state) {
onDrawOver(c, parent);
}

3. 设置网格线的位置，根据 view 的位置来确定分割线画在哪，同时设置网格线的矩形区域，及设置宽高和位置
public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state) {
getItemOffsets(outRect, ((LayoutParams) view.getLayoutParams()).getViewLayoutPosition(),
parent);
}

```

```java

public class DividerGridItemDecoration extends RecyclerView.ItemDecoration {

private static final int[] ATTRS = new int[]{android.R.attr.listDivider};
private Drawable mDivider;

public DividerGridItemDecoration(Context context) {
final TypedArray a = context.obtainStyledAttributes(ATTRS);
mDivider = a.getDrawable(0);
a.recycle();
}

@Override
public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {

// 绘制水平分割线和垂直分割线
drawHorizontal(c, parent);
drawVertical(c, parent);

}

private int getSpanCount(RecyclerView parent) {
// 列数
int spanCount = -1;
RecyclerView.LayoutManager layoutManager = parent.getLayoutManager();
if (layoutManager instanceof GridLayoutManager) {
spanCount = ((GridLayoutManager) layoutManager).getSpanCount();
} else if (layoutManager instanceof StaggeredGridLayoutManager) {
spanCount = ((StaggeredGridLayoutManager) layoutManager)
.getSpanCount();
}
return spanCount;
}

public void drawHorizontal(Canvas c, RecyclerView parent) {
int childCount = parent.getChildCount();
for (int i = 0; i < childCount; i++) {
final View child = parent.getChildAt(i);
final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child
.getLayoutParams();
final int left = child.getLeft() - params.leftMargin;
final int right = child.getRight() + params.rightMargin
+ mDivider.getIntrinsicWidth();
final int top = child.getBottom() + params.bottomMargin;
final int bottom = top + mDivider.getIntrinsicHeight();
mDivider.setBounds(left, top, right, bottom);
mDivider.draw(c);
}
}

public void drawVertical(Canvas c, RecyclerView parent) {
final int childCount = parent.getChildCount();
for (int i = 0; i < childCount; i++) {
final View child = parent.getChildAt(i);

final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child
.getLayoutParams();
final int top = child.getTop() - params.topMargin;
final int bottom = child.getBottom() + params.bottomMargin;
final int left = child.getRight() + params.rightMargin;
final int right = left + mDivider.getIntrinsicWidth();

mDivider.setBounds(left, top, right, bottom);
mDivider.draw(c);
}
}

private boolean isLastColum(RecyclerView parent, int pos, int spanCount,
int childCount) {
RecyclerView.LayoutManager layoutManager = parent.getLayoutManager();
if (layoutManager instanceof GridLayoutManager) {
if ((pos + 1) % spanCount == 0)// 如果是最后一列，则不需要绘制右边
{
return true;
}
} else if (layoutManager instanceof StaggeredGridLayoutManager) {
int orientation = ((StaggeredGridLayoutManager) layoutManager)
.getOrientation();
if (orientation == StaggeredGridLayoutManager.VERTICAL) {
if ((pos + 1) % spanCount == 0)// 如果是最后一列，则不需要绘制右边
{
return true;
}
} else {
childCount = childCount - childCount % spanCount;
if (pos >= childCount)// 如果是最后一列，则不需要绘制右边
return true;
}
}
return false;
}

private boolean isLastRaw(RecyclerView parent, int pos, int spanCount,
int childCount) {
RecyclerView.LayoutManager layoutManager = parent.getLayoutManager();
if (layoutManager instanceof GridLayoutManager) {
childCount = childCount - childCount % spanCount;
if (pos >= childCount)// 如果是最后一行，则不需要绘制底部
return true;
} else if (layoutManager instanceof StaggeredGridLayoutManager) {
int orientation = ((StaggeredGridLayoutManager) layoutManager)
.getOrientation();
// StaggeredGridLayoutManager 且纵向滚动
if (orientation == StaggeredGridLayoutManager.VERTICAL) {
childCount = childCount - childCount % spanCount;
// 如果是最后一行，则不需要绘制底部
if (pos >= childCount)
return true;
} else
// StaggeredGridLayoutManager 且横向滚动
{
// 如果是最后一行，则不需要绘制底部
if ((pos + 1) % spanCount == 0) {
return true;
}
}
}
return false;
}

@Override
public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
int spanCount = getSpanCount(parent);
int childCount = parent.getAdapter().getItemCount();

int itemPosition = ((RecyclerView.LayoutParams) view.getLayoutParams()).getViewLayoutPosition();
RecyclerView.LayoutManager layoutManager = parent.getLayoutManager();

if (layoutManager instanceof GridLayoutManager){
int orientation = ((GridLayoutManager) layoutManager).getOrientation();

}
if (isLastRaw(parent, itemPosition, spanCount, childCount))// 如果是最后一行，则不需要绘制底部
{
outRect.set(0, 0, mDivider.getIntrinsicWidth(), 0);
} else if (isLastColum(parent, itemPosition, spanCount, childCount))// 如果是最后一列，则不需要绘制右边
{
outRect.set(0, 0, 0, mDivider.getIntrinsicHeight());
} else {
outRect.set(0, 0, mDivider.getIntrinsicWidth(),
mDivider.getIntrinsicHeight());
}
}
}

```
![multiple_divider](http://pfk1gcktm.bkt.clouddn.com/RecyclerView_GridLayoutManager_divider.png)

### 2. RecyclerView xml 属性设置

xml 布局中有些属性我们要熟悉，像高度设置，滚动条设置，底部的阴影效果等。

```java
<!--重点部分，使用RecyclerView，高度设置，-->
<!--如果是垂直布局，使用match_parent-->
<!--如果是水平布局，使用wrap_content -->
<android.support.v7.widget.RecyclerView
android:id="@+id/rv_meizhi"
android:layout_width="match_parent"
android:layout_height="match_parent"
<!--底部虚化效果的方向设置-->
android:requiresFadingEdge="vertical"
<!--底部虚化效果的高度设置-->
android:fadingEdgeLength="200dp" 
<!--底部阴影效果设置，never 为去掉-->
android:overScrollMode="always"
<!--滚动条设置，none 为去掉-->
android:scrollbars="vertical"
android:scrollbarThumbVertical="@drawable/my_bar"/>

```
还有一个点，对于滚动条是可以改变样式的，通过设置自定义的 drawable ,注意这个 自定义的 drawble 有点特别，不是普通的shape，而是  layer-list 开头的，主要是为了将滚动条到屏幕边设置一个像素的距离。如果不设置距离，直接使用普通的shape就行

```java

<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
<item android:right="1dp">
<shape>
<solid android:color="#999999"/>
<corners android:radius="1dp"/>
<stroke android:color="#111111"    android:width="1px"/>
<size android:width="2dp"/>
</shape>
</item>
</layer-list>

```

然后在 xml 属性中设置一下就好了，RecyclerView 布局常用的一些设置就是这些，下面看一下效果，实际中根据自己产品设计要求来变动。

![scroll_bar](http://pfk1gcktm.bkt.clouddn.com/RecyclerView_scrollbar.png)

### 3. setHasFixedSize 作用

相信一部分童鞋，遇到过这个方法吗，但是没有仔细分析它的作用，或者说是不知道什么时候应该使用这个方法，下面就来分析一下。
方法调用很简单，就是设置 true 或 false 
```java
mRecyclerView.setHasFixedSize(true);
```

看一下令人迷惑的官方介绍

```java

/**
* RecyclerView can perform several optimizations if it can know in advance that RecyclerView's
* size is not affected by the adapter contents. RecyclerView can still change its size based
* on other factors (e.g. its parent's size) but this size calculation cannot depend on the
* size of its children or contents of its adapter (except the number of items in the adapter).
* <p>
* If your use of RecyclerView falls into this category, set this to {@code true}. It will allow
* RecyclerView to avoid invalidating the whole layout when its adapter contents change.
*
* @param hasFixedSize true if adapter changes cannot affect the size of the RecyclerView.
*/
```

大概意思：如果我们能够提前知道 RecyclerView 的大小不受 Adapter 内容的影响时，RecyclerView 能够通过设置 setHasFixedSize 来进行优化。这里指的是不受 Adapter 内容的影响，如果是 RecyclerView 父布局大小的改变不算，也就是只看 Adapter 的内容或者它的 item 大小的改变，item数量的改变不算。如果你的 RecyclerView的使用属于这种情况，那么就可以通过设置 setHasFixedSize 来进行优化， 当 adapter 的内容变化时，RecyclerView 不需要重新绘制全部布局。

看完可能不是很明白，不知道怎么使用？

举个例子：当我使用 RecyclerView
时，adapter 的item的布局大小是一定的，与数量无关,这时每一屏上的数量是一定的，即使没有铺满屏幕也不要紧，照样可以使用 setHasFixedSize。练习中假设每张图片的大小是 50 * 50的，那么就满足条件。假设 ImageView 中的使用了 wrap_content,这时所有的图片大小不一定相同，就不符合条件了，所以不能使用 setHasFixedSize。

### 4. GridLayoutManager 的网格合并

使用 GridLayoutManager 时可能需要进行网格的合并，如想图中想要将第一行的网格合并成一个，该怎么做，方法比较简单，设置每一行所占的格子数目。

```java

// SpanSizeLookup
// A helper class to provide the number of spans each item occupies.
layoutManager.setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
@Override
public int getSpanSize(int position) {
int type = mRecyclerView.getAdapter().getItemViewType(position);
// 实际中最好根据 item 的类型来判断
if (position == 0 || position == 1) {
// 相当于 LinearLayout
return 2;
}

return 1;
}
});

```

![SpanSizeLookup](http://pfk1gcktm.bkt.clouddn.com/RecyclerView_grid_add.png)


[**Demo 地址**](https://github.com/RalfNick/AndroidPractice/tree/master/Common/RecyclerViewTest)

### 参考

[ListView/RecyclerView/ScrollView/View等的滚动条scrollbar设置](https://www.jianshu.com/p/52f9a758d1b0)


---
layout: post
title: "自定义 View - layout 过程详解"
date: 2019-04-13
description: "自定义 View - layout 过程详解"
tag: Android View
---
在上一篇文章 [**自定义 View - Measure 详解**](https://www.jianshu.com/p/ef922ca6055e) 中讲了 View 的 Measure 过程，还不熟悉的童鞋可以翻过去看看。View 的 3 个过程是按照顺序执行的：

**measure --> layout --> draw**

测量(measure)过程是确定这个 View 的大小，布局(layout)过程是确定 View 的位置。layout 过程相比于 measure 过程简单，因为它主要就是确定自己的位置，即 left、top、right、bottom 四个点的位置。布局过程的入口是 layout() 方法，该方法用于确定 View 自身的位置，它里面还调用了 onLayout 方法，这个方法主要用于确定子 View 的位置。对于单一 View 来说，就是就在 layout() 中确定自身的位置，onLayout 是一个空实现；对于 ViewGroup，在 layout() 中需要先确定自身的位置，然后需要执行 onLayout() 方法，确定子 View 的位置，继承自 ViewGroup 的父容器需要自己实现 onLayout() 方法，如果有子 View，遍历所有可见的子 View，执行单一 View 的布局过程。下面就针对这两种情况进行分析。

### 1. 单一 View layout 过程

单一 View 的 layout 过程就是确定自身的位置，那么开看下源码中是如何确定的。

```java
public void layout(int l, int t, int r, int b) {
...

// View 的 4 个顶点位置
int oldL = mLeft;
int oldT = mTop;
int oldB = mBottom;
int oldR = mRight;

// 判断当前 View 的位置是否改变，确定位置通过 setOpticalFrame 或 setFrame 来实现
// setOpticalFrame 实际上也是调用 setFrame
boolean changed = isLayoutModeOptical(mParent) ?
setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
// 视图的大小 或 位置发生变化
// 会重新确定该 View 所有子 View 在父容器的位置
// 单一View 的 onLayout() 是一个空实现
// 对于 ViewGroup 来说，onLayout() 是抽象方法，不同的父容器需要自己实现，以为不同的父容器特性不同
onLayout(changed, l, t, r, b);

if (shouldDrawRoundScrollbar()) {
if(mRoundScrollbarRenderer == null) {
mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
}
} else {
mRoundScrollbarRenderer = null;
}

...
}

...
}
```

```java

// 通过该方法设置 View 的四个顶点位置
protected boolean setFrame(int left, int top, int right, int bottom) {
boolean changed = false;

...

// Invalidate our old position
invalidate(sizeChanged);

mLeft = left;
mTop = top;
mRight = right;
mBottom = bottom;
mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
...
}

```

```java

// 当前 View 的位置是否改变
private boolean setOpticalFrame(int left, int top, int right, int bottom) {
Insets parentInsets = mParent instanceof View ?
((View) mParent).getOpticalInsets() : Insets.NONE;
Insets childInsets = getOpticalInsets();
// View 的位置发生改变，需要作调整，之后实际还是调用 setFrame() 方法
return setFrame(
left   + parentInsets.left - childInsets.left,
top    + parentInsets.top  - childInsets.top,
right  + parentInsets.left + childInsets.right,
bottom + parentInsets.top  + childInsets.bottom);
}

```

总结：对于 View，在 layout 方法中确定自身位置，通过left、top、right、bottom 四个参数确定，无需实现 onLayout 方法。

### 2. ViewGroup layout 过程

ViewGroup 的 layout 基本过程是相对比较简单，首先通过重写 layout 方法，确定自身的位置，这点与测量过程不同， 测量过程是先测量子 View，最后再测量自身。ViewGroup 自身位置确定之后，如果有子 View，再遍历所有可见性不为 GONE 的子 View，对每个子 View 进行 layout 过程，即单一 View 的 layout 过程。

那么为什么说是基本过程简单，因为ViewGroup 中 的 onLayout 方法是一个抽象方法，并未给出实现，实现是由继承自 ViewGroup 的其他 ViewGroup 自己来实现，它们内部的实现就相对复杂了，需要考虑各种细节，但是基本的 layout 过程还是 ViewGroup 的过程.

```java
@Override
public final void layout(int l, int t, int r, int b) {
if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
if (mTransition != null) {
mTransition.layoutChange(this);
}
// 调用 View layout 方法，确定自身位置
super.layout(l, t, r, b);
} else {
// record the fact that we noop'd it; request layout when transition finishes
mLayoutCalledWhileSuppressed = true;
}
}
```

```java
// 抽象方法，由子 ViewGroup 来实现
@Override
protected abstract void onLayout(boolean changed,
int l, int t, int r, int b);
```

下面以 LinearLayout 来看下 layout 的具体过程。

LinearLayout 继承自 ViewGroup，没有重写 layout 方法，布局时，直接调用上面 ViewGroup 的 layout 方法，确定自身位置。接下来看一下 onLayout 方法，遍历子 View，进行子 View 的 layout 过程。

```java

@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
// 我们知道 LinearLayout 有水平和垂直两个方向，所以布局时有两种情况，这里看下垂直的情况
if (mOrientation == VERTICAL) {
layoutVertical(l, t, r, b);
} else {
layoutHorizontal(l, t, r, b);
}
}

// 在垂直方向布局
void layoutVertical(int left, int top, int right, int bottom) {
final int paddingLeft = mPaddingLeft;

int childTop;
int childLeft;

// Where right end of child should go
final int width = right - left;
int childRight = width - mPaddingRight;

// Space available for child
int childSpace = width - paddingLeft - mPaddingRight;

final int count = getVirtualChildCount();

final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;

// 根据 gravity 确定 childTop
switch (majorGravity) {
case Gravity.BOTTOM:
// mTotalLength contains the padding already
childTop = mPaddingTop + bottom - top - mTotalLength;
break;

// mTotalLength contains the padding already
case Gravity.CENTER_VERTICAL:
childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
break;

case Gravity.TOP:
default:
childTop = mPaddingTop;
break;
}

for (int i = 0; i < count; i++) {
final View child = getVirtualChildAt(i);
if (child == null) {
childTop += measureNullChild(i);
} else if (child.getVisibility() != GONE) {

// 测量的宽和高
final int childWidth = child.getMeasuredWidth();
final int childHeight = child.getMeasuredHeight();

final LinearLayout.LayoutParams lp =
(LinearLayout.LayoutParams) child.getLayoutParams();

int gravity = lp.gravity;
if (gravity < 0) {
gravity = minorGravity;
}
final int layoutDirection = getLayoutDirection();
final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);

// 根据 gravity 确定 childLeft
switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
case Gravity.CENTER_HORIZONTAL:
childLeft = paddingLeft + ((childSpace - childWidth) / 2)
+ lp.leftMargin - lp.rightMargin;
break;

case Gravity.RIGHT:
childLeft = childRight - childWidth - lp.rightMargin;
break;

case Gravity.LEFT:
default:
childLeft = paddingLeft + lp.leftMargin;
break;
}

if (hasDividerBeforeChildAt(i)) {
childTop += mDividerHeight;
}

childTop += lp.topMargin;

// 以上部分实际上就是确定 childLeft, childTop 这两个位置点，
// childRight = childLeft + childWidth;
// childBottom = childTop + getLocationOffset(child) + childHeight;

setChildFrame(child, childLeft, childTop + getLocationOffset(child),
childWidth, childHeight);
childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

i += getChildrenSkipCount(child, i);
}
}
}

// setChildFrame 中调用的 layout，这样就能够完成对子 View 的布局过程。
private void setChildFrame(View child, int left, int top, int width, int height) {
child.layout(left, top, left + width, top + height);
}
```

### 3. getWidth() 和 getMeasuredWidth() 区别

getMeasuredWidth() 得到的是测量过程的宽度，在测量过程通过 setMeasuredDimension() 方法，保存了 View 的宽和高，getMeasuredWidth() 得到的就是这个保存的宽度尺寸。

getWidth() 是在 layout 过程完成后得到的一个宽度，在上面 LinearLayout 布局过程中 我们知道，childRight = childLeft + childWidth; 而 childWidth = child.getMeasuredWidth()，那么通过 getWidth() 方法得到的宽度实际上就是测量的宽度，即等于 getMeasuredWidth()。但是在上一篇 Measure 过程分析时，也提到了这两个方法得到尺寸不一致，可能是由于子 View 在 layout 过程中改变了尺寸，但是这种做法很少，也没有实际意义，一般情况下，我们可以认为 getWidth() 和 getMeasuredWidth() 得到的尺寸是一致的。同理，对于 getHeight() 和 getMeasuredHeight() 也是一样的。

```java
public final int getWidth() {
return mRight - mLeft;
}
```

好了，layout 过程分析就完了，相比于 Measure 过程简单多了，建议可以再看看其他继承 ViewGroup 的布局过程，以便有助我们在自定义 View 的，能够更好的完成 layout 的重写过程。


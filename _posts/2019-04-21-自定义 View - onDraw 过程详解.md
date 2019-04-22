---
layout: post
title: "自定义 View - onDraw 过程详解"
date: 2019-04-21
description: "自定义 View - onDraw 过程详解"
tag: Android View
---
之前两篇文章分析了 [**onMeasure**](https://www.jianshu.com/p/ef922ca6055e) 过程和 [**onLayout**](https://www.jianshu.com/p/af7c6cb6939c) 过程，不熟悉的童鞋可以回头去复习下，本篇文章来分析绘制过程的最后一个 onDraw 过程。这个过程的绘制使用到的 Paint 和 Canvas 在之前也有讲解到，在本篇的练习代码中有使用到，不会具体讲解这些知识点，不熟悉的话可以看看我之前的文章

[自定义 View - Paint 详解](https://www.jianshu.com/p/b4aa78c96f66)

[自定义 View - Canvas 详解](https://www.jianshu.com/p/133921a24011)

### View 绘制过程

绘制过程也不复杂，在 View draw 方法的源码中注释写的也很详细，先给出一张图。

图中的绘制过程也是代码中给出的绘制过程，对于 单一 View 和 ViewGroup 来说，差别仅仅在于绘制子 View 上， 因为单一 View 没有子 View，所以不需要绘制子 View，即 dispatchDraw(canvas) 是一个空实现；那对于 ViewGroup 来说，如果有子 View 的话，就需要实现该方法，绘制子 View，过程循环遍历子 View，然后调用 子 View 的 draw 方法，这样就回到了子 View 的绘制过程，具体分析看后面。

![process](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/draw_process.png)

#### 1.单一 View 绘制

单一 View 的绘制过程就是上面图中的流程，下面主要看下代码中的注释。

```java

public void draw(Canvas canvas) {
final int privateFlags = mPrivateFlags;
final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
(mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

/*
* Draw traversal performs several drawing steps which must be executed
* in the appropriate order:
*
*      1. Draw the background
*      2. If necessary, save the canvas' layers to prepare for fading
*      3. Draw view's content
*      4. Draw children
*      5. If necessary, draw the fading edges and restore layers
*      6. Draw decorations (scrollbars for instance)
*/
上面指出绘制过程的 6 个步骤，其中 2 和 5 是对图层的操作，不是必要的
1、绘制背景，该方法发生在一个叫 drawBackground() 的方法里，但这个方法是 private 的，不能重写，如果要设置背景，只能用自带的 API 去设置（xml 布局文件的 android:background 属性以及 Java 代码的 View.setBackgroundXxx() 方法

3、绘制主体部分，例如，对于 TextView，绘制文字部分

4、绘制子 View，单一 View 没有子 View，空实现，对于继承 ViewGroup 的 Layout，如果有子 View 需要实现该方法。

6、绘制滑动边缘渐变和滑动条、以及前景，这些东西是在一个方法中绘制的，不能分开

int saveCount;

// Step 1：绘制背景
if (!dirtyOpaque) {
drawBackground(canvas);
}

// skip step 2 & 5 if possible (common case)
final int viewFlags = mViewFlags;
boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
if (!verticalEdges && !horizontalEdges) {
// Step 3：绘制主体部分
if (!dirtyOpaque) onDraw(canvas);

// Step 4, 绘制子 View
dispatchDraw(canvas);

drawAutofilledHighlight(canvas);

// Overlay is part of the content and draws beneath Foreground
if (mOverlay != null && !mOverlay.isEmpty()) {
mOverlay.getOverlayView().dispatchDraw(canvas);
}

// Step 6, 绘制装饰部件，滚动条，前景等
onDrawForeground(canvas);

// Step 7, 具有焦点时，绘制焦点高亮
drawDefaultFocusHighlight(canvas);

if (debugDraw()) {
debugDrawFocus(canvas);
}

// 其实到这里已经结束了
return;
}

// 但是源码中后面还有一大堆代码，这些代码一般很少能够执行到，
// 可能是用来测试，后面这部分有兴趣的自己看看吧
/*
* Here we do the full fledged routine...
* (this is an uncommon case where speed matters less,
* this is why we repeat some of the tests that have been
* done above)
*/
...
}
```

上面就是 draw 总调度函数绘制的过程，下面看下每个过程的具体函数。

**绘制背景**

背景绘制过程采用 Drawable 的 draw 方法，注意这个方法是 private，也就是说背景绘制我们不能重写，如果要设置背景，只能用自带的 API 去设置（xml 布局文件的 android:background 属性以及 Java 代码的 View.setBackgroundXxx() 方法

```java
private void drawBackground(Canvas canvas) {
final Drawable background = mBackground;
if (background == null) {
return;
}
// 设定背景的范围
setBackgroundBounds();

// Attempt to use a display list if requested.
if (canvas.isHardwareAccelerated() && mAttachInfo != null
&& mAttachInfo.mThreadedRenderer != null) {
mBackgroundRenderNode = getDrawableRenderNode(background, mBackgroundRenderNode);

final RenderNode renderNode = mBackgroundRenderNode;
if (renderNode != null && renderNode.isValid()) {
setBackgroundRenderNodeProperties(renderNode);
((DisplayListCanvas) canvas).drawRenderNode(renderNode);
return;
}
}

// background 是一个 Drawable，背景绘制过程采用 Drawable 的 draw 方法
final int scrollX = mScrollX;
final int scrollY = mScrollY;
if ((scrollX | scrollY) == 0) {
background.draw(canvas);
} else {
canvas.translate(scrollX, scrollY);
background.draw(canvas);
canvas.translate(-scrollX, -scrollY);
}
}
```

**onDraw**

```java
// 绘制 View 本身的内容
// 继承自 View 的单一 View 实现各不相同，需要自己实现，如 TextView 绘制文字
protected void onDraw(Canvas canvas) {

}
```

**dispatchDraw**

```java

/**
* dispatchDraw 用来绘制子 View，
* 因为单一 View 没有子 View，所以不需要绘制子 View，即 dispatchDraw(canvas) 是一个空实现；那对于
* ViewGroup 来说，如果有子 View 的话，就需要实现该方法，绘制子 View，过程循环遍历子 View
*/
protected void dispatchDraw(Canvas canvas) {

... // 空实现

}
```

**onDrawForeground**

绘制前景，这个过程也会绘制滚动条和指示器等小装饰，然后绘制前景，前景绘制过程和背景过程类似，也是通过 Drawable 的 draw 方法绘制的，装饰和前景一起绘制，该方法不能拆分。

```java
public void onDrawForeground(Canvas canvas) {
// 绘制指示器和滚动条
onDrawScrollIndicators(canvas);
onDrawScrollBars(canvas);

final Drawable foreground = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
if (foreground != null) {
if (mForegroundInfo.mBoundsChanged) {
mForegroundInfo.mBoundsChanged = false;
final Rect selfBounds = mForegroundInfo.mSelfBounds;
final Rect overlayBounds = mForegroundInfo.mOverlayBounds;

if (mForegroundInfo.mInsidePadding) {
selfBounds.set(0, 0, getWidth(), getHeight());
} else {
selfBounds.set(getPaddingLeft(), getPaddingTop(),
getWidth() - getPaddingRight(), getHeight() - getPaddingBottom());
}

final int ld = getLayoutDirection();
Gravity.apply(mForegroundInfo.mGravity, foreground.getIntrinsicWidth(),
foreground.getIntrinsicHeight(), selfBounds, overlayBounds, ld);
foreground.setBounds(overlayBounds);
}
// 绘制前景，前景和背景绘制过程基本一致
foreground.draw(canvas);
}
}
```

#### 2. View Group 绘制过程

由于 ViewGroup 继承 View，所以上面几个过程基本是一致的，除了 dispatchDraw 方法，一般情况下 ViewGroup 是有子 View 的，所以通过 dispatchDraw 方法来进行子 View 的绘制，下面主要分析一下 dispatchDraw 方法。

```java
@Override
protected void dispatchDraw(Canvas canvas) {
final int childrenCount = mChildrenCount;
final View[] children = mChildren;
...

for (int i = 0; i < childrenCount; i++) {
...

final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
// 调用子 View 的绘制方法
more |= drawChild(canvas, child, drawingTime);
}
}
}

// 调用子 View 的绘制方法
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
return child.draw(canvas, this, drawingTime);
}
```
dispatchDraw 实际上就是对子 View 的遍历，然后依次调用子 View 的绘制方法，就按照上面的 View 的绘制方法执行，当然也有可能还是 ViewGroup，那么还会继续调用 dispatchDraw 方法，遍历子 View。

### 绘制方法重写顺序分析

上面过程分析了 View 的绘制过程，那么我们进行自定义 View 时，会有这样一个问题，有时我们会只用 super 调用 父类方法，我们自己重写的部分代码，是放在上面，还是下面呢，有何影响？下面就来对上面的方法进行分析，写在 super 上面和下面有何影响。

**(1) super.onDraw() 前**

```java
public class MyView extends View {  
...
protected void onDraw(Canvas canvas) {
super.onDraw(canvas);
... // 自定义绘制代码
}
...
}
```

写在 onDraw 的下面，自定义的绘制部分会盖住控件原有的内容，如在 ImageView 的图片上显示图片的尺寸信息等。

![before_draw](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/before_draw.jpg)

**(2) super.onDraw() 后**

写在 onDraw 的下面，自定义的绘制部分被控件原有的内容盖住，如在文字下面绘制强调色。

![after_draw](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/after_draw.jpg)

**(3) ViewGroup 子类的 onDraw() 中****

一个继承自 ViewGroup 的子类，本身在 onDraw 是一个空实现，在 onDraw 中进行绘制，如果没有子 View，那么会显示出绘制的内容，如果有子 View，子 View 会遮住 onDraw 中绘制的内容。

![ondraw_layout](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/ondraw_layout.jpg)

**摘自 Hencoder**

另外有一点，出于效率的考虑，ViewGroup 默认会绕过 draw() 方法，换而直接执行 dispatchDraw()，以此来简化绘制流程。所以如果自定义了某个 ViewGroup 的子类（比如 LinearLayout）并且需要在它的除  dispatchDraw() 以外的任何一个绘制方法内绘制内容，可能会需要调用  View.setWillNotDraw(false) 这行代码来切换到完整的绘制流程（是「可能」而不是「必须」的原因是，有些 ViewGroup 是已经调用过 setWillNotDraw(false) 了的，例如 ScrollView）

**(4) 在 dispatchDraw 下面**

写在 dispatchDraw 下面，会遮住子 View 部分

![dispatchDraw](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/dispatch_draw.jpg)

**(5) 在onDrawForeground 后**

写在 在onDrawForeground 后面会会遮住滑动边缘渐变、滑动条和前景

![after_foreground](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/after_foreground.jpg)

**(6) 在onDrawForeground 前**

写在 在onDrawForeground 前面会被前景部分遮住

![before_foreground](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/before_foreground.jpg)

最后给出一张 Hencoder 大神总结的一张图

![position](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Draw/draw_position.png)

### 参考

[HenCoder Android 开发进阶：自定义 View 1-5 绘制顺序](https://hencoder.com/ui-1-5/)


[**练习代码地址**](https://github.com/RalfNick/AndroidPractice/tree/master/CustomView_onDraw)

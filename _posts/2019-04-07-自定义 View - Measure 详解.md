---
layout: post
title: "自定义 View - Measure 详解"
date: 2019-04-07
description: "自定义 View - Measure 详解"
tag: Android View
---
![content](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Measure/measure_content.png)

上图就是 View 绘制的主要过程，View 的绘制是从顶层的 DecoraView -- ViewGroup（可能多个 ViewGroup）再到 View，按照这个流程从上往下，依次measure(测量),layout(布局),draw(绘制)。其中 Measure 过程是相对复杂的一个，但是其实我们梳理出来需要掌握测量过程的知识点，就很清楚了，下面就来一起看看 Measure 过程。

Measure 过程实际上就是确定这个 View 或者 ViewGroup 的宽高，确定了宽高，然后进行布局，最后进行绘制。这个做个比喻，我们有一只张纸（类似于手机屏幕），要在这张纸上进行画图，首先要做的是选定范围（对应测量过程），选好范围后需要确定想要画的图形在所选定范围的位置，如画中太阳放在哪个位置，树木、草地放在哪个位置等。确定位置后才开始画图，这样就完成了一幅图的绘制过程。 View 的绘制过程就可以想象为这个画画的过程。

那么对于 Measure 过程，我们需要掌握的点，我在下面的脑图中给出了，掌握这几个点，对于测量的过程基本就够了。

![progress](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Measure/measure_process.png)

### 1. MeasureSpec

MeasureSpec 翻译为：规格。它用来确定一个 View 的尺寸规格，当然一个 View 的尺寸同时也受父 View 的影响，由两者共同来决定子 View 的宽高。当然，除顶层的 DecorView 之外，因为 DecorView 没有父容器，所以 DecorView 由窗口的尺寸和其自身的 LayoutParams 共同决定 MeasureSpec，MeasureSpec 决定以后，在 onMeasure 中就可以确定 View 的宽和高。

这里要说明一下 View 的 LayoutParams，子 View 可以通过 getLayoutParams() 获得这个参数，所有的 View 都会有 LayoutParams，父 View 会通过这个参数来对子 View 进行布局设置。LayoutParams 有 3 种常量，FILL_PARENT、MATCH_PARENT 和 WRAP_CONTENT

```
FILL_PARENT     使子视图的大小扩展至与父视图大小相等（不含 padding )
match_parent    与fill_parent相同，用于Android 2.3 & 之后版本
wrap_content    自适应大小，使视图扩展以便显示其全部内容(含 padding )
```
我们在 XML 布局中使用这几个来设置 View 的宽和高，或者指定具体的数值，下测量过程中，系统会会转到 View 的 LayoutParams 中的 width 和 height。举个例子，如果在 XML 中设置宽为 match_parent,高设置为 wrap_content,那么对应 这个 View 的 LayoutParams 中的 为 width 为 match_parent, height 为 wrap_content。有了 LayoutParams，还需要配合父容器的 MeasureSpec 共同确定子 View 的 MeasureSpec。

MeasureSpec 包含两部分，SpecMode 和 SpecSize，将这两个打包在一起实际是为了减少在内存中的内对，提高效率，MeasureSpec 是一个 int 值，低 30 位是 SpecSize，高 2 位是 SpecMode。SpecMode 有以下 3 种：

| 模式          | 二进制数值 | 描述                                     |
| ----------- | :---: | -------------------------------------- |
| UNSPECIFIED |  00   | 默认值，父控件没有给子view任何限制，子View可以设置为任意大小。    |
| EXACTLY     |  01   | 表示父控件已经确切的指定了子View的大小。                 |
| AT_MOST     |  10   | 表示子View具体大小没有尺寸限制，但是存在上限，上限一般为父View大小。 |

MeasureSpec 的操作实际上就是二进制的操作，可以看看注释部分，写的很详细。

```java

public static class MeasureSpec {

// 移位数
private static final int MODE_SHIFT = 30;
// 遮罩位，二进制 11 左移 30 位，
// MeasureSpec 通过与 MODE_MASK 做 & 运算能够获取 SpecMode
// MeasureSpec 通过与 ~MODE_MASK 做 & 运算能够获取 SpecSize
private static final int MODE_MASK  = 0x3 << MODE_SHIFT;

// UNSPECIFIED 的数值
public static final int UNSPECIFIED = 0 << MODE_SHIFT;

// EXACTLY 的数值
public static final int EXACTLY     = 1 << MODE_SHIFT;

// AT_MOST 的数值
public static final int AT_MOST     = 2 << MODE_SHIFT;

// 将 SpecMode 和 SpecSize 合成 MeasureSpec
public static int makeMeasureSpec(int size, int mode) {
if (sUseBrokenMakeMeasureSpec) {
return size + mode;
} else {
return (size & ~MODE_MASK) | (mode & MODE_MASK);
}
}

// 获取 SpecMode， MeasureSpec 通过与 MODE_MASK 做 & 运算
public static int getMode(int measureSpec) {
//noinspection ResourceType
return (measureSpec & MODE_MASK);
}

// MeasureSpec 通过与 ~MODE_MASK 做 & 运算
public static int getSize(int measureSpec) {
return (measureSpec & ~MODE_MASK);
}
}
```

那么 View 的 MeasureSpec 是如何确定的呢？这里不再分析 DecorView 的 MeasureSpec 确定过程，主要看一下普通 View 的 MeasureSpec 确定过程。先看下 ViewGroup 的 measureChildWithMargins 方法。

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
从这个方法中可以看出：在测量子 View 之前，会先获取子 View 的 MeasureSpec，它与子 View 的 LayoutParams 和 父容器的 MeasureSpec有关，同时，还与子 View 的 margin 及 padding 有关。下面再看下 getChildMeasureSpec 方法。

```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
// 获取父容器的 specMode 和 specSize
int specMode = MeasureSpec.getMode(spec);
int specSize = MeasureSpec.getSize(spec);

// 如果子 View 有 padding，需要减去 padding 的大小
int size = Math.max(0, specSize - padding);

// 子 View 的 size 和 mode 变量定义，也作为结果
int resultSize = 0;
int resultMode = 0;

// 需要看父容器的测量模式
switch (specMode) {
// (1) 父容器是精准测量模式
case MeasureSpec.EXACTLY:

// 子 View 设定了具体的尺寸，即我们在 xml 布局中指定了具体的 dp，子 View 的尺寸就是childDimension，测量模式为 EXACTLY
if (childDimension >= 0) {
resultSize = childDimension;
resultMode = MeasureSpec.EXACTLY;
} else if (childDimension == LayoutParams.MATCH_PARENT) {
// 子 View 在 xml 中设定的是 match_parent,这时子 View 的大小就是父容器大小，即 size，测量模式为 EXACTLY
resultSize = size;
resultMode = MeasureSpec.EXACTLY;
} else if (childDimension == LayoutParams.WRAP_CONTENT) {
// 子 View 在 xml 中设定的是 wrap_content,这时子 View 的大小不能超过父容器大小，即 size，测量模式为 AT_MOST
resultSize = size;
resultMode = MeasureSpec.AT_MOST;
}
break;

// (2) 父容器是 AT_MOST
case MeasureSpec.AT_MOST:
if (childDimension >= 0) {
// 子 View 设定了具体的尺寸，即我们在 xml 布局中指定了具体的 dp，子 View 的尺寸就是childDimension，测量模式为 EXACTLY
resultSize = childDimension;
resultMode = MeasureSpec.EXACTLY;
} else if (childDimension == LayoutParams.MATCH_PARENT) {
// 子 View 在 xml 中设定的是 match_parent,这时子 View 的大小不能超过父容器大小，即 size，测量模式为 AT_MOST
resultSize = size;
resultMode = MeasureSpec.AT_MOST;
} else if (childDimension == LayoutParams.WRAP_CONTENT) {
// 子 View 在 xml 中设定的是 wrap_content,这时子 View 的大小不能超过父容器大小，即 size，测量模式为 AT_MOST,可以看到父容器是 AT_MOST，子 View 如果不指定具体的数值，那么测量模式均为 AT_MOST，且大小为不超过父容器剩余空间的大小。
resultSize = size;
resultMode = MeasureSpec.AT_MOST;
}
break;

// (3) 父容器是 UNSPECIFIED 模式
case MeasureSpec.UNSPECIFIED:
if (childDimension >= 0) {
// 子 View 设定了具体的尺寸，即我们在 xml 布局中指定了具体的 dp，子 View 的尺寸就是childDimension，测量模式为 EXACTLY
resultSize = childDimension;
resultMode = MeasureSpec.EXACTLY;
} else if (childDimension == LayoutParams.MATCH_PARENT) {
// 父容器是 UNSPECIFIED 模式，子 View 一般为 0 
resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
resultMode = MeasureSpec.UNSPECIFIED;
} else if (childDimension == LayoutParams.WRAP_CONTENT) {
// 父容器是 UNSPECIFIED 模式，子 View 一般为 0 
resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
resultMode = MeasureSpec.UNSPECIFIED;
}
break;
}
return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}

```

以上就是通过父容器来获取子 View 的测量规格，分类看似很多，实际上也是有规律的，下面给出这个总结规律：

![spec](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Measure/measure_spec.png)

**总结起来主要就是 3 条，对于 UNSPECIFIED，不需要考虑：**

```
(1) 当子 View 是具体的数值时，无论父 View 是哪种测量模式，子 View 测量模式都是 EXACTLY，大小是子 View 设定的大小。

(2) 当父容器是 EXACTLY 模式，子 View 的 LayoutParams 的布局格式是 match_parent，子 View 测量模式是 EXACTLY，大小是父容器剩余大小。

(3) 当子 View 的 LayoutParams 的布局格式是 wrap_content，子 View 测量模式都是 AT_MOST，大小是父容器剩余大小。

```

### 2. View 的测量过程

View 的测量过程有两种情况：

```
(1) 单一 View 的测量过程，这个过程相对简单，通过 onMeasure 方法就完成了测量过程。

(2) ViewGroup 测量过程，ViewGroup 除了要完成自己的测量之外，还需要遍历完成各个子 View 的测量，这样才算测量过程。
```

#### 2.1 单一 View 的测量过程

单一 View 的测量过程流程图：

![view](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Measure/measure_view.png)

View 测量过程的入口函数，该方法是 final 的，不允许被重写，其内部主要对 widthMeasureSpec 和 heightMeasureSpec，然后调用 onMeasure 方法完成测量过程。

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

...

if (forceLayout || needsLayout) {
...
int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
if (cacheIndex < 0 || sIgnoreMeasureCache) {

onMeasure(widthMeasureSpec, heightMeasureSpec);

} else {
long value = mMeasureCache.valueAt(cacheIndex);
// Casting a long to int drops the high 32 bits, no mask needed
setMeasuredDimensionRaw((int) (value >> 32), (int) value);
mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
}
...
}
...
}
```

这个是我们熟知的测量过程的方法，方法内部先通过 getDefaultSize 获取宽高，然后再通过 setMeasuredDimension 将宽高信息保存到成员变量中。

```java

protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

```

传入的参数 measureSpec，对于普通 View 来说，是通过父容器和 View 自身的 LayoutParams得到的规格，有了这个规格，对于 AT_MOST 和 EXACTLY，宽高就是规格中的尺寸。对于 UNSPECIFIED 模式下，尺寸来源于 getSuggestedMinimumWidth() 这个方法，这个方法就不仔细分析了，它给出的尺寸首先看 View 是否设置了背景，如果背景图是有尺寸的（自定义的 Shape木有尺寸，Bitmap 有尺寸），那么就取背景的宽高，背景图没尺寸，就取 mMinWidth 和 mMinHeight 的尺寸，这两个是在 xml 属性中设置的，如果没有指定，那就默认为 0 了。

```java

public static int getDefaultSize(int size, int measureSpec) {
int result = size;
int specMode = MeasureSpec.getMode(measureSpec);
int specSize = MeasureSpec.getSize(measureSpec);

switch (specMode) {
case MeasureSpec.UNSPECIFIED:
result = size;
break;
case MeasureSpec.AT_MOST:
case MeasureSpec.EXACTLY:
result = specSize;
break;
}
return result;
}

```

将测量的宽高保存到成员变量中

```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {
boolean optical = isLayoutModeOptical(this);
if (optical != isLayoutModeOptical(mParent)) {
Insets insets = getOpticalInsets();
int opticalWidth  = insets.left + insets.right;
int opticalHeight = insets.top  + insets.bottom;

measuredWidth  += optical ? opticalWidth  : -opticalWidth;
measuredHeight += optical ? opticalHeight : -opticalHeight;
}
setMeasuredDimensionRaw(measuredWidth, measuredHeight);
}

private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
mMeasuredWidth = measuredWidth;
mMeasuredHeight = measuredHeight;

mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
}

```

通过上述分析，我们能够知道 View 的尺寸由 getDefaultSize 中的 measureSpec 决定，从上面的总结的表中可以看出，当我们自定义 View 时，如果设置的宽或高的尺寸是 wrap_content 时，最后的效果是和 match_parent 一样，都是父容器剩余的空间大小，而对于 wrap_content，实际上我们想要刚好包裹住 View 内部的内容，不是父容器剩余空间那么大，那么需要处理呢？做法也很简单，判断 measureSpec 的规格中模式，在 AT_MOST 时，给出一个默认的具体尺寸就可以了。

```java


// 设置 wrap_content 的默认宽 / 高值
// 可根据需要自己调整具体的数值
int mWidth = 200;
int mHeight =200;

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
super.onMeasure(widthMeasureSpec, heightMeasureSpec);

// 获取宽-测量规则的模式和大小
int widthMode = MeasureSpec.getMode(widthMeasureSpec);
int widthSize = MeasureSpec.getSize(widthMeasureSpec);
// 获取高-测量规则的模式和大小
int heightMode = MeasureSpec.getMode(heightMeasureSpec);
int heightSize = MeasureSpec.getSize(heightMeasureSpec);

// 当布局参数设置为 wrap_content 时，设置默认值
if (getLayoutParams().width == ViewGroup.LayoutParams.WRAP_CONTENT && getLayoutParams().height == ViewGroup.LayoutParams.WRAP_CONTENT) {
setMeasuredDimension(mWidth, mHeight);
// 其中一个布局参数为 wrap_content 时，设置默认值
} else if (getLayoutParams().width == ViewGroup.LayoutParams.WRAP_CONTENT) {
setMeasuredDimension(mWidth, heightSize);
} else if (getLayoutParams().height == ViewGroup.LayoutParams.WRAP_CONTENT) {
setMeasuredDimension(widthSize, mHeight);
}
}
```

#### 2.2 ViewGroup 测量过程

ViewGroup测量过程流程图：

![ViewGroup](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Measure/measure_viewgroup.png)

流程图中已经很清晰的展示了 ViewGroup 的测量过程，对于 ViewGroup 来说，没有重写 onMeasure 方法，只是给出了 measureChildren 方法，如果子 View 仍旧是 ViewGroup，重复这个这个过程；如果是单一的 View，那么直接走上面的子 View 的测量过程。ViewGroup 之所以没有重写 onMeasure 方法，是因为 ViewGroup 有很多种，LinearLayout，RelativeLayout等，各个 ViewGroup 的测量规则一致，所以无法给出统一的重写方法，所以就需要我们自己重写（当我们自定义的 View 是继承自 ViewGroup 时），注意既然需要重写 onMeasure 方法，那么最后还要测量 ViewGroup 自身，这样才能够得到 ViewGroup 完整的尺寸测量信息。

```java
// 遍历所有子 View，对所有子 View 进行测量
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
final int size = mChildrenCount;
final View[] children = mChildren;
for (int i = 0; i < size; ++i) {
final View child = children[i];
if ((child.mViewFlags & VISIBILITY_MASK) != GONE) {
measureChild(child, widthMeasureSpec, heightMeasureSpec);
}
}
}

// 每个子 View 的具体测量过程，最终调用的是子 View 的 measure 方法
protected void measureChild(View child, int parentWidthMeasureSpec,
int parentHeightMeasureSpec) {
final LayoutParams lp = child.getLayoutParams();

// 获取子 View 的 LayoutParams,并结合父容器的 MeasureSpec，来得到子 View 的 MeasureSpec
final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
mPaddingLeft + mPaddingRight, lp.width);
final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
mPaddingTop + mPaddingBottom, lp.height);

// 最终调用的是子 View 的 measure 方法
child.measure(childWidthMeasureSpec, childHeightMeasureSpec);

}
```

### 3. LinearLayout 测量示例

```java

@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
if (mOrientation == VERTICAL) {
measureVertical(widthMeasureSpec, heightMeasureSpec);
} else {
measureHorizontal(widthMeasureSpec, heightMeasureSpec);
}
}

// 以垂直方向上为例进行主要部分的分析
void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
mTotalLength = 0;
int maxWidth = 0;
int childState = 0;
int alternativeMaxWidth = 0;
int weightedMaxWidth = 0;
boolean allFillParent = true;
float totalWeight = 0;

final int count = getVirtualChildCount();

final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

...

// 对每个子 View 进行测量
for (int i = 0; i < count; ++i) {
final View child = getVirtualChildAt(i);

// 子 View 为空，measureNullChild 实际也是 0
if (child == null) {
mTotalLength += measureNullChild(i);
continue;
}
// 当子 View 不显示时，不进行测量
if (child.getVisibility() == View.GONE) {
i += getChildrenSkipCount(child, i);
continue;
}

...

final int usedHeight = totalWeight == 0 ? mTotalLength : 0;
// 测量子 View
measureChildBeforeLayout(child, i, widthMeasureSpec, 0,
heightMeasureSpec, usedHeight);

final int childHeight = child.getMeasuredHeight();

// 在垂直方向上相加，同时需要考虑子 View 上下方向的 margin 以及 两个子 View 的间隔
final int totalLength = mTotalLength;
mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
lp.bottomMargin + getNextLocationOffset(child));

}

...

// 需要 ViewGroup 自身的 Padding 值
maxWidth += mPaddingLeft + mPaddingRight;

// Check against our minimum width
maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
// 保存测量值
setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
heightSizeAndState);
}
```

### 4. 获取测量尺寸

上面讲了 View 和 ViewGroup 的测量过程，通过这个过程之后就得到一个测量尺寸，实际上，在进行 onLayout 之后也会得到一个宽高，这个是最终 View 的宽和高，所以好的习惯就是在 onLayout 中获取宽和高。一般情况下，onMeasure 中的得到的宽和高和 OnLayout 中的宽和高是相等的，但是也有特殊情况，比如在 onLayout 中修改了宽和高，那么这个时候就不相等了。

如何获取一个 View 的宽和高呢?这里分为两种情况：

（1）在 Activity 的生命周期中获取 View 的宽和高

（2）在 View 的 onDraw 方法中获取。

#### 4.1 在 Activity 的生命周期中获取 View 的宽和高

这里先给出结论，在 onCreat、onStart、onResume 中均不能够保证能够获取到 View 的宽和高，为什么呢？因为 View 的 measure 过程和 Activity 的生命周期方法不是同步的，所以无法保证在 onCreat、onStart、onResume 这些生命周期的方法中，测量过程能够完成，所以不能在这些方法中获取 View 的宽和高。这里给出 2 种常用的方法。

（1）Activity 或 View 的 onWindowFocusChanged 中获取

在 onWindowFocusChanged 方法中可以获取到 View 的宽和高，因为这时候 View 已经初始化完毕。onWindowFocusChanged 方法是指窗口焦点改变时回调的方法，如 Activity 的 onPause 方法和 onResume 方法执行时，onWindowFocusChanged 方法均会被调用。

```java
@Override
public void onWindowFocusChanged(boolean hasWindowFocus) {
super.onWindowFocusChanged(hasWindowFocus);
if(hasWindowFocus){
// view 可以通过 findViewById 的方式
int width = view.getMeasuredWidth();
int height = view.getMeasuredHight();
}
}
```

（2）通过 view.post(runnable) 方式获取

通过 view.post(runnable) 方式，将获取宽高的时间放到 主线程 Looper 的循环消息队列对尾，当执行到这个 Runnable 时，view 已经初始化好了，所以能够获取到 view 的宽和高。

```java
protected void onStart(){
super.onStart();
view.post(new Runnable() {
@Override
public void run() {
// view 可以通过 findViewById 的方式
int width = view.getMeasuredWidth();
int height = view.getMeasuredHight();
}
});
}
```


#### 4.2 在 View 的 onDraw 方法中获取

这种情况下就比较简单了，因为 onMeasure 过程一定在 onDraw 的前面，所以直接获取 view 的宽和高即可。

```java
@Override
public void draw(Canvas canvas) {
super.draw(canvas);
int width = view.getMeasuredWidth();
int height = view.getMeasuredHight();  
}
```

另外还有一种方法：

因为View的大小不仅由 View 本身控制，而且受父控件的影响，所以我们在确定 View 大小的时候最好使用系统提供的 onSizeChanged 回调函数。

onSizeChanged 如下：

```java

@Override
protected void onSizeChanged(int w, int h, int oldw, int oldh) {
super.onSizeChanged(w, h, oldw, oldh);
mWidth = w;
nHeight = h;
}
```
它有四个参数，分别为 宽度，高度，上一次宽度，上一次高度。

### 参考

**《安卓开发艺术探索》**

[图解View测量、布局及绘制原理](https://www.jianshu.com/p/3d2c49315d68)

[自定义View Measure过程 - 最易懂的自定义View原理系列（2）](https://www.jianshu.com/p/1dab927b2f36)

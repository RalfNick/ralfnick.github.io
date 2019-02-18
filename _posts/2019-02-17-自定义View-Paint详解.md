---
layout: post
title: "自定义View-Paint详解"
date: 2019-02-17
description: "Android 自定义 View"
tag: Android View
---

[**上一篇**](https://www.jianshu.com/p/007a791cd474) 文章对自定义单一 View 进行了初步的介绍，对绘制的流程有了一个概念，本篇将对 Paint 部分进行详细 的介绍。

![Custom_View_Paint](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/Custom_View_Paint.jpg)

上一篇中给出一张图，自定义 View 部分设计的知识点，对于 Paint 部分，也是很大的一块，Paint 的 API 很多，有各种样式，各种效果，颜色设置，抗锯齿，文字设置等。绘制的过程可以想象为画画，有画布和画笔。画布就是 Canvas，画笔是 Paint。

```java

canvas.drawCircle(300, 300, 200, paint);

```
像这样一个过程，就是用这两样东西就完成了。可能感觉有些别扭，不是应该是 paint.DrawXX 么，这个我没去深究。为了便于记忆，个人觉得可以这样理解，所有的动作时在画布上执行的，那么就需要准备各种材料（画笔，位置等），执行的动作就是画，即函数名。

下面就对 Paint 的基本用法和各种样式、效果进行介绍。

### Paint 基本使用方法

Paint 的使用方法相对简单，构造一个 Paint 对象， 然后在使用 Canvas 绘制时将对象传入即可。

```java

Paint()

Paint(int flags)

Paint(Paint paint)

```

构造函数有 3 个，第三个是使用一个已有的 Paint 对象来构造一个画笔，相当于复制一个 Paint 对象。

第二个是设置一些标志位，如常用的 ANTI_ALIAS_FLAG（抗锯齿）、DITHER_FLAG（开启抖动）。在构造时可以直接传入。

```java
Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG | Paint.DITHER_FLAG);
```
有一点需要注意：paint 对标示位有 setFlags 方法，这个方法使用时需要注意，如果分为多个设置，后面的会覆盖前面的属性。

```java
Paint paint = new Paint();
paint.setFlags(Paint.ANTI_ALIAS_FLAG);
paint.setFlags(Paint.DITHER_FLAG);

```

结果保留的属性是 Paint.DITHER_FLAG，Paint.ANTI_ALIAS_FLAG 属性被覆盖了，这个容易出错的地方。解决方法可以在构造中一次性传入，或者在 setFlags 方法中一次传入

```java
paint.setFlags(Paint.ANTI_ALIAS_FLAG | Paint.DITHER_FLAG);
```
还有一种是使用各种 flag 的属性：

```
setAntiAlias(boolean aa)

setDither(boolean dither)
```
### 颜色

像素颜色设置，可以根据绘制内容来进行设置，如果绘制颜色和 bitmap，可以不设置 Paint 的颜色，可以直接使用 Canvas 的颜色填充类方法  drawColor/RGB/ARGB() ，是直接写在方法的参数里，通过参数来设置；  drawBitmap() 的颜色，是直接由 Bitmap 对象来提供。对于文字和形状、path 绘制，需要对画笔来进行设置颜色。

![Canvas](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/canvas_draw.jpg)

Paint 设置颜色的方法有两种：一种是直接用 Paint.setColor/ARGB() 来设置颜色，另一种是使用  Shader 来指定着色方案。

#### 直接设置颜色

setColor 方法

Color 获取方法在 [**自定义 View 的基础篇**](https://www.jianshu.com/p/8dd97246650f) 已经介绍了，不熟悉的可以在看看。

```java
paint.setColor(Color.parseColor("#009688"));  
canvas.drawRect(30, 30, 230, 180, paint);
```

setARGB 方法
```java
paint.setARGB(100, 255, 0, 0);  
canvas.drawRect(0, 0, 200, 200, paint);  
```

#### 使用 Shader

使用方法比较简单，通过 setShader(Shader shader) 设置 Shader。

**注意：**如果同时设置了 Shader 和 ARGB，Shader起作用，而设置的 ARGB 的颜色无效。 

学过 OpenGL 的人对 Shader 会比较了解，叫作着色器，用于给每个像素点进行着色用的。在安卓中，追色器一般有这几种：

> LinearGradient 
>
> RadialGradient 
>
> SweepGradient 
>
> BitmapShader 
>
> ComposeShader

下面就来试一下不同着色器的效果。

**LinearGradient** 

LinearGradient(float x0, float y0, float x1, float y1, int color0, int color1, Shader.TileMode tile)

线性渐变 Shader，指定两个点和两个颜色，颜色在这两个点之间线性渐变。还有个参数 tile：端点范围之外的着色规则。通过设置规则来显示两个端点之外的颜色。


**RadialGradient** 

RadialGradient(float centerX, float centerY, float radius, int centerColor, int edgeColor, TileMode tileMode)

辐射渐变，指定中心点，从圆心到半径处，从颜色1渐变到颜色2，呈辐射状，最后一个参数也是指定圆外的颜色的方式

**SweepGradient** 

SweepGradient(float cx, float cy, int color0, int color1)

扫描渐变，制定一个点，颜色1和颜色2，在360度的圆周上从颜色1渐变到颜色2

---

上面三种 Shader 是通过设定指定点的颜色，来实现绘制范围内所有像素点颜色的过渡，下面看一下效果。

![Shader](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_gradient.jpg)


**BitmapShader** 
BitmapShader(Bitmap bitmap, Shader.TileMode tileX, Shader.TileMode tileY)

用 Bitmap 的像素来作为图形或文字的填充，也就是说取图片中像素的颜色来作为每个点的颜色，bitmap是图片，tileX 和 tileY 指定X方向和Y方向端点范围之外的着色规则。

这里通过 BitmapShader 展示三种规则下的效果：

>* CLAMP 超过端点的颜色，取端点处像素的颜色值
>
>* MIRROR 超过端点的颜色，以端点处为界限，对称的颜色值
>
>* REPEAT 超过端点的颜色，重复从端点1到端点2之间的颜色值

三种效果图如下：

CLAMP

![Clamp](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/clamp.jpg)

MIRROR

![mirror_shader](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/mirror_shader.jpg)

REPEAT

![repeat_shader](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/repeat_shader.jpg)


**ComposeShader** 
ComposeShader(Shader shaderA, Shader shaderB, PorterDuff.Mode mode)


混合着色器，就是使用两个 shader，通过混合模式来给每个点着色，shaderA 和 shaderB 就是两个 shader，PorterDuff.Mode 指的是混合重叠的模式，这个在 [自定义View基础篇](https://www.jianshu.com/p/8dd97246650f) 介绍过混合的颜色的不同模式。这个就不再展开介绍每种模式的效果，使用到是可以查一下，多试几次就了解了。

**注意：** ComposeShader() 在硬件加速下是不支持两个相同类型的 Shader 的，所以这里也需要关闭硬件加速才能看到效果。

![compose](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/compose.jpg)

#### 设置颜色过滤

上面讲的都是如何设置颜色，即给目标设置一个颜色，这一部分说说颜色过滤，在已经有颜色的基础上，对已有的颜色进行颜色过滤，通过 setColorFilter(ColorFilter colorFilter) 方法设置颜色过滤器，以便得到期望的颜色。对颜色进行过滤有三种方式：

>* LightingColorFilter（）
>
>* PorterDuffColorFilter
>
>* ColorMatrixColorFilter

**LightingColorFilter** 是用来模拟简单的光照效果，计算颜色的公式如下：

LightingColorFilter 的构造方法是 LightingColorFilter(int mul, int add) ，参数里的 mul 和  add 都是和颜色值格式相同的 int 值，其中 mul 用来和目标像素相乘，add 用来和目标像素相加：

```
R' = R * mul.R / 0xff + add.R 
G' = G * mul.G / 0xff + add.G  
B' = B * mul.B / 0xff + add.B  
```

一个「保持原样」的 LightingColorFilter ，mul 为 0xffffff，add 为 0x000000（也就是0），那么对于一个像素，它的计算过程就是：

```
R' = R * 0xff / 0xff + 0x0 = R // R' = R  
G' = G * 0xff / 0xff + 0x0 = G // G' = G  
B' = B * 0xff / 0xff + 0x0 = B // B' = B  
```

假如想去掉原像素中的红色，可以把它的 mul 改为 0x00ffff （红色部分为 0 ） ，那么它的计算过程就是：

```
R' = R * 0x0 / 0xff + 0x0 = 0 // 红色被移除  
G' = G * 0xff / 0xff + 0x0 = G  
B' = B * 0xff / 0xff + 0x0 = B  
```
设置 LightingColorFilter 颜色过滤器

```java
ColorFilter lightingColorFilter = new LightingColorFilter(0x00ffff, 0x000000);  
paint.setColorFilter(lightingColorFilter);  
```

**PorterDuffColorFilter** 是通过指定一个颜色和混叠模式来影响最终的颜色效果的，通过构造 PorterDuffColorFilter(int color, PorterDuff.Mode mode) 来得到一个 ColorFilter，第一个参数是一个 color，第二个参数上面已经介绍过了，就是颜色的混叠模式。

**ColorMatrixColorFilter** 是通过一个4*5的矩阵来控制每一个像素的ARGB，更加精确和细致。
```
[ a, b, c, d, e,
f, g, h, i, j,
k, l, m, n, o,
p, q, r, s, t ]
```
通过计算， ColorMatrix 可以把要绘制的像素进行转换。对于颜色 [R, G, B, A] ，转换算法：

```
R’ = a*R + b*G + c*B + d*A + e;  
G’ = f*R + g*G + h*B + i*A + j;  
B’ = k*R + l*G + m*B + n*A + o;  
A’ = p*R + q*G + r*B + s*A + t;  
```
得到的 [R’, G’, B’, A’] 就是最终的颜色。

上述三种效果：

![paint_colorfilter](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_colorfilter.jpg)

#### setXfermode

Xfermode 指的是你要绘制的内容和 Canvas 的目标位置的内容应该怎样结合计算出最终的颜色。如果对 OpenGL 了解的话，这个很好理解。就是 Canvas 绘制的是一部分内容，另外在离屏渲染的缓冲区也有一部分内容， 这两部分内容通过指定的模式来进行混合，得到最终的效果。


![paint_xfermode](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_xfermode.jpg)

有一点需要注意的是：

通过使用离屏缓冲，把要绘制的内容单独绘制在缓冲层， Xfermode 的使用就不会出现奇怪的结果。

![paint_xfermode_savelayer](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_xfermode_savelayer.jpg)


使用离屏缓冲有两种方式：

1. Canvas.saveLayer()

saveLayer() 可以做短时的离屏缓冲。使用方法很简单，在绘制代码的前后各加一行代码，在绘制之前保存，绘制之后恢复 canvas.restoreToCount(saved)

```java
int saved = canvas.saveLayer(null, null, Canvas.ALL_SAVE_FLAG);

canvas.drawBitmap(rectBitmap, 0, 0, paint); // 画方
paint.setXfermode(xfermode); // 设置 Xfermode
canvas.drawBitmap(circleBitmap, 0, 0, paint); // 画圆
paint.setXfermode(null); // 用完及时清除 Xfermode

canvas.restoreToCount(saved);
```

2. View.setLayerType()

View.setLayerType() 是直接把整个 View 都绘制在离屏缓冲中。  setLayerType(LAYER_TYPE_HARDWARE) 是使用 GPU 来缓冲，  setLayerType(LAYER_TYPE_SOFTWARE) 是直接直接用一个 Bitmap 来缓冲。

### Paint 的各种效果

设置 Paint 的一些效果，如抗锯齿、填充/轮廓、线条宽度等

#### 抗锯齿

当图片放大时会有颗粒感，即能够看到一个一个的像素，通过抗锯齿光滑过渡，降低颗粒感。

方式一：setAntiAlias (boolean aa)

方式二：Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);  

#### setStyle(Paint.Style style)

>* Paint.Style.FILL 填充模式
>
>* Paint.Style.STROKE 画线模式
>
>* Paint.Style.FILL_AND_STROKE 填充 + 画线 
>

#### 线条

主要这4种方法：

```java
setStrokeWidth(float width)
setStrokeCap(Paint.Cap cap)
setStrokeJoin(Paint.Join join)
setStrokeMiter(float miter) 

```

![paint_style](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_style.jpg)

#### 色彩优化

主要有两个方法：

setDither(dither) ，设置抖动来优化色彩深度降低时的绘制效果； 

![paint_dither](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_dither.jpg)

setFilterBitmap(filterBitmap) ，设置双线性过滤来优化 Bitmap 放大绘制的效果。

![paint_bitmap_filter](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_bitmap_filter.jpg)

#### 图形轮廓效果

主要通过 setPathEffect(PathEffect effect) 来设置，对 Canvas 所有的图形绘制有效，如 drawLine() drawCircle() drawPath() 这些方法。

Android 中的 6 种 PathEffect。PathEffect 分为两类，单一效果的  CornerPathEffect DiscretePathEffect DashPathEffect PathDashPathEffect ，和组合效果的  SumPathEffect ComposePathEffect。

轮廓效果一般不常用，这里仅仅给出效果展示，如果用到，看着那个像，对应查找一下，写出来好像也记不住。。。


![paint_patheffect](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_patheffect.jpg)

#### setShadowLayer(float radius, float dx, float dy, int shadowColor)

绘制内容下面加一层阴影

注意：

如果要清除阴影层，使用 clearShadowLayer() 。

在硬件加速开启的情况下， setShadowLayer() 只支持文字的绘制，文字之外的绘制必须关闭硬件加速才能正常绘制阴影。

如果 shadowColor 是半透明的，阴影的透明度就使用 shadowColor 自己的透明度；而如果  shadowColor 是不透明的，阴影的透明度就使用 paint 的透明度。

![paint_shadow](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_shadow.jpg)

#### setMaskFilter(MaskFilter maskfilter)

setShadowLayer() 是设置的在绘制层下方的附加效果；而这个 MaskFilter 和它相反，设置的是在绘制层上方的附加效果。

setColorFilter(filter) ，是对每个像素的颜色进行过滤；setMaskFilter(filter) 则是基于整个画面来进行过滤。

MaskFilter 有两种： BlurMaskFilter 和 EmbossMaskFilter。

BlurMaskFilter 有四种模式：

>* NORMAL: 内外都模糊绘制
>
>* SOLID: 内部正常绘制，外部模糊
>
>* INNER: 内部模糊，外部不绘制
>
>* OUTER: 内部不绘制，外部模糊

![paint_mask_filter](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_mask_filter.jpg)

#### 获取绘制的 Path

Path ，指的就是 drawPath() 的绘制内容的轮廓，要算上线条宽度和设置的 PathEffect。

默认情况下（线条宽度为 0、没有 PathEffect），原 Path 和实际 Path 是一样的；而在线条宽度不为 0 （并且模式为 STROKE 模式或 FLL_AND_STROKE ），或者设置了 PathEffect 的时候，实际 Path 就和原 Path 不一样了：

通过 getFillPath(src, dst) 方法就能获取这个实际 Path。方法的参数里，src 是原 Path ，而 dst 就是实际 Path 的保存位置。 getFillPath(src, dst) 会计算出实际 Path，然后把结果保存在 dst 里。

![paint_path](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_path.jpg)

#### 获取文字的 Path  

```java
getTextPath(String text, int start, int end, float x, float y, Path path) / getTextPath(char[] text, int index, int count, float x, float y, Path path)
```

![paint_text_path](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Paint/paint_text_path.jpg)

### 参考

[HenCoder](https://hencoder.com/ui-1-2/) 主要参考 HenCoder 的博客，建议没看过的都看下，讲的很详细，这里只是抛砖引玉

[Paint 官方文档](https://developer.android.com/reference/android/graphics/Paint)

[练习代码地址](https://github.com/RalfNick/AndroidPractice/tree/master/CustomView_Paint)




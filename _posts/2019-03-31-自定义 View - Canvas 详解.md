---
layout: post
title: "自定义 View - Canvas 详解"
date: 2019-03-31
description: "Android 自定义 View - Canvas"
tag: Android View
---

![content](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/canvas_content.png)

### 1.Canvas 

Canvas 是我们绘制各种图形或文字时主要的操作对象，因为绘制绘制过程调用的都是它的 drawXX 方法。官方给出的 Canvas 的解释：

> The Canvas class holds the "draw" calls. To draw something, you need 4 basic components: A Bitmap to hold the pixels, a Canvas to host the draw calls (writing into the bitmap), a drawing primitive (e.g. Rect, Path, text, Bitmap), and a paint (to describe the colors and styles for the drawing).

> Canvas 提供各种 draw 方法，为了绘制，需要 4 个基本组件，持有像素的位图，拥有绘制方法的画布（Canvas,能够往位图中写入像素），绘制的基本元素，如矩形，路径，文字，图片等，还有一个画笔（Paint），画笔用来给出所画事物的颜色、样式等。

在之前一篇文章 [**自定义 View - Paint 详解**](https://www.jianshu.com/p/b4aa78c96f66)，讲解了 Paint 的用法和各种属性设置，在其中也提到很别扭的一点，绘制过程按照常规想法，不是应该是 paint.DrawXX 么。为了便于记忆，个人觉得可以这样理解，所有的动作时在画布上执行的，那么就需要准备各种材料（画笔，位置等），执行的动作就是画，即函数名。

Canvas 的方法很多，用的不熟练的话，也记不住这时候就需要整理一下，需要时查一下表。

### 2.Canvas 的常用操作速查表

| 操作类型       | 相关API                                    | 备注                                       |
| ---------- | ---------------------------------------- | ---------------------------------------- |
| 绘制颜色       | drawColor, drawRGB, drawARGB             | 使用单一颜色填充整个画布                             |
| 绘制基本形状     | drawPoint, drawPoints, drawLine, drawLines, drawRect, drawRoundRect, drawOval, drawCircle, drawArc | 依次为 点、线、矩形、圆角矩形、椭圆、圆、圆弧                  |
| 绘制图片       | drawBitmap, drawPicture                  | 绘制位图和图片                                  |
| 绘制文本       | drawText,    drawPosText, drawTextOnPath | 依次为 绘制文字、绘制文字时指定每个文字位置、根据路径绘制文字          |
| 绘制路径       | drawPath                                 | 绘制路径，绘制贝塞尔曲线时也需要用到该函数                    |
| 顶点操作       | drawVertices, drawBitmapMesh             | 通过对顶点操作可以使图像形变，drawVertices直接对画布作用、 drawBitmapMesh只对绘制的Bitmap作用 |
| 画布剪裁       | clipPath,    clipRect                    | 设置画布的显示区域                                |
| 画布快照       | save, restore, saveLayerXxx, restoreToCount, getSaveCount | 依次为 保存当前状态、 回滚到上一次保存的状态、 保存图层状态、 回滚到指定状态、 获取保存次数 |
| 画布变换       | translate, scale, rotate, skew           | 依次为 位移、缩放、 旋转、错切                         |
| Matrix(矩阵) | getMatrix, setMatrix, concat             | 实际上画布的位移，缩放等操作的都是图像矩阵Matrix， 只不过Matrix比较难以理解和使用，故封装了一些常用的方法。 |

******

### 3.Canvas 操作

画布常见的操作有以下 3 种：几何变换、范围裁剪和画布保存和回滚

先来看几何变换

#### 3.1 几何变换

画布的几何变换实际上是对画布位置的变换，变换之后在进行绘制操作，这样就能够得到我们希望图形正确的形状和位置，也能够避免我们自己去进行复杂的计算来得到从开始到最终的坐标。

(1) 平移

平移是坐标系的移动，可以为图形绘制选择一个合适的坐标系。

**注意，位移是基于当前位置移动，而不是每次基于屏幕左上角的(0,0)点移动**可以看到下面两次移动，可以叠加。
``` java
// 平移
mPaint.setColor(Color.GREEN);
canvas.translate(200,200);
canvas.drawCircle(0,0,100, mPaint);

mPaint.setColor(Color.BLUE);
canvas.translate(200,200);
canvas.drawCircle(0,0,100, mPaint);
```

![translate1](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/translate.png)

(2) 缩放

缩放提供了两个方法，如下：

``` java
public void scale (float sx, float sy)

public final void scale (float sx, float sy, float px, float py)
```
这两个方法中前两个参数是相同的分别为x轴和y轴的缩放比例。而第二种方法比前一种多了两个参数，用来控制缩放中心位置。

缩放比例(sx,sy)取值范围详解：

| 取值范围(n) | 说明                                           |
| ----------- | ---------------------------------------------- |
| (-∞, -1)    | 先根据缩放中心放大n倍，再根据中心轴进行翻转    |
| -1          | 根据缩放中心轴进行翻转                         |
| (-1, 0)     | 先根据缩放中心缩小到n，再根据中心轴进行翻转    |
| 0           | 不会显示，若sx为0，则宽度为0，不会显示，sy同理 |
| (0, 1)      | 根据缩放中心缩小到n                            |
| 1           | 没有变化                                       |
| (1, +∞)     | 根据缩放中心放大n倍                            |

如果在缩放时稍微注意一下就会发现<b>缩放的中心默认为坐标原点,而缩放中心轴就是坐标轴</b>，如下：

```java
// 2.缩放
//        mPaint.setStyle(Paint.Style.STROKE);
//        mPaint.setStrokeWidth(2);
//        mPaint.setColor(Color.GREEN);
//        canvas.drawCircle(0, 0, 200, mPaint);
// 缩放方法一
////        canvas.scale(2, 2);
// 缩放方法二
//        canvas.scale(0.5f, 0.5f,0,200);
//        mPaint.setColor(Color.BLUE);
//        canvas.drawCircle(0, 0, 200, mPaint);

// 缩小效果
// 将坐标系原点移动到画布正中心
//        canvas.translate(getWidth() / 2, getHeight() / 2);
//        for (int i = 0; i <= 20; i++) {
//            mPaint.setStyle(Paint.Style.STROKE);
//            mPaint.setStrokeWidth(5);
//            mPaint.setColor(Color.BLUE);
//            canvas.scale(0.9f, 0.9f);
//            canvas.drawCircle(0, 0, 400, mPaint);
//        }
```
![scale1](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/scale.png)

![scale2](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/scale_demo.png)

(3) 旋转

旋转提供了两种方法：

``` java
public void rotate (float degrees)

public final void rotate (float degrees, float px, float py)
```
和缩放一样，第二种方法多出来的两个参数依旧是控制旋转中心点的。

```java
// 3.旋转
RectF rect = new RectF(0,-400,400,0);
mPaint.setColor(Color.BLACK);
canvas.drawRect(rect,mPaint);

// 默认旋转一
//        canvas.rotate(180);
//        mPaint.setColor(Color.BLUE);
//        canvas.drawRect(rect,mPaint);

// 旋转二
canvas.rotate(180,100,0);
mPaint.setColor(Color.BLUE);
canvas.drawRect(rect,mPaint);
```
将黑色矩形围绕原点旋转 180°

![rotate1](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/rotate_1.png)

旋转 180° 后向右移动 100 的距离

![rotate2](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/rotate_2.png)

(4) 错切


skew 这里翻译为错切，错切是特殊类型的线性变换。

错切只提供了一种方法：
``` java
public void skew (float sx, float sy)
```

<b>参数含义：<br/>
float sx:将画布在x方向上倾斜相应的角度，sx倾斜角度的tan值，<br/>
float sy:将画布在y轴方向上倾斜相应的角度，sy为倾斜角度的tan值.</b>

变换后:
```
X = x + sx * y
Y = sy * x + y
```

下面看一下效果：

```java

```

![skew](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/skew.png)

(5) 矩阵变换

使用矩阵需要了解一些数学相关的知识，这里给出平移、旋转和缩放的几张图，也很容易理解。

![transtalte](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/translate_matrix.png)

![rotate](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/rotate_matrix.png)

![scale](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/scale_matrix.png)

使用矩阵变换来控制画布的步骤：

```
(1) 创建 Matrix 对象；

(2) 调用 Matrix 的 pre/postTranslate/Rotate/Scale/Skew() 方法来设置几何变换；

(3) 使用 Canvas.setMatrix(matrix) 或 Canvas.concat(matrix) 来把几何变换应用到 Canvas。
```
其中，Canvas.setMatrix(matrix)：用 Matrix 直接替换 Canvas 当前的变换矩阵。Canvas.concat(matrix)：用 Canvas 当前的变换矩阵和 Matrix 相乘，即基于 Canvas 当前的变换，叠加上 Matrix 中的变换。

这里仅仅示例一下平移和缩放，其他效果自己试试：

```java
canvas.translate(getWidth() / 2, getHeight() / 2);
//        Matrix matrix = new Matrix();
//        matrix.postTranslate(200,200);
//        canvas.setMatrix(matrix);
//        // = canvas.translate(200, 200);

//        mPaint.setColor(Color.BLUE);
//        canvas.drawCircle(0, 0, 100, mPaint);

for (int i = 0; i < 20; i++) {
mPaint.setStyle(Paint.Style.STROKE);
mPaint.setColor(Color.BLUE);
mPaint.setStrokeWidth(3);
Matrix matrix = new Matrix();
matrix.postScale(0.9f, 0.9f);
canvas.concat(matrix);
canvas.drawCircle(0, 0, 400, mPaint);
}
```

#### 3.2 范围裁剪

这里说明一下，范围裁剪是不可逆的，所以要加上 Canvas.save() 和 Canvas.restore() 来及时恢复绘制范围。

(1) clipRect()

裁剪指定矩形区域部分，也就是将矩形区域部分去掉。

```java
canvas.save();
// 裁剪是裁掉的部分，也就是说原图 - 这部分
canvas.clipOutRect(0, 200, 300, 400);
canvas.drawBitmap(bitmap, 0, 0, paint);
canvas.restore();
```
![rect](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/clip_rect.png)

(2) clipPath()

路径裁剪，通过路径裁剪可以使用更多的形状，矩形、圆形以及其他自定义的形状等。

```java
Point point1 = new Point( 150,  200);
Point point2 = new Point( 150,  200);

path1.addCircle(point1.x,point1.y,150,Path.Direction.CW);
path2.addCircle(point1.x,point1.y,150,Path.Direction.CCW);
path2.setFillType(Path.FillType.INVERSE_WINDING);
// path1
//        canvas.save();
//        // 裁剪是裁掉的部分，也就是说原图 - 这部分
//        canvas.clipPath(path1);
//        canvas.drawBitmap(bitmap, 0, 0, paint);
//        canvas.restore();

// path2
canvas.save();
// 裁剪是裁掉的部分，也就是说原图 - 这部分
canvas.clipPath(path2);
canvas.drawBitmap(bitmap, 0, 0, paint);
canvas.restore();
```

![path1](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/clip_path1.png)

![path2](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/clip_path2.jpg)

#### 3.3 画布保存和回滚

(1) 保存

画布操作是不可逆过程，所以对画布操作前，先对状态进行保存，以便在需要之前的状态时能够恢复

常用的保存于恢复操作比较简单：

```java
1.save() 或 save(int flag) 保存

2.画布操作，以及绘制过程

3.restore();恢复
```

saveLayerXxx
saveLayerXxx有比较多的方法：实际上就是根据选定的区域，以及透明度等设定来保存图层，操作相对于 save 更细。

```java

// 无图层alpha(不透明度)通道
public int saveLayer (RectF bounds, Paint paint)
public int saveLayer (RectF bounds, Paint paint, int saveFlags)
public int saveLayer (float left, float top, float right, float bottom, Paint paint)
public int saveLayer (float left, float top, float right, float bottom, Paint paint, int saveFlags)

// 有图层alpha(不透明度)通道
public int saveLayerAlpha (RectF bounds, int alpha)
public int saveLayerAlpha (RectF bounds, int alpha, int saveFlags)
public int saveLayerAlpha (float left, float top, float right, float bottom, int alpha)
public int saveLayerAlpha (float left, float top, float right, float bottom, int alpha, int saveFlags)
```

(2) 恢复

![save_restore](https://github.com/RalfNick/PicRepository/raw/master/CustomView/Canvas/canvas_stack.png)

restore

状态回滚，从栈顶取出一个状态然后根据内容进行恢复。

调用一次 restore 方法则从状态栈中取出一次，根据里面保存的状态进行状态恢复。

restoreToCount

弹出指定位置以及以上所有状态，并根据指定位置状态进行恢复。如果调用restoreToCount(2) 则会弹出 2 3 4 5 的状态，并根据第2次保存的状态进行恢复。

getSaveCount

获取保存的次数，即状态栈中保存状态的数量，使用该函数的返回值为5。该函数的最小返回值为1，即使弹出了所有的状态，返回值依旧为1，代表默认状态。


### 4 参考

[Canvas](https://developer.android.com/reference/android/graphics/Canvas?hl=en)

[Canvas之画布操作](https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/%5B03%5DCanvas_Convert.md)

[自定义 View 1-4 Canvas 对绘制的辅助 clipXXX() 和 Matrix](https://hencoder.com/ui-1-4/)


[**练习代码地址**](https://github.com/RalfNick/AndroidPractice/tree/master/CustomView_Canvas)

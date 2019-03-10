---
layout: post
title: "动画还可以这么玩？使用 Toast 显示 or Dialog 显示"
date: 2019-03-10
description: "动画还可以这么玩？使用 Toast 显示 or Dialog 显示"
tag: 动画
---

本篇不是讲解动画的设计，而是分析动画在使用过程中，如何合理显示遇到的一些坑，主要是由于特定场景引起的。

### 问题

相信大家都见过这样的点赞动画，点赞之后图片能够飘一会。

思路：动画其实并不难，通过一个自定义 View,大小为显示动画的范围，通过一个 ImageView显示图片，然后通过动画根据设计的路径改变位置，透明度和大小，显示特定的时长。

思路有了，然后就是实现，实现完成之后就出现了坑，坑不在这个动画上，而是使用的场景。看一下这个图

![Item](https://github.com/RalfNick/PicRepository/raw/master/Anima_Show_Dialog_Toast/RecyclerView_Item.jpg)

典型的 RecyclerView 的多布局，这样的布局如果设计：

```java

思路一：在每个 Item 中，如果设计成一个布局，这个布局会很大，也就是 1、 2、3、4 这几部分在一个子布局中，那么这个动画没有问题，点击按钮后，正常显示，自定义 View 可以放在子布局中的最上层。但存在一个问题，当需要更新每个 Item 时，难道要刷新整个 Item？举个例子，用户在看视频，点赞之后，因为需要刷新头像列表，所以整个 Item 刷新了一下，视频也重新加载，屏幕会闪一下，体验非常不好，所以这种思路不能接受。

```

```java
思路二：将 Item 拆分的更细，1、2、3、4 每个部分各自为一个小模块，刷新时只刷新单独一个模块，这样不会出现屏幕闪一下的问题（闪一下主要是由于视频或者大图部分刷新导致的），这样做的好处还有使得布局更加清晰，这部分的实现不具体展开了，采用的 BRVAH 封装的 RecyclerView。但是存在一个问题，动画属于第 3 部分，而第 3 部分的高度不足以装下动画布局的大小，我试过之后，动画仅仅展示一小部分，因为头像这部分的高度不够。这就是问题的关键，这个问题怎么解决？将动画部分放在第 2 部分，但是不同布局模块间的通讯怎么实现，又是问题？

```

所有也就引出了本文的标题，在布局最上层显示动画，可以采用 Dialog 或者 Toast。开始想到的是 Dialog，通过自定义 Dialog 来显示动画

### 自定义 View

采用 Dialog 或者 Toast显示动画都需要有一个自定义 View，这个 View 才是显示动画实际载体，然后将这个 View 放到 Dialog 或者 Toast 上。自定义 View 这部分就不细讲了，大体思路就是通过继承 RelativeLayout，然后添加一个 ImageView，用来显示图片，并通过设置动画，让图片动起来，并自己设置估值器，设置图片飘动的路径，动画由移动，缩放，透明度 3˙种动画组合的，下面给出代码。

```java

public class PraiseAnimView extends RelativeLayout {

private static final int TIME_ANIMATION = 4000;
protected PointF pointFStart, pointFEnd, pointFFirst, pointFSecond;
private Bitmap mBitmap;
private AnimatorSet mAnimatorSet;
private ImageView mImageView;

public PraiseAnimView(Context context) {
super(context);
init();
}

public PraiseAnimView(Context context, AttributeSet attrs) {
super(context, attrs);
init();
}

public PraiseAnimView(Context context, AttributeSet attrs, int defStyleAttr) {
super(context, attrs, defStyleAttr);
init();
}

private void init() {
setBackground(getResources().getDrawable(R.mipmap.bg_praise_anim, null));
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.mipmap.praise_anim_pic);
int height = Double.valueOf(bitmap.getHeight() * 1.5).intValue();
int width = Double.valueOf(bitmap.getWidth() * 1.5).intValue();
mBitmap = BitmapUtil.zoomImg(bitmap, width, height);
pointFStart = new PointF();
pointFFirst = new PointF();
pointFSecond = new PointF();
pointFEnd = new PointF();
initAnim();
}

private void initView() {
mImageView = new ImageView(getContext());
LayoutParams params = new LayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT);
params.addRule(CENTER_HORIZONTAL);
params.addRule(ALIGN_PARENT_BOTTOM);
mImageView.setImageBitmap(mBitmap);
addView(mImageView, params);
}

public void startAnim() {
removeAllViews();
initView();
mAnimatorSet.setStartDelay(0);
mAnimatorSet.start();
}

private void initAnim() {
mAnimatorSet = new AnimatorSet();
// 位置动画
ValueAnimator beiAnim = ValueAnimator.ofObject(new MyTypeEvaluator(pointFFirst, pointFSecond),
pointFStart, pointFEnd);
beiAnim.addUpdateListener(animation -> {
PointF value = (PointF) animation.getAnimatedValue();
mImageView.setX(value.x - mImageView.getWidth() / 2);
mImageView.setY(value.y + mImageView.getHeight() / 2);
});
beiAnim.setInterpolator(new AccelerateDecelerateInterpolator());
// 缩放动画
PropertyValuesHolder pl = PropertyValuesHolder.ofFloat("scaleY", 1f, 1.2f, 1f);
PropertyValuesHolder p2 = PropertyValuesHolder.ofFloat("scaleX", 1f, 1.2f, 1f);
ObjectAnimator scaleAnim = ObjectAnimator.ofPropertyValuesHolder(mImageView, pl, p2).setDuration(500);
// 透明度动画
ObjectAnimator alphaAnim = ObjectAnimator.ofFloat(mImageView, "alpha", 0.8f, 0).setDuration(500);
mAnimatorSet.addListener(new AnimatorListenerAdapter() {
@Override
public void onAnimationStart(Animator animation) {
super.onAnimationStart(animation);
}

@Override
public void onAnimationEnd(Animator animation) {
super.onAnimationEnd(animation);
PraiseAnimView.this.removeView(mImageView);
}
});
mAnimatorSet.setDuration(TIME_ANIMATION).play(beiAnim).with(alphaAnim).with(scaleAnim);
}

@Override
public void draw(Canvas canvas) {
super.draw(canvas);

pointFStart.x = mBitmap.getWidth() / 2;
pointFStart.y = getMeasuredHeight() - mBitmap.getHeight() * 3 / 2;
pointFEnd.y = 10;
pointFEnd.x = getMeasuredWidth() - mBitmap.getWidth() / 2;

pointFFirst.x = 10;
pointFFirst.y = getMeasuredHeight() / 2;
pointFSecond.x = getMeasuredWidth();
pointFSecond.y = getMeasuredHeight() / 2;
}

/**
* 估值器
*/
static class MyTypeEvaluator implements TypeEvaluator<PointF> {

private PointF pointFFirst, pointFSecond;

public MyTypeEvaluator(PointF start, PointF end) {
this.pointFFirst = start;
this.pointFSecond = end;
}

@Override
public PointF evaluate(float fraction, PointF startValue, PointF endValue) {
PointF result = new PointF();
float left = 1 - fraction;
result.x = (float) (startValue.x * Math.pow(left, 3) + 3 * pointFFirst.x * Math.pow(left, 2)
* fraction + 3 * pointFSecond.x * Math.pow(fraction, 2) * left + endValue.x * Math.pow(fraction, 3));
result.y = (float) (startValue.y * Math.pow(left, 3) + 3 * pointFFirst.y * Math.pow(left, 2)
* fraction + 3 * pointFSecond.y * Math.pow(fraction, 2) * left + endValue.y * Math.pow(fraction, 3));
return result;
}
}

}

```

### 使用 Dialog 显示动画

有了自定义的 View，那么接下来就简单了，将这个 view 放到自定义 Dialog 的布局中，然后自定义 Dialog

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:orientation="vertical">

<wang.ralf.showanimation.PraiseAnimView
android:id="@+id/praise_anim_view"
android:layout_width="100dp"
android:layout_height="200dp" />

</LinearLayout>
```
Dialog 的代码也直接给出

```java
public class PraiseDialog extends Dialog {

private Context mContext;
private PraiseAnimView mAnimView;
private int mWidth;
private int mHeight;
private Handler mHandler;

public PraiseDialog(Builder builder) {
super(builder.context);
mContext = builder.context;
mWidth = builder.width;
mHeight = builder.height;
mHandler = new Handler();
}

public PraiseDialog(Builder builder, int themeResId) {
super(builder.context, themeResId);
mContext = builder.context;
mWidth = builder.width;
mHeight = builder.height;
mHandler = new Handler();
}

protected PraiseDialog(Builder builder, boolean cancelable, @Nullable OnCancelListener cancelListener) {
super(builder.context, builder.cancelTouchOut, builder.cancelListener);
mContext = builder.context;
mWidth = builder.width;
mHeight = builder.height;
mHandler = new Handler();
}

@Override
protected void onCreate(Bundle savedInstanceState) {
super.onCreate(savedInstanceState);
View view = View.inflate(mContext, R.layout.layout_dialog_praise_animation, null);
setContentView(view);

Window win = getWindow();
WindowManager.LayoutParams lp = win.getAttributes();
lp.height = mHeight;
lp.width = mWidth;
win.setAttributes(lp);
mAnimView = view.findViewById(R.id.praise_anim_view);
}

@Override
public void show() {
super.show();
mAnimView.startAnim();
// 隐藏
mHandler.postDelayed(this::dismiss, 3500);
}

/**
* 设置对话框的位置，在屏幕上的位置
*
* @param x 横坐标
* @param y 纵坐标
*/
public void setPosition(int x, int y) {
Window win = getWindow();
WindowManager.LayoutParams lp = win.getAttributes();
win.setGravity(Gravity.TOP | Gravity.START);
lp.x = x;
lp.y = y;
win.setAttributes(lp);
}

public static final class Builder {

private Context context;
private int height, width;
private boolean cancelTouchOut;
private View view;
private int resStyle = -1;
private OnCancelListener cancelListener;


public Builder(Context context) {
this.context = context;
}

public Builder view(int resView) {
view = LayoutInflater.from(context).inflate(resView, null);
return this;
}

public Builder heightdp(int val) {
height = SizeUtils.dip2px(context, val);
return this;
}

public Builder widthdp(int val) {
width = SizeUtils.dip2px(context, val);
return this;
}

public Builder heightDimenRes(int dimenRes) {
height = context.getResources().getDimensionPixelOffset(dimenRes);
return this;
}

public Builder widthDimenRes(int dimenRes) {
width = context.getResources().getDimensionPixelOffset(dimenRes);
return this;
}

public Builder style(int resStyle) {
this.resStyle = resStyle;
return this;
}

public Builder cancelTouchout(boolean val) {
cancelTouchOut = val;
return this;
}

public Builder cancelListener(OnCancelListener listener) {
cancelListener = listener;
return this;
}

public Builder addViewOnclick(int viewRes, View.OnClickListener listener) {
view.findViewById(viewRes).setOnClickListener(listener);
return this;
}


public PraiseDialog build() {
if (resStyle != -1) {
return new PraiseDialog(this, resStyle);
} else {
return new PraiseDialog(this);
}
}
}
}
```

通过 Builder 模式设置 Dialog 的参数，其中有几点需要注意的：

>* 1、Dialog 的宽高设置
>
>* 2、动画开启以及显示时长控制
>
>* 3、Dialog 背景透明设置
>
>* 4、Dialog 显示位置控制
>

第 1 点：代码中已经给出

```java
Window win = getWindow();
WindowManager.LayoutParams lp = win.getAttributes();
lp.height = mHeight;
lp.width = mWidth;
win.setAttributes(lp);
```
通过 Window 来设置宽高，也要注意 dp 和 pixel 的转换关系


第 2 点：动画开启比较简单，重写 show 方法，获取自定义 view，即可开启动画。显示时长这里采用的是使用 Handler 发送一个延时消息来是 Dialog 销毁掉。

第 3 点：设置 Dialog 为透明，可以通过设置 Dialog 的主题即可。

```java
<style name="TransparentDlgTheme" parent="@android:style/Theme.Dialog">
<item name="android:windowBackground">@android:color/transparent</item>
<item name="android:windowIsTranslucent">false</item>
<item name="android:backgroundDimEnabled">false</item>
</style>
```

第 4 点：自定义 Dialog 中已经给出接口来设置位置

```java
/**
* 设置对话框的位置，在屏幕上的位置
*
* @param x 横坐标
* @param y 纵坐标
*/
public void setPosition(int x, int y) {
Window win = getWindow();
WindowManager.LayoutParams lp = win.getAttributes();
win.setGravity(Gravity.TOP | Gravity.START);
lp.x = x;
lp.y = y;
win.setAttributes(lp);
}
```
假设需求是要显示在点击按钮的右上角位置，我们需要获取窗口或者屏幕的绝对坐标，然后将坐标的横坐标向右移动 控件的宽度，纵坐标向上 Dialog 的高度 + 按钮的高度。但是有一点问题，高度上和宽度上有一些偏差，具体问题原因还没找出来？知道的大佬可以在评论区指正。

```java
int dialogBtnWidth = mDialogBtn.getWidth();
int height = mDialogBtn.getHeight();
//获取在整个窗口内的绝对坐标
int[] location = new int[2];
mDialogBtn.getLocationInWindow(location);
PraiseDialog dialog = new PraiseDialog.Builder(this)
.heightdp(200)
.widthdp(160)
.style(R.style.TransparentDlgTheme)
.cancelTouchout(true)
.build();
dialog.setPosition(location[0] + dialogBtnWidth,
location[1] - SizeUtils.dip2px(this, 200) - height / 2);
dialog.show();
```
这个显示在本例中没问题，但是在开篇的布局中，仍旧会有问题，这个坐标在 BRVAH 的 convert 方法中是获取不到的，因为布局还没有绘制完，只有在点击按钮时将坐标传过去，这里有一个小窍门，可以通过 Entity 中增加坐标字段，通过这两个字段，在刷新 item 时传过去。

本例子中使用 Dialog 显示的效果如下：

![Dialog_Anim](https://github.com/RalfNick/PicRepository/raw/master/Anima_Show_Dialog_Toast/anim_dialog.gif)

那么使用 Dialog 的问题在哪里，为什么还需要使用 Toast 么?仅仅是为了增加一种思路么？是因为它和 Toast 是有区别的。区别在于 Dialog 会获取焦点，当显示动画的3秒钟之间：

（1）如果设置触摸 Dialog 外部 Dialog消失，这时候点击点赞按钮就被拦截掉了，需要 Dialog 消失后再点击才会起作用。这里我没有继续往下探索，有兴趣的童鞋可以继续研究下。比如，将背景设置透明的， Dialog 显示时，点击点赞按钮，仍能够做出取消点赞的响应。

（2）如果 Dialog 外部不能点击，那只有动画展示完毕，Dialog 消失，用户才能够点击，也不够友好。

左右尝试到这里，我就使用 Toast 了。

### 使用 Toast 显示动画

自定义 Toast 比较简单，只需通过 LayoutInflater 加载布局，然后调用 Toast 的 setView 方法即可。这里引入了 Handler 和 弱引用，保证每次只显示一个 Toast，并且不会造成内存泄露等问题。

```java
public class AnimToast {

private static WeakReference<Toast> sWeakToast;
private static final Handler HANDLER = new Handler(Looper.getMainLooper());
private static final int COLOR_DEFAULT = 0xFEFFFFFF;
private static int sGravity = -1;
private static int sXOffset = -1;
private static int sYOffset = -1;

private AnimToast() {
throw new UnsupportedOperationException("u can't instantiate me...");
}


/**
* Cancel the toast.
*/
public static void cancel() {
final Toast toast;
if (sWeakToast != null && (toast = sWeakToast.get()) != null) {
toast.cancel();
sWeakToast = null;
}
}

/**
* reset default position
*/
public static void resetDefaultPosition(Context context) {
setGravity(Gravity.BOTTOM | Gravity.CENTER_HORIZONTAL,
0, SizeUtils.dip2px(context, 24));
}

/**
* Show custom toast for a short period of time.
*
* @param layoutId ID for an XML layout resource to load.
*/
public static View showCustomShort(Context context, @LayoutRes final int layoutId) {
final View view = getView(context, layoutId);
show(context, view, Toast.LENGTH_SHORT);
return view;
}

/**
* Show custom toast for a long period of time.
*
* @param layoutId ID for an XML layout resource to load.
*/
public static View showCustomLong(Context context, @LayoutRes final int layoutId) {
final View view = getView(context, layoutId);
show(context, view, Toast.LENGTH_LONG);
return view;
}

public static void show(Context context, final View view, final int duration) {
HANDLER.post(() -> {
cancel();
final Toast toast = new Toast(context);
sWeakToast = new WeakReference<>(toast);

toast.setView(view);
toast.setDuration(duration);
if (sGravity != -1 || sXOffset != -1 || sYOffset != -1) {
toast.setGravity(sGravity, sXOffset, sYOffset);
}
toast.show();
});
}

/**
* Set the gravity.
*
* @param gravity The gravity.
* @param xOffset X-axis offset, in pixel.
* @param yOffset Y-axis offset, in pixel.
*/
public static void setGravity(final int gravity, final int xOffset, final int yOffset) {
sGravity = gravity;
sXOffset = xOffset;
sYOffset = yOffset;
}

private static View getView(Context context, @LayoutRes final int layoutId) {
LayoutInflater inflate =
(LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
return inflate != null ? inflate.inflate(layoutId, null) : null;
}
}
```

如果自己使用的是全局的 Toast 工具类，那么调整位置后，需要重置 Toast 的位置，

```java
/**
* reset default position
*/
public static void resetDefaultPosition(Context context) {
setGravity(Gravity.BOTTOM | Gravity.CENTER_HORIZONTAL,
0, SizeUtils.dip2px(context, 24));
}

/**
* Set the gravity.
*
* @param gravity The gravity.
* @param xOffset X-axis offset, in pixel.
* @param yOffset Y-axis offset, in pixel.
*/
public static void setGravity(final int gravity, final int xOffset, final int yOffset) {
sGravity = gravity;
sXOffset = xOffset;
sYOffset = yOffset;
}
```
为什么是 24 dp，这个可以从 Toast 的源码中可以得知，这个自己查看一下吧。

```java
int toastBtnWidth = mToastBtn.getWidth();
int height = mToastBtn.getHeight();
//获取在整个窗口内的绝对坐标
int[] location = new int[2];
mToastBtn.getLocationInWindow(location);
int x = location[0] + toastBtnWidth;
int y = location[1] - SizeUtils.dip2px(this, 200) - height / 2;
AnimToast.setGravity(Gravity.START | Gravity.TOP, x, y);
View view = AnimToast.showCustomLong(this, R.layout.layout_toast_praise_animation);
PraiseAnimView praiseAnimView = view.findViewById(R.id.praise_anim_view);
praiseAnimView.startAnim();
```

这里代码应该能够看懂，和上面的 Dialog 位置的计算一样。下面来看一下效果：

![Toast_Anim](https://github.com/RalfNick/PicRepository/raw/master/Anima_Show_Dialog_Toast/anim_toast.gif)


这回可以随便点击点赞和取消点赞，再也不用担心了，而且相比之下，使用 Toast 更加优美，哈哈！

### Demo 代码地址

[练习代码](https://github.com/RalfNick/AndroidPractice/tree/master/ShowAnimation)


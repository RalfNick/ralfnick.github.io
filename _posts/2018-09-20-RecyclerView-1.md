---
layout: post
title: "RecyclerView-1"
date: 2018-09-20
description: "Android 常用 RecyclerView"
tag: Android 常用
---
### 一、RecyclerView 介绍

![RecyclerView—Recycling](http://pfk1gcktm.bkt.clouddn.com/recyclerview_structure.png)

在 RecyclerView 出来之前，大家都在使用 ListView、GridView，当然 RecyclerView 出来之后，基本上都转向了 RecyclerView，从名字上可以看出，它能够实现view 的复用，同样 ListView 在使用时我们自己也可以通过 converView 来实现复用，但是 RecyclerView 已经帮我们做好了，我们只需要给出需要装载 view 的 ViewHolder 就行，同时允许我们添加 ItemDecoration，ItemAnimator，设置多样的LayoutManager。

有这样一段英文：


>the RecyclerView itself doesn't care about visuals at all. It doesn't care about placing the elements at the right place, it doesn't care about separating any items and not about the look of each individual item either. To exaggerate a bit: All RecyclerView does, is recycle stuff. Hence the name.
>
>Anything that has to do with layout, drawing and so on, that is anything that has to do with how your data set is presented, is delegated to pluggable classes. That makes the new RecyclerView API extremely flexible.

大概意思是：RecyclerView 不关心视图，不关心元素位置，只关心复用，也就是说 RecyclerView 帮我们做好了复用的实现， 其他方面我们自己来配置，这样更加灵活，达到所谓的“插拔式”效果，即可添加 分割线，可添加动画效果，可设置布局效果等等。

其实复用，就是多创建一个或者几个视图，当滑动到底部时，加载更多的视图时，复用已经不可见的视图。至于多创建视图是几个，每一屏上的视图是几个，暂时我也不知道，后面进行源码分析时，再来揭晓，先知道它大概的原理。

![RecyclerView—Recycling](http://pfk1gcktm.bkt.clouddn.com/recycling_of_views.png)

### 二、基本使用

RecyclerView 使用很频繁，其实主要场景就是要展示的视图数量很多，无法在一个屏幕之下展示出来，当然你想到 ScrollView，ScrollView 展示的视图也是有限的，也不会很多，否则会造成卡顿，甚至 OOM。RecyclerView 一般的用法就是做分页加载，上拉加载，下拉刷新，然后将数据铺到界面上，复杂一些时，界面上有几种不同的视图，通过不同的 ViewHolder 来装载。

下面就先开看看最基本的用法：

![Steps](http://pfk1gcktm.bkt.clouddn.com/recyclerview_use_step.png)

首先在 gradle 中引入 RecyclerView ，这里同时引入 cardview，不是必须的，只是为了在添加每个Item 视图时使用 cardview ，更加美观一些。

```java
implementation 'com.android.support:cardview-v7:27.1.1'
implementation 'com.android.support:recyclerview-v7:27.1.1'
```
### 1. XML 布局设置

**引用 recyclerview**

activity_main.xml

```
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:app="http://schemas.android.com/apk/res-auto"
xmlns:toolbar="http://schemas.android.com/apk/res-auto"
android:layout_width="match_parent"
android:layout_height="match_parent">

<android.support.design.widget.CoordinatorLayout
android:layout_width="match_parent"
android:layout_height="match_parent">

<!--使用 toolbar 设置菜单-->
<include layout="@layout/view_toolbar" />

<!--用于刷新-->
<com.ralf.www.recyclerviewtest.widget.MultiSwipeRefreshLayout
android:id="@+id/swipe_refresh_layout"
android:layout_width="match_parent"
android:layout_height="match_parent"
app:layout_behavior="@string/appbar_scrolling_view_behavior">

<!--重点部分，使用RecyclerView，高度设置，-->
<!--如果是垂直布局，使用match_parent-->
<!--如果是水平布局，可使用wrap_content -->
<android.support.v7.widget.RecyclerView
android:id="@+id/rv_meizhi"
android:layout_width="match_parent"
android:layout_height="match_parent" />

</com.ralf.www.recyclerviewtest.widget.MultiSwipeRefreshLayout>

<android.support.design.widget.FloatingActionButton
android:id="@+id/main_fab"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:layout_gravity="right|bottom"
android:layout_marginBottom="24dp"
android:layout_marginRight="16dp"
android:clickable="true"
android:onClick="onFab"
android:src="@mipmap/ic_refresh_white_24dp"
app:borderWidth="0dp"
app:elevation="4dp"
app:layout_anchor="@id/swipe_refresh_layout"
app:layout_anchorGravity="right|bottom"
app:layout_behavior="com.ralf.www.recyclerviewtest.widget.FABAutoHideBehavior" />

</android.support.design.widget.CoordinatorLayout>

</FrameLayout>

```

**子项的 XML 布局**

item_main_activity_rv.xml

子项布局在重写 Adapter 时使用,外面包了一层 CardView，更加美观一些

```
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.CardView
android:id="@+id/meizhi_card"
xmlns:android="http://schemas.android.com/apk/res/android"
xmlns:app="http://schemas.android.com/apk/res-auto"
xmlns:tools="http://schemas.android.com/tools"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:layout_margin="5dp"
android:clickable="true"
android:foreground="?attr/selectableItemBackground"
app:cardCornerRadius="2dp"
app:cardElevation="4dp"
tools:minWidth="160dp">

<LinearLayout
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:orientation="vertical">

<ImageView
android:id="@+id/iv_meizhi"
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:adjustViewBounds="true"
android:scaleType="fitXY"
tools:src="@mipmap/ic_launcher"/>

<LinearLayout
android:layout_width="match_parent"
android:layout_height="wrap_content"
android:orientation="horizontal"
android:paddingBottom="10dp"
android:paddingLeft="10dp"
android:paddingRight="10dp"
android:paddingTop="10dp">

<TextView
android:id="@+id/tv_title"
android:layout_width="wrap_content"
android:layout_height="wrap_content"
android:textAppearance="?android:attr/textAppearanceSmall"
tools:text="Title"/>

</LinearLayout>

</LinearLayout>

</android.support.v7.widget.CardView>

```

### 2. LayoutManager 设置

RecyclerView 设置 LayoutManager，系统提供了3种 LayoutManager：LinearLayoutManager（线性布局）、GridLayoutManager（网格布局）、StaggeredGridLayoutManager（瀑布布局），根据场景需要自己设定，也可以自己实现 LayoutManager。
这里先给出简单的使用，对于详细的分析以及自己实现 **后面的文章**（加粗下，提醒自己。。）再给出分析。

```java

//        final StaggeredGridLayoutManager layoutManager = new StaggeredGridLayoutManager(
//                2, StaggeredGridLayoutManager.VERTICAL
//        );
final GridLayoutManager layoutManager = new GridLayoutManager(this, 2
, GridLayoutManager.VERTICAL, false);

//        final LinearLayoutManager layoutManager = new LinearLayoutManager(this,
//                LinearLayoutManager.VERTICAL, false);

// 设置布局管理器        
mRecyclerView.setLayoutManager(layoutManager);
```

### 3. 设置数据源

数据源一般就用 List 传递给 Adapter，作为其构造函数中的一个参数给出。

```
private List<GanHuo> mPicList = new ArrayList<>();
```
直接在 成员变量声明时给出 List 对象，一般数据源 都是通过网络求来的，在 demo 中我集成了 Retrofit + Rxjava 进行网络请求，加载图片，对这部分没接触的童鞋可以看看 Retrofit + Rxjava，或者自己替换成本地的图片也行。

有一点，需要注意下：对于初始化时，设置布局管理器，设置 Adapter，设置数据源，没有严格的先后顺序。开始没有在 List 中加入数据，初始化时候，加载网络数据或者本地数据，然后通过刷新列表即可。

### 4. 实现 Adapter 并设置

PictureAdapter 需要集成 RecyclerView.Adapter，并需要声明泛型类型 ViewHolder，否则是 Object 类型，并实现 三个方法：


>生成用于持有每个 View 的 ViewHolder，实现复用
>onCreateViewHolder
>
>将ViewHolder绑定，即将数据绑定到视图上
>onBindViewHolder
>
>获取子 View 的数量，即传过来的 List 的大小
>getItemCount

```java

public class PictureAdapter extends RecyclerView.Adapter<PictureAdapter.PictureViewHolder> {

private List<GanHuo> picList;
private ItemClickListener mClickListener;
private Context mContext;

public PictureAdapter(Context context, List<GanHuo> picList) {
this.picList = picList;
mContext = context;
}

@NonNull
@Override
public PictureViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
// 加载布局 item_main_activity_rv.xml
View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item_main_activity_rv,
parent, false);
return new PictureViewHolder(view);
}

@Override
public void onBindViewHolder(@NonNull final PictureViewHolder holder, final int position) {
if (picList != null) {
GanHuo ganHuo = picList.get(position);
String url = ganHuo.getUrl();
String who = ganHuo.getWho();

// Glide图片加载
RequestOptions requestOptions = RequestOptions.placeholderOf(R.mipmap.ic_launcher)
.error(R.mipmap.ic_refresh_white_24dp)
.useAnimationPool(true);
Glide.with(mContext)
.load(url)
.apply(requestOptions)
.into(holder.imageView);
holder.textView.setText(who);
// 点击事件
if (mClickListener != null) {
holder.textView.setOnClickListener(new View.OnClickListener() {
@Override
public void onClick(View v) {
mClickListener.onItemClick(PictureAdapter.this, v, position);
}
});
holder.textView.setOnLongClickListener(new View.OnLongClickListener() {
@Override
public boolean onLongClick(View v) {
mClickListener.onItemLongClick(PictureAdapter.this, v, position);
return true;
}
});

holder.imageView.setOnClickListener(new View.OnClickListener() {
@Override
public void onClick(View v) {
mClickListener.onItemClick(PictureAdapter.this, v, position);
}
});
holder.imageView.setOnLongClickListener(new View.OnLongClickListener() {
@Override
public boolean onLongClick(View v) {
mClickListener.onItemLongClick(PictureAdapter.this, v, position);
return true;
}
});
}
} else {
// default url
RequestOptions requestOptions = RequestOptions.placeholderOf(R.mipmap.ic_launcher)
.error(R.mipmap.ic_refresh_white_24dp);
Glide.with(mContext)
.load(R.mipmap.ic_launcher)
.apply(requestOptions)
.into(holder.imageView);
holder.textView.setText("看不到的妹纸");
}
}

@Override
public int getItemCount() {
if (picList == null || picList.size() < 1) {
return 0;
}
return picList.size();
}

// ViewHolder 用于存放 View，这个View也就是 前面设置的 .xml，RecyclerView 通过 ViewHolder进行复用
static class PictureViewHolder extends RecyclerView.ViewHolder {

private View mView;
private ImageView imageView;
private TextView textView;

public PictureViewHolder(View itemView) {
super(itemView);
mView = itemView;
initView();
}

private void initView() {
imageView = mView.findViewById(R.id.iv_meizhi);
textView = mView.findViewById(R.id.tv_title);
}
}

public ItemClickListener getClickListener() {
return mClickListener;
}

public void setClickListener(ItemClickListener clickListener) {
mClickListener = clickListener;
}

// 点击事件接口
public interface ItemClickListener {

void onItemClick(RecyclerView.Adapter adapter,
View view, int position);

void onItemLongClick(RecyclerView.Adapter adapter,
View view, int position);
}
}
```
实现 Adapter 之后，创建实例，然后用 RecyclerView setAdapter。

```java

// mPicList 就是用于存放数据的
mAdapter = new PictureAdapter(this, mPicList);
mRecyclerView.setAdapter(mAdapter);
```

请求数据，通过网络请求数据 Retrofit + Rxjava
```java
private void requestData(int index) {
// 开始刷新
setRefreshing(true);
RetrofitClient.getRetrofitClientInstance()
.requestNetForData(NetApi.BASE_URL, RequestDataService.class)
.getGanHuoData("福利", 10, index)
.observeOn(AndroidSchedulers.mainThread())
.subscribeOn(Schedulers.io())
.subscribe(new DefaultObserver<BaseEntity<GanHuo>>() {
@Override
public void onNext(BaseEntity<GanHuo> ganHuo) {
if (ganHuo != null && !ganHuo.isError()) {
// 刷新列表
refreshRecyclerView(ganHuo.getResults());
}
}

@Override
public void onError(Throwable e) {
setRefreshing(false);
}

@Override
public void onComplete() {
// 刷新结束，如果是下拉刷新，滚动到最顶部
setRefreshing(false);
if (mIndex == 1) {
mRecyclerView.smoothScrollToPosition(0);
}
}
});
}
```
刷新 RecyclerView，有上拉加载和下拉刷新，对于不同的情况做了简单的处理，防止更新数据时，有闪烁现象。
```java
private void refreshRecyclerView(List<GanHuo> ganHuoList) {
if (ganHuoList == null || ganHuoList.size() < 1) {
return;
}
if (mIndex == 1) {
mPicList.clear();
mPicList.addAll(ganHuoList);
mAdapter.notifyDataSetChanged();
} else {
mPicList.addAll(ganHuoList);
mAdapter.notifyItemRangeInserted(mAdapter.getItemCount()
, ganHuoList.size());
}
}
```
此时，就可以运行，能够看到加载的图片列表了。

![load_refresh](http://pfk1gcktm.bkt.clouddn.com/recyclerview-refresh-load.gif)

这里面有 RecyclerView的滑动事件监听，用来滑动到底部时，做上拉加载更多数据。

```java

mRecyclerView.addOnScrollListener(
new RecyclerView.OnScrollListener() {
@Override
public void onScrolled(RecyclerView rv, int dx, int dy) {
// 判断是否滑动到最底部
if (!mSwipeRefreshLayout.isRefreshing() &&
layoutManager.findLastCompletelyVisibleItemPosition() >= mAdapter.getItemCount() - 1) {

mIndex += 1;
requestData(mIndex);
}
}
}
);
```

### 5. 设置分割线

在图中看出，已经设置了分割线。设置分割线相对简单，主要有两种方法：

>* 在 xml 布局中添加 0.5 dp的View
>* 使用 ItemDecoration

第一种就不介绍了，自己尝试弄一下。第二种实现ItemDecoration，系统默认只给出来了一种实现 DividerItemDecoration，
一般情况下，也够用了，除了改一下颜色之类的。

```java

// 添加分割线
// 第二个参数设置方向，垂直布局设置垂直分割线，水平布局设置水平分割线
mRecyclerView.addItemDecoration(new DividerItemDecoration(
this, DividerItemDecoration.VERTICAL));
```

分割线的颜色怎么修改的？
DividerItemDecoration 的颜色采用的应是主题的颜色，所以要改变颜色，需要自己定义一个颜色，然后加到应用的主题中。

在 drawable 中定义一个 shape 
```
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
android:shape="rectangle" >

// 渐变颜色
<gradient
android:centerColor="#ff00ff00"
android:endColor="#ff0000ff"
android:startColor="#ffff0000"
android:type="linear" />

// 分割线宽度
<size android:height="4dp"/>

</shape>
```
加到应用主题中

```
<!-- Application theme. -->
<style name="AppTheme1" parent="Theme.AppCompat.NoActionBar">
<item name="colorPrimary">@color/theme_primary</item>
<item name="colorPrimaryDark">@color/theme_primary_dark</item>
<item name="colorAccent">@color/md_red_400</item>
// 分割线主题颜色设置
<item name="android:listDivider">@drawable/divider_shape</item>
<!-- 加入toolbar溢出【弹出】菜单的风格 -->
<item name="actionOverflowMenuStyle">@style/OverflowMenuStyle</item>

</style>
```
这样就能改变分割线颜色，同时宽度也可以 shape 中修改


![DividerItemDecoration](http://pfk1gcktm.bkt.clouddn.com/recyclerview-linearlayoutmanager.png)

### 6. 设置动画

设置动画相对比较简单，调用 setItemAnimator 即可，动画可以使用提供的默认的动画，或者自己实现 RecyclerView.ItemAnimator，达到自己想要的结果。
```java

// 添加动画
mRecyclerView.setItemAnimator(new DefaultItemAnimator());

```
动画用于数据改变时显示的动画效果，这里在 Adapter 中添加两个方法，测试一下
```java

/**
* 添加测试，用于展现动画
* @param position
*/
public void indertTest(int position){
picList.add(1,new GanHuo());
// 这里更新数据集不是用adapter.notifyDataSetChanged()而是
// notifyItemInserted(position)与notifyItemRemoved(position)
// 否则没有动画效果。
notifyItemInserted(position);
}

/**
* 删除测试，用于展现动画
* @param position
*/
public void removeTest(int position){
picList.remove(picList.get(position));
notifyItemRemoved(position);
}
```
![animatirs](http://pfk1gcktm.bkt.clouddn.com/recyclerview_add_remove.gif)

想看看到更多的动画效果，可以参考 github 上的开源项目 [RecyclerViewItemAnimators](https://github.com/wasabeef/recyclerview-animators)

### 7. 设置点击事件

对于 RecyclerView 的点击事件，系统没有提供接口 ClickListener和 LongClickListener，需要自己实现。常用的方式一般有两种：

>*  mRecyclerView.addOnItemTouchListener(listener);根据手势动作判断
>* 第二种是自己在 Adapter 中设置接口，然后将实现传递进去

这里给出第二种的方式，看一下效果：

在 onBindViewHolder 方法中代码有点多，主要看setOnClickListener 部分，实际上还是给普通的控件设置点击事件，在 onClick 中回调我们设置的接口，这样执行的方法就是我们想要的动作了。
```java
public class PictureAdapter extends RecyclerView.Adapter<PictureAdapter.PictureViewHolder> {

...
@Override
public void onBindViewHolder(@NonNull final PictureViewHolder holder, final int position) {
if (picList != null) {
GanHuo ganHuo = picList.get(position);
String url = ganHuo.getUrl();
String who = ganHuo.getWho();
RequestOptions requestOptions = RequestOptions.placeholderOf(R.mipmap.ic_launcher)
.error(R.mipmap.ic_refresh_white_24dp)
.useAnimationPool(true);
Glide.with(mContext)
.load(url)
.apply(requestOptions)
.into(holder.imageView);
holder.textView.setText(who);
// 点击事件
if (mClickListener != null) {
holder.textView.setOnClickListener(new View.OnClickListener() {
@Override
public void onClick(View v) {
mClickListener.onItemClick(PictureAdapter.this, v, position);
}
});
holder.textView.setOnLongClickListener(new View.OnLongClickListener() {
@Override
public boolean onLongClick(View v) {
mClickListener.onItemLongClick(PictureAdapter.this, v, position);
return true;
}
});

holder.imageView.setOnClickListener(new View.OnClickListener() {
@Override
public void onClick(View v) {
mClickListener.onItemClick(PictureAdapter.this, v, position);
}
});
holder.imageView.setOnLongClickListener(new View.OnLongClickListener() {
@Override
public boolean onLongClick(View v) {
mClickListener.onItemLongClick(PictureAdapter.this, v, position);
return true;
}
});
}
} else {
// default url
RequestOptions requestOptions = RequestOptions.placeholderOf(R.mipmap.ic_launcher)
.error(R.mipmap.ic_refresh_white_24dp);
Glide.with(mContext)
.load(R.mipmap.ic_launcher)
.apply(requestOptions)
.into(holder.imageView);
holder.textView.setText("看不到的妹纸");
}
}

// ClickListener和 LongClickListener 回调接口
public interface ItemClickListener {

void onItemClick(RecyclerView.Adapter adapter,
View view, int position);

void onItemLongClick(RecyclerView.Adapter adapter,
View view, int position);
}
```

在 MainActivity 中添加接口的具体实现

```java

// 设置点击事件
mAdapter.setClickListener(new PictureAdapter.ItemClickListener() {
@Override
public void onItemClick(RecyclerView.Adapter adapter, View view, int position) {
GanHuo ganHuo = mPicList.get(position);
String who = ganHuo.getWho();
String desc = ganHuo.getDesc();
if (view.getId() == R.id.tv_title) {
Toast.makeText(MainActivity.this, "你点击了" + who, Toast.LENGTH_SHORT).show();
} else if (view.getId() == R.id.iv_meizhi) {
Toast.makeText(MainActivity.this, "你点击了" + desc, Toast.LENGTH_SHORT).show();
}
}

@Override
public void onItemLongClick(RecyclerView.Adapter adapter, View view, int position) {
GanHuo ganHuo = mPicList.get(position);
String who = ganHuo.getWho();
String desc = ganHuo.getDesc();
if (view.getId() == R.id.tv_title) {
Toast.makeText(MainActivity.this, "你长按了" + who, Toast.LENGTH_SHORT).show();
} else if (view.getId() == R.id.iv_meizhi) {
Toast.makeText(MainActivity.this, "你长按了" + desc, Toast.LENGTH_SHORT).show();
}
}

});
```
这样执行的方法就是我们想要的效果了。

![click](http://pfk1gcktm.bkt.clouddn.com/recyclerview_item_click.gif)

### 总结

主要介绍了 RecyclerView 的主要特点，与ListView 相比的优势，在于 view 的复用上；同时详细介绍了基本的使用方法。使用 RecyclerView 可以很方便的使用 动画，分割线，以及布局管理器的样式改变。例子中的也给出了上拉加载和下拉刷新，但是仅供参看，使用起来还是存在一定问题的，后面会做修正，在项目中使用的话，可以使用开源框架 [Android智能下拉刷新框架-SmartRefreshLayout](https://github.com/RalfNick/SmartRefreshLayout)

好了，就到这里，对 RecyclerView 不熟悉的赶紧试试吧，后面会再写一篇，介绍使用过程中的一些小细节，避免采坑！

[Demo地址](https://github.com/RalfNick/AndroidPractice/tree/master/Common)

### 参考

[Android RecyclerView 使用完全解析 体验艺术般的控件](https://blog.csdn.net/lmj623565791/article/details/45059587)

[动画开源库](https://github.com/wasabeef/recyclerview-animators)

[A First Glance at Android’s RecyclerView](https://www.grokkingandroid.com/first-glance-androids-recyclerview/)

[Android智能下拉刷新框架-SmartRefreshLayout](https://github.com/RalfNick/SmartRefreshLayout)

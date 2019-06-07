---
layout: post
title: "Android-Divider"
date: 2018-01-18
description: "android中的小技巧"
tag: Android
---

在android中创建布局时，发现有些控件之间加一些分割线，会很美观，上网搜索了下，找到了三种方式创建分割线，下面就来分别来试一下。

### 1. 使用View
也是最简单的一种方式，直接定义宽度和高度，设置颜色即可。
但是，分割线较多的布局中，这种不太适合，会占用较多内存

```java
<View
  android:layout_width="match_parent"
  android:layout_height="1dp"
  android:background="#303F9F"/>
```

### 2. 使用ImageView

方法与View类似，也是设置高度、宽度和颜色即可

```java
<ImageView
  android:layout_width="match_parent"
  android:layout_height="0.5dp"
  android:background="@color/colorAccent"/>
```

### 3. 自定义xml

自定义一个分割线的divider.xml，放置drawable目录下

```java
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android">

    <solid android:color="@color/colorAccent"/>
    <size android:height="1dp"/>

</shape>
```

使用时，一般在垂直布局中设置，水平布局中不能显示

```java
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="10dp"
    android:divider="@drawable/divider"
    android:showDividers="end"
    android:dividerPadding="1.5dp">

    ...
    </LinearLayout>
```

注意点：

（1）垂直布局

（2）android:divider="@drawable/divider" ，不能直接设置颜色，否则不显示，divider就是自定义的xml

（3）android:showDividers="end" 设置显示位置

* end：末端

* beginning：前端

* middle：中间

* none：不显示

（4）android:dividerPadding="1.5dp"，可以更改分割线的宽度

### 4.垂直分割线

有的童鞋可能需要使用在水平的布局中使用分割线，那么如何创建呢？

其实方式是相同的，只不过改变一下宽度和高度，高度匹配父布局，宽度设置为线宽,这里仅View的方式为例，在两个TextView之间加入一个分割线。

```java
<LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:layout_margin="10dp">

    <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginLeft="20dp"
            android:layout_weight="1"
            android:text="Android2"
            android:textSize="18sp"/>

    <View
            android:layout_width="1.5dp"
            android:layout_height="match_parent"
            android:background="@color/colorAccent"/>

    <TextView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:gravity="end"
            android:textSize="18sp"
            android:text="Android21"/>
    </LinearLayout>
```

### 总结

（1）以上就是分割线的三种创建方式，需要根据自己的布局选择合适方式，若分割线使用数量不多，1 和 2 是较为简单的方式；

（2）若分割线数量较多，可以采用 3，能够较少布局所占内存，并较少布局中控件的数量，达到复用的效果！

### 附效果及代码

效果图布局代码如下，需要的小伙伴可以试试额！

![分割线效果图](https://github.com/RalfNick/PicRepository/raw/master/view/%E5%88%86%E5%89%B2%E7%BA%BF%E7%9A%84%E4%B8%89%E7%A7%8D%E6%96%B9%E5%BC%8F-2018-01-18.png)


```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:padding="10dp"
        android:divider="@drawable/divider"
        android:showDividers="">

    <LinearLayout

            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_margin="10dp">

        <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:textSize="18sp"
                android:layout_gravity="center_vertical"
                android:layout_marginLeft="20dp"
                android:layout_weight="1"
                android:text="Android1"/>

        <Button
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginEnd="10dp"
                android:text="button1"/>

    </LinearLayout>

    <!--方式一：ImageView-->
    <ImageView
            android:layout_width="match_parent"
            android:layout_height="0.5dp"
            android:background="@color/colorAccent"/>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_margin="10dp">

        <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="20dp"
                android:layout_weight="1"
                android:text="Android2"
                android:textSize="18sp"/>

        <View
                android:layout_width="1.5dp"
                android:layout_height="match_parent"
                android:background="@color/colorAccent"/>

        <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:gravity="end"
                android:textSize="18sp"
                android:text="Android21"/>
    </LinearLayout>

    <!--方式二：使用View-->
    <View
            android:layout_width="match_parent"
            android:layout_height="1dp"
            android:background="#303F9F"/>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_margin="10dp"
            android:divider="@drawable/divider"
            android:showDividers="end"
            android:dividerPadding="5dp">

        <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="20dp"
                android:layout_weight="1"
                android:text="Android3"
                android:textSize="18sp"/>

        <Button
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginEnd="10dp"
                android:text="button3"/>
    </LinearLayout>

    <!--方式二：使用View-->
    <View
            android:layout_width="match_parent"
            android:layout_height="0.5dp"
            android:background="#303F9F"/>

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:layout_margin="10dp"
            >

        <TextView
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginLeft="20dp"
                android:layout_weight="1"
                android:text="Android4"
                android:textSize="18sp"/>

        <Button
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginEnd="10dp"
                android:text="button4"/>
    </LinearLayout>


</LinearLayout>

```

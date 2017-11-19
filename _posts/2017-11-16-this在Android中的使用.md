---
layout: post
title: "this在Android中的使用"
date: 2017-11-16
description: "android学习过程中遇到的小知识点"
tag: Android
---

## **1.this**
java 中 this 是一个隐含参数，代表一个对象本身，下面以一个简单的例子说明 this参数。

```java

class Fruit{

  private String name;
  private int size;

  public Fruit(){}

  public String getInfo(){
    return this.name + this.size;//this 可以省略
  }
}
```
从这个例子中看出，this 代表当前对象本身，而且 this 可以省略，可以理解为 this 的活动范围是整个类中，注意这里没有内部类。

## **2. Andrroid 中 this 的使用**

下面看一个常用的例子,如：在 MainActivity 中开启一个 AnotherActivity

```java

protected void onCreate(Bundle savedInstenceState){
    super.onCreate(savedInstenceState);

    Intent intent = new Intent(this,Another.class);
    startActivity(intent);

}
```
通过 Intent 开启新的 Activity ，Intent 构造函数中的有两个参数，Context 指向当前所在的 Activity，cls 指向要开启的 Activity 的class

```java
public Intent(Context packageContext, Class<?> cls) {
      throw new RuntimeException("Stub!");
}

```

Intent第一个参数使用this，相当于MainActivity.this，所以使用MainActivity.this，同样可以, this 所在的范围是 MainActivity 类。

下面再看一下另外一个例子

```java
button.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View view) {
        Intent inte  = new Intent(this,LoginActivity.class);//需要将this.改成MainActivity.this
       startActivity(intent);
    }
});

```
看看这段能否执行?
答案是否定的，这里的 this 指向匿名类内部类 new View.OnClickListener()对象，而不是我们想要的MainActivity对象，在内部类中想要使用外部
类方法或访问变量，需要获取外部类实例，外部类名.this。

如果这段难以理解，那么将上述方法换一个写法：

```java

class Listener implements View.OnClickListener{

      @Override
      public void onClick(View view) {

          Intent intent = new Intent("www.ralf.com.broadcastprectice.OFFLINE");
          Intent inte  = new Intent(this,LoginActivity.class);//需要将this.改成MainActivity.this
          sendBroadcast(intent);
      }
  }
    Listener listenter = new Listener();
    button.setOnClickListener(listenter);

```
这样写之后应该就比较清晰了，this指向Listener，这里实际需要传入MainActivity.this,所以代码会报错。

实际使用中需要注意区分，看当前处于哪个类中，然后就可以确定this所指定的对象了。

好了，就这些，虽然简单，对初学者可能需要加强理解。

写的不准确的地方，请指出哈！

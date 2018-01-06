---
layout: post
title: "Singleton Design Pattern"
date: 2018-01-06
description: "学习设计模式，提升内功"
tag: Java
---

### 1. 什么是单例模式？
单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。
这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

```Java

注意：
1、单例类只能有一个实例。
2、单例类必须自己创建自己的唯一实例。
3、单例类必须给所有其他对象提供这一实例。
```


```Java

意图：保证一个类仅有一个实例，并提供一个访问它的全局访问点。

```

```java

主要解决：一个全局使用的类频繁地创建与销毁,节省系统资源的时候。

```

```java

关键代码：构造函数是私有的。

```

```Java

应用实例： 比如 I/O 读写、数据库连接等

```

```java

优点：
1、在内存里只有一个实例，减少了内存的开销;
2、避免对资源的多重占用（比如写文件操作）。


缺点：
1.没有接口，不能继承，与单一职责原则冲突;
2.一个类应该只关心内部逻辑，而不关心外面怎么样来实例化。

```

### 2. 单例模式的几种类型

#### (1) 懒汉式

单例对象初始化时赋值为 null

```Java

public class Singleton {
    private Singleton() {}  //私有构造函数
    private static Singleton instance = null;  //单例对象
    //静态工厂方法
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}

```

#### (2) 饿汉式

顾名思义，饿汉式，就是在类加载时就创建实例。但是他有一个缺点，如果你需要在传进来参数时，这种方式就不适合，可以换成懒汉式，在 getInstance() 方法和构造函数中都加上构造参数。

```Java
public class Singleton {
    private Singleton() {}  //私有构造函数
    private static final Singleton instance = new Singleton();  //单例对象
    //静态工厂方法
    public static Singleton getInstance() {
        return instance;
    }
}
```

#### (3) 线程安全

>使用双重检验锁

>使用 volatile 关键字限制 JVM 指令重排序


在synchronized作用域中，JVM主要做了三件事：

1、在堆空间里分配一部分空间；

2、执行FileIO的构造方法进行初始化；

3、把fileIO对象指向在堆空间里分配好的空间。

如果不加 volatile 关键字，JVM 为了优化性能重排指令的话，如，线程 1 按 1-3-2 顺序，线程 2 可能会得到一个没有初始化话的实例，导致崩溃。


```Java

public class Singleton {

   private Singleton() {}  //私有构造函数
   private static volatile Singleton instance = null;  //单例对象
   //静态工厂方法
   public static Singleton getInstance() {
        if (instance == null) {      //双重检测机制
         synchronized (Singleton.class){  //同步锁
           if (instance == null) {     //双重检测机制
             instance = new Singleton();
               }
            }
         }
        return instance;
    }
}

```

#### (4) 线程安全-静态内部类


> 既保证线程安全，又保证具有懒汉式的风格

> 优化类加载


```Java
public class Singleton{

	private static final class SingletonHolder{
			private static final Singleton INSTANCE = new Singleton();
	}
	private Singleton(){}

	public static Singleton getInstance(){
			return SingletonHolder.INSTANCE;
	}
}

```

#### (5) 线程安全-静态内部类-反序列化


> 既保证线程安全，又保证具有懒汉式的风格

> 优化类加载

> 反序列化


```Java

public class Singleton{

	private static final class SingletonHolder{
			private static final Singleton INSTANCE = new Singleton();
	}
	private Singleton(){}

	public static Singleton getInstance(){
			return SingletonHolder.INSTANCE;
	}

	private Object readResolve(){
		return SingletonHolder.INSTANCE;
	}
}

```


#### (6) 枚举法


> 保证线程安全，防止通过反射创建多个实例

> 缺点：枚举被加载时就进行初始化


```Java

public enum Singleton {  
    INSTANCE;  
    public void whateverMethod() {  
    }  
}  

```

### 3. 总结

1 和2 方式在没有线程安全要求时可以使用，3,4,5,6 均能够保证线程安全，对于有反序列化要求可以使用 5 和 6 .

对于5不熟悉，可以使用6，简单明了，一步到位，只不过牺牲一点加载时的时间。


### 参考


[菜鸟教程之单例模式](http://www.runoob.com/design-pattern/singleton-pattern.html)

[码农翻身之单例模式](https://mp.weixin.qq.com/s/q7-GWt87S7uJ9NY-s1z4SA)

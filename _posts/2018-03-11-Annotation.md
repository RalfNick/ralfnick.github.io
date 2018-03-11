---
layout: post
title: "Annotation"
date: 2018-03-11
description: "基础学习"
tag: Java
---

在android中有很多开源框架，比如Retrofit，Butter Knife，ActiveAndroid等都有些自己定义的注解，实用到了Annotion，中文译为“注解”，所以很有必要学习下注解，要不框架源码很不容易看懂
，另外，因为注解中用到了反射，所以可以对反射不熟悉的同学先去了解下反射的基本用法。

### 1. 什么是Annotation？

官方说法：

> An annotation is a form of metadata, that can be added to Java source code.
> Classes, methods, variables, parameters and packages may be annotated. Annotations have no direct effect on the operation of the code they annotate.

>注解是一种源数据，能够添加到java源代码上，类、方法、变量、参数、包名都能够被注解。注解对代码的执行没有直接影响。
>

乍一听，很难懂，我也不懂，哈哈。看了篇博客 [**秒懂，Java 注解 （Annotation）你可以这样学**](http://blog.csdn.net/briblue/article/details/73824058)，比喻的不错，将注解比喻成标签，比
如,将标签比喻成一个贴了标签的瓶子，标签对瓶子没有影响，瓶子还是瓶子，拿到瓶子我看看上面有没有标签，有标签(注解了)，那我就可以根据标签做一些事情（反射操作），比如，看看标签上贴了哪些文字，写了什么
，甚至可以根据做一些对瓶子的操作，向瓶子里加点水之类。
![注解-瓶子](http://p0c1ordis.bkt.clouddn.com/%E6%B3%A8%E8%A7%A31.jpg)

个人的一点理解：结合反射机制，注解应该算是一个标记，该标记可能记录了该注解的包名，让后通过发射就可以获取到该注解的信息，进行做一些其他操作，待之后深入分析注解的原理。

注解的作用：

>* a. 标记，用于告诉编译器一些信息

>* b. 编译时动态处理，如动态生成代码

>* c. 运行时动态处理，如得到注解信息

注解的基本语法：注意和接口的区别，interface前面多了一个@

```Java
public @interface MyAnnotation {
}
```

### 2. 注解的类型

**（1）元注解**

元注解是什么？作用是啥？

其实元注解是一种基本注解，用来对注解进行注解的注解，可能有点绕，说白了，它只能用于对注解进行注解，记住这一点就够了。

元注解一共有5种:

**@Retention**

该注解是声明注解的生命周期，它的取值如下：

- RetentionPolicy.SOURCE 注解只在源码阶段保留，在编译器进行编译时它将被丢弃忽视。

- RetentionPolicy.CLASS 注解只被保留到编译进行的时候，它并不会被加载到 JVM 中。

- RetentionPolicy.RUNTIME 注解可以保留到程序运行的时候，它会被加载进入到 JVM 中，所以在程序运行时可以获取到它们。

**@Documented**

 作用是告诉JavaDoc工具，当前注解本身也要显示在Java Doc中

**@Target**

 该注解是说明注解的目的，用标签形容的话，就是你希望对那些事物进行注解，为哪些东西贴上标签

 @Target 有下面的取值：

- ElementType.ANNOTATION_TYPE 可以给一个注解进行注解，
- ElementType.CONSTRUCTOR 可以给构造方法进行注解
- ElementType.FIELD 可以给属性进行注解
- ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
- ElementType.METHOD 可以给方法进行注解
- ElementType.PACKAGE 可以给一个包进行注解
- ElementType.PARAMETER 可以给一个方法内的参数进行注解
- ElementType.TYPE 可以给一个类型进行注解，比如类、接口、枚举

**注意：**

1.ElementType.ANNOTATION_TYPE:元注解类型，只能用来注解其它的注解

2.ElementType.TYPE：可以用来注解任何类型的java类，如类、接口、枚举



**@Inherited**

Inherited是继承的意思，并是说该注解可以继承，而是用@Inherited继承的类可以继承，如果子类没有被注解
的话那么它继承父类的注解。

下面用代码表示一下：
```Java
@Inherited
public @interface MyAnnotation {

}

//fu类
@MyAnnotation
public class Fu{}

//zi类
public class Zi extends Fu{}
```

此时，Zi类就有了Fu类的注解属性了

**@Repeatable**

Repeatable 是重复的意思，一般是注解的值可以取多个的时候使用@Repeatable注解

```Java

//福袋
@interface FuDai {
    FuZi[]  value();
}

//福字
@Repeatable(FuDai.class)
@interface FuZi{
    String what default "";
}


//万能福
@FuZi(what="jingye")
@FuZi(what="aiguo")
@FuZi(what="hexie")
@FuZi(what="fuqiang")
@FuZi(what="youshan")
public class WanNengFu{
}
```
上面的代码中类FuZi被注解成@Repeatable，后面括号中的类，相当于注解的容器，
注解容器意思就是里面可以有多个注解，所以注解中变量value是一个数组类型，注解容器其本身也是一个注解。这里用过年前集五福类形容，
注解容器是一个福袋，福字本身是可重复的，万能福的话可以是任意一张福。这样应该更好理解@Repeatable了。


（2）自定义注解

自定义注解，上面也提到了，写法很简单，类似于接口，在interface前面加上@就可以了。不过，这里强调一下，**每个注解都需要有自己单独的一个文件，也就是说，不能再同一个文件中定义两个或多个注解。**

下面来看一下，注解中的一些属性，前面自定义的注解中没有设置属性，属性类似于成员变量，写法上有些不同。

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {

    int id();

    String msg();
}
```
这样，就在注解中定义了两个属性，int类型和String类型，写法上有些类似方法，但注意区分，括号中没有参数。使用时，需要在注解后给属性赋值。

```Java
@MyAnnotation(id=1,msg="hello")
public class Test{
}

```

```Java

注解中的属性还可以设置默认值，在属性后面使用default 设置默认值，这样在使用注解时，就可以不用属性赋值，当然赋值也是可以的，根据需求来决定。

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {

    int id() default 0;

    String msg() default "";
}
```

```Java
//属性赋值步骤可以省略
@MyAnnotation()
public class Test{
}


另外，如果只有一个属性，在赋值时可以不写属性名，直接在括号中写上值即可

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {

    int id() default 0;
}
```

```Java
@MyAnnotation(1)
public class Test{
}
```

如果注解中没有属性，那么注解后面的括号也可以省略掉。

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface MyAnnotation {
}
```

```Java
@MyAnnotation
public class Test{
}
```


（3）内置注解

**@Deprecated**
>这个元素是用来标记过时的元素，想必大家在日常开发中经常碰到。编译器在编译阶段遇到这个注解时会发出提醒警告，告诉开发者正在调用一个过时的元素比如过时的方法、过时的类、过时的成员变量。

**@Override**
>提示子类要复写父类中被 @Override 修饰的方法

**@SuppressWarnings**
>阻止警告的意思。有时编译器会警告提醒，而有时候开发者会忽略这种警告，他们可以在调用的地方通过 @SuppressWarnings 达到目的。

**@SafeVarargs**
>参数安全类型注解。它的目的是提醒开发者不要用参数做一些不安全的操作,它的存在会阻止编译器产生 unchecked 这样的警告。它是在 Java 1.7 的版本中加入的。

**@FunctionalInterface**

函数式接口注解，这个是 Java 1.8 版本引入的新特性。

函数式接口 (Functional Interface) 就是一个具有一个方法的普通接口。
使用函数式接口注解，可以转成Lambda 表达式。

### 注解的获取

有了注解，可以是注解了方法，属性，或者一个类，那么注解之后可以做些什么呢，怎么获取你注解的属性，方法或者类呢？

答案是通过**反射**

> 要想获取一个注解，需要先判断一个类是否被注解，通过调用Class对象的isAnnotationPresent() 方法判断它是否应用了某个注解

```Java
public boolean isAnnotationPresent(Class<? extends Annotation> annotationClass)

```

>然后通过 getAnnotation() 方法来获取 Annotation 对象。

```Java
 public <A extends Annotation> A getAnnotation(Class<A> annotationClass)
```

>或者通过getAnnotations() 方法。

下面用实例验证一下：

```Java
//类注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface TestAnnotation{
  int id() default -1;
  String msg default "";
}
```

```Java
//方法注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Perform{

}
```

```Java
//字段注解
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Check{
  int value();
}
```

```Java
@TestAnnotation(msg="hello")
public class Test {

    @Check(value=1)
    int var;

    @Perform
    public void testMethod(){}

    public static void main(String[] args) {

        boolean isAnnotation = Test.class.isAnnotationPresent(TestAnnotation.class);

        if ( isAnnotation ) {
            TestAnnotation testAnnotation = Test.class.getAnnotation(TestAnnotation.class);
            //获取类的注解
            System.out.println("id:"+testAnnotation.id());
            System.out.println("msg:"+testAnnotation.msg());
        }


        try {
            Field a = Test.class.getDeclaredField("a");
            a.setAccessible(true);
            //获取一个成员变量上的注解
            Check check = a.getAnnotation(Check.class);

            if ( check != null ) {
                System.out.println("check value:"+check.value());
            }

            Method testMethod = Test.class.getDeclaredMethod("testMethod");

            if ( testMethod != null ) {
                // 获取方法中的注解
                Annotation[] ans = testMethod.getAnnotations();
                for( int i = 0;i < ans.length;i++) {
                    System.out.println("method testMethod annotation:"+ans[i].annotationType().getSimpleName());
                }
            }
        } catch (NoSuchFieldException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            System.out.println(e.getMessage());
        } catch (SecurityException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            System.out.println(e.getMessage());
        } catch (NoSuchMethodException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
            System.out.println(e.getMessage());
        }
    }

}

```

### 总结

对注解有了基本的了解，包括注解的类型，元注解，自定义注解，java内置的几种注解，在掌握注解的基本写法后，对注解的提取简单介绍了利用反射的操作。有了这些后面对一些框架源码的分析会有很大帮助。

实际上，对注解的方法，字段，或者类，就可以在与编译时期，运行时期做些操作，框架的原理也是基于此进行的操作，由于注解提取是通过反射进行的，反射会使得运行效率降低，但是优化好了，同样很好用，像ButterKnife，Retrofit等框架，这些是需要我们学习的地方，后面在对一些开源框架进行分析！

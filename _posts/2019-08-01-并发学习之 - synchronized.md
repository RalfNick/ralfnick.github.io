---
layout: post
title: "并发学习之 - synchronized"
date: 2019-08-01
description: "并发学习之 - synchronized"
tag: 并发
---
上一篇文章 [**并发基础知识扫盲**](https://www.jianshu.com/p/5e3529b36196) 讲了一些 java 中 并发相关的基础性的东西，这篇来了解下同步中常使用的关键字 synchronized。

synchronized 关键字是随着 Java 的诞生就有的的，它对于开发者来说，使用起来非常方便，无需关心底层的复杂实现。但是在使用过程中开发者往往担心 synchronized 带来的性能问题，认为它太重了，获得锁和释放锁的确会带来性能上的消耗。在 Java SE 1.6 之后，synchronized 进行了很大的优化，引入了偏性锁和轻量级锁，来减少获取锁和释放锁带来的性能问题，所以一般要求同步操作时，建议使用 synchronized。

### 1. synchronized 用法

synchronized 使用主要就是 3 种形式：

> - 普通同步方法，锁是当前实例对象
>
> - 静态同步方法，锁是当前类的 Class 对象
>
> - 同步代码块，锁是 synchronized 括号中的对象

下面看下不同步的情况：

```java
public class ThreadSync {
private static class NoSyncRunnable implements Runnable {
private int count = 1000;

@Override
public void run() {
while (count >= 0) {
System.out.println(Thread.currentThread().getName() + " --- " + count);
count--;
}
}
}
public static void main(String[] args) throws InterruptedException {        testNoSyncMethod();
}

private static void testNoSyncMethod() throws InterruptedException {
NoSyncRunnable noSyncRunnable = new NoSyncRunnable();
Thread thread1 = new Thread(noSyncRunnable);
Thread thread2 = new Thread(noSyncRunnable);
thread1.start();
thread2.start();
thread1.join();
thread2.join();
}

```
这个未加同步的例子中，我们期望 count 能够从 1000 一直减少到 0，但实际结果中可能会出现一个线程打印的值，会大于或者等于前一个线程给出的值。出现这种情况主要是由于缓存的原因，在上一篇文章 [**并发基础知识扫盲**](https://www.jianshu.com/p/5e3529b36196) 已经提到，由于Java 虚拟机的工作内存形式，线程首先会从工作内存中取值，所以读取值和修改值是基于工作内存，并未及时更新到主内存中，所以不能保证线程之间的可见性。要想保证线程间同步，需要满足同步的 3 个条件：原子性、可见性和顺序性，synchronized 能够保证这三个条件。

```java
Thread-1 --- 1000
Thread-1 --- 999
Thread-1 --- 998
Thread-1 --- 997
Thread-1 --- 996
Thread-1 --- 995
Thread-1 --- 994
Thread-1 --- 993
Thread-1 --- 992
Thread-0 --- 1000
Thread-0 --- 990
Thread-0 --- 989
Thread-0 --- 988
Thread-0 --- 987
Thread-0 --- 986
Thread-0 --- 985
Thread-0 --- 984
...
```
使用同步操作

```java
private static class SyncRunnable implements Runnable {

private static int count = 10;
// 特殊的instance变量
// 零长度的byte数组对象创建起来将比任何对象都经济――查看编译后的字节码：
// 生成零长度的byte[]对象只需3条操作码，而Object lock = new Object()则需要7行操作码。
private final byte[] lock = new byte[0];

@Override
public void run() {
// 普通同步方法
//            countMethod0();
// 同步代码块 this 锁
//            countMethod1();
// 同步代码块 非 this 锁
//            countMethod2();
// 静态同步方法
//            countMethod3();
// 同步代码块 类的 Class 对象
countMethod4();
}

/**
* 同步方法
*/
private synchronized void countMethod0() {
for (int i = 0; i < 5; i++) {
System.out.println(Thread.currentThread().getName() + " --- " + count);
count--;
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}


/**
* 同步代码块（this 锁）
*/
private void countMethod1() {
synchronized (this) {
for (int i = 0; i < 5; i++) {
System.out.println(Thread.currentThread().getName() + " --- " + count);
count--;
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}

/**
* 同步代码块（非 this 锁）
*/
private void countMethod2() {
synchronized (lock) {
for (int i = 0; i < 5; i++) {
System.out.println(Thread.currentThread().getName() + " --- " + count);
count--;
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}

/**
* 静态同步方法
*/
private static synchronized void countMethod3() {
for (int i = 0; i < 5; i++) {
System.out.println(Thread.currentThread().getName() + " --- " + count);
count--;
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}

/**
* 同步代码块（类的 Class 对象锁）
*/
private static void countMethod4() {
synchronized (SyncRunnable.class) {
for (int i = 0; i < 5; i++) {
System.out.println(Thread.currentThread().getName() + " --- " + count);
count--;
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}

}
```

上述几个操作都能够保证线程之间的同步操作，其中，

（1）countMethod0 是普通同步方法，它获取的当前 SyncRunnable 实例对象的锁，countMethod1 是同步代码块，同样是当前 SyncRunnable 实例对象的锁，并且 countMethod1 和 countMethod0 等同，因为 countMethod1 同步代码块包裹的范围是整个方法。同步代码块尽可能的保证同步位置的最小化，这样能够提高线程间的运行效率，使用更加灵活。

（2）方法 countMethod2 是一个同步代码块，获取的锁对象是非 this 锁，注释中给出一个提示，使用 byte[] 对象只需 3 条操作码，而Object lock = new Object() 则需要 7 行操作码，所以使用 byte[] 对象 更加经济。使用非 this 对象锁的优势，如果是同步方法或者 锁是 this 的代码块，会阻塞其他同步方法和 this 的代码块，而 非 this 锁不会阻塞同步方法和 this 的代码块，因为是不同的锁对象。

（3）方法 countMethod3 是一个静态同步方法，它的锁是 SyncRunnable 的 Class 对象锁，相当于 countMethod4，属于类的同步方法，这个锁和普通同步方法的锁是不一样的。

### 2. synchronized 使用细节

上面演示了 synchronized 的 3 种用法，下面看看 synchronized 使用过程中的几种阻塞情况。

阻塞和非阻塞关注的是程序在等待调用结果（消息，返回值）时的状态。阻塞调用是指调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会返回。非阻塞调用指在不能立刻得到结果之前，该调用不会阻塞当前线程。

使用 synchronized 同步时，一个线程没有获取到锁，就是处于阻塞状态。

这里演示的例子，通过建立一个同步类，里面包括各种同步方法，普通同步方法，同步代码块（this 锁和非 this 锁）、静态同步方法，然后通过建立不同的线程，来演示这些方法之间组合时的阻塞情况。

同步类 SyncFun，包含多种同步方法。

```java
public class SyncFun {

private final Object lock = new Object();

public synchronized void print0() {
int i = 0;
while (i < 10) {
System.out.println("SyncFun - 同步方法 - " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}

public synchronized void print1() {
int i = 0;
while (i < 10) {
System.out.println("SyncFun - 同步方法 - " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}

public void print2() {
synchronized (this) {
int i = 0;
while (i < 10) {
System.out.println("SyncFun - 同步代码块 this - " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}

public void print3() {
synchronized (lock) {
int i = 0;
while (i < 5) {
System.out.println("SyncFun - 同步代码块 lock- " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}

public void print4(Object o) {
synchronized (o) {
int i = 0;
while (i < 10) {
System.out.println("SyncFun - 同步方法 参数 lock - " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}

public static synchronized void print5() {
int i = 0;
while (i < 10) {
System.out.println("SyncFun - 类同步方法 - " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}

public static synchronized void print6() {
int i = 0;
while (i < 10) {
System.out.println("SyncFun - 类同步方法 - " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}

public void print7() {
synchronized (SyncFun.class) {
int i = 0;
while (i < 10) {
System.out.println("SyncFun - 类同步代码块- 类锁 - " + Thread.currentThread().getName() + " - " + i++);
try {
Thread.sleep(100);
} catch (InterruptedException e) {
e.printStackTrace();
}
}
}
}
}

```

这里就直接把代码贴出来，主要针对普通同步方法，不同对象锁的同步代码块，和静态同步方法这几种情况的组合来进行实验。

```java
public class ThreadSync1 {

private static class Runnable0 implements Runnable {

private SyncFun syncFun;

public Runnable0(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
syncFun.print0();
}
}

private static class Runnable1 implements Runnable {

private SyncFun syncFun;

public Runnable1(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
syncFun.print1();
}
}

private static class Runnable2 implements Runnable {

private SyncFun syncFun;

public Runnable2(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
syncFun.print2();
}
}

private static class Runnable3 implements Runnable {

private SyncFun syncFun;

public Runnable3(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
syncFun.print3();
}
}

private static class Runnable4 implements Runnable {

private SyncFun syncFun;
private static final Object o = new Object();

public Runnable4(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
syncFun.print4(o);
}
}

private static class Runnable5 implements Runnable {

private SyncFun syncFun;

public Runnable5(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
SyncFun.print5();
}
}

private static class Runnable6 implements Runnable {

private SyncFun syncFun;

public Runnable6(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
SyncFun.print6();
}
}

private static class Runnable7 implements Runnable {

private SyncFun syncFun;

public Runnable7(SyncFun syncFun) {
this.syncFun = syncFun;
}

@Override
public void run() {
syncFun.print7();
}
}


public static void main(String[] args) throws InterruptedException {
// （1）不同对象的同一个同步方法- 不阻塞
//        test1_0();
// （2）同一个对象的不同的同步方法- 阻塞
//        test1_1();
// （3）访问同一个同步方法- 阻塞
//        test1_2();
// （4）访问同步方法 和 this 锁- 阻塞
//        test1_3();
// （5）访问同一个非 this 锁的同步代码块 - 阻塞
//        test1_4();
// （6）同步方法和非 this 对象 - 不阻塞
//        test1_5();
// （7）同步方法和参数 lock,和非 this 对象锁一样 - 不阻塞
//        test1_6();
// （8）访问不同静态同步方法， - 阻塞
//        test6();
// （9）访问静态同步方法和该类对象的同步代码块（类锁）- 阻塞
//        test7();
// （10）同步方法和静态同步方法, - 不阻塞
//        test8();
// （11）不同对象访问不同静态同步方法, - 阻塞
test9();
}

private static void test1_0() throws InterruptedException {
SyncFun syncFun0 = new SyncFun();
SyncFun syncFun1 = new SyncFun();
Thread thread0 = new Thread(new Runnable0(syncFun0));
Thread thread1 = new Thread(new Runnable0(syncFun1));

thread0.start();
thread1.start();

Thread.sleep(3000);
}


private static void test1_1() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable0(syncFun));
Thread thread2 = new Thread(new Runnable1(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test1_2() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable1(syncFun));
Thread thread2 = new Thread(new Runnable1(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test1_3() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable1(syncFun));
Thread thread2 = new Thread(new Runnable2(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test1_4() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable3(syncFun));
Thread thread2 = new Thread(new Runnable3(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test1_5() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable1(syncFun));
Thread thread2 = new Thread(new Runnable3(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test1_6() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable1(syncFun));
Thread thread2 = new Thread(new Runnable4(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test6() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable5(syncFun));
Thread thread2 = new Thread(new Runnable6(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test7() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread1 = new Thread(new Runnable5(syncFun));
Thread thread2 = new Thread(new Runnable7(syncFun));

thread1.start();
thread2.start();

Thread.sleep(3000);
}

private static void test8() throws InterruptedException {
SyncFun syncFun = new SyncFun();
Thread thread2 = new Thread(new Runnable5(syncFun));
Thread thread1 = new Thread(new Runnable1(syncFun));

thread2.start();
thread1.start();

Thread.sleep(3000);
}

private static void test9() throws InterruptedException {
SyncFun syncFun1 = new SyncFun();
SyncFun syncFun2 = new SyncFun();
Thread thread1 = new Thread(new Runnable5(syncFun1));
Thread thread2 = new Thread(new Runnable6(syncFun2));

thread1.start();
thread2.start();

Thread.sleep(3000);
}
}
```

（1）、（2）和（3）好理解，（1）调用不同对象的同步方法，自然不会阻塞，因为获取的不是同一个锁，（2）和（3）调用同一个对象的同步方法，无论是同一个同步方法还是不同的同步方法，都会阻塞，因为一个对象的同步方法的锁对象时 this，所以同一个对象的同步方法都是互斥的。

（4）访问同一个对象的同步方法和 this 代码块，会阻塞。因为当前对象的同步方法和 this 代码块锁的都是当前对象，所以会互斥。

（5）访问同一个对象的非 this 同步代码块也会阻塞，这个没什么解释的，仅仅为了说明使用非 this 锁也是阻塞的。

（6）访问同一个对象的同步方法和非 this 对象代码块，不会阻塞，因为不是同一个锁对象，也就不会阻塞。

（7）访问同一个对象的同步方法和参数传递的锁对象,和（6）中非 this 对象锁一样，同样不阻塞。

（8）访问一个类的不同静态同步方法，会阻塞，这是锁对象是这个类的 Class 对象，所以所有的静态同步方法会互斥，同时调用时会进行阻塞。

（9）访问静态同步方法和该类对象的同步代码块（Class 对象锁），会阻塞，此时锁对象是同一个，都是 Class 对象锁，所以会阻塞。


（10）访问一个对象的同步方法和该类静态同步方法，不会阻塞，这个可能有些疑惑，但是只要找到这两个方法的锁，就很容易理解了，对象的同步方法的锁是 this，该类静态同步方法的锁是该类的 Class 对象，属于不同的锁，所以同时调用时不会阻塞。

（11）不同对象访问不同静态同步方法，会阻塞，此时与对象没有关系，因为是静态方法，属于类的，相当于直接调用类静态的同步方法，会阻塞。

### 3.总结

1、synchronized 同步方法

> (1)对其他 synchronized 同步方法或 synchronized(this) 同步代码块呈阻塞状态
>
> (2)同一时间只有一个线程可以执行 synchronized 同步方法中的代码

2、synchronized 同步代码块

> (1)对其他 synchronized 同步方法或 synchronized(this) 同步代码块呈阻塞状态
>
> (2)同一时间只有一个线程可以执行 synchronized(this) 同步代码块中的代码
>
> (3)当一个线程访问对象的 synchronized 代码块的时候，另一个线程依然可以访问对象方法中其余非 synchronized 块的部分


3、synchronized 同步方法 VS synchronized 同步代码块

(1) 锁非 this 对象具有一定的优点：如果在一个类中有很多 synchronized 方法，这时虽然能实现同步，但会受到阻塞，从而影响效率。但如果同步代码块锁的是非 this 对象，则 synchronized(非 this 对象 x)代码块中的程序与同步方法是异步的，不与其他锁 this 同步方法争抢 this 锁，大大提高了运行效率。

(2) synchronized(非 this 对象 x) 格式的写法是将x对象本身作为对象监视器，有三个结论得出：

> 当多个线程同时执行 synchronized(x){} 同步代码块时呈同步效果
>
> 当其他线程执行 x 对象中的 synchronized 同步方法时呈同步效果
>
> 当其他线程执行 x 对象方法中的 synchronized(this) 代码块时也呈同步效果
>
> synchronized(非 this 对象 x)，这个对象如果是实例变量的话，指的是对象的引用，只要对象的引用不变，即使改变了对象的属性，运行结果依然是同步的。

总结：无论是方法锁还是代码锁都是要以一个对象监视器来锁定，锁定的代码是同步的，锁 this 是当前对象，锁 String 是 String 这个对象，锁 Object 是 Object 这个对象，互不干扰，如果有其它线程调用同样用到跟上面锁 this、Objcet、String 相同对象的方法或代码，就需要等待同步，锁代码块比锁方法更加灵活。因为锁方法锁的是 this 也就是当前对象,当一个线程正在调用当前这个对象的锁方法时，导致其它线程调用不了该对象的其它锁 this 的代码，也调不了所有该对象的锁方法。


### 参考

[静态方法加锁，和非静态方法加锁区别](https://www.jianshu.com/p/8327c5c15cb8)

[java 多线程9 : synchronized锁机制 之 代码块锁](https://www.cnblogs.com/signheart/p/0a8548258725cb8812768d2b3e1a2aef.html)

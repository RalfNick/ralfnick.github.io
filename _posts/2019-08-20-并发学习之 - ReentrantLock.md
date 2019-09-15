---
layout: post
title: "并发学习之 - ReentrantLock"
date: 2019-08-20
description: "并发学习之 - ReentrantLock"
tag: 并发
---
### synchronized 和 ReentrantLock

[上一篇文章 并发学习之 - synchronized](https://www.jianshu.com/p/03332e66f27b) 中我们讲解了如何使用关键字 synchronized 来实现同步访问。从 Java 5 之后，JDK 提供了另外一种方式来实现同步访问，那就是 ReentrantLock。ReentrantLock 增加了一些高级功能，主要是这 3 项：等待可中断、可实现公平锁，以及锁可以绑定多个条件。

在 JDK 6 之前，尽管 synchronized 关键字实现同步很方便，但是这种同步操作很重量级，很大程度上影响程序的执行效率，所以对于开发者来说，使用起来会有点畏惧。但是在 JDK 6 之后，对 synchronized 进行了很大的优化， 引入了偏向锁，适应性自旋，轻量级锁和重量级锁，锁粗化等手段，目前 synchronized 和 ReentrantLock 的性能基本能够持平，所以一般情况下，还是建议使用 synchronized 关键字，会使得程序简洁和易读。

JDK 6 之前

![jdk6before](https://github.com/RalfNick/PicRepository/raw/master/Thread/synchronized_jdk5_before.jpeg)

JDK 6 之后

![jdk6later](https://github.com/RalfNick/PicRepository/raw/master/Thread/synchronized_jdk5_later.jpeg)

但是 ReentrantLock 也有很多优点，ReentrantLock 是 Lock 的实现类，是一个互斥的同步器，在多线程高竞争条件下，ReentrantLock 比 synchronized 有更加优异的性能表现。

(1) Lock 使用起来比较灵活，但是必须有释放锁的配合动作
Lock 必须手动获取与释放锁，而 synchronized 不需要手动释放和开启锁 Lock 只适用于代码块锁，而 synchronized 可用于修饰方法、代码块等。

(2) ReentrantLock的优势体现在：
具备尝试非阻塞地获取锁的特性：当前线程尝试获取锁，如果这一时刻锁没有被其他线程获取到，则成功获取并持有锁
能被中断地获取锁的特性：与synchronized不同，获取到锁的线程能够响应中断，当获取到锁的线程被中断时，中断异常将会被抛出，同时锁会被释放
超时获取锁的特性：在指定的时间范围内获取锁；如果截止时间到了仍然无法获取锁，则返回

(3) 在使用 ReentrantLock 时，要注意:在 finally 中释放锁，目的是保证在获取锁之后，最终能够被释放
不要将获取锁的过程写在 try 块内，因为如果在获取锁时发生了异常，异常抛出的同时，也会导致锁无法被释放。

(4) ReentrantLock 提供了一个 newCondition 的方法，以便用户在同一锁的情况下可以根据不同的情况执行等待或唤醒的动作。

### 2. ReentrantLock 用法

ReentrantLock 是 Lock 接口的实现类，Lock 提供了以下几个方法：

```java
// 获取锁，获取过程中不可中断
void lock();

// 获取锁，获取过程中可中断，中断后抛出 InterruptedException 异常
void lockInterruptibly() throws InterruptedException;

// 非阻塞获取锁，无论是否获取到均返回，true 获取成功，false 获取失败
boolean tryLock();

// 尝试获取锁，在 time 时间内获取到返回 true，超时则返回 false
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

// 释放锁，一般在 finally 块中
void unlock();

// 创建等待条件
Condition newCondition();

```
#### 2.1 Lock 锁使用

ReentrantLock 锁使用起来也比较简单，主要根据场景来进行选择，有一点需要注意，如果不是使用 tryLock 的方法，需要在 finally 块中释放锁。下面给出几种获取锁的使用示例代码。


```java
public class ReentrantLockTest {

private static final ReentrantLock reentrantLock = new ReentrantLock();
private static int sCount = 0;

/**
* lock
*/
private static class MyRunnable implements Runnable {

@Override
public void run() {
reentrantLock.lock();
try {
for (int i = 0; i < 5; i++) {
System.out.println(Thread.currentThread().getName() + " --- > " + sCount++);
}
Thread.sleep(3500);
} catch (Exception e) {

} finally {
reentrantLock.unlock();
}

}
}

/**
* tryLock
*/
private static class MyRunnable1 implements Runnable {

@Override
public void run() {
boolean isLock = reentrantLock.tryLock();
if (isLock) {
execute();
} else {
System.out.println(Thread.currentThread().getName() + " --- > 获取锁【失败】");
}
}
}

/**
* tryLock(long timeout, TimeUnit unit)，超时获取失败，中间可以被中断
*/
private static class MyRunnable2 implements Runnable {

@Override
public void run() {
boolean isLock;
try {
//                isLock = reentrantLock.tryLock(3, TimeUnit.SECONDS);
isLock = reentrantLock.tryLock(4, TimeUnit.SECONDS);
} catch (InterruptedException e) {
System.out.println(Thread.currentThread().getName() + " --- > 获取锁【被中断】");
return;
}
if (isLock) {
execute();
} else {
System.out.println(Thread.currentThread().getName() + " --- > 获取锁【失败】");
}
}
}

/**
* lockInterruptibly 中间可以被中断,如果不中断就一直尝试获取锁
*/
private static class MyRunnable3 implements Runnable {

@Override
public void run() {
try {
reentrantLock.lockInterruptibly();
execute();
} catch (InterruptedException e) {
System.out.println(Thread.currentThread().getName() + " --- > 获取锁【被中断】");
}
}
}

private static void execute() {
try {
for (int i = 0; i < 5; i++) {
System.out.println(Thread.currentThread().getName() + " --- > " + sCount++);
}
} catch (Exception e) {

} finally {
reentrantLock.unlock();
}
}

public static void main(String[] args) throws InterruptedException {
sCount = 0;
// lock
//        test1();
// tryLock
//        test2();
// tryLock(long timeout, TimeUnit unit)
//        test3();
// lockInterruptibly
test4();
}

private static void test1() {
MyRunnable runnable = new MyRunnable();
Thread thread1 = new Thread(runnable);
Thread thread2 = new Thread(runnable);

thread1.start();
thread2.start();
}

private static void test2() {
Thread thread1 = new Thread(new MyRunnable());
Thread thread2 = new Thread(new MyRunnable1());

thread1.start();
thread2.start();
}

private static void test3() {
Thread thread1 = new Thread(new MyRunnable());
Thread thread2 = new Thread(new MyRunnable2());

thread1.start();
thread2.start();

// 中断测试
thread1.interrupt();
thread2.interrupt();
}

private static void test4() throws InterruptedException {
Thread thread1 = new Thread(new MyRunnable());
Thread thread2 = new Thread(new MyRunnable3());

thread1.start();
thread2.start();

Thread.sleep(2000);
// 中断测试
thread1.interrupt();
thread2.interrupt();
}
}
```

#### 2.2 Condition 使用

对应一个 ReentrantLock 锁，可以有多个条件，这是 synchronized 不具备的灵活性。synchronized 中方法或者块中，只能使用锁的 wait() 和 notify()、notifyAll()。而 ReentrantLock 的一个锁可以有多个条件，ReentrantLock 的一个锁对应一个同步队列，多个条件对应多个等待队列。下面使用 ReentrantLock 的条件实现生产者 - 消费者模型。

```java
public class ProducerConsumer1 {

private static final ReentrantLock sLock = new ReentrantLock();
private static final Condition sProducerCondition = sLock.newCondition();
private static final Condition sConsumerCondition = sLock.newCondition();

private static final int MAX = 10;
private static int count = 0;

private static class Producer implements Runnable {

@Override
public void run() {
while (!Thread.currentThread().isInterrupted()) {
sLock.lock();
try {
if (count == MAX) {
sProducerCondition.await();
}
count++;
System.out.println("生产者 ---- > " + count);
Thread.sleep(500);
sConsumerCondition.signal();
} catch (InterruptedException e) {
e.printStackTrace();
} finally {
sLock.unlock();
}
}
}
}

private static class Consumer implements Runnable {

@Override
public void run() {
while (!Thread.currentThread().isInterrupted()) {
sLock.lock();
try {
if (count == 0) {
sConsumerCondition.await();
}
count--;
System.out.println("消费者 ---- > " + count);
Thread.sleep(300);
sProducerCondition.signal();
} catch (InterruptedException e) {
e.printStackTrace();
} finally {
sLock.unlock();
}
}
}
}

public static void main(String[] args) throws InterruptedException {
Thread producer = new Thread(new Producer());
Thread consumer = new Thread(new Consumer());
producer.start();
consumer.start();

Thread.sleep(10000);
}
}

```

#### 2.3 其他说明

ReetrantLock 和 synchronized 都是一种支持重进入的锁，即该锁可以支持一个线程对资源重复加锁。但是 synchronized 不支持公平锁，ReetrantLock 支持公平锁与非公平锁。公平与非公平指的是在请求先后顺序上，先对锁进行请求的就一定先获取到锁，那么这就是公平锁，反之，如果对于锁的获取并没有时间上的先后顺序，如后请求的线程可能先获取到锁，这就是非公平锁，一般而言非，非公平锁机制的效率往往会胜过公平锁的机制，但在某些场景下，可能更注重时间先后顺序，那么公平锁自然是很好的选择。需要注意的是 ReetrantLock 支持对同一线程重加锁，但是加锁多少次，就必须解锁多少次，这样才可以成功释放锁。

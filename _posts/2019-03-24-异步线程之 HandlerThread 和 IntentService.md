---
layout: post
title: "异步线程之 HandlerThread 和 IntentService"
date: 2019-03-24
description: "异步线程之 HandlerThread 和 IntentService"
tag: Handler
---
本篇主要讲解一下 HandlerThread 和 IntentService，其中 IntentService 内部使用了 HandlerThread，而 HandlerThread 是一个 Thread，内部使用到了 Handler 消息机制，对 Handler 消息机制还不熟悉的话，可以看看之前的两篇文章 [**消息机制 - Handler 使用**](https://www.jianshu.com/p/b88e32c2477b) 和 [**深入理解 Handler 消息机制**](https://www.jianshu.com/p/43e5bcf1c05a)

下面就来看下 HandlerThread 和 IntentService 的用法和原理分析。

### 1 HandlerThread

#### 1.1 HandlerThread 使用

首先继承 HandlerThread，创建一个类，其实可以不用继承，直接使用 HandlerThread也可以，这里主要为了看一下 Log 信息。

```java
public class MyHandlerThread extends HandlerThread {

private static final String TAG = MyHandlerThread.class.getSimpleName();

public MyHandlerThread(String name) {
super(name);
}

public MyHandlerThread(String name, int priority) {
super(name, priority);
}

@Override
protected void onLooperPrepared() {
super.onLooperPrepared();
Log.e(TAG,"onLooperPrepared");
}

@Override
public void run() {
Log.e(TAG,"run");
super.run();
}

@Override
public Looper getLooper() {
return super.getLooper();
}

@Override
public boolean quit() {
Log.e(TAG,"quit");
return super.quit();
}
}
```

在 MainActivity 中创建线程和 Handler

```java
private MyHandlerThread myHandlerThread;
private Handler mHandler;

private void createHandlerThread() {
if (myHandlerThread != null && myHandlerThread.isAlive()) {
return;
}
Log.e(TAG, "myHandlerThread is not alive,need to create one");
myHandlerThread = new MyHandlerThread("HandlerThread");
myHandlerThread.start();
mHandler = new Handler(myHandlerThread.getLooper());
}
```

**注意：** 这里创建线程时做判断，是因为一旦调用线程的 quit() 方法后，现场就结束了，里面的消息队列也就退出了。所以该线程也就无法使用，只能重新创建一个。

```java
// 开启线程按钮点击事件
// 创建线程
createHandlerThread();
// 发送消息
mHandler.postDelayed(new Runnable() {
@Override
public void run() {
Log.e(TAG, "start");
try {
Log.e(TAG, "正在执行...");
Thread.sleep(5000);
} catch (InterruptedException e) {
e.printStackTrace();
}
Log.e(TAG, "end");
}
}, 100);
```

所以 HandlerThread 使用比较简单：

**步骤**

> * (1) 创建 HandlerThread 和 Handler
> * (2) 通过 Handler 发送消息

**注意点**

> * (1) 调用线程的 quit()，线程销毁，若想使用，需要重新创建 HandlerThread 和 Handler
> * (2) 任务执行后，需要手动停止线程，即调用 quit() 或者 quitSafely(),节省资源

**疑问点**

> * (1) Handler 能否通过 HandlerThread 获取到，因为其内部有 Handler？

#### 1.2 HandlerThread 原理分析

```java
public class HandlerThread extends Thread {
int mPriority;
int mTid = -1;
Looper mLooper;
private @Nullable Handler mHandler;

public HandlerThread(String name) {
super(name);
mPriority = Process.THREAD_PRIORITY_DEFAULT;
}

public HandlerThread(String name, int priority) {
super(name);
mPriority = priority;
}

protected void onLooperPrepared() {
}

@Override
public void run() {
mTid = Process.myTid();
Looper.prepare();
synchronized (this) {
mLooper = Looper.myLooper();
notifyAll();
}
Process.setThreadPriority(mPriority);
onLooperPrepared();
Looper.loop();
mTid = -1;
}

// 获取当前线程的 Looper
public Looper getLooper() {
if (!isAlive()) {
return null;
}

// If the thread has been started, wait until the looper has been created.
// 因为此时还没走 run 方法时，mLooper 还没创建，所以需要等待，创建之后唤醒。
synchronized (this) {
while (isAlive() && mLooper == null) {
try {
wait();
} catch (InterruptedException e) {
}
}
}
return mLooper;
}

/**
* @return a shared {@link Handler} associated with this thread
* @hide  注意这个 hide 注解
*/
@NonNull
public Handler getThreadHandler() {
if (mHandler == null) {
mHandler = new Handler(getLooper());
}
return mHandler;
}

// 退出消息循环
public boolean quit() {
Looper looper = getLooper();
if (looper != null) {
looper.quit();
return true;
}
return false;
}

public boolean quitSafely() {
Looper looper = getLooper();
if (looper != null) {
looper.quitSafely();
return true;
}
return false;
}

...
```

HandlerThread 的源码比较少，主要理解了 Handler 机制，HandlerThread 也很简单。

(1) 开始时两个构造函数，没什么可说的，可以设置线程的名字，可以设置线程的优先级。

(2) 执行过程，当创建线程之后，只有调用了 Thread 的 start 方法后线程才开始进入就绪状态，当该线程获取 CPU 的时间片后，进入执行执行状态，进入 run 方法。

(3) 在 run 方法中和传统的线程有区别，不是直接执行一个 Runnable，而是创建当前线程 Looper，通过 Looper.prepare(); 创建 Looper 的同时也创建了 MessageQueue，这个 Looper 是保存在 ThreadLocal 中的，获取时也是通过 ThreadLocal 来获取。创建完之后，开始消息循环，Looper.loop()，这样就进入一个无限循环状态，当有消息进来时，就会处理消息，消息的执行过程是 Handler 的 handlerMessage 或者是 post 方法的 Runnable 的 run 方法。

(4) 注意一下，创建 Looper 和获取 Looper 方法中的同步机制，如果还没走 run 方法时就开始获取 Looper，mLooper 还没创建，所以需要等待，等待 Looper 创建之后唤醒。

(5) 回到上面的疑问点：HandlerThread 提供了 getThreadHandler() 方法，但是你可以试试，在 外面调用这个方法时，会报错，看不到这个方法，咋一看这个方法没问题，是 public 的，为啥不能调用？是谷歌在搞笑么？搜索了一下：

[看看这个解释](https://stackoverflow.com/questions/31908205/what-exactly-does-androids-hide-annotation-do)

里面的解释大概是：

> * private @Nullable Handler mHandler; 可能为空，但是这个解释有点牵强，方法中 已经做了非空判断
> * 方法中有个注解 @hide，解释中给出的是，因为引入时，会删除标有@hide的所有类，方法，字段等，估计是 Google 不希望我们直接调用这个方法，那能怎么办？自己创建 Handler 呗。


### 2 IntentService

#### 2.1 IntentService 使用

IntentService 内部使用了 HandlerThread 和 Handler，同时它继承了 Service，可以在子线程中执行任务，通过由于也成为四大组件之一，具有比普通线程的优先级，后台执行时不容易被杀死，所以 IntentService 比较适合后台执行优先级比较高的任务
。

IntentService 使用起来也比较简单,只需要继承 IntentService，重写 onHandleIntent() 这个抽象方法即可，这里重写其他方法，主要也是为了看 Log 信息，实际中可以不重写。

```java
public class MyIntentService extends IntentService {

private static final String TAG = MyIntentService.class.getSimpleName();

public MyIntentService() {
super(TAG);
}

public MyIntentService(String name) {
super(name);
}

@Override
public void onCreate() {
super.onCreate();
Log.e(TAG, "onCreate");
}

@Override
public void onStart(@Nullable Intent intent, int startId) {
super.onStart(intent, startId);
Log.e(TAG, "onStart");
}

@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
Log.e(TAG, "onStartCommand");
return super.onStartCommand(intent, flags, startId);
}

@Override
public void onDestroy() {
super.onDestroy();
Log.e(TAG, "onDestroy");
}

@Override
protected void onHandleIntent(@Nullable Intent intent) {
Log.e(TAG, "onHandleIntent");
String tag = intent != null ? intent.getStringExtra("tag") : "";
switch (tag) {
case "task1":
try {
Thread.sleep(3000);
Log.e(TAG, "task1");
} catch (InterruptedException e) {
e.printStackTrace();
}
break;
case "task2":
try {
Thread.sleep(3000);
Log.e(TAG, "task2");
} catch (InterruptedException e) {
e.printStackTrace();
}
break;
default:
break;
}

}
}
```
开启服务

```java
// 设置点击按钮，来开启两个后台任务
Intent intent1 = new Intent(MainActivity.this, MyIntentService.class);
intent1.putExtra("tag", "task1");
startService(intent1);

Intent intent2 = new Intent(MainActivity.this, MyIntentService.class);
intent2.putExtra("tag", "task2");
startService(intent2);
```

// 停止服务

```java
private void stopService() {
// 会把正在执行的任务完成
Intent intent = new Intent(this, MyIntentService.class);
this.stopService(intent);
}
```

别忘记在 AndroidManifest.xml 中注册服务

#### 2.2 IntentService 原理分析

主要分析两个问题：

(1) IntentService 在什么时候开启线程 HandlerThread？

(2) 如何接收任务和执行任务？

```java
public abstract class IntentService extends Service {

private volatile Looper mServiceLooper;
private volatile ServiceHandler mServiceHandler;
private String mName;
private boolean mRedelivery;

private final class ServiceHandler extends Handler {
public ServiceHandler(Looper looper) {
super(looper);
}

@Override
public void handleMessage(Message msg) {
onHandleIntent((Intent)msg.obj);
stopSelf(msg.arg1);
}
}

public IntentService(String name) {
super();
mName = name;
}

@Override
public void onCreate() {

super.onCreate();
// 创建 HandlerThread，并开启线程
HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
thread.start();

// 创建 Handler
mServiceLooper = thread.getLooper();
mServiceHandler = new ServiceHandler(mServiceLooper);
}

@Override
public void onStart(@Nullable Intent intent, int startId) {
// 将消息发送到消息队列的当中
Message msg = mServiceHandler.obtainMessage();
msg.arg1 = startId;
msg.obj = intent;
mServiceHandler.sendMessage(msg);
}

@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
onStart(intent, startId);
return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}

@Override
public void onDestroy() {
mServiceLooper.quit();
}

@Override
@Nullable
public IBinder onBind(Intent intent) {
return null;
}

@WorkerThread
protected abstract void onHandleIntent(@Nullable Intent intent);
```

IntentService 的源码也比较少，分析起来也不难。

(1)由于继承 Service，自然就有了 Service 的生命周期，当此一次开启时，会走 onCreate 方法，在这个方法中完成**创建 HandlerThread，并开启线程** 和 **创建 Handler**。

(2)接着会走 onStartCommand 方法，其中参数 intent 就是 startService() 中的参数，然后调用 onStart 方法，并通过 IntentService 内部的 Handler 将消息发送到消息队列中，参数 startId 代表请求的特殊标记，通过这个标记可用于 stopSelf(int startId)，通过这个方法停止任务，会等待所有的消息执行完之后，服务才会停止。

(3)消息的执行具体过程是在调用 ServiceHandler 的 handleMessage()，其内部又会调用 onHandleIntent((Intent)msg.obj)，即我们重写的方法，参数 msg.obj，也就是我们开启任务是传进来的 intent。执行任务之后通过 stopSelf(msg.arg1)，但是消息队列中还有任务时，会等待其他消息执行完毕，才会结束。

(4)由于采用的 Looper 执行任务，所有消息执行过程是按照顺序执行，只要这个 IntentService 还在执行，后面发过来的消息，就会排在消息队列的后面。

通过分析之后，上边的两个问题清晰了，理解了 IntentService 如何开启线程以及如何接收消息，和执行消息的过程。理解这部分的前提，是要把 Handler 的消息机制搞清楚。

### 代码地址

[练习代码](https://github.com/RalfNick/AndroidPractice/tree/master/AndroidThreadTest)

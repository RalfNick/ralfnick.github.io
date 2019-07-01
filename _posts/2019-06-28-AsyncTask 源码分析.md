---
layout: post
title: "AsyncTask 源码分析"
date: 2019-06-28
description: "AsyncTask 源码分析"
tag: 线程
---
### AsyncTask 简介

在 Android 中执行耗时任务时，我们一般不直接自己 new 一个 Thread,而且在 Android Studio 中也会给出提示，不建议使用传统的 Thread，那么有哪些方式呢？

> - HandlerThread 是一个 Thread，内部使用 Handler,它与普通 Thread 的区别是通过 handler 向消息队列中添加消息，优势是可以利用 Message 做相关控制，而且 HandlerThread 不调用 quit()，线程一直存在
>
> - IntentService 是对 HandlerThread 的一个应用，同时也是 Service，优先级更高，在后台执行时，不容易被杀死，执行完毕自动退出
>
> - AsyncTask 是一种轻量级的异步任务，底层执行利用线程池，同时也利用 Handler 在执行任务时可以在 UI 线程更新进度，执行结束后，也可以在 UI 线程更新结果，对于特别耗时的任务，不建议使用
>
> - 线程池 特别耗时任务时建议使用线程，多线程执行效率更高，充分利用 CPU

HandlerThread、IntentService、AsyncTask 中都用到了 Handler。如果对消息机制还不是很熟悉的话，可以看看我之前写的两篇关于 Handler 的文章

[消息机制 - Handler 使用](https://www.jianshu.com/p/b88e32c2477b)

[深入理解 Handler 消息机制](https://www.jianshu.com/p/43e5bcf1c05a)

对于 HandlerThread 和 IntentService 的使用及源码分析，可以看看之前的文章

[异步线程之 HandlerThread 和 IntentService](https://www.jianshu.com/p/83d28838d460)

不同的线程有不同的特点及使用场景，那么 AsyncTask 的使用场景是怎么样的？怎么评估呢？主要是从两点出发，一个是在后台执行过程中，需要有进度显示，类似于下载进度条，执行后更新结果；另一个是轻量级，特别耗时的任务不太适合，任务特别耗时，建议使用线程池。

下面就来看看 AsyncTask 的使用以及 AsyncTask 的主要源码

### AsyncTask 使用

这里使用 AsyncTask 完成一个下载的示例，后台下载，显示进度。

AsyncTask 是一个抽象的泛型类，具有 3 个泛型参数，Params, Progress, Result，其中 Params 表示参数类型，执行时，传递给 doInBackground() 方法，Progress 表示后台任务执行进度的类型，Result 表示后台任务执行完是返回的结果类型，当然未执行完时也会返回该类型的结果，但不一定是最终我们期望的结果。如果不需要参数时，可以使用 Void 类型代替。

(1) 继承 AsyncTask，重写 onPreExecute()、onPostExecute(String s)、onProgressUpdate(Float... values)、doInBackground(String... strings) 四个方法。

这里没有将 MyAsyncTask 写在 MainActivity 中，而是单拿出来，所以需要有回调接口 TaskCallBack，进行进度更新和结果返回。

```java
public class MyAsyncTask extends AsyncTask<String, Float, String> {

private static final int COUNT = 1000;
private TaskCallBack mCallBack;

public static MyAsyncTask newTask() {
return new MyAsyncTask();
}

private MyAsyncTask() {
super();
}

@Override
protected void onPreExecute() {
super.onPreExecute();
mCallBack.onPrepare();
}

@Override
protected void onPostExecute(String s) {
super.onPostExecute(s);
mCallBack.onFinished(s);
}

@Override
protected void onProgressUpdate(Float... values) {
super.onProgressUpdate(values);
mCallBack.onUpdate(values);
}

@Override
protected void onCancelled(String s) {
super.onCancelled(s);
mCallBack.onCancelled(s);
}

@Override
protected String doInBackground(String... strings) {
int i = 1;
while (i < COUNT) {
publishProgress(i * 1.0f / COUNT);
// 每完成 10%，睡眠 2 秒
if (i % 100 == 0) {
try {
Thread.sleep(2000);
} catch (InterruptedException e) {
e.printStackTrace();
return "未下载完成 - interrupt - " + i * 1.0f / COUNT;
}
}
i++;
if (isCancelled()) {
return "未下载完成 - " + i * 1.0f / COUNT;
}
}
return "下载完成";
}

public void setCallBack(TaskCallBack callBack) {
mCallBack = callBack;
}

public interface TaskCallBack {

void onPrepare();

void onUpdate(Float[] values);

void onCancelled(String s);

void onFinished(String s);
}
}
```

(2)设置显示进度对话框

```java
private void initDialog() {
mDownLoadDialog = new DownLoadDialog(this);
mDownLoadDialog.setCancelable(false);
mDownLoadDialog.updateProgress(0);
mDownLoadDialog.setOnDismissListener(new DialogInterface.OnDismissListener() {
@Override
public void onDismiss(DialogInterface dialog) {
mAsyncTask.cancel(true);
}
});
}
```
(3)创建 AsyncTask，设置监听回调

```java
/**
* 主线程中开启 AsyncTask
*/
private void startNewTask() {
mAsyncTask = MyAsyncTask.newTask();
mAsyncTask.setCallBack(new MyAsyncTask.TaskCallBack(){
@Override
public void onPrepare() {
mDownLoadDialog.show();
}

@Override
public void onUpdate(Float[] values) {
mDownLoadDialog.updateProgress(values[0]);
}

@Override
public void onCancelled(String s) {
Toast.makeText(MainActivity.this, "已经取消下载 - " + s, Toast.LENGTH_SHORT).show();
}

@Override
public void onFinished(String s) {
if (mDownLoadDialog.isShowing()) {
mDownLoadDialog.dismiss();
}
Toast.makeText(MainActivity.this, "下载完成 - " + s, Toast.LENGTH_SHORT).show();
}
});
mAsyncTask.execute("开始下载");
}
```
(4)设置点击事件，开启任务

```java
findViewById(R.id.start_btn).setOnClickListener(new View.OnClickListener(){
@Override
public void onClick(View view) {
initDialog();
startNewTask();
}
})
```

![gif](https://github.com/RalfNick/PicRepository/raw/master/AsyncTask/asynctask_demo.gif)

以上就是模拟下载进度的一个小例子，中间执行过程中，每完成 10%，睡眠 2 秒钟，模拟停顿一下。onPreExecute()、onPostExecute(String s)、onProgressUpdate(Float... values) 这 3 个方法是在主线程中执行的，其中 onPreExecute()在子线程开启前已经执行，没有用到 Handler，onProgressUpdate 和 onPostExecute 方法是通过 Handler 切换到主线程中的，所以更新 UI 也不会出现问题，doInBackground(String... strings) 执行耗时任务，在子线程中执行。

**有一点需要注意：** 
例子中的 doInBackground 方法每完成 10%，睡眠 2 秒钟，这过程中如果取消任务，会**大概率**抛出一个 InterruptedException，为什么是大概率呢？如果在睡眠时，被打断，就会抛出 InterruptedException，此时如果不返回，线程会继续执行；如果不是在睡眠过程中被打断，那么就不会抛出 InterruptedException，但是通过 isCancelled() 判断，线程处于 interrupt 状态，执行返回操作，如果不做返回操作，会继续执行，这里涉及线程的中断的知识，不做详细讲解。大概率抛出 InterruptedException 异常，因为在 while循环中，其他操作耗时相对于睡眠的 2 秒钟来说，基本可以忽略，所以取消任务是，几乎都会中断睡眠，抛出 InterruptedException 异常。

### AsyncTask 源码分析

在 AsyncTask 的注释中，提到使用 AsyncTask 的使用注意事项：

> - (1)AsyncTask 类需要 UI 线程中加载，在 Android 4.1 以上版本，这个加载过程已经由系统来完成了，在 ActivityThread 中使用到了 AsyncTask，自然也就完成了加载过程。
>
> - (2)AsyncTask 实例需要 UI 线程中创建
>
> - (3)execute 方法必须在 UI 线程中被调用
>
> - (4)不要直接调用 AsyncTask 的四个复写方法：onPreExecute、onPostExecute、onProgressUpdate、doInBackground
>
> - (5)一个 AsyncTask 实例，只能调用一次，重复调用会报错，也就是说，如果想再次调用 AsyncTask，需要重新创建实例。

我们根据这几个问题来进行源码分析。首先来看 AsyncTask 的执行过程，从 execute(Params... params) 方法开始。

```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params){
return executeOnExecutor(sDefaultExecutor, params);
}
```
接着调用 executeOnExecutor(sDefaultExecutor, params)，sDefaultExecutor 是 AsyncTask 默认的线程执行器，一个进程中都会使用这个执行器。

```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
Params... params) {
if (mStatus != Status.PENDING) {
switch (mStatus) {
case RUNNING:
throw new IllegalStateException("Cannot execute task:"
+ " the task is already running.");
case FINISHED:
throw new IllegalStateException("Cannot execute task:"
+ " the task has already been executed "
+ "(a task can be executed only once)");
}
}

mStatus = Status.RUNNING;

onPreExecute();

mWorker.mParams = params;
exec.execute(mFuture);

return this;
}
```
表明 AsyncTask 运行状态用一个 Status 的枚举来表示，有三种状态 PENDING --> RUNNING --> FINISHED。当创建 AsyncTask 后，表示运行状态的变量 mStatus 初始化为 Status.PENDING，执行开始后为 RUNNING，执行结束后更改为 FINISHED，中间过程不能修改为之前的状态。

一旦调用过 execute() 方法一次，状态就会变成 RUNNING 或者 FINISHED，所以第二次调用后，就会抛出异常，也就上面的两种异常，这也验证了上述使用注意事项 (5),execute 方法只能调用一次。

线程开始执行前 onPreExecute() 被调用，onPreExecute() 是用来在子线程执行前可以做一些 UI 提示，如显示进度提示框，所以 onPreExecute() 应该在主线程中被调用，也就是说 execute() 要在主线程中执行，这也验证了注意事项: (3) execute 方法必须在 UI 线程中被调用, 这里留一个疑问，一定是必须要在 UI 线程么，如果不在主线程，会出现什么情况，有没有什么方式可以解决？后面会对这个问题验证。

接着是开始执行子线程任务，这里有两个变量，mWorker 和 mFuture，其中 mWorker 是对 Callable 的一个封装，主要是为了添加参数 mParams，能够在线程执行时将参数传递进去，也就是 Params 类型的参数，传递到 doInBackground(Params... params) 方法中。

mWorker 和 mFuture 是在 AsyncTask 构建时创建的，下面看一下线程执行时通过这两个完成了哪些任务。

```java
public AsyncTask(@Nullable Looper callbackLooper) {
// Handler 创建
mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
? getMainHandler()
: new Handler(callbackLooper);

// mWorker 创建
mWorker = new WorkerRunnable<Params, Result>() {
public Result call() throws Exception {
// 表示 call() 方法已经被调用
mTaskInvoked.set(true);
Result result = null;
try {
// 设置线程优先级，设置为后台线程 Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
//noinspection unchecked
// 后台执行方法 doInBackground
result = doInBackground(mParams);
Binder.flushPendingCommands();
} catch (Throwable tr) {
mCancelled.set(true);
throw tr;
} finally {
// 返回结果给 UI 线程
postResult(result);
}
return result;
}
};
// mFuture 创建
mFuture = new FutureTask<Result>(mWorker) {
@Override
protected void done() {
try {
// 如果 mWorker 的 call() 方法未执行，会走这里
postResultIfNotInvoked(get());
} catch (InterruptedException e) {
android.util.Log.w(LOG_TAG, e);
} catch (ExecutionException e) {
throw new RuntimeException("An error occurred while executing doInBackground()",
e.getCause());
} catch (CancellationException e) {
postResultIfNotInvoked(null);
}
}
};
}
```
mWorker 的 call 方法是线程中执行的具体内容，在执行开始时，会通过 mTaskInvoked.set(true);来表示 call 方法已经被调用，指定的具体内容是我们自己重写的 doInBackground(mParams); 方法，无论是否被取消，最终都会调用 postResult(result); 方法，将结果发送给 UI 线程。

mFuture 对 mWorker 又进行包装了一层，mFuture 同时实现了 Runnable 和 Future，它能够在线程执行完获取结果，同时也能够知道线程是否被取消。这里对 mWorker 包装的意义在于如果 mWorker 的 call 方法因为异常没有被执行时（即 mTaskInvoked.get() 为 false 时），也能够通过 postResult(result); 通知到主线程。

```java
private void postResultIfNotInvoked(Result result) {
final boolean wasTaskInvoked = mTaskInvoked.get();
if (!wasTaskInvoked) {
postResult(result);
}
}
```
到这里线程的执行过程我们清楚了，还有两个地方没有串联起来，一个是执行器 exec.execute(mFuture);如何管理任务；一个是 执行时和执行后如何切换线程，通知主线程

先来看下线程执行器。

通过上面代码我们知道执行过程中使用的 AsyncTask 的默认执行器 sDefaultExecutor，它是一个 static volatile 变量，通过指向 SerialExecutor 实例，意味着一个进程中加载 AsyncTask 后已经初始化，所有的 AsyncTask 的任务都是通过它来执行的。

```java
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

private static class SerialExecutor implements Executor {
final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
Runnable mActive;

public synchronized void execute(final Runnable r) {
mTasks.offer(new Runnable() {
public void run() {
try {
r.run();
} finally {
scheduleNext();
}
}
});
if (mActive == null) {
scheduleNext();
}
}

protected synchronized void scheduleNext() {
if ((mActive = mTasks.poll()) != null) {
THREAD_POOL_EXECUTOR.execute(mActive);
}
}
}
```
当 exec.execute(mFuture); 方法执行时，SerialExecutor 会将任务 mFuture 添加到 mTasks 中，mTasks 是一个 ArrayDeque，意味着任务的执行是按照加入到队列中的顺序执行的，也就是串行的。SerialExecutor 的作用首先是将任务加入到 mTasks 中，然后找到一个可执行的任务 mActive 交给 THREAD_POOL_EXECUTOR 来执行，所以任务的真正执行是通过 THREAD_POOL_EXECUTOR 这个线程池来完成的。执行过程中就会调用 mWorker 的 call 方法，结束后通过 postResult(result); 将结果传递给主线程，另外，在 doInBackground 方法中，通过调用 publishProgress 可以将进度传递给 UI 线程，其原理和 postResult(result); 方法是一样的，都是通过其内部的 InternalHandler 来完成的，这里我们就只看一下 postResult(result); 方法。

```java
private Result postResult(Result result) {
@SuppressWarnings("unchecked")
Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
new AsyncTaskResult<Result>(this, result));
message.sendToTarget();
return result;
}
```

通过 Handler 创建一个 Message，Message 中含有 result 信息和标识信息  MESSAGE_POST_RESULT，然后发送到消息队列中，当处理消息时，会调用 Handler 的handleMessage 方法。

```java 
private static class InternalHandler extends Handler {
public InternalHandler(Looper looper) {
super(looper);
}
@SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
@Override
public void handleMessage(Message msg) {
AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
switch (msg.what) {
case MESSAGE_POST_RESULT:
// There is only one result
result.mTask.finish(result.mData[0]);
break;
case MESSAGE_POST_PROGRESS:
result.mTask.onProgressUpdate(result.mData);
break;
}
}
}
```
在 handleMessage 方法中，根据 msg.what 标识来执行对应的方法，如 MESSAGE_POST_RESULT，会执行 result.mTask.finish(result.mData[0]); 方法，result.mTask 就是 AsyncTask 本身，也就是会执行 AsyncTask 的 finish 方法，最终会调用 onPostExecute(result); 也就是我们自己重写的方法。

```java
private void finish(Result result) {
if (isCancelled()) {
onCancelled(result);
} else {
onPostExecute(result);
}
mStatus = Status.FINISHED;
}
```
AsyncTask 的四个需要重写的方法 onPreExecute()、onPostExecute(String s)、onProgressUpdate(Float... values)、doInBackground(String... strings) 在执行过程中 AsyncTask 会自动在合适的时机调用，我们不需要自己去调用，就像 Activity 的生命周期一样。

既然通过 InternalHandler 来切换到 UI 线程，那么 InternalHandler 的 构造中需要的 Looper 应该是主线程的 Looper，即 sMainLooper。那么 InternalHandler 是在什么时候构造的呢？在创建 AsyncTask 时。

```java
public AsyncTask(@Nullable Looper callbackLooper) {
mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
? getMainHandler()
: new Handler(callbackLooper);

....
}
```
我们在创建 AsyncTask 时，并没有传递 Looper，所以 callbackLooper 为 null，所以会调用 getMainHandler() 方法。

该方法为 AsyncTask.class 线程安全做了同步，保证一个进程中所有 AsyncTask 实例只有一个 Handler 实例，且只有应用在第一次实例化 AsyncTask 时才会创建 Handler 对象。

创建 InternalHandler 传入了主线程的 Looper 对象，即无论我们在哪个线程实例化 AsyncTask 实例，AsyncTask 中的 Handler 都是用主线程的 Looper 来实例化的，所以也就会切换到了 UI 线程。

```java
private static Handler getMainHandler() {
synchronized (AsyncTask.class) {
if (sHandler == null) {
sHandler = new InternalHandler(Looper.getMainLooper());
}
return sHandler;
}
}
```

通过上述分析，可以知道注意事项中 **AsyncTask 类需要 UI 线程中加载** 和 **AsyncTask 实例需要 UI 线程中创建** 可以不按照这个要求来。

```java
/**
* 子线程中开启 AsyncTask
*/
private void startTaskInNewThread() {
Thread asyncThread = new Thread(new Runnable() {
@Override
public void run() {
mAsyncTask = MyAsyncTask.newTask();
mAsyncTask.setCallBack(new MyAsyncTask.TaskCallBack() {
@Override
public void onPrepare() {
mDownLoadDialog.show();
}

@Override
public void onUpdate(Float[] values) {
mDownLoadDialog.updateProgress(values[0]);
}

@Override
public void onCancelled(String s) {
Toast.makeText(MainActivity.this, "已经取消下载 - " + s, Toast.LENGTH_SHORT).show();
}

@Override
public void onFinished(String s) {
if (mDownLoadDialog.isShowing()) {
mDownLoadDialog.dismiss();
}
Toast.makeText(MainActivity.this, "下载完成 - " + s, Toast.LENGTH_SHORT).show();
}
});
}
});
asyncThread.start();
// 等待mAsyncTask创建完毕，否则 mAsyncTask.execute("开始下载"); 无法执行，空指针
try {
asyncThread.join();
} catch (InterruptedException e) {
e.printStackTrace();
}
mAsyncTask.execute("开始下载");
}
```
这样创建 AsyncTask 一样可以正常执行，为什么会这样的? 我这里分析的 AsyncTask 的源码是基于 android-28 中的 AsyncTask，在 android 26 起 AsyncTask 就已经能够保证即使不在主线程加载 AsyncTask 和创建 AsyncTask 一样可以正常运行。通过上述代码可以知道， AsyncTask 中的 InternalHandler 也能够获取主线程的 Looper，自然也就可以切换到 UI 线程。但是在 android 26 之前，InternalHandler 在 AsyncTask 加载时就已经创建，而且没有传入 Looper,这时候会获取当前线程的 Looper，如果子线程没有调用 Looper.prepare()，就会报错，而且即使子线程创建了 Looper，也不能切换到 UI 线程，因为不是主线程的 Looper。

```java
private static final InternalHandler sHandler = new InternalHandler();

private static class InternalHandler extends Handler {  
@SuppressWarnings({unchecked, RawUseOfParameterizedType})  
@Override  
public void handleMessage(Message msg) {  
...
}  
}
```

上面我们验证 AsyncTask 类可以不在 UI 线程中加载，实例也不一定要在 UI 线程中创建，但是 execute 方法最好还是在 UI 线程中调用，如果不在主线程，会出现什么情况，有没有什么方式可以解决？

下面就是在一个子线程中创建 AsyncTask，在子线程中执行 execute 方法

```java
private void startNewTask1() {
new Thread(new Runnable() {
@Override
public void run() {
//                Looper.prepare();
mAsyncTask = MyAsyncTask.newTask();
mAsyncTask.setCallBack(new MyAsyncTask.TaskCallBack() {
@Override
public void onPrepare() {
// 子线程执行会报错
//                        mDownLoadDialog.show();
// mDownLoadDialog.show(); 换成 button.setText("123");
//                        button.setText("123");
// 切换到 UI 线程
runOnUiThread(new Runnable() {
@Override
public void run() {
mDownLoadDialog.show();
}
});
}

@Override
public void onUpdate(final Float[] values) {
runOnUiThread(new Runnable() {
@Override
public void run() {
mDownLoadDialog.updateProgress(values[0]);
}
});
}

@Override
public void onCancelled(String s) {
Toast.makeText(MainActivity.this, "已经取消下载 - " + s, Toast.LENGTH_SHORT).show();
}

@Override
public void onFinished(String s) {
runOnUiThread(new Runnable() {
@Override
public void run() {
if (mDownLoadDialog.isShowing()) {
mDownLoadDialog.dismiss();
}
}
});
Toast.makeText(MainActivity.this, "下载完成 - " + s, Toast.LENGTH_SHORT).show();
}
});
mAsyncTask.execute("开始下载");
//                Looper.loop();
}
}).start();
}
```
由于例子中使用的 Dialog 显示进度，Dialog 的显示过程需要 ViewRootImpl 中初始化 ViewRootHandler，需要获取一个 Looper，但是在子线程中没有 调用 Looper.prepare() 和 Looper.loop(),自然没有 Looper，所以会报错。

```java
java.lang.RuntimeException: Can't create handler inside thread that has not called Looper.prepare()
at android.os.Handler.<init>(Handler.java:204)
at android.os.Handler.<init>(Handler.java:118)
at android.view.ViewRootImpl$ViewRootHandler.<init>(ViewRootImpl.java:3665)
at android.view.ViewRootImpl.<init>(ViewRootImpl.java:3998)
at android.view.WindowManagerGlobal.addView(WindowManagerGlobal.java:346)
at android.view.WindowManagerImpl.addView(WindowManagerImpl.java:93)
at android.app.Dialog.show(Dialog.java:330)
```

在注释中，给出另外一个更加清晰辨别的方法，更新 Button 的文字，也是需要在主线程中执行，所以会报下面的错误，因为不是在主线程中更新 UI。对上面 Dialog 的显示，如果在 run 方法中加上 Looper.prepare() 和 Looper.loop()，同样会报下面的错误。

```java
android.view.ViewRootImpl$CalledFromWrongThreadException: Only the original thread that created a view hierarchy can touch its views.
```

所以不在 UI 线程中调用 execute 方法，更新 UI 操作会报错，那么有没有什么解决方法呢？方法就是将更新 UI 的操作切换到主线程即可。可以采用 runOnUiThread 方法。

上面一系列操作看似有点非常规，但是对于我们理解 AsyncTask 的使用很有帮助，正常情况下，我们按照 AsyncTask 使用注意事项中的建议使用，不会引出一些不必要的麻烦。

### 总结

AsyncTask 是利用线程池和 Handler 来完操作，线程池在后台子线程执行耗时任务，更新进度和结果返回时，通过 Handler 来进行更新 UI 操作，需要理解两个任务变量 mWorker 和 mFuture,以及两个执行器 sDefaultExecutor 和 THREAD_POOL_EXECUTOR 各自的作用。此外，AsyncTask 的 execute 方法是串行执行的，如果想并行，可以使用 executeOnExecutor 方法，使用自定义的执行器或者直接使用 AsyncTask 中的 THREAD_POOL_EXECUTOR，能够达到并行处理的目的。

### 代码地址

[AsyncTask 练习](https://github.com/RalfNick/AndroidPractice/tree/master/AsyncTaskTest)


### 参考

[程序媛说源码：AsyncTask在子线程创建与调用的那些事儿](http://www.10tiao.com/html/227/201905/2650245764/1.html)




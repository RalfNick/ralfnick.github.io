---
layout: post
title: "Android ANR"
date: 2021-04-10
description: "Android ANR"
tag: ANR
---
### 1.ANR

#### 1.1 什么是 ANR？

![anr](https://github.com/RalfNick/PicRepository/raw/master/anr/anr-example-framed.png)

我们知道安卓应用中 UI 渲染是在主线程中，所以对于一些点击事件，以及和用户交互相关的事件需要能够及时，否则对于用户来说就是一个很不好的体验。安卓系统中对于这类在主线程中没有及时作出反馈的问题叫作 ANR(Application Not Responding)。安卓不同组件对于 ANR 的超时限制是不同的。

![anr_time](https://github.com/RalfNick/PicRepository/raw/master/anr/anr_time.png)

安卓中 ANR 的机制采用类似于『看门狗』，发送事件的同时，会发一个计时的消息，如果在计时的时间内没有完成事件，则触发 ANR。gityuan 博客中形容的比较贴切，比喻成安装炸弹和拆炸弹的过程。

![anr_how](https://github.com/RalfNick/PicRepository/raw/master/anr/broadcast_anr_2.jpeg)

对于输入事件的过程和安卓组件的过程还是有区别，输入事件的 ANR 触发需要下一次输入，也很容易理解，上一次事件在处理中，虽然可能超时，但是下一次事件还没到来，如果下一次输入事件到来，则检测上一次事件是否超时。

#### 1.2 ANR 难点

用户在应用内的绝大部分操作，比如按钮点击，加载资源，页面跳转等操作，都需要有 App 的主动反馈，但 ANR 发生时，在用户等待数秒后，仅会弹出一个“应用无响应”的弹窗给用户，这会给用户带来“应用难用”的感觉，极其影响用户体验。但是，现网中的 ANR 问题又很难处理，问题包括但不限于：

- 平时的测试难以覆盖，毕竟ANR经常出现在老设备、弱网络环境的场景下，测试难以做到全场景覆盖。
- 对于现网应用的ANR问题，如果问题非必现，则定位难度较高，需要有可以复现问题的实际设备在身边，才能获取到具体日志trace等信息。
- ANR问题定位复杂，影响因素多，一些新负责定位ANR问题的同学，上手困难，问题解决比较依赖经验。

### 2.ANR 日志分析

#### 2.1 ANR 日志

之前在低版本手机还能获取到 anr 日志,可以直接使用 adb pull 命令将 anr 文件导出

```
adb pull /data/anr/anr文件 导出路径xxx
```

如下 SP 导致的 ANR 实例：

```java
06-16 16:16:28.590  1853  2073 E ActivityManager: ANR in com.android.camera (com.android.camera/.Camera)
06-16 16:16:28.590  1853  2073 E ActivityManager: PID: 27661
06-16 16:16:28.590  1853  2073 E ActivityManager: Reason: Input dispatching timed out (com.android.camera/com.android.camera.Camera, Waiting to send non-key event because the touched window has not finished processing certain input events that were delivered to it over 500.0ms ago.  Wait queue length: 24.  Wait queue head age: 5511.1ms.)
06-16 16:16:28.590  1853  2073 E ActivityManager: Load: 16.25 / 29.48 / 38.33
06-16 16:16:28.590  1853  2073 E ActivityManager: CPU usage from 0ms to 8058ms later:
06-16 16:16:28.590  1853  2073 E ActivityManager:   58% 291/mediaserver: 51% user + 6.7% kernel / faults: 2457 minor 4 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   27% 317/mm-qcamera-daemon: 21% user + 5.8% kernel / faults: 15965 minor
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.4% 288/debuggerd: 0% user + 0.3% kernel / faults: 21615 minor 87 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   17% 27661/com.android.camera: 10% user + 6.8% kernel / faults: 2412 minor 34 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   16% 1853/system_server: 10% user + 6.4% kernel / faults: 1754 minor 87 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   10% 539/sensors.qcom: 7.8% user + 2.6% kernel / faults: 16 minor
06-16 16:16:28.590  1853  2073 E ActivityManager:   4.4% 277/surfaceflinger: 1.8% user + 2.6% kernel / faults: 14 minor
06-16 16:16:28.590  1853  2073 E ActivityManager:   4% 203/mmcqd/0: 0% user + 4% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   2.6% 3510/com.android.phone: 1.9% user + 0.6% kernel / faults: 1148 minor 8 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   2.1% 2902/com.android.systemui: 1.6% user + 0.4% kernel / faults: 1272 minor 32 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   1.6% 3110/com.miui.whetstone: 1.6% user + 0% kernel / faults: 2614 minor 22 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.8% 99/kswapd0: 0% user + 0.8% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   1.4% 217/jbd2/mmcblk0p25: 0% user + 1.4% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   1.4% 223/logd: 0.7% user + 0.7% kernel / faults: 4 minor
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.9% 12808/kworker/0:1: 0% user + 0.9% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.8% 35/kworker/u:2: 0% user + 0.8% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0% 3222/com.miui.sysbase: 0% user + 0% kernel / faults: 1314 minor 12 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.8% 3446/com.android.nfc: 0.4% user + 0.3% kernel / faults: 1223 minor 9 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.7% 10866/kworker/u:1: 0% user + 0.7% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.6% 642/mdss_fb0: 0% user + 0.6% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.6% 29336/kworker/u:7: 0% user + 0.6% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.4% 6/kworker/u:0: 0% user + 0.4% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.4% 22924/kworker/u:6: 0% user + 0.4% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.3% 4421/mpdecision: 0% user + 0.3% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.2% 276/servicemanager: 0.1% user + 0.1% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.2% 289/rild: 0.2% user + 0% kernel / faults: 20 minor
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 4161/mcd: 0% user + 0% kernel / faults: 9 minor 1 major
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 3/ksoftirqd/0: 0% user + 0.1% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 5/kworker/0:0H: 0% user + 0.1% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 7/kworker/u:0H: 0% user + 0.1% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0% 215/flush-179:0: 0% user + 0% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 321/displayfeature: 0.1% user + 0% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 368/irq/33-cpubw_hw: 0% user + 0.1% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 403/qmuxd: 0% user + 0.1% kernel / faults: 60 minor
06-16 16:16:28.590  1853  2073 E ActivityManager:   0% 3491/com.xiaomi.finddevice: 0% user + 0% kernel / faults: 706 minor
06-16 16:16:28.590  1853  2073 E ActivityManager:   0.1% 29330/ksoftirqd/1: 0% user + 0.1% kernel
06-16 16:16:28.590  1853  2073 E ActivityManager: 96% TOTAL: 56% user + 29% kernel + 6.3% iowait + 4.1% softirq
```

可以初步定为发生 ANR 的进程以及发生日志时的负载情况。这是之前版本的 ANR 日志，从日志中有明显的 ANR 标识，以及 CPU 负载情况。

- 51% user + 6.7% kernel / faults: 2457 minor 4 major 用户态和内核态使用情况，以及当前 IO 情况，minor 缺页时不从硬盘读取，major 缺页时从硬盘读取。
- Load: 16.25 / 29.48 / 38.33，这里输出了CPU的负载情况

CPU 负载是指某一时刻系统中运行队列长度之和加上当前正在CPU上运行的进程数，而 CPU 平均负载可以理解为一段时间内正在使用和等待使用 CPU 的活动进程的平均数量。在Linux中『活动进程』是指当前状态为运行或不可中断阻塞的进程。通常所说的负载其实就是指平均负载。

用一个从网上看到的很生动的例子来说明（不考虑CPU时间片的限制），把设备中的一个单核CPU比作一个电话亭，把进程比作正在使用和等待使用电话的人，假如有一个人正在打电话，有三个人在排队等待，此刻电话亭的负载就是 4。使用中会不断的有人打完电话离开，也会不断的有其他人排队等待，为了得到一个有参考价值的负载值，可以规定每隔 5 秒记录一下电话亭的负载，并将某一时刻之前的一分钟、五分钟、十五分钟的的负载情况分别求平均值，最终就得到了三个时段的平均负载。
实际上我们通常关心的就是在某一时刻的前一分钟、五分钟、十五分钟的CPU平均负载，例如以上日志中这三个值分别是 16.25 / 29.48 / 38.33，说明前一分钟内正在使用和等待使用CPU的活动进程平均有 16.25 个，依此类推。在大型服务器端应用中主要关注的是第五分钟和第十五分钟的两个值，但是 Android 主要应用在便携手持设备中，有特殊的软硬件环境和应用场景，短时间内的系统的较高负载就有可能造成 ANR，所以一分钟内的平均负载相对来说更具有参考价值。

CPU 的负载和使用率没有必然关系，有可能只有一个进程在使用CPU，但执行的是复杂的操作；也有可能等待和正在使用 CPU 进程很多，但每个进程执行的都是简单操作。实际处理问题时偶尔会遇到由于平均负载高引起的 ANR，典型的特征就是系统中应用进程数量多，CPU 总使用率较高，但是每个进程的 CPU 使用率不高，当前应用进程主线程没有异常阻塞，一分钟内的 CPU 平均负载较高。

目前高版本手机已经不好拿到之前的 ANR 日志，现在一般都是获取 trace 日志来分析。一些新版本手机如果没有 root，获取不到文件权限，导致无法导出 anr 文件，即使用 chmod 改变权限也不行。

```
OnePlus7Pro:/ $ su
/system/bin/sh: su: inaccessible or not found
127|OnePlus7Pro:/ $ chmod 777 /data
chmod: chmod '/data' to 40777: Permission denied
1|OnePlus7Pro:/ $ chmod 777 data/anr/anr_2021-04-03-12-25-07-134
chmod: chmod 'data/anr/anr_2021-04-03-12-25-07-134' to 100777: Permission denied
1|OnePlus7Pro:/ $ cd data
OnePlus7Pro:/data $ ls
ls: .: Permission denied
```

此时可以通过 adb bugreport 获取 trace 文件

```
~ adb bugreport
/data/user_de/0/com.android.shell/files/bugreports/bugreport-OnePlus7Pro_CH-QKQ1.190716.003-2021-04-03-12-25-23.zip: 1 file pulled, 0 skipped. 35.3 MB/s (9218520 bytes in 0.249s)
➜  ~ adb pull /data/user_de/0/com.android.shell/files/bugreports/bugreport-OnePlus7Pro_CH-QKQ1.190716.003-2021-04-03-12-25-23.zip /Users/xxx/Desktop
/data/user_de/0/com.android.shell/files/bugreports/bugreport-OnePlus7Pro_CH-QKQ1.190716.003-2021-04-03-12-25-23.zip: 1 file pulled, 0 skipped. 38.0 MB/s (9218520 bytes in 0.231s)
```

- 也可以不通过命令，在手机的开发者选项中也能生成报告。打开开发者选项 -- > 错误报告

#### 2.2 ANR 例子

简单模拟 ANR 的例子：

```kotlin
class AnrActivity : AppCompatActivity() {

    private val mReceiver = MyBoardCastReceiver()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_anr)
        val filter = IntentFilter()
        registerReceiver(mReceiver, IntentFilter("MyBoardCastReceiver"))
        text_click.setOnClickListener {
//            deadLock()
//            sleep()
//            deadLoop()
            waitNotify()
            Toast.makeText(this, "点击事件", Toast.LENGTH_SHORT).show()
        }
        board_cast_click.setOnClickListener {
            sendBroadcast(Intent("MyBoardCastReceiver"))
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        unregisterReceiver(mReceiver)
    }

    private fun deadLock() {
        val runnable1 = Runnable {
            synchronized(lock1) {
                Thread.sleep(500)
                synchronized(lock2) {
                    Log.d(TAG, "run: task 1 execute")
                }
            }
        }

        val runnable2 = Runnable {
            synchronized(lock2) {
                Thread.sleep(500)
                synchronized(lock1) {
                    Log.d(TAG, "run: task 2 execute")
                }
            }
        }

        Thread(runnable1).start()
        runOnUiThread(runnable2)
    }

    private fun sleep() {
        Thread.sleep(6000)
    }

    private fun waitNotify() {
        val runnable1 = Runnable {
            synchronized(lock1) {
                Thread.sleep(500)
//                lock1.notify()
            }
        }

        val runnable2 = Runnable {
            synchronized(lock1) {
                lock1.wait()
            }
        }

        Thread(runnable1).start()
        Thread.sleep(100)
        runOnUiThread(runnable2)
    }

    private fun deadLoop() {
        while (true) {
            Log.d(TAG, "deadLoop")
        }
    }

    private fun sendBroadCastEvent() {

    }

    private class MyBoardCastReceiver : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            Log.d(TAG, "onReceive: " + intent?.action)
            // 耗时操作
            Thread.sleep(20000)
            // 更新 UI
        }
    }

    companion object {
        val lock1 = Object()
        val lock2 = Object()
        const val TAG = "AnrActivity"
    }
}
```

- wait

```java
"main" prio=5 tid=1 Waiting
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72fb4638 self=0x7773810800
  | sysTid=28945 nice=-10 cgrp=default sched=0/0 handle=0x77fa042ed0
  | state=S schedstat=( 393128629 45675994 245 ) utm=33 stm=5 core=0 HZ=100
  | stack=0x7fe273a000-0x7fe273c000 stackSize=8192KB
  | held mutexes=
  at java.lang.Object.wait(Native method)
  - waiting on <0x0c45f993> (a java.lang.Object)
  at java.lang.Object.wait(Object.java:442)
  at java.lang.Object.wait(Object.java:568)
  at com.ralf.textspantest.AnrActivity$waitNotify$runnable2$1.run(AnrActivity.kt:63)
  - locked <0x0c45f993> (a java.lang.Object)
  at android.app.Activity.runOnUiThread(Activity.java:6922)
  at com.ralf.textspantest.AnrActivity.waitNotify(AnrActivity.kt:69)
  at com.ralf.textspantest.AnrActivity.access$waitNotify(AnrActivity.kt:9)
  at com.ralf.textspantest.AnrActivity$onCreate$1.onClick(AnrActivity.kt:17)
  at android.view.View.performClick(View.java:7201)
  at android.view.View.performClickInternal(View.java:7170)
  at android.view.View.access$3500(View.java:806)
  at android.view.View$PerformClick.run(View.java:27582)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7710)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:516)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950)
```

- blocked

```java
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72fb4638 self=0x7773810800
  | sysTid=13350 nice=-10 cgrp=default sched=0/0 handle=0x77fa042ed0
  | state=S schedstat=( 413722705 41562546 258 ) utm=33 stm=8 core=3 HZ=100
  | stack=0x7fe273a000-0x7fe273c000 stackSize=8192KB
  | held mutexes=
  at com.ralf.textspantest.AnrActivity$deadLock$runnable2$1.run(AnrActivity.kt:33)
  - waiting to lock <0x0c45f993> (a java.lang.String) held by thread 2
  - locked <0x06883dd0> (a java.lang.String)
  at android.app.Activity.runOnUiThread(Activity.java:6922)
  at com.ralf.textspantest.AnrActivity.deadLock(AnrActivity.kt:40)
  at com.ralf.textspantest.AnrActivity.access$deadLock(AnrActivity.kt:9)
  at com.ralf.textspantest.AnrActivity$onCreate$1.onClick(AnrActivity.kt:14)
  at android.view.View.performClick(View.java:7201)
  at android.view.View.performClickInternal(View.java:7170)
  at android.view.View.access$3500(View.java:806)
  at android.view.View$PerformClick.run(View.java:27582)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7710)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:516)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950)
```
- sleep

```java
"main" prio=5 tid=1 Sleeping
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72fb4638 self=0x7773810800
  | sysTid=10722 nice=-10 cgrp=default sched=0/0 handle=0x77fa042ed0
  | state=S schedstat=( 903861783 108686515 436 ) utm=81 stm=9 core=3 HZ=100
  | stack=0x7fe273a000-0x7fe273c000 stackSize=8192KB
  | held mutexes=
  at java.lang.Thread.sleep(Native method)
  - sleeping on <0x05680e82> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:440)
  - locked <0x05680e82> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:356)
  at com.ralf.textspantest.AnrActivity.sleep(AnrActivity.kt:48)
  at com.ralf.textspantest.AnrActivity.access$sleep(AnrActivity.kt:9)
  at com.ralf.textspantest.AnrActivity$onCreate$1.onClick(AnrActivity.kt:15)
  at android.view.View.performClick(View.java:7201)
  at android.view.View.performClickInternal(View.java:7170)
  at android.view.View.access$3500(View.java:806)
  at android.view.View$PerformClick.run(View.java:27582)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7710)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:516)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950)
```
- 死循环

```java
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72fb4638 self=0x7773810800
  | sysTid=24799 nice=-10 cgrp=default sched=0/0 handle=0x77fa042ed0
  | state=S schedstat=( 285148804810 93710258 3056 ) utm=17382 stm=11132 core=7 HZ=100
  | stack=0x7fe273a000-0x7fe273c000 stackSize=8192KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/24799/stack)
  native: #00 pc 000000000007067c  /apex/com.android.runtime/lib64/bionic/libc.so (syscall+28)
  native: #01 pc 000000000014c1f4  /apex/com.android.runtime/lib64/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+148)
  native: #02 pc 0000000000372074  /apex/com.android.runtime/lib64/libart.so (art::(anonymous namespace)::CheckJNI::ReleaseStringCharsInternal(char const*, _JNIEnv*, _jstring*, void const*, bool, bool)+472)
  native: #03 pc 0000000000151f60  /system/lib64/libandroid_runtime.so (android::android_util_Log_println_native(_JNIEnv*, _jobject*, int, int, _jstring*, _jstring*)+208)
  at android.util.Log.println_native(Native method)
  at android.util.Log.d(Log.java:155)
  at com.ralf.textspantest.AnrActivity.deadLoop(AnrActivity.kt:74)
  at com.ralf.textspantest.AnrActivity.access$deadLoop(AnrActivity.kt:9)
  at com.ralf.textspantest.AnrActivity$onCreate$1.onClick(AnrActivity.kt:16)
  at android.view.View.performClick(View.java:7201)
  at android.view.View.performClickInternal(View.java:7170)
  at android.view.View.access$3500(View.java:806)
  at android.view.View$PerformClick.run(View.java:27582)
  at android.os.Handler.handleCallback(Handler.java:883)
  at android.os.Handler.dispatchMessage(Handler.java:100)
  at android.os.Looper.loop(Looper.java:214)
  at android.app.ActivityThread.main(ActivityThread.java:7710)
  at java.lang.reflect.Method.invoke(Native method)
  at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:516)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:950)
```

#### 2.3 trace 文件内容说明：

```
第0行:
线程名: main（如有daemon则代表守护线程)
prio: 线程优先级
tid: 线程内部id
线程状态: NATIVE

第1行:
group: 线程所属的线程组
sCount: 线程挂起次数
dsCount: 用于调试的线程挂起次数
obj: 当前线程关联的java线程对象
self: 当前线程地址

第2行：
sysTid：线程真正意义上的tid
nice: 调度有优先级
cgrp: 进程所属的进程调度组
sched: 调度策略
handle: 函数处理地址

第3行：
state: 线程状态
schedstat: CPU调度时间统计, 见proc/[pid]/task/[tid]/schedstat
utm/stm: 用户态/内核态的CPU时间(单位是jiffies), 见proc/[pid]/task/[tid]/stat
core: 该线程的最后运行所在核
HZ: 时钟频率

第4行：
stack：线程栈的地址区间
stackSize：栈的大小

第5行：
mutex: 所持有mutex类型，有独占锁exclusive和共享锁shared两类
schedstat含义说明：
binder_cpu

nice值越小则优先级越高。此处nice=-10, 可见优先级还是比较高的;

schedstat括号中的3个数字依次是Running、Runable、Switch，紧接着的是utm和stm

Running时间：CPU运行的时间，单位ns
Runable时间：RQ队列的等待时间，单位ns
Switch次数：CPU调度切换次数

utm: 该线程在用户态所执行的时间，单位是jiffies，jiffies定义为sysconf(_SC_CLK_TCK)，默认等于10ms
stm: 该线程在内核态所执行的时间，单位是jiffies，默认等于10ms

jiffies是记录着从电脑开机到现在总共的时钟中断次数。也就是说每10ms一次中断。 所以一般来说Linux的精确度是10毫秒。 硬件给内核提供一个系统定时器用以计算和管理时间，内核通过编程预设系统定时器的频率，即节拍率（tick rate),每一个周期称作一个tick(节拍）

可见，该线程Running=285148804810ns,也约等于285140ms。在CPU运行时间包括用户态(utm)和内核态(stm)。 utm + stm = (17382 + 11132) ×10 ms = 285140ms。

结论：utm + stm = schedstat第一个参数值。
```

对于上述 sleep、block、wait 等实例都很容看出，都是主线程出现阻塞导致出现的 ANR 问题，此时只要根据 trace 信息基本能够定位到我们代码中出现问题的位置。

对于死循环或者其他类似导致 CPU 负载出现异常时的情况，相对不是很容易分析起，因为影响 CPU 负载的情况很多。当然这个 demo 中还是很容易看出的。此类属于资源紧张导致的：
- 如内存比较紧张，kernel log 中 D 状态的用户进程比较多，就可能 block.
- 如 CPU 资源紧张，像这个死循环，频繁调用，导致其他地方的方法不能执行，CPU 过高也不一定完全是用户态线程导致，内核态也可能出现 CPU 过高的情况，如频繁出现 ANR、OOM 时需要频繁 dump 文件，都会导致资源紧张的情况。

```java
"main" prio=5 tid=1 Native
  | group="main" sCount=1 dsCount=0 flags=1 obj=0x72fb4638 self=0x7773810800
  | sysTid=24799 nice=-10 cgrp=default sched=0/0 handle=0x77fa042ed0
  | state=S schedstat=( 285148804810 93710258 3056 ) utm=17382 stm=11132 core=7 HZ=100
  | stack=0x7fe273a000-0x7fe273c000 stackSize=8192KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/24799/stack)
  native: #00 pc 000000000007067c  /apex/com.android.runtime/lib64/bionic/libc.so (syscall+28)
  native: #01 pc 000000000014c1f4  /apex/com.android.runtime/lib64/libart.so (art::ConditionVariable::WaitHoldingLocks(art::Thread*)+148)
  native: #02 pc 0000000000372074  /apex/com.android.runtime/lib64/libart.so (art::(anonymous namespace)::CheckJNI::ReleaseStringCharsInternal(char const*, _JNIEnv*, _jstring*, void const*, bool, bool)+472)
  native: #03 pc 0000000000151f60  /system/lib64/libandroid_runtime.so (android::android_util_Log_println_native(_JNIEnv*, _jobject*, int, int, _jstring*, _jstring*)+208)
    at android.util.Log.println_native(Native method)
  at android.util.Log.d(Log.java:155)
```

从日志中看出，utm 和 stm 都很高，用户态时间 utm = 17382 * 10 ms = 173 s，内核态 stm = 11132 * 10ms = 111 s。此刻的内存使用也很好 stackSize 有 8M。再结合上下文日志看出，用户态转到内核态是在使用日志打印，单纯的日志打印不至于使 CPU 过高，那么就要看在此处的函数调用具体做了哪些工作导致的。可以看到该处有一个死循环，导致 CPU 过忙，而循环中一直调用日志打印，转到了内核态，也就导致内核态的时间也很长。

### 4.遇到的实际问题及解决方案

代码问题出在 RecyclerView 的 itemdecoration 绘制分割线时，获取资源 drawable 的过程中。可以从代码中可以看到问题的出现是由于获取资源时 AssetManager 打开 XML 时的同步锁导致的。由于资源文件还是要获取，那么如何解决资源获取问题带来的 ANR 问题呢？

```java
ContextCompat.getDrawable(context(), res) --> context.getResources().getDrawable()

--> mResourcesImpl.loadDrawable(this, value, id, density, theme);  -- > loadDrawableForCookie

-- > loadXmlResourceParser() --> mAssets.openXmlBlockAsset


@NonNull XmlBlock openXmlBlockAsset(int cookie, @NonNull String fileName) throws IOException {
    Objects.requireNonNull(fileName, "fileName");
    // 同步锁
    synchronized (this) {
        ensureOpenLocked();

        final long xmlBlock = nativeOpenXmlAsset(mObject, cookie, fileName);
        if (xmlBlock == 0) {
            throw new FileNotFoundException("Asset XML file: " + fileName);
        }
        final XmlBlock block = new XmlBlock(this, xmlBlock);
        incRefsLocked(block.hashCode());
        return block;
    }
}
```
实际上项目中处理时并没有完全解决这个问题，因为可能是其他位置的代码也正在获取资源，导致本处的代码出现了阻塞。既然是在绘制分割线时导致的资源获取阻塞，那么能否调整一下时机，使得资源提前准备好，是不就能够大大减少在绘制时出现阻塞的情况。按照这个思路，将资源获取时机挪到构造函数中,实例代码如下：

```java
public class MyItemDecoration extends ItemDecoration {

    private final Drawable mDivider;

    public MyItemDecoration(){
    // 构造方法中获取 Drawable
      mDivider = CommonUtil.drawable(R.drawable.divider);
    }

    @Override
    public void onDraw(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull State state) {
        // 挪到构造函数中
//    if (mDivider == null) {
//      mDivider = CommonUtil.drawable(R.drawable.news_divider_normal);
//    }

    }
}

```
![]()

最终通过观察线上数据，ANR 发生的次数大幅度降低。理论上虽然这种解决方式不能完全消除 ANR，但是也是行之有效的方案，实际线上观察基本上已经消除该处的 ANR 问题。通过该问题的解决思路可以看出：尽量避开在出现视图集中绘制的时机做一些耗时操作，选择一些 CPU 不是繁忙或者更新视图操作较少的时机做一些轻量的耗时操作，这样也有助于提高视图的渲染效率，较少页面卡顿情况的发生。

### 5.总结

导致 ANR 的原因可以分为两大类，一是系统资源不足导致，二是自身代码逻辑导致。而因为我们自身代码导致 ANR 的常见操作有以下几种情况：

- 在主线程做耗时操作，如 IO 操作
- 在主线程做大量运算，导致 CPU 过忙
- 在主线程通过 binder 调用其他进程，而其他进程的操作耗时导致调用线程阻塞
- 主线程在等待同步锁，同步锁被其他线程所持有
- 主线程和其他线程陷入死锁状态，或者是和其他进程通讯时出现死锁，不仅仅等待时间过长的问题

简单概括起来就是主线程被阻塞，IO 操作、锁、CPU 过忙，被其他进程阻塞等，所以分析 ANR 问题时也主要从这几方面入口，首先定位出现问题的大致位置，定位到应用进程，函数方法等，如果能够定位到这些，问题也基本能够解决了，只不过为了确保问题的准确性，可以通过 trace 文件再确认一下。如果定位不到，那么就需要详细分析一下当前的 trace 文件，当前线程的执行情况，哪里导致 CPU 阻塞、锁住、用户态和内核态时间过长等现象，再去找到堆栈中可能导致出现这些现象的原因。

![anr_handle](https://github.com/RalfNick/PicRepository/raw/master/anr/anr_handle.png)

### 参考

[ANRs](https://developer.android.com/topic/performance/vitals/anr.html)

[Capture and read bug reports](https://developer.android.com/studio/debug/bug-report)

[彻底理解安卓应用无响应机制](http://gityuan.com/2019/04/06/android-anr/)

[今日头条 ANR 优化实践系列 - 设计原理及影响因素](https://juejin.cn/post/6940061649348853796)

[Android 调试系列-Bugreport实战篇
](https://www.jianshu.com/p/907028bac94a)

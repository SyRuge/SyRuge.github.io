---
title: Handler相关知识
date: 2018-10-20 20:49:55
tags: Android开发艺术探索笔记
categories: [Android开发艺术探索笔记]
---

### Handler

handler相关的几个类:Handler、Message、MessageQueue、Looper。

<!--more-->

### 主线程的Handler

ActivityThread类的Main方法中:

```java
public static void main(String[] args) {
    // 创建looper
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    if (sMainThreadHandler == null) {
        // 获取主线程的Handler
        sMainThreadHandler = thread.getHandler();
    }
    // 开始无限循环
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

Looper中：

```java
public static void prepare() {
    prepare(true);
}
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {// 确保ThreadLocal中只有一个Looper
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

new Looper中：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

### 发送消息和取出(MessageQueue)

* 插入消息采用的是链表结构，插入到链表尾部。

* 取对头消息

以上针对消息时间when依次递增，若为延时消息，则根据消息时间when来确定插入顺序。插入尾部或中间。

### 子线程能否使用Handler呢？

答案是肯定的：

```kotlin
private fun initThreadHandler(){
    thread{
        Looper.prepare()
        threadHandler=object:Handler(){
            override fun handleMessage(msg: Message) {
                Log.d("xcx","${msg.what}")
            }
        }
        Looper.loop()
    }
}
```

### 总结：

ActivityThread中初始化了当前线程的Looper，因为采用了ThreadLocal。looper内部有一个MessageQueue，最后开启无限循环。

### Handler内存泄漏

原因：内存泄漏时长生命周期对象持有短生命周期对象，导致短生命周期对象无法释放。

handler内部若有延时任务而此时activity准备销毁就会导致内存泄漏

解决方法:

* 继承Handler弱引用Activity

```kotlin
class MyHandler(activity: Activity) : Handler() {
    // 使用WeakReference弱引用持有Activity实例
    private val reference: WeakReference<Activity> = WeakReference(activity)
    override fun handleMessage(msg: Message) {
        Log.d("xcx", "${msg.what}")
    }
}
```

* onDestroy中移除handler中所有消息：

```kotlin
override fun onDestroy() {
    super.onDestroy()
    threadHandler.removeCallbacksAndMessages(null)
}
```
---
title: HandlerThread和IntentService
date: 2018-11-02 17:38:32
tags: Android开发艺术探索笔记
categories: [Android开发艺术探索笔记]
---

### HandlerThread

HandlerThread源码如下：

<!--more-->

```java
public class HandlerThread extends Thread {
    
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
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        // If the thread has been started, wait until the looper has been created.
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
    
    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }
}
```

核心代码与子线程使用Handler原理一样，就不多解释辣。

### IntentService

IntentService使用跟service几乎没有区别，不同的是IntentService帮我们实现了onBind和onStartCommand方法，因此如要重写这些方法，记得super一下。IntentService适用于执行后台耗时任务，任务完成后会自动停止，同时因为继承Service，比一般的后台线程要高，不容易被杀死。

既然继承Service，以此看一下onCreate、onstartCommand

```java
@Override
public void onCreate() {
    // TODO: It would be nice to have an option to hold a partial wakelock
    // during processing, and to have a static startService(Context, Intent)
    // method that would launch the service & hand off a wakelock.
    super.onCreate();
    HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
    thread.start();
    mServiceLooper = thread.getLooper();
    mServiceHandler = new ServiceHandler(mServiceLooper);
}
```

初始化了一个HandlerThread用于执行耗时任务。onStartCommand最终执行了onStart方法：

```java
@Override
public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
    onStart(intent, startId);
    return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
}
@Override
public void onStart(@Nullable Intent intent, int startId) {
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent;
    mServiceHandler.sendMessage(msg);
}
```

发送了一个消息，最终会在IntentService的内部类ServiceHandler中：

```java
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
protected abstract void onHandleIntent(@Nullable Intent intent);
```

最终会调用onHandleIntent，在这里我们去处理自定义的耗时任务，任务完成会自动销毁，由于是基于消息队列的，所以是串行执行。
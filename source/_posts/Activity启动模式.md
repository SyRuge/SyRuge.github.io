---
title: Activity启动模式
date: 2019-01-09 19:00:19
tags: Android
categories: [Android]

---

### Activity启动模式

<!--more-->

#### TaskAffinity任务相关性

* 标识了启动一个Activity需要的任务栈的名字
* 默认为应用的包名(如com.xcx.activity)
* 可以为每个Activity指定TaskAffinity属性(不能跟包名相同，否则等于没指定)
* TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用，在其他情况下没有意义
* TaskAffinity和allowTaskReparenting配合使用时，应用A启动应用B的Activity C，若C的allowTaskReparenting=true，
  C会从A的任务栈转移到B的任务栈(摘自Android开发艺术探索)

#### Activity指定启动模式

##### AndroidMenifest指定

``

```xml
<activity
    android:name=".activity.TestTabActivity"
    android:launchMode="singleTask"
    android:screenOrientation="portrait"
    />
```

##### 代码指定

``

```kotlin
val intent=Intent()
intent.setClass(this@MainActivity,TestTabActivity::class.java)
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
startActivity(intent)
```

##### 优先级和缺点

* 代码优先级更高，同时存在以代码为主
* AndroidMenifest无法直接指定FLAG_ACTIVITY_CLEAR_TOP
* 代码无法设置singleInstance模式
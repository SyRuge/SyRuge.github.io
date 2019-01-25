---
title: Activity生命周期
date: 2018-10-17 20:49:55
tags: Android开发艺术探索笔记
categories: [Android开发艺术探索笔记]
---

### 典型状态下：(Mi8 UD MIUI10)

home键：onPause->onSaveInstanceState->onsTop

未回收：onRestart->onStart->onResume

旧Activity先onPause，新Activity才启动(onCreate->...)

### 异常状态(Mi8 UD MIUI10)

#### 屏幕旋转

<!--more-->

切横屏和切竖屏都会引起activity销毁并重建，并调用onSaveInstanceState和onRestoreInstanceState方法。onSaveInstanceState在onStop之前，和onPause没有既定时序关系。onRestoreInstanceState在onStart之后，onResume之前。

#### 指定屏幕旋转不重建Activity

xml中指定： android:configChanges="orientation|screenSize|smallestScreenSize"

```xml
<activity android:name=".activity.CustomActivity"
          android:configChanges="orientation|screenSize|smallestScreenSize"
    >
    <intent-filter>
        <action android:name="android.intent.action.VIEW"/>
        <action android:name="android.intent.action.MAIN"/>

        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>
</activity>
```

最终会调用Activity的onConfigurationChanged方法，处理屏幕旋转问题。

```kotlin
override fun onConfigurationChanged(newConfig: Configuration?) {
    super.onConfigurationChanged(newConfig)
    Log.d("xcx","onConfigurationChanged")
}
```
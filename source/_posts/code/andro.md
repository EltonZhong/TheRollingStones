---
title: Android Instrumentation Test
cover: /imgs/11.png
tags:
  - Code
categories:
  - Code
date: 2019-08-07 21:10:27
---
# Android 测试

Android测试随着时间变化，很多工具被废弃，也有许多新的工具发展。

不同时期的android测试工具、框架![AndroidTest.png](https://upload-images.jianshu.io/upload_images/14858485-6a16ad5c1bc928b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![AndroidTest2.png](https://upload-images.jianshu.io/upload_images/14858485-1d8308f8d27d5283.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


简单地介绍以下其中几种比较重要或者常用的android 测试工具:
- [Mockito](https://site.mockito.org/): Mock android 平台相关依赖, 单元测试.
- [Robolectric](http://robolectric.org/): Run android tests inside the JVM on your workstation. Android 程序是跑在 Dalvik 虚拟机上的， 直接实现了android.jar的api， 参考[这里](https://www.jianshu.com/p/f17fa7e2253c)
- [MonkeyRunner](https://developer.android.com/studio/test/monkeyrunner): Like appium, use hidden api. [Chinese doc](https://yq.aliyun.com/articles/63501)

- [UiAutomator1]: 给UI做黑盒测试， 进程和和app进程独立, 写测试用例打包成jar， push到android设备内使用`uiautomator runtest 你的jar.jar`跑

- [ATSL](https://developer.android.com/training/testing/): Android Test Support Library
  - [Espresso](https://developer.android.com/training/testing/espresso)
  - [UiAutomator2](https://developer.android.com/training/testing/ui-automator)

## Instrumentation test
Instrumentation test现在是android测试的标准。
UIAutomator1也迁移到Instrumentation based这个框架下。

那什么是 Instrumentation test呢？

[文档自然是最吼的](https://developer.android.com/reference/android/app/Instrumentation.html)， 但是如果不愿意看文档， 可以继续往下看本篇文章

### What's instrumentation

- [Official doc](https://developer.android.com/reference/android/app/Instrumentation.html)
- [Chinese doc](https://yoyoyoky.github.io/2017/01/14/Instrumentation%E5%88%B0%E5%BA%95%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F/)
- [Chinese doc](https://testerhome.com/topics/6592)

App 创建时都会实例化一个Instrumentation对象, 在Activity之间共享实例， 控制Activity的生命周期, 使得Activity与控件的生命周期不需要被系统控制, 并且可以随意控制activity的创建与销毁。

如下面的代码:

```java
    // Start the main activity of the application under test
    mActivity = getActivity();

    // Get a handle to the Activity object's main UI widget, a Spinner
    mSpinner = (Spinner)mActivity.findViewById(com.android.example.spinner.R.id.Spinner01);

    // Set the Spinner to a known position
    mActivity.setSpinnerPosition(TEST_STATE_DESTROY_POSITION);

    // Stop the activity - The onDestroy() method should save the state of the Spinner
    mActivity.finish();

    // Re-start the Activity - the onResume() method should restore the state of the Spinner
    mActivity = getActivity();

    // Get the Spinner's current position
    int currentPosition = mActivity.getSpinnerPosition();

    // Assert that the current position is the same as the starting position
    assertEquals(TEST_STATE_DESTROY_POSITION, currentPosition);
```

几句话概括以下：
> Android instrumentation is a set of control methods or hooks in the Android system.

> These hooks control an Android component independently of its normal lifecycle.

> They also control how Android loads applications.

如何指定instrumentation?

在一个测试apk的 manifest中， 声明 targetPackage 和 name， 然后打包编译后使用
`adb shell am instrument ...`命令启动测试apk， 然后android系统会做两件事:

1. 将 targetPackage app进程重启
2. 然后实例化 name声明的 instrumentation类， 创建instrumentation线程

然后测试代码跑在instrumentation线程中， 并且可以控制、操作app进程、app上下文来实现自动化测试， 这就是 Android instrumentation test.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
      xmlns:tools="http://schemas.android.com/tools"
      package="com.example.android.testing.uiautomator.BasicSample.test"
      android:versionCode="1"
      android:versionName="1.0">
    <uses-sdk android:minSdkVersion="18" android:targetSdkVersion="28" />

    <instrumentation android:targetPackage="com.example.android.testing.uiautomator.BasicSample"
                     android:name="androidx.test.runner.AndroidJUnitRunner"/>

    <application tools:replace="label" android:label="BasicSampleTest" />
</manifest>
```

### Espresso VS UiAutomator v2 VS UiAuotomator v1

Refrences:

1. [Uiautomator1 vs espresso](https://stackoverflow.com/questions/31076228/android-testing-uiautomator-vs-espresso) (Remeber the answers are talking about uiautomator v1, in 2016)

2. [Uiautomator2 vs 1](https://kkboxsqa.wordpress.com/2017/09/18/appium-and-uiautomator2-part1/)

3. [Espresso](https://developer.android.com/training/testing/espresso)

   ```sh
   Each time your test invokes onView(), Espresso waits to perform the corresponding UI action or assertion until the following synchronization conditions are met:

   The message queue is empty.
   There are no instances of AsyncTask currently executing a task.
   All developer-defined idling resources are idle.
   By performing these checks, Espresso substantially increases the likelihood that only one UI action or assertion can occur at any given time. This capability gives you more reliable and dependable test results.
   ```

   - [Chinese doc](https://juejin.im/entry/589d39f361ff4b006b384055)

---

Below are some snippets:

1. A key benefit of using Espresso is that it provides automatic synchronization of test actions with the UI of the app you are testing. Espresso detects when the main thread is idle, so it is able to run your test commands at the appropriate time, improving the reliability of your tests. This capability also relieves you from having to add any timing workarounds, such as Thread.sleep() in your test code.
   The Espresso testing framework is an instrumentation-based API and works with the AndroidJUnitRunner test runner.
2. UiAutomator is powerful and has good external OS system integration e.g. can turn WiFi on and off and access other settings during test.
3. Based on uiAutomator
4. Espresso is targeted at developers, who believe that automated testing is an integral part of the development lifecycle. While it can be used for black-box testing, Espresso’s full power is unlocked by those who are familiar with the codebase under test.

UiAutomator1.0 use `uiautomator runtest yourtest.jar` cmd.

While uiautomator use `adb shell am instrument -w -r -e debug false -e class ...`

```sh
# special case pre-processing for 'runtest' command
if [ "${cmd}" == "runtest" ]; then
  # Print deprecation warning
  echo "Warning: This version of UI Automator is deprecated. New tests should be written using"
  echo "UI Automator 2.0 which is available as part of the Android Testing Support Library."
  echo "See https://developer.android.com/training/testing/ui-testing/uiautomator-testing.html"
  echo "for more details."
```

## Appium vs ATSL(Android native test)

Appium is based on ATSL now. UiAutomator2 is widely used now. The support for espresso is released, but due to espresso is more suitable for white box test, so it's not

1. Speed:

   - Espresso is suitable to cover simple cases, unit tests. It's much faster than uiautomator, and appium. (About 7 times faster than robotium).
   - UiAutomator is faster and than appium, cause http requests take time.

2. Ability:

   - Appium has access to the server connecting android devices, so you can call cmds like `adb install` when running cases. While you can not install or uninstall apps when using uiautomator directly. With `andro`'s helper, you can install/uninstall apps at the `beggining` of a case, or at the `end` of a case. While all `adb shell` commands are available in native test.
   - Appium is not suitable to write grey-box test, also you can access android methods through reflection, but without code base of android, it's hard. While with native-test, you're free to get the context of app, activities, and call special lifecycle methods in activities.
   - With appium, you can use almost all language to write your tests. While with the native test, you can only write with kotlin/java

3. Stability:

   - Appium use uiautomator-server apk and uiautomator-server-test apk. It should start a tcp port on the device, and recieve commands to controll the device. Compared to android test which is maintained by google team, appium is not as stable as native test, but acceptable.
   - Cases wrote in espresso don't need to poll the state of ui, so they are more stable. While comparing cases wrote in appium with cases wrote in uiautomator, they are almost the same except for that appium use network connection, which is not reliable.

---
title: Monkey：参数说明
date: 2018-04-07 09:19:33
categories:
- 技术向
tags:
- Android
- Test
---

> 本文旨在简单介绍说明 Monkey 及其参数，内容不一定完全正确，本文将随着实践及源码阅读深入而更新。

Monkey Test 是一个 Android 系统内置的工具，用来进行稳定性测试。它通过向系统注入伪随机的用户事件流（如点击、触摸、手势以及系统级的事件）来进行压力测试。

- Monkey 在 Android 系统中的存放路径是：/system/framework/monkey.jar
- Monkey 命令脚本在 Android 系统中的存放路径是：/system/bin/monkey
- Monkey 源码在 aosp/development/cmds/monkey

## 基本使用
Monkey 运行在 Android 设备上，所以要使用设备上的 shell 来启动。

`adb shell monkey [options] <event-count>`

一个典型的命令如下所示：

```
adb shell monkey -p com.android.contacts --throttle 500 --ignore-crashes --ignore-timeouts --ignore-security-exceptions -- ignore-native-crashes --monitor-native-crashes --kill-process-after-error -v -v -v 1200
```

<!--more-->

## Monkey 参数
Monkey 有很多选项，可以分为四个主要类别：General，Events，Constraints，Debugging。


```
➜  ~ adb shell monkey -h
  bash arg: -h
args: [-h]
 arg: "-h"
usage: monkey [-p ALLOWED_PACKAGE [-p ALLOWED_PACKAGE] ...]
              [-c MAIN_CATEGORY [-c MAIN_CATEGORY] ...]
              [--ignore-crashes] [--ignore-timeouts]
              [--ignore-security-exceptions]
              [--monitor-native-crashes] [--ignore-native-crashes]
              [--kill-process-after-error] [--hprof]
              [--match-description TEXT]
              [--pct-touch PERCENT] [--pct-motion PERCENT]
              [--pct-trackball PERCENT] [--pct-syskeys PERCENT]
              [--pct-nav PERCENT] [--pct-majornav PERCENT]
              [--pct-appswitch PERCENT] [--pct-flip PERCENT]
              [--pct-anyevent PERCENT] [--pct-pinchzoom PERCENT]
              [--pct-permission PERCENT]
              [--pkg-blacklist-file PACKAGE_BLACKLIST_FILE]
              [--pkg-whitelist-file PACKAGE_WHITELIST_FILE]
              [--wait-dbg] [--dbg-no-events]
              [--setup scriptfile] [-f scriptfile [-f scriptfile] ...]
              [--port port]
              [-s SEED] [-v [-v] ...]
              [--throttle MILLISEC] [--randomize-throttle]
              [--profile-wait MILLISEC]
              [--device-sleep-time MILLISEC]
              [--randomize-script]
              [--script-log]
              [--bugreport]
              [--periodic-bugreport]
              [--permission-target-system]
              COUNT

```

### General：基础参数

| 参数      | 说明                                 |
| --------- | ------------------------------------ |
| -h        | help 命令                            |
| -v [-v] … | 命令行中的每个 -v 都会增加详细级别。 |

-v 分为三个级别，详细说明如下：

- 级别0（默认值）：除了启动通知，测试完成和最终结果之外，几乎不提供其他信息。
- 级别1：提供更多详细信息，例如发送给 Activity 的每个事件信息。
- 级别2：提供更详细的设置信息，例如选中或未选中的 Activity 信息。

由于 Monkey 的结果通常使用脚本进行信息分析与提取，所以最常使用 `-v -v -v` 来收集详细的信息。

### Constraints：操作限制

如果只测试一个 App，可以指定只访问对应的包名。测试多个 App，可以多次指定或通过文件指定包名。

| 参数                                         | 说明                                                         |
| -------------------------------------------- | ------------------------------------------------------------ |
| -p ALLOWED_PACKAGE [-p  ALLOWED_PACKAGE] ... | 白名单：指定 Monkey 可以访问的包名，如果不指定默认可以访问所有的包。 |
| --pkg-blacklist-file PACKAGE_BLACKLIST_FILE  | 黑名单：通过文件来指定不可以访问的包名                       |
| --pkg-whitelist-file PACKAGE_WHITELIST_FILE  | 白名单：通过文件来指定可以访问的包名                         |
| --permission-target-system                   | 指定是否可以启动系统的应用，默认情况下不启动非白名单中的系统应用 |

> Note：黑名单跟白名单只能设置一个，不能同时使用。

从源码中可以看到判断能否启动某个 Package 与黑白名单及能否启动系统应用选项相关：

```java
private boolean shouldTargetPackage(PackageInfo info) {
    // target if permitted by white listing / black listing rules
    if (MonkeyUtils.getPackageFilter().checkEnteringPackage(info.packageName)) {
        return true;
    }
    if (mTargetSystemPackages
            // not explicitly black listed
            && !MonkeyUtils.getPackageFilter().isPackageInvalid(info.packageName)
            // is a system app
            && (info.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0) {
        return true;
    }
    return false;
}
```

要记住 Monkey 运行在设备上，所以黑白名单的文件需要放置到设备上，例如：

```
com.android.contacts
com.android.dialer
com.android.launcher
```

具体包名列表可从设备上读取，存成如上格式的文件后， push 到设备上：`/sdcard/whitelist.txt`

Monkey 启动时会开启可访问的包名的某个 Activity，具体来说，是启动 -c 参数指定的 category 的 Activity。一般不设置，即启动桌面图标对应启动的 Activity。

| 参数                                    | 说明                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| -c MAIN_CATEGORY [-c MAIN_CATEGORY] ... | category类别，如不手动指定，Monkey 会选择启动 Intent.CATEGORY_LAUNCHER 和  Intent.CATEGORY_MONKEY 的 Activity 为初始 Activity。 |

### Events：事件参数

Monkey 的可重复性体现在其事件流的生成取决于设置的种子数。

| 参数                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| -s SEED              | 伪随机数的种子值。 如果使用相同的种子运行，每次的事件序列都是相同的。 |
| --throttle MILLISEC  | 事件之间的时延，默认为 0 ，即尽快地产生事件                  |
| --randomize-throttle | 事件之间的时延随机为 [0,  throttle  设置值]                  |

为了模拟真实用户的操作频率，Monkey 测试时需要控制事件之间的时间，如果间隔时间太短则与实际操作相差太大。但为了提高应用的稳定性，必要时还是要考虑快速操作导致的问题。 另外，在低端手机上，由于系统响应慢，反映出来的真实用户的操作也确实可能存在间隔短的情况。

> Note：一般间隔时间取 300ms ~ 500ms

另外，事件参数中有一系列以 --pct 开头的控制每种事件百分比的参数，当测试不同类型 APP 时可以针对性地调整百分比。

> Note：--pct 数值后不用加百分号 %


| 参数                     | 说明                                                         |
| ------------------------ | ------------------------------------------------------------ |
| --pct-touch PERCENT      | 百分比，点击事件                                             |
| --pct-motion PERCENT     | 百分比，直线滑动事件                                         |
| --pct-trackball PERCENT  | 百分比，曲线滑动事件                                         |
| --pct-nav PERCENT        | 百分比，导航事件：上下左右；一般设备没有该事件               |
| --pct-majornav PERCENT   | 百分比，导航事件：返回、确认、菜单；一般设备没有该事件       |
| --pct-syskeys PERCENT    | 百分比，导航栏：Home、Back、音量键等                         |
| --pct-appswitch PERCENT  | 百分比，各Activity的启动比率，启动得越多，越容易覆盖更多的Activity |
| --pct-flip PERCENT       | 百分比，不清楚，模拟器适用的事件                             |
| --pct-permission PERCENT | 百分比，权限事件                                             |
| --pct-anyevent PERCENT   | 百分比，其他一些不常用的事件，各种按键之类的                 |
| --pct-pinchzoom PERCENT  | 百分比，多点手势缩放                                         |

源码中可以查看到默认的事件百分比：

```java
mFactors[FACTOR_TOUCH] = 15.0f;
mFactors[FACTOR_MOTION] = 10.0f;
mFactors[FACTOR_TRACKBALL] = 15.0f;
// Adjust the values if we want to enable rotation by default.
mFactors[FACTOR_ROTATION] = 0.0f;
mFactors[FACTOR_NAV] = 25.0f;
mFactors[FACTOR_MAJORNAV] = 15.0f;
mFactors[FACTOR_SYSOPS] = 2.0f;
mFactors[FACTOR_APPSWITCH] = 2.0f;
mFactors[FACTOR_FLIP] = 1.0f;
// disbale permission by default
mFactors[FACTOR_PERMISSION] = 0.0f;
mFactors[FACTOR_ANYTHING] = 13.0f;
mFactors[FACTOR_PINCHZOOM] = 2.0f;
```



### Debugging：调试选项

| 参数            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| --dbg-no-events | 使用此参数时 Monkey 会启动待测应用，但不发送任何事件，建议与 -v，-p，-throttle 一起使用来观测  Activity 之间的跳转 |
| --wait-dbg      | Monkey 停止执行，等待 Debugger 连接                          |
| --hprof         | 使用此参数时会在 Monkey 启动前后生成内存的快照，在设备的 /data/misc 目录下生成一个 hprof 文件。 |
| --port port     | 分配一个 TCP 端口给 Monkey，可用于远程调试                   |

| 参数                         | 说明                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| --ignore-crashes             | 通常，当应用程序崩溃或遇到任何类型的未处理的异常时，Monkey 将停止。 如果指定此选项，Monkey  将继续向系统发送事件。 |
| --ignore-timeouts            | 通常，当应用程序遇到任何类型的超时错误（如 ANR）时，Monkey 将停止。 如果指定此选项，Monkey  将继续向系统发送事件。 |
| --ignore-security-exceptions | 通常，当应用程序遇到任何类型的权限错误时，Monkey 将停止，例如，如果它尝试启动需要某些权限的活动。  如果指定此选项，Monkey 将继续向系统发送事件。 |
| --ignore-native-crashes      | 通常，当应用程序崩溃或遇到任何类型的未处理的异常时，Monkey 将停止。 如果指定此选项，Monkey  将继续向系统发送事件。 |
| --monitor-native-crashes     | 设置该选项来监控系统的 Native 层错误。如果设置了 kill-process-after-error  ，那么在发生错误时系统就会停止运行。 |
| --kill-process-after-error   | 一般情况下，Monkey 发生了某个错误将停止，而出问题的应用会留在系统上继续执行，设置该选项它会通知系统停止发生错误的过程。 |
| --bugreport                  | 将上报任何时候发生的 Crash                                   |
| --periodic-bugreport         | 生成报告的周期                                               |
| --match-description TEXT     | 仅报告与描述匹配的信息                                       |

| 参数                              | 说明                               |
| --------------------------------- | ---------------------------------- |
| --setup scriptfile                | 脚本指定用户的操作事件             |
| -f scriptfile [-f scriptfile] ... | 多个脚本指定用户的操作事件         |
| --profile-wait MILLISEC           | 使用脚本运行事件时的间隔时间       |
| --device-sleep-time MILLISEC      | 使用脚本运行事件时的设备空闲的时间 |
| --randomize-script                | 随机执行多个脚本来输入事件         |
| --script-log                      | （这个参数从源码中看没具体的作用） |

## 参考链接

* [UI/Application Exerciser Monkey](https://developer.android.com/studio/test/monkey.html)
* [android自动化测试之Monkey--从参数讲解、脚本制作到实战技巧](http://www.cnblogs.com/yajing-zh/p/4340795.html)
* [Monkey学习过程与个人见解_参数](https://www.jianshu.com/p/b1aa68f09de1)
* [Monkey学习过程与个人见解_报告分析](https://www.jianshu.com/p/2df7b1c3b0e9)
* [Android稳定性测试-- Monkey源码分析](https://blog.csdn.net/tobetheender/article/details/53448949)
* [Android稳定性测试-- Monkey二次开发](https://blog.csdn.net/tobetheender/article/details/54693874)
* [Android自动化测试之monkey工具使用(一)](http://blog.sina.com.cn/s/blog_e03267d30102wk0b.html)
* [Android monkey 资料](http://www.cnblogs.com/wfh1988/archive/2010/11/16/1878224.html)

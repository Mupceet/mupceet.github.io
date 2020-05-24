---
title: Android 性能优化——启动时间优化指南
date: 2017-04-15 21:26:33
categories:
- 技术向
tags:
- Android
- 性能优化
---

>请保持淡定，分析代码，记住：性能很重要。

毫无疑问，应用的启动速度越快越好。

本文可以帮助你优化应用的启动时间：首先描述应用启动过程的内部机制；然后讨论如何分析启动性能；最后，列举了一些常见的影响启动时间的问题，并就如何解决这些问题给出一些提示。

## 第 1 部分：启动过程内部机制
应用的启动可能为三种状态之一，不同状态的启动时长是不一样的。三种状态分别为：冷启动(cold start)，暖启动(warm start)，热启动(lukewarm start)。冷启动即应用从零开始加载运行，而其它状态则是应用从后台运行回到前台运行。建议始终基于冷启动的假设进行优化，因为这样做同样提升了另两种启动状态的表现。

要使得应用能快速启动，首先要理解应用以不同状态启动时，系统和应用内发生了什么，以及它们是如何交互的。

<!--more-->

### 1) 冷启动 (cold start)
冷启动状态：系统不存在该应用的进程，启动应用才创建出应用的进程。冷启动一般指的就是应用在开机后或者被系统停止后的第一次启动过程。因为系统和应用在冷启动时需要做更多的工作，所以减少它的启动时间的挑战是最大的。

冷启动初始时，系统完成三个任务：

- 启动和加载应用(这里泛指的是应用本身)
- 创建应用的专属进程
- 启动后立刻显示启动视图(通常是个空白屏)

一旦系统创建了应用的专属进程，该进程开始创建应用：

1. 创建应用对象
1. 启动主线程 (MainThread)
1. 创建 Main Activity
1. 加载视图 (Inflating views)
1. 渲染布局 (Laying out)
1. 执行初始绘制

一旦应用完成了第一次绘制，系统进程就把当前显示的启动视图切换为应用的界面，这时用户就可以开始使用应用了。

下图展示了系统和应用启动时相互之间的关系：

![系统和应用启动流程](http://upload-images.jianshu.io/upload_images/2946447-a2d44bd16a41271f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上流程中的大部分由系统来控制，**出现性能问题的地方往往在 Application 和 Activity 的创建 (`onCreate`) 过程中**。我们先仔细看下这两个创建过程。

#### a) Application 的创建

![Application onCreate](http://upload-images.jianshu.io/upload_images/2946447-b481411d09b82669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上文说到，当你启动应用时，屏幕将“立即”出现空白屏幕，这个空白屏幕将在应用完成首屏的绘制时切换为应用的首屏视图，然后允许用户开始与应用进行交互。而应用的创建是从 `Application.onCreate()` 开始的。

如果你在应用中重载了 Application.onCreate()，系统将先调用应用的该方法。大型的 App 通常会在这里做大量的通用组件、三方 SDK 的初始化操作。

然后应用程序生成主线程——也被称为 UI 线程，并开始创建 Main Activity。

#### b) Activity 的创建

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/2946447-03fb7f519e58b919.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

应用创建 Activity 的过程为：

1. 初始化(Activity init)
1. 调用构造函数
1. 调用当前生命周期的回调方法，例如 `Activity.onCreate()`

通常情况下，`onCreate()` 方法对加载时间的影响最大，因为它执行了开销最重的工作：加载、渲染和初始化 Activity 所需要的对象，如果布局过于复杂很可能导致严重的启动性能问题。

在这之后，系统和应用按各自的[生命周期](https://developer.android.google.cn/guide/topics/processes/process-lifecycle.html)运行着。

### 2) 暖启动(warm start)
应用程序的暖启动与冷启动类似，但比冷启动开销低。在暖启动中，系统只需要把 Activity 切换到前台运行。如果应用的该 Activity 之前驻留在内存中，那么应用程序就不用重新初始化对象和渲染布局。

但是，如果由于响应了低内存事件，例如在 `onTrimMemory()` 方法中清除了资源对象，那么这些对象就需要在热启动时重新创建。

暖启动与冷启动的显示情况是一致的：系统进程显示空白屏幕，直到应用程序已经完成 Activity 的渲染。

### 3) 热启动(lukewarm start)
热启动为冷启动的过程操作的子集，而且开销比暖启动稍小。以下这些情况可以认为是热启动：

1. 用户退出应用，但随后重新启动它。应用的进程还在运行，但应用必须重新从 `onCreate()` 开始创建 Activity。

2. 系统从内存中清除了应用(非用户主动)，然后用户重新启动它。进程和 Activity 需要重新启动，但 `onCreate()` 将接收到保存状态的 Bundle。事实上，`savedInstanceState` 在用户未主动销毁 Activity 时系统就会调用。

## 第 2 部分：剖析启动性能
为了正确评估启动时的表现，你需要跟踪应用启动到显示需要多长时间。下图展示了应用初始显示的时间和完全显示的时间的定义。

![两种显示时间](http://upload-images.jianshu.io/upload_images/2946447-84919dc240c6aac0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 1) 查看初始显示的时间
#### a) Displayed
从 Android 4.4(API 19) 开始，logcat 的输出包括了一行 `Displayed` 的值。这个值表示了应用启动进程到 Activity 完成屏幕绘制经过的时间。经过的时间包括以下事件，按顺序为：

1. 启动进程
1. 初始化对象
1. 创建和初始化 Activity
1. 布局渲染
1. 完成第一次绘制

报告的日志行看起来类似于下面的例子：
```
I/ActivityManager: Displayed com.android.contacts/.activities.PeopleActivity: +612ms
```

如果您在终端使用 logcat，可以直接找到这一行，当然，为了方便需要使用 `grep` 进行查找。而如果使用 Android Studio 查看，你必须在你的 logcat 视图中禁用过滤器，因为这是系统打的日志而不是应用本身。一旦您完成了过滤器设置，就可以轻松地搜索到该行查看时间。下图展示了如何禁用过滤器，及 logcat 窗口显示 Displayed 时间的例子。

![Android Studio logcat 窗口](http://upload-images.jianshu.io/upload_images/2946447-db47892d50dabaf5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Displayed 显示的时间是到第一次绘制完成的时候，它并不包括不被布局文件及初始化对象所引用的资源的加载时间，因为这个加载是一个内部过程，不阻塞应用初始内容的显示。

#### b) ADB Shell Activity Manager
你也可以使用 [ADB Shell Activity Manager](https://developer.android.google.cn/studio/command-line/shell.html#am) 测量启动到显示的时间。下面是一个例子：
```
adb shell am start -S -W com.android.contacts/.activities.PeopleActivity
-c android.intent.category.LAUNCHER
-a android.intent.action.MAIN
```
你的终端窗口就像显示 Displayed 一样地显示如下内容：
```
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.android.contacts/.activities.PeopleActivity }
Status: ok
Activity: com.android.contacts/.activities.PeopleActivity
ThisTime: 701
TotalTime: 701
WaitTime: 718
Complete

```
通过可选参数 -c 和 -a 可以指定 Intent 的 &lt;category> 和 &lt;action>。
* ThisTime:最后一个启动的 Activity 的启动耗时
* TotalTime:现在的所有的 Activity 的启动耗时
* WaitTime:ActivityManagerService 启动 App 的 Activity 时的总时间，包括前 Activity 的 onPause() 和现在 Activity 的启动

### 2) 查看完全显示的时间
#### a) reportFullyDrawn()
你可以使用 `reportFullyDrawn()` 方法来测量应用启动到所有资源和视图层次结构的完整显示之间所经过的时间，该方法在应用使用延迟加载的情况下是很有用的。

在延迟加载时，应用在初始的绘图之后，异步加载资源，然后更新视图。如果由于延迟加载，应用的初始显示并不包括所有的资源，你可能会考虑将所有的资源和视图的完全加载和显示作为一个单独的指标。例如：你的用户界面可能已经完成了文本的加载，但又必须从网络获取图像。

为了解决这个问题，你可以手动调用`reportFullyDrawn()`，让系统知道你的 Activity 完成了它的延迟加载。当您使用此方法，logcat 将显示出从创建应用对象到调用 `reportFullyDrawn()` 方法的时间。下面是 logcat 的输出的例子：
```
system_process I/ActivityManager: Fully drawn {package}/.MainActivity: +1s54ms
```

#### b) screenrecord
还有一种测量启动时间的方法值得一提，因为这种方法虽然繁琐但可以很直观查看起止位置的时间，那就是使用 `screenrecord` 命令。该命令可以直接录制屏幕，通过以下命令启动：
```
adb shell screenrecord --bugreport /sdcard/launch.mp4
```
在手机上操作，点击 App，等待其显示，必要时可以多等待一会儿，然后使用 `Ctrl + c` 停止命令，就得到了想要的视频了。使用命令导出视频：
```
adb pull /sdcard/launch.mp4
```
接着就可以使用一个能逐帧查看的视频播放器——例如 QuickTime 播放器来查看视频，一般地，认为 App 的图标高亮时为启动计时的起点，记录此时刻到你想要的停止的时刻之间的时间就可以了。简单来说，就是录制一个视频，使用逐帧查看的视频播放器方便地记录下你想查看的任意起止时刻。

如果通过以上四种方法测量出应用启动时间，你发现启动时间比预期要慢，你可以尝试着找出启动过程中的瓶颈。

### 3) 识别性能瓶颈
定位性能问题需要用到以下两种工具：Method Tracer 工具和 Systrace 工具。

#### a) Method Tracer 工具
在 Android Studio 的 CPU Monitor 栏中，提供了 Method Tracer 工具。

![Android Monitor 中的 Method Tracer ](http://upload-images.jianshu.io/upload_images/2946447-aebcefb26eba3aaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

首先需要启动要监控的应用，在 Android Studio 下方的 Android Monitor 中选择该应用的进程（图中长方框位置），就可以看到 Memory Monitor / CPU Monitor / Network Monitor 都开始工作起来。

如果要使用 Method Trace 功能，只需要点击 Start Method Tracing（图中小方框），在手机上进行操作之后，再次点击它停止 Method Trace，稍等片刻就能在工程的 captures 文件夹中找到 .trace 文件了。

由以上流程可以知道对于冷启动而言是无法在正确的时间启动该工具以获得日志信息的。这种情况下可以在代码中合适的位置，例如 `onCreate()` 和 `onWindowFocusChanged` 中，分别添加 `android.os.Debug.startMethodTracing()` 和 `android.os.Debug.stopMethodTracing()` 方法来生成 trace 文件，该文件生成在 sdcard 根目录下或者应用的目录中。

> Note: 运行 Method Trace 将明显地影响应用的运行速率。 所以 Method Trace 可以用来了解程序的流程及方法的运行时间的比例，其计时时间不可直接作为应用性能的表现。

使用 Android Studio 打开 trace 文件，如果是用 CPU Monitor 生成的 trace 文件，Android Studio 会自动打开它，你将得到如下形式的视图：

![Android Studio 中 trace 文件的展示](http://upload-images.jianshu.io/upload_images/1857887-96b3cc96d35f9a3f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

| 列名 | 具体含义|
| :--- | :--- |
| Name | 方法名 |
| Invocation Count | 方法调用次数 |
| Inclusive Time (microseconds) | 该方法及其调用的子方法的耗时 |
| Exclusive Time (microseconds) | 该方法（不包含调用的子方法）的耗时 |

图表的 x 坐标可以选择 `Wall Clock Time` 或者 `Thread Time`，其中前者表示方法调用到返回结果真实的 CPU 时间，后者表示线程调度的时间，如果线程不连续执行，那么被中断的时间将被排除，所以将小于前者的统计。另外，一般通过搜索方法名称以快速定位到图表中该方法的位置。关于 Method Tracer 及其视图的更多信息，请参阅：[Method Tracer](https://developer.android.google.cn/studio/profile/am-methodtrace.html)。

也可以使用 DDMS 打开 trace 文件，其展示的视图如下所示：

![DDMS 中 trace 文件的展示](http://upload-images.jianshu.io/upload_images/1857887-8579d47f882d95ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

各列名称及其含义与 Android Studio 的图示基本类似。

还可以使用 dmtracedump 工具解析生成 html 文件如下图（dmtracedump 可以生成图片，但往往混乱到看不出顺序，有兴趣的可以自行查阅相关资料）：

![dmtracedump 解析 trace 文件](http://upload-images.jianshu.io/upload_images/1857887-9456bc40135a7746.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从以上三种方式展示的 trace 文件结果来看，结果中包含了 JDK 函数，第三方库函数，以及 Android SDK 中函数，如果想仅分析应用中的方法调用顺序信息，可以根据 trace 文件过滤出当前应用下的方法信息。目前 GitHub 上有一个 Windows 平台下的[分析应用方法耗时的 swing 工具](https://github.com/Harlber/Method_Trace_Tool)，其使用方法很简单：

* 将 sdk\platform-tools 下的 dmtracedump 添加到系统环境变量
* 基于 jdk 1.8 环境运行 Method-trace-analysis.jar
* 直接导入 .trace 文件，一键分析（注意：trace 文件路径不要包含空格）

该工具的思路基于：[一个能让你了解所有函数调用顺序以及函数耗时的 Android 库（无需侵入式代码）](https://github.com/zjw-swun/AppMethodOrder)，该库核心就是 2 个 build.gradle 中的 task 基于 dmtracedump 工具对 trace 文件进行解析、过滤。

Method Trace Tool 得到了良好的展示效果，如图：

![Method Trace Tool 中 trace 文件的展示](http://upload-images.jianshu.io/upload_images/2946447-870b40a6df6766a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以上 trace 文件的几种展示方式可以让你了解到关于应用中方法的调用顺序及耗时占比信息（注意：该耗时信息不代表真正使用场景下的耗时，所以时间比例是个更有用的信息），基于以上信息可以分析出一个方法或者一个环节是否成为了性能瓶颈。

#### b) Systrace 工具
另一个跟踪的方法就是 Systrace 的使用了。

Systrace 是 Android 4.1 及以上版本提供的性能数据采样和分析工具。它可以帮助开发者收集 Android 关键子系统（如：surfaceflinger、WindowManagerService 等 Framework 部分关键模块、服务， View 系统）的运行信息，从而帮助开发者更直观地分析系统瓶颈，改进性能。

Systrace 的功能包括跟踪系统的 I/O 操作、内核工作队列、 CPU 负载等，很好收集分析 UI 显示性能的数据。 Systrace 工具可以跟踪、收集、检查定时信息，可以很直观地查看 CPU 周期消耗的具体时间，显示每个线程和进程的跟踪信息，使用了不同的颜色来突出问题的严重性，并提供了解决这些问题的一些建议。

使用方法：

1. 收集 trace 数据（具体可查看：[Systrace Walkthrough](https://developer.android.google.cn/studio/profile/systrace-walkthru.html)）
![Steps for starting Systrace](http://upload-images.jianshu.io/upload_images/2946447-01c6d79a0f928633.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![Steps for creating a trace.png](http://upload-images.jianshu.io/upload_images/2946447-d9f63b6282e1df4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 收集 trace 数据还可以通过命令行的方式，使用命令行配置好后多次使用可以快速得到数据，不用每次手动点击去收集。
```console
$ cd android-sdk/platform-tools/systrace
$ python systrace.py --time=10 -o mynewtrace.html sched gfx view wm
```
关于命令行的参数及配置请查看：[Systrace command reference](https://developer.android.google.cn/studio/profile/systrace-commandline.html)
2. 使用 Chrome 打开 trace.html 文件，使用 WASD 进行缩放、移动查看
![Systrace display after zooming in on a long-running frame.png](http://upload-images.jianshu.io/upload_images/2946447-54640c1e43a36e62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体的相关的信息分析可从网上查找经验总结博客。

> 注意：由于 Systrace 是以系统的角度返回一些信息，并不能定位到具体的耗时的方法，要进一步获取 CPU 被占用的原因，就需要使用另一个分析工具 Traceview。

刚才说到 Systrace 收集展示的是系统的信息，实际上在 4.3 之后，可以通过插入代码的方式，在 Systrace 里显示想要查看的 API 的耗时以及调用关系。举个例子：

```Java
public class MyAdapter extends RecyclerView.Adapter<MyViewHolder> {

    ...

    @Override
    public MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        Trace.beginSection("MyAdapter.onCreateViewHolder");
        MyViewHolder myViewHolder;
        try {
            myViewHolder = MyViewHolder.newInstance(parent);
        } finally {
            Trace.endSection();
        }
        return myViewHolder;
    }

   @Override
    public void onBindViewHolder(MyViewHolder holder, int position) {
        Trace.beginSection("MyAdapter.onBindViewHolder");
        try {
            try {
                Trace.beginSection("MyAdapter.queryDatabase");
                RowItem rowItem = queryDatabase(position);
                mDataset.add(rowItem);
            } finally {
                Trace.endSection();
            }
            holder.bind(mDataset.get(position));
        } finally {
            Trace.endSection();
        }
    }

…

}
```
通过 Trace.beginSection 和 Trace.endSection 来追踪应用的代码片段，有两个需要注意的地方：

1. 这两个 API 需要放在同一个线程里
2. 这两个 API 需要成对出现，而且每一个 endSection 都只会与最近的 beginSection 对应

了解更多关于 Systrace 的信息，请参阅 [Trace](https://developer.android.google.cn/reference/android/os/Trace.html) 功能的参考文档，以及 [Systrace](https://developer.android.google.cn/studio/profile/systrace-commandline.html) 工具的介绍。

## 第 3 部分：常见问题
本节讨论几个常见的影响应用启动性能的问题。主要是关注应用与 Activity 对象的初始化以及画面的加载。

### 1) Application 初始化开销大
正如上文所述，Application 的创建过程中，如果执行复杂的逻辑或者初始化大量的对象，将会影响应用的启动体验。具体而言，就是你继承了 Application 并在初始化时执行了不必要的代码，比如：初始化 MainActivity 的状态信息；创建了大量临时变量导致 GC（GC 在 ART 下影响很小）；执行磁盘 I/O 操作（这甚至就会直接阻塞应用的执行）；反序列化操作；多重循环等等。

#### 解决问题的方法
懒加载：只初始化那些必要的对象，而其他的全局静态对象移动到一个单例模式中。此外，可以考虑依赖注入框架 [Dagger2](http://google.github.io/dagger/) 来创建对象及其依赖关系。

### 2) Activity 初始化开销大
Activity 的创建中除了要避免 Application 创建中提到的问题，还需要注意以下问题：
* 加载极其复杂的布局
* 主线程中出现磁盘或网络 I/O
* 加载和解码 Bitmap
* 渲染多个 VectorDrawable 对象。

#### 解决问题的方法
这部分的问题要具体分析解决，常见的共通的两个问题的解决办法如下：

1. 视图层次过深：
  * 减少冗余、嵌套的布局层次。
  * 不布局绘制不可见的 UI，而是使用 ViewStub 对象在适当的时间布局绘制。
2. 大量的资源初始化：
  * 调整资源初始化的位置，可以在不同的线程执行懒加载。
  * 加载部分视图，然后再加载大的位图和其他资源。

### 3) 启动界面
文章开始就说到应用启动时会立即显示启动界面，而这通常是个白屏，你不妨给应用设置一个与主界面类似的启动画面，这样做可以向用户隐藏这个启动过程，用户会感受到应用已经在运行了，显示的界面就是应用的一部分或者说是流程的一部分。

有一个粗暴的办法是使用 windowDisablePreview 主题属性来去除应用启动时的空白屏。但这种方法会让用户点击之后觉得没有响应，而不知道应用已经开始启动了，这种体验不好，基本不会采用该方法。

#### 解决问题的方法
使用 Activity 的 windowBackground 属性，在启动时显示简单的自定义的画面。

首先创建一个要在启动时显示的画面，可以像如下所示：

```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" android:opacity="opaque">
  <!-- The background color, preferably the same as your normal theme -->
  <item android:drawable="@android:color/white"/>
  <!-- Your product logo - 144dp color version of your app icon -->
  <item>
    <bitmap
      android:src="@drawable/product_logo_144dp"
      android:gravity="center"/>
  </item>
</layer-list>
```
然后在自定义一个 style:
``` xml
<style name="AppTheme.Launcher" parent="@style/PeopleTheme">
    <item name="android:windowBackground">@drawable/start_activity_background</item>
</style>
```
在 AndroidManifest 文件中 Activity 的属性里设置该 style：
```xml
<activity ...
android:theme="@style/AppTheme.Launcher" />
```
这样子在应用启动时显示的画面就是你的自定义的画面了。但进入 Activity 后要正确的设置回正确的 style。

最简单的方法是在 `super.onCreate()` 之前调用 `setTheme(R.style.AppTheme)`，如下所示：
```java
public class MyMainActivity extends AppCompatActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    // Make sure this is before calling super.onCreate
    setTheme(R.style.Theme_MyApp);
    super.onCreate(savedInstanceState);
    // ...
  }
}
```

>参考文章：
* [Launch-Time Performance](https://developer.android.google.cn/topic/performance/launch-time.html)

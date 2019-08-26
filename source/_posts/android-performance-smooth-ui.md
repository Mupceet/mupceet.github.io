---
title: Android 性能优化——UI 优化指南
date: 2017-07-06 21:27:33
updated: 2017-07-09 00:13:33
categories:
- 技术向
tags:
- Android
- 性能优化
---

>请保持淡定，分析代码，记住：性能很重要。

人是一种视觉动物，如果你的应用长得不美，下场自不必多说，但这可是 UI 设计师的专属工作呀。作为工程师，对于视觉的体验上自然是从 UI 的性能优化入手了，从启动速度到应用 UI 优化，都是工程师讨好用户的手段。本文作为一份优化指南将帮助你优化你的应用的渲染性能，主要从两个方面入手：减少过度绘制，优化视图层级，内容包括了问题原因的阐述、问题的检测以及问题的解决方案。

## 减少过度绘制
过度绘制（Overdraw）是指应用在渲染一帧的时间内对屏幕某个像素进行多次绘制。应用应该尽可能避免过度绘制，因为让 GPU 去绘制用户不可见的内容完全是一种浪费。

<!--more-->

### 为什么会过度绘制
例如，在多层次的重叠的 UI 结构中，上层的 UI 遮挡了下层的 UI，系统是从后往前进行绘制（[painter's algorithm](https://en.wikipedia.org/wiki/Painter%27s_algorithm)），被遮挡的部分仍然会被绘制。为什么系统不直接绘制需要表现的 UI 而是从后往前绘制呢？系统采用此绘制算法是为了能给半透明的对象如阴影添加合适的透明度。

一般情况下，UI 元素由 XML 布局及自定义控件中定义。因此导致过度绘制的主要原因为：

* XML 布局 → 控件重叠；多次设置了背景
* 自定义 View → `onDraw()` 方法中同一个区域被多次绘制

### 诊断过度绘制
#### 初步查看
在设置中的开发者选项提供了过度绘制检测工具，此工具能够展示页面上哪些区域出现了不必要的过度绘制，可以直观查看应用当前页面是否存在过度绘制的现象，并且可以直观对比优化前后的显示效果。

按照以下步骤开启：

1. 点击设置中的**开发者选项**
2. 点击**调试 GPU 过度绘制**
3. 弹出框中选择**显示过度绘制区域**

这时可以看到页面出现了不同颜色的色块。不同的颜色代表不同程度的过度绘制：


![过度绘制区域颜色输出](http://upload-images.jianshu.io/upload_images/2946447-843ec9de1c4127da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


| 颜色 | Overdraw 倍数 | 像素点绘制次数 | 可接受区域大小 |
| ------------- |:-------------:| :-----:| :-----: |
| 无色 | 0 倍| 1 | 全部 |
| 蓝色 | 1 倍| 2 | 大片 |
| 绿色 | 2 倍| 3 | 中等 |
| 粉色 | 3 倍| 4 | 小于 1/4 |
| 红色 | 4 倍| 5 | 避免红色 |

过度绘制的优化目标是使得显示区域的过度绘制色块大部分为无色或者为蓝色，当然这个是比较理想的效果，不大面积出现红色即可视为达到目标了。

以天气应用为例，开启显示过度绘制区域后显示如下：


![天气应用过度绘制](http://upload-images.jianshu.io/upload_images/2946447-6d31186378969706.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/260)

可以看到列表区域存在红色程度的过度绘制的情况。

#### 详细定位
通过初步查看，看到了存在过度绘制的区域，但是这时候还不知道过度绘制的区域是由哪几层重叠形成的，这时候我们可以借助工具进一步定位。

最简便的方法便是借助常用的 Hierarchy View 工具进行分析。

1. 打开 Android Studio 的 Tools → Android → Android Device Monitor
2. 打开 Hierarchy View

  ![Hierarchy View](http://upload-images.jianshu.io/upload_images/2946447-15a01300022c0fee.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. Windows 窗口中选择相应的 Activity

  ![Activity Tree View](http://upload-images.jianshu.io/upload_images/2946447-0bbd0856441b38ff.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800)

4. 使用图片中圆圈标注的功能：Capture the window layers as a Photoshop document 将当前界面导出为一个带图层信息的 Photoshop 的文件

  ![导出.psd文件](http://upload-images.jianshu.io/upload_images/2946447-993b613ebde857ae.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5. 使用 GIMP 软件（Ubuntu）打开该 .psd 文件查看图层信息

  ![psd.jpg](http://upload-images.jianshu.io/upload_images/2946447-f37d8fb4be64fc2b.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/520)

  可以看到城市列表的背景部分被设置了两次。查看代码，发现是给列表每一项的layout都设置了背景，但是由于该列表的内容都会填充满该项的位置，故可以去掉该背景。

### 修复过度绘制
通过工具进行定位，找到了需要优化的页面，如上原因所述，过度绘制的成因分为两个方面，因此修复过度绘制分别从两方面着手。

#### XML 布局优化
##### 去除不必要的背景
首先要做的就是去除不必要的背景，多个有背景的布局控件放在一起就有可能导致过度绘制。

被上层视图背景覆盖下的内容可能永远都不会被用户看到，当子视图具有背景覆盖了父视图，特别是它们如果使用了相同的背景色时你将很不容易发现，这就需要上面的检测工具来定位过度绘制区域是由哪些层级的元素所覆盖形成的。一般的，优化布局移除背景可以总结为以下几点：

* 移除 XML 中不必要的背景，或根据条件设置
* 移除 Window 默认的背景
* 按需显示占位图片

其中，第二点指的是使用Android的自带的主题时，往往设置了一个默认的背景，这个背景由DecorView持有。当 App 的布局拥有另外的全局背景的时候，这个主题带的背景就是多余的，因此可以移除：

* 可以在Style里添加：
```xml
<item name="android:windowBackground">@null</item>
```

* 或者在 Activity 的 `onCreate` 方法中添加：
```Java
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    getwindow().setBackgroundDrawable(null);
}
```

举个去除背景的例子，便签应用优化前，存在一层不必要的过度绘制。

![未去除背景前](http://upload-images.jianshu.io/upload_images/2946447-db9b16a344cc1665.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/260)  

通过以上导出图层的方法进行查看后发现子视图 id/sv_note_editor  和 ﻿id/statusBarBackground 重复设置了背景。此处可以去除子 View  的背景，去除之后查看过度绘制情况，可以发现应用减少了一层过度绘制。

![去除背景后](http://upload-images.jianshu.io/upload_images/2946447-e5a8558e0451975f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/260)

##### 减少透明元素
在屏幕上显示透明的像素，称为 α 渲染，该情况也是导致过度绘制的一个关键因素。与标准的过度绘制不同，标准的过度绘制中是在上面绘制不透明的像素来完全覆盖已绘制的像素，而一个透明的对象必须要多层次绘制才能实现透明效果。如下图所示，每个有悬浮按钮控件的页面上可以看到一坨过度绘制。

![悬浮按钮：带阴影](http://upload-images.jianshu.io/upload_images/2946447-30016276b177c735.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/260)

事实上，透明动画，渐隐效果和阴影效果等视觉效果都涉及到了某种程度的透明，而这很有可能会导致过度绘制，因此通过减少渲染透明对象的数量可以减少过度绘制。例如，你可能会通过给一个字体颜色为黑色的 TextView 设置一个透明度来得到一个灰色的文本，可以替换成直接绘制灰色的文本，这样能得到同样的效果，但是性能更好。

##### 减少视图的层级
现在的设计的布局在每个视图对象都不透明的情况下，可能在屏幕上已经发生了重叠，如果是因为这种情况产生了过度绘制，可以通过优化视图层次结构来提高性能，以减少重叠的 UI 对象的数量。具体的减少视图层级的分析后文会继续提到。

#### 自定义 View 优化
在自定义 View 中的 `onDraw` 方法通过两个常用的方法来避免卡片式重叠（矩形式重叠）导致的过度绘制。关于以下这两个方法的使用，在你需要使用的时候上网查一下例子就知道了。

##### 快速判断是否需要绘制
在绘制一个区域之前，首先通过 `canvas.quickReject()` 方法判断该区域是否不和 Canvas 的剪切域（指定绘制区域）相交。返回 true 表示该区域与指定绘制区域不相交，这时直接绘制无须 GPU 的计算与渲染即不产生过度绘制；返回 false 即表示该区域与指定绘制区域相交了，这时可以指定自己的绘制区域为与原先剪切域 diff 的区域：`canvas.clipRect(rectF, Region.Op.DIFFERENCE)`。

##### 指定绘制区域
每个绘制单元都有自己的绘制区域，绘制前，`canvas.clipRect(Region.Op.INTERSECT) ` 帮助系统识别那些可见的区域。这个方法可以指定一块矩形区域，只有在这个区域内才会被绘制，其他的区域会被忽视。这个API 可以很好的帮助那些有多组重叠组件的自定义View来控制显示的区域。同时 clipRect 方法还可以帮助节约 CPU 与 GPU 资源，在clipRect区域之外的绘制指令都不会被执行，那些部分内容在矩形区域内的组件，仍然会得到绘制。

通过以上几项措施，可以有效修复过度绘制。当然，正如前文所述，完全消除过度绘制是一种理想化，业务中不可避免地需要对控件进行复用、封装，很容易在不知不觉中产生过度绘制，对此情况只要保持警惕，时刻检查，选择合适的手段，就能最大可能地避免严重的过度绘制现象。例如在 ListView 中每个 ItemView 如果相当复杂，就可以实现成一个自定义 View，通过指定绘制区域和重写 `requestLayout `、`onSizeChanged` 等措施来优化，可以使得滑动更加流畅。

## 优化视图层级
视图绘制过程包括一个测量（Measure）和布局（Layout）的过程。测量部分决定了 View 的大小：尺寸和边界；布局部分决定了 View 在屏幕上的位置。

大多数时候，每个 View 在这两个过程的消耗都很少，不会影响到性能。然而，当应用添加或者移除一些 View 对象的时候，例如当一个 Recyclerview 重用条目的时候，这个消耗会变大。另外当一个 View 对象是自适应时消耗也会更高，例如，一个 wrapcontent 的 TextView 对象调用了 `setText()`  方法时候，它需求重新计算尺寸。如果上述情况消耗时间过长，就会导致一帧无法在规定的 16ms 中完成绘制，那么这些帧就会被丢弃，用户就可能觉得卡顿。

但是因为 UI 操作只能在主线程中执行，你不能将它们移到子线程中去执行，所以最好还是对视图进行优化，减少它们的时间消耗。

以下先介绍两种会影响绘制性能的问题。

### 绘制性能
#### 复杂度：不同布局与深度
在 Android 系统中绘制源码是在 ViewRootImp 类的 performTraversals() 方法中 ，可以看到 Measure 和 Layout 都是通过以深度优先的递归来完成的，需要遍历子层级的 View，因此，层级越深，元素越多，耗时也就越长，特别在层级太深时，每增加一层会增加更多的耗时。

常见的绘制耗时长的布局的特点就是视图层级深，进行了多层的嵌套。每一层嵌套都给布局增加了消耗，因此解决问题的根本办法就是使视图层级变得扁平。举个例子，使用 RelativeLayout 进行的布局与嵌套的无权重的 LinearLayout 效果相同，由于 RelativeLayout 有下文提到的 Double Taxation 现象，所以 LinearLayout 布局的性能更好，但如果此时深度很深，就要考虑增加层级是否是正确的。

#### Double Taxation
通常情况下，布局或者测量的过程只需要进行一次，这个过程一般都很快。然而，在一些复杂的布局情况下，可能需要多次遍历层次结构的各个部分，这些部分需要经过多次测量才能最终定位。这种需要执行超过一次的布局和测量的迭代叫做Double Taxation。以下是不同的布局的 Double Taxation 现象：

##### RelativeLayout
当使用 RelativeLayout 时候，它需要根据一个 View 的位置来确定另一个 View 的位置：

1. 执行第一次布局-测量的过程，在这个过程中，根据每个子 View 的需求计算它们的位置和尺寸
2. 通过这些数据，再结合这些 View 对象的权重，确定它们的合适的位置
3. 执行第二次的布局过程来最终确定这些视图的位置

也就是说 RelativeLayout 布局一定会做两次测量。

##### LinearLayout
LinearLayout 如果为横向布局时候，需要执行两次布局-测量过程。在竖向布局时候，如果添加了  measureWithLargestChild 属性，也有可能会需要执行两次的布局-绘制过程，因为在这种情况下，Framework 可能需要执行两次流程来确定对象的合适尺寸。

##### GridLayout
GridLayout 也允许相对放置 View，它通常是通过预处理子 View 之间的位置关系来避免双倍消耗。然而，当它使用 Weight 或者 Gravity 属性时，就会失去预处理的优势，如果再包含有 RelativeLayout 的话，此时可能就要更多次的布局测量流程。

事实上，多次的布局-测量流程即 Double Taxation 本身并不一定是负担。但是 Double Taxation 发生在以下布局层级中就要注意了：

* 布局的根元素
* 有一个深层级的结构
* 有很多实例填充在屏幕上，类似 ListView 中的子条目

在以上情况中就要尽可能地避免出现 Double Taxation 现象。

通过以上两点，可以看出选择 RelativeLayout 还是 LinearLayout 并不是绝对的，本身层级太深的话就推荐使用 RelativeLayout 减少布局本身的层次，否则使用性能更好的 LinearLayout 更合适。

##### ConstraintLayout
当然，如果应用面向 7.0 开发，可以使用 ConstraintLayout 代替 RelativeLayout，可以避免本节描述的许多问题。ConstraintLayout 提供了与 RelativeLayout 相似的布局控制功能，但是性能更好。因为它与普通的布局不同，就是它使用自己的 constraint-solving 系统来解决视图之间的关系。

### 诊断绘制性能
#### 初步查看
##### Profile GPU rendering
与过度绘制的诊断类似，在设置中的开发者选项提供了 GPU 呈现模式分析（Profile GPU rendering）工具，此工具能够展示绘制一帧时，布局-测量流程花费了多少时间。

按照以下步骤开启：

1. 点击设置中的**开发者选项**
2. 点击**GPU 呈现模式分析**
3. 弹出框中选择**在屏幕上显示为条形图**

这时可以看到页面出现了条形图。每一条条形图代表每一帧的绘制情况，条形图上的不同的颜色代表绘制的不同过程：

![Profile GPU rendering](http://upload-images.jianshu.io/upload_images/2946447-0b8e5d105e3fc66a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/260)

可以看到屏幕上有一条绿线，条形图在绿线之下代表该帧的绘制时间在 16ms 之内，如果一个应用的大部分条形图都超过了绿线，那么该应用给用户的感受就是明显的卡顿感。

条形图的颜色在不同的系统版本上是不一样的，在 6.0 及更高的版本有 8 种颜色，在 4.0 (API level 14) 到  5.0 (API level 21) 之间只有 4 种颜色。用[表格来描述一下](https://developer.android.google.cn/studio/profile/dev-options-rendering.html)：


![6.0以上条形图](http://upload-images.jianshu.io/upload_images/2946447-0b23a1a283eb06c2.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![4.0 and 5.0 条形图.jpg](http://upload-images.jianshu.io/upload_images/2946447-892b0b81ba604d34.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体的颜色的含义这里就不详细说了，只需要关注是否超过了绿线以及是否是绘制的测量、布局流程耗时占比很大。GPU Profile 工具可以很简便地帮助你找到渲染有问题的页面。

##### Systrace
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
![Clicking the Alert button to the right reveals the alert tab.png](http://upload-images.jianshu.io/upload_images/2946447-1f0ceb9f4cef4daf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

与 UI 性能相关主要是右上角的 Alerts 选项以及对应的 Frame 数据，Alerts 选项中将列出渲染时间超时的帧，选中该 Alert 可以看到窗口下方展示了该 Frame 问题的详细数据描述以及相关的建议，并且会定位出对应的 Frame 行的对应位置。

Frame 行上有圆圈，如果是绿色的，表示该帧渲染满足性能要求，即在 16ms 内渲染完毕，如果是黄色、红色则代表渲染时间超过了 16ms。使用 W 键放大后可以看到系统在这一帧中具体做了什么。具体的相关的信息分析可从网上查找经验总结博客。

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

更多关于 Systrace 的信息请查看 [Analyzing UI Performance with Systrace](https://developer.android.google.cn/studio/profile/systrace.html) 。

这样子通过查看 Systrace 就可以查看到应用的页面是否存在渲染问题，并且可以初步定位到问题的原因所在，然后可以通过插入代码增加 trace 的方式去分析例如 ListView 中的 getView 方法的耗时，相对于打 Log 的方式会更加地直观方便查看耗时数据。

#### 详细定位
##### Hierarchy View
上文提到Android Studio 的Hierarchy Viewer有着强大的视图调试功能 ，它使用图形化来表现视图的结构。它呈现的视图可以用来分析由Double Taxation引起的性能问题。它也可以很容易定位到因为深层嵌套或者嵌套了大量子类的布局导致布局-测量流程非常耗时引起的性能问题。

这里介绍它的另一个功能：Profile Node。
![profile node](http://upload-images.jianshu.io/upload_images/2946447-8b01d5d448d2c808.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/780)

选择上图中红框圈中的最后一个图标：obtain layout times for tree rooted as selected node，可以获得布局-测量流程所消耗的相对时间信息。如下图所示，需要注意的是，图中的圆圈的颜色是与同级的视图相比较得出的，与 Systrace 中颜色的含义有所不同。
![profile node result](http://upload-images.jianshu.io/upload_images/2946447-a03907d74dfef56f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### Lint
Lint 扫描通过静态扫描检查代码的方式，能够发现在代码中潜在的问题，同时给出问题的原因和在代码中的位置，并给出相应的优化建议。

Lint 的功能非常强大，开发者应该深入学习使用方法，可进行配置检查选项甚至自定义检查规则。扫描规则和缺陷级别的配置在 File → Settings → Inspections → Android Lint。这里我们只关注 Performance 规则。

![Lint Performance](http://upload-images.jianshu.io/upload_images/2946447-21fb15affbfb21f5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

共有 29 项规则，默认选中 27 项，举些例子：

1. Layout has too many views：表示控件太多，默认超过 80 个控件会提示该问题
2. Layout hierarchy is too deep：表示层级太深，默认超过 10 层会提示该问题
3. Useless parent layout：表示无用的父布局，应该移除避免加深布局层级
4. Node can be replaced by a TextView with compound drawables：表示可优化的布局，即一个 ImageView 和一个 TextView 线程布局可以使用 CompoundDrawable 的 TextView 代替

一般通过 Lint 扫描都会扫描出代码中存在性能问题，但是对于具体的问题是否要解决是要衡量一下的，不是说每一个提示都需要去解决。

### 优化布局层级
#### 布局复用
可重用布局这项功能特别强大，它可以使你创建那些复杂的可重用布局，一个相同的布局可以在很多页面使用。比方说，可以用来创建一个含有 yes 和 no 按钮的容器或者一个含有 progressBar 及一个文本框的容器。虽然说你可以通过自定义 View 的方式来实现更为复杂的 UI 组件，但是重用布局的方法更简便一些，修改起来不会有遗漏。Android 的布局复用通过 include 标签来实现。
1. 创建一个可重用的布局
如果你已经知道哪一个布局需要重用，那么就创建一个新的 xml 文件用来定义这个布局。下面就定义了一个 ActionBar 的布局文件，众所周知，ActionBar 是会在每个 Activity 中统一出现的：
```xml
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width=”match_parent”
    android:layout_height="wrap_content"
    android:background="@color/titlebar_bg">
    <ImageView android:layout_width="wrap_content"
               android:layout_height="wrap_content"
               android:src="@drawable/gafricalogo" />
</FrameLayout>
```
2. 使用 include 标签
在希望添加重用布局的布局内，添加 include 标签。下面的例子就是将上面的布局加入到了当前的布局中：
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:layout_width=”match_parent”
    android:layout_height=”match_parent”
    android:background="@color/app_bg"
    android:gravity="center_horizontal">
    <include layout="@layout/titlebar"/>
    <TextView android:layout_width=”match_parent”
              android:layout_height="wrap_content"
              android:text="@string/hello"
              android:padding="10dp" />    ...
</LinearLayout>
```
你也可以重写布局的参数，但只仅限于以 `android:layout_*` 开头的布局参数。就像下面这样：
```xml
<include android:id=”@+id/news_title”
         android:layout_width=”match_parent”
         android:layout_height=”match_parent”
         layout=”@layout/title”/>
```
如果你要重写 include 标签指定布局的布局属性，那么必须重写 android:layout_height 及 android:layout_width 这两个属性，以便使其它属性的作用生效。


#### 减少层级
使用 Hierarchy View 查找 UI 布局不合理的地方主要关注两个问题：

1. 冗余的父布局。
  在视图树上可以定位此问题，也就是看到一长串的没有分支的布局就要注意了，是不是有的布局：没有背景绘制、没有大小限制，这种布局就是无用的父布局，如果是在布局文件里不小心写出了这样的父布局，使用 Lint 工具可以直接提示你。一般来说引入此问题的原因是使用了 include 标签，使用此标签很容易导致不注意的情况下多了一个无用的父布局。对于该问题，可以通过 merge 标签合并来减少UI层级。
2. LinearLayout 带来过深的层级
  同样的，看到一长串的没有分支的布局时可以看一下是不是使用 LinearLayout 嵌套布局了，如果有此类问题，可以使用 RelativeLayout 来代替 LinearLayout。但正如上面所说，它们的相互替代的效果是需要衡量的，可以使用 profile node 来检测前后的效果。

##### RelativeLayout、LinearLayout、ConstraintLayout
正如上面所说，使用不同的布局可以达到相同的效果，但最终由于层级与测量流程的不同，哪种布局下的性能最好是需要修改后再测量考虑的。但有几个原则是可以遵循的。
1. 能使用 ConstraintLayout 就使用它，不能的话尽量使用 RelativeLayout 和 LinearLayout
2. 在布局层级相同的情况下，使用 LinearLayout
3. 在层级过深时，使用 RelativeLayout 使界面扁平化

##### 合理使用 merge
在将一个布局内嵌进另一个布局时，merge 标签可以帮助消除冗余的 View 容器。举个例子，如果你的主布局是一个垂直的 LinearLayout，在它的内部含有两个 View，并且这两个 View 需要在多个布局中重用，那么重用这两个 View 的布局需要有一个 root View。然而，使用单独的 LinearLayout 作为这个 root View 会导致在一个垂直的 LinearLayout 中又嵌了一个垂直的 LinearLayout。其实这个内嵌的 LinearLayout 并不是我们真正想要的，此外它还会降低UI性能。

为了避免出现这种冗杂的 View 容器，你可以使用 merge 标签作为这两个 View 的 root View：
```xml
<merge xmlns:android="http://schemas.android.com/apk/res/android">
    <Button
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:text="@string/add"/>
    <Button
        android:layout_width="fill_parent"
        android:layout_height="wrap_content"
        android:text="@string/delete"/>
</merge>
```
那么现在再使用这个布局的时候，系统会自动忽略 merge 标签，并会将两个 Button View 直接加入到布局 < include/> 标签所指定的位置。
>注意：如果 Merge 代替的布局元素为 LinearLayout，在自定义布局代码中要将 LinearLayout 的属性添加到引用上，如垂直、水平布局、背景色等。Merge 不是哪里都可以用的，显然它只能在 xml 文件的根元素上，而且还要注意以下两点：
1. 使用 Merge 来加载一个布局时，必须指定一个 ViewGroup 作为其父元素，并且要设置加载的 attachToRoot 参数为 true（参照 inflate(int, ViewGroup, Boolean)）；
2. 不能在 ViewStub 中使用 Merge 标签，原因就是 ViewStub 的 inflate 方法中根本没有 attachToRoot 的设置。

这里讲了如何减少层级，那么多少层才是合理的呢?从 Lint 的检查配置上来看，超过 10 层才会报警，所以我们可以认为超过 15 层就必须重视开始准备优化，再多层就是一定要修改的了。

#### 提高绘制速度
上文提到绘制需要进行布局与测量，并且层级越深，元素越多，耗时也就越长。在实践中，有时候我们的布局文件中存在很多只在特定情况下才会使用到的 View。针对这些 View，很多时候我们会使用设置可见性的方式来确保只在需要时才会显示。

但是设置为 Gone 并不能解决性能问题，绘制流程中还是会测试和解析这些布局的。在 inflate 布局文件的时候，依然会去创建 GONE 属性的实例，初始化对象。我们知道创建对象以及测量-布局流程耗费很高，如果创建大量当前不需要显示的 View 对象，会很大程度上增加启动时间。对于减少内存使用来说，设置可见性也没有任何用处。

在上述情况下，推迟资源加载是非常重要的解决手段，通过“在需要时才去加载”的方式来降低内存使用和加快绘制速度。Android 提供了 ViewStub 控件来解决这个场景，我们可以通过声明 ViewStub 来实现推迟 View 加载。ViewStub 是一个轻量级的 View，它的构造函数简单、成员变量很少，对象创建时间更短。而且它的尺寸为0，并且不会绘制任何东西。

```Java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(0, 0);
}

@Override
public void draw(Canvas canvas) {
}

@Override
protected void dispatchDraw(Canvas canvas) {
}
```

ViewStub 在布局文件中的使用与其他的 View 一致，只是需要增加 layout 属性：
```xml
<ViewStub
    android:id="@+id/ll_draw"
    android:layout_width="@dimen/drawer_width"
    android:layout_height="match_parent"
    android:layout_gravity="start"
    android:layout="@layout/stub_activity_main_drawer" />
```
当需要使用到 ViewStub 对应的 Views 时，只需要调用 ViewStub#inflate() 方法或将其可见性设置为 VISIBLE 即可。需要注意的是，ViewStub#inflate() 只能调用一次，因为 inflate 之后，ViewStub 会被对应的 Views 替换，ViewStub 会从原来的 Parent 中被移除，如果再次调用 ViewStub 就会抛出异常。
```Java
ViewStub stub = (ViewStub) findViewById(R.id.ll_draw);
View drawer = stub.inflate();
```

## 布局原则总结
通过以上的分析过程，可以得出一些通用的准则，在 Android UI 布局过程中，遵守这些惯用、有效的布局原则，可以制作出高效且复用性高的 UI。

1. 尽量多使用 ConstraintLayout、RelativeLayout、LinearLayout
  * 尽量使用 ConstraintLayout
  * 在布局层级相同的情况下，使用 LinearLayout 代替 RelativeLayout
  * 在布局复杂或层级过深时，使用 RelativeLayout 代替 LinearLayout 使界面层级扁平化
2. 将可复用的组件抽取出来并通过 include 标签使用
3. 使用 merge 减少布局的嵌套层级
4. 使用 ViewStub 加载一些不常用的布局
5. 尽可能少用 layout_weight
6. 去除不必要的背景，减少过度绘制
  * 有多层背景重叠的，保留最上层。或者可以统一的使用一个大的背景
  * 对于 Selector 当背景的，可以将 normal 状态的 color 设置为 @android:color/transparent

>参考文章：
* [Reducing Overdraw](https://developer.android.google.cn/topic/performance/rendering/overdraw.html)
* [Debug GPU Overdraw Walkthrough](https://developer.android.google.cn/studio/profile/dev-options-overdraw.html)
* [Profile GPU Rendering Walkthrough](https://developer.android.google.cn/studio/profile/dev-options-rendering.html)
* [Systrace Walkthrough](https://developer.android.google.cn/studio/profile/systrace-walkthru.html)
* [Analyzing UI Performance with Systrace](https://developer.android.google.cn/studio/profile/systrace.html)
* 罗彧成.Android应用性能优化最佳实践.机械工业出版社,2017：12-49
* TMQ专项测试团队.移动App性能评测与优化,2016：71-91

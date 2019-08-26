---
title: Android 内存管理机制（入门）
date: 2018-04-03 22:47:33
categories:
- 技术向
tags:
- Android
- GC
---

>**不建议在知识深度广度不够的时候尝试深入理解当前版本的 Android 的内存管理机制，如果一定要这样做，请考虑候世达定律**

所谓内存管理，就是指内存资源的分配、使用、释放与回收。在 Android 中指的就是对象分配与垃圾回收的过程。

[老罗的Android之旅](http://blog.csdn.net/luoshengyang)博客中对相关的内容有很深入的解析，篇幅宏大。以下内容为学习相关内容博客的摘要，想了解更多或对本文有不理解的地方请移步老罗的博客，下文给了参考的具体博客的文章链接。

## Dalvik 虚拟机的内存机制

因为 Dalvik 虚拟机的内存机制相对比较简单，并且与 ART 使用了共同或相通的技术，所以先理解 Dalvik 下的内存机制。

>参考链接：
1. [Dalvik虚拟机垃圾收集机制简要介绍和学习计划 ]( http://blog.csdn.net/luoshengyang/article/details/41338251)
2. [Dalvik虚拟机堆的创建过程](http://blog.csdn.net/luoshengyang/article/details/41581063)
3. [Dalvik虚拟机的对象分配过程](http://blog.csdn.net/luoshengyang/article/details/41688319)
4. [Dalvik虚拟机的垃圾收集过程](http://blog.csdn.net/luoshengyang/article/details/41822747)

<!--more-->

### 基本概念

![Dalvik虚拟机垃圾收集机制的基本概念](https://upload-images.jianshu.io/upload_images/2946447-876e2a8cec040937.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Dalvik 虚拟机的堆

Dalvik 虚拟机用来分配对象的堆划分为两部分，一部分叫做 Active Heap，另一部分叫做 Zygote Heap。Android 系统的第一个 Dalvik 虚拟机是由 Zygote 进程创建的，而应用程序进程是由 Zygote 进程 fork 出来的。也就是说，应用程序进程使用了一种写时拷贝技术（COW）来复制了 Zygote 进程的地址空间。这意味着一开始的时候，应用程序进程和 Zygote 进程共享了同一个用来分配对象的堆。然而，当 Zygote 进程或者应用程序进程对该堆进行写操作时，内核就会执行真正的拷贝操作，使得 Zygote 进程和应用程序进程分别拥有自己的一份拷贝。

拷贝是一件费时费力的事情。因此，为了尽量地避免拷贝，Dalvik 虚拟机将自己的堆划分为两部分。事实上，Dalvik 虚拟机的堆最初是只有一个的，也就是 Zygote 进程在启动过程中创建 Dalvik 虚拟机的时候，只有一个堆。但是当 Zygote 进程在 fork 第一个应用程序进程之前，会将已经使用了的那部分堆内存划分为一部分，还没有使用的堆内存划分为另外一部分。前者就称为 Zygote 堆，后者就称为 Active 堆。以后无论是 Zygote 进程，还是应用程序进程，当它们需要分配对象的时候，都在 Active 堆上进行。这样就可以使得 Zygote 堆尽可能少地被执行写操作，因而就可以减少执行写时拷贝的操作。在 Zygote 堆里面分配的对象其实主要就是 Zygote 进程在启动过程中预加载的类、资源和对象了。这意味着这些预加载的类、资源和对象可以在 Zygote 进程和应用程序进程中做到长期共享。这样既能减少拷贝操作，还能减少对内存的需求。

明白了 Dalvik 虚拟机为什么要把用来分配对象的堆划分为 Active 堆和 Zygote 堆之后，我们再看到底堆是个什么东西：

![Dalvik虚拟机Java堆](https://upload-images.jianshu.io/upload_images/2946447-29d6ef9fdee0d20d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在 Dalvik 虚拟机中，堆实际上就是一块匿名共享内存。Dalvik 虚拟机并不是直接管理这块匿名共享内存，而是将它封装成一个 mspace，交给 C 库来管理。为什么要这样做呢？因为内存碎片问题其实是一个通用的问题，不单止 Dalvik 虚拟机在 Java 堆为对象分配内存时会遇到，C 库的 malloc 函数在分配内存时也会遇到。Android 系统使用的 C 库 bionic 使用了 Doug Lea 写的 dlmalloc 内存分配器，也就是说，我们调用函数 malloc 的时候，使用的是 dlmalloc 内存分配器来分配内存。这是一个成熟的内存分配器，可以很好地解决内存碎片问题。于是，Dalvik 虚拟机就很机智地利用 C 库里面的 dlmalloc 内存分配器来解决内存碎片问题！关于 dlmalloc 内存分配器的设计，可以参考这篇文章：[A Memory Allocator](http://gee.cs.oswego.edu/dl/html/malloc.html)。

#### GC 简述

Dalvik 虚拟机使用标记-清除（Mark-Sweep）算法进行 GC （garbage collection）。 顾名思义，Mark-Sweep 垃圾收集算法主要分为两个阶段：Mark 和 Sweep。Mark 阶段从对象的根集开始标记被引用的对象。标记完成后，就进入到 Sweep 阶段，而 Sweep 阶段所做的事情就是回收没有被标记的对象占用的内存。为了管理 Java 堆，Dalvik 虚拟机需要一些辅助数据结构，包括一个 Card Table、两个 Heap Bitmap 和一个 Mark Stack。Card Table 是为了记录在垃圾收集过程中对象的引用情况的，以便可以实现 Concurrent GC。图 1 的两个 Heap Bitmap，一个称为 Live Heap Bitmap，用来记录上次 GC 之后，还存活的对象，另一个称为 Mark Heap Bitmap，用来记录当前 GC 中有被引用的对象的状态。这样，上次 GC 后存活的但是当前 GC 不存活的对象，就是需要释放的对象。

#### 堆的大小

Java 堆的起始大小（Starting Size）指定了 Davlik 虚拟机在启动的时候向系统申请的物理内存的大小。后面再根据需要逐渐向系统申请更多的物理内存，直到达到最大值（Maximum Size）为止。这是一种按需要分配策略，可以避免内存浪费。在默认情况下，Java 堆的起始大小（Starting Size）和最大值（Maximum Size）等于 4M 和 16M。但是厂商会通过 dalvik.vm.heapstartsize 和 dalvik.vm.heapsize 这两个属性将它们设置为适合设备的值的。

注意，虽然 Java 堆使用的物理内存是按需要分配的，但是它使用的虚拟内存的总大小却是需要在 Dalvik 启动的时候就确定的。这个虚拟内存的大小就等于 Java 堆的最大值（Maximum Size）。想象一下，如果不这样做的话，会出现什么情况。假设开始时创建的虚拟内存小于Java堆的最大值（Maximum Size），由于实际情况是允许虚拟内存的大小是达到Java堆的最大值（Maximum Size）的，因此，当开始时创建的虚拟内存无法满足需求时，那么就需要重新创建另外一块更大的虚拟内存。这样就需要将之前的虚拟内存的内容拷贝到新创建的更大的虚拟内存去，并且还要相应地修改各种辅助数据结构。这样太麻烦了，而且效率也太低了。因此就在一开始的时候，就创建一块与 Java 堆的最大值（Maximum Size）相等的虚拟内存。

但是，Dalvik 虚拟机又希望能够动态地调整Java堆的可用最大值，于是就出现了一个称为增长上限的值（Growth Limit）。这个增长上限值（Growth Limit），我们可以认为它是 Java 堆大小的软限制，而前面所描述的最大值（Maximum Size），是 Java 堆大小的硬限制。通过动态地调整增长上限值（Growth Limit），就可以实现动态调整 Java 堆的可用最大值，但是这个增长上限值必须要小于等于最大值（Maximum Size）。如果没有指定 Java 堆的增长上限的值（Growth Limit），那么它的值就等于 Java 堆的最大值（Maximum Size）。

### Memory 请求

下图表明了 Dalvik 虚拟机为对象分配内存的过程：

![Dalvik虚拟机为对象分配内存的过程](https://upload-images.jianshu.io/upload_images/2946447-42d2c0fc08bac943.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 在Java堆上分配指定大小的内存。如果分配成功，那么就将分配得到的地址直接返回给调用者了。此次分配不改变 Java 堆当前的大小，属于轻量级的内存分配动作。

2. 如果上一步内存分配失败，这时候就需要执行一次 GC 了。如果 GC 线程已经在运行中，等到GC执行完成执行再进行下一步。否则就执行一次不回收软引用对象引用的对象的 GC。

3. GC 执行完毕后，再次尝试轻量级的内存分配操作。如果分配成功，那么就将分配得到的地址直接返回给调用者了。

4. 如果上一步内存分配失败，这时候就得考虑先将 Java 堆的当前大小设置为 Dalvik 虚拟机启动时指定的 Java 堆最大值，再进行内存分配了。

5. 如果增长堆后分配内存成功，则将分配得到的地址直接返回给调用者。

6. 如果上一步内存分配还是失败，这时候就得出狠招了。执行回收软引用对象引用的对象的 GC。

7. GC执行完毕，再次内存分配。这是最后一次努力了，成功与否都到此为止。失败了则抛出 OOM 异常。

### Garbage Collection

下图展示了 Dalvik 虚拟机中 GC 的流程：

![Dalvik虚拟机垃圾收集过程](https://upload-images.jianshu.io/upload_images/2946447-a3343a41bfa7ed05.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图中可以看到，GC 有四类：

* GC_FOR_MALLOC: 表示是在堆上分配对象时内存不足触发的 GC。

* GC_CONCURRENT: 表示是在已分配内存达到一定量之后触发的 GC。

* GC_EXPLICIT: 表示是应用程序调用 System.gc、VMRuntime.gc 接口或者收到 SIGUSR1 信号时触发的 GC。

* GC_BEFORE_OOM: 表示是在准备抛 OOM 异常之前进行的最后努力而触发的GC。

实际上，GC_FOR_MALLOC、GC_CONCURRENT 和 GC_BEFORE_OOM 三种类型的 GC 都是在分配对象的过程触发的。

Dalvik 虚拟机支持非并行和并行两种 GC。在上图中，左边是非并行 GC 的执行过程，而右边是并行 GC 的执行过程。它们的总体流程是相似的，主要差别在于前者在执行的过程中一直是挂起非 GC 线程的，而后者是有条件地挂起非 GC 线程。挂起所有非 GC 进程的影响就是会导致线程卡顿、停止。

## ART 内存机制

老罗将 ART （Android Runtime）当成一个名词来使用，这里大家理解一下它的说法。实际上官方文档也是将 ART 与 Dalvik 的说法并列的，当成名词来使用。

以下部分是 4.4 的 ART 的分析总结，后面有 5.0 引入的新的 Compacting GC。

>参考链接：
1. [ART运行时垃圾收集机制简要介绍和学习计划 ]( http://blog.csdn.net/luoshengyang/article/details/42072975)
2. [ART运行时堆的创建过程](http://blog.csdn.net/luoshengyang/article/details/42379729)
3. [ART运行时的对象分配过程](http://blog.csdn.net/luoshengyang/article/details/42492621)
4. [ART运行时的垃圾收集过程](http://blog.csdn.net/luoshengyang/article/details/42555483)

### 基本概念

![ART运行时垃圾收集机制的基本概念](https://upload-images.jianshu.io/upload_images/2946447-75835c30d38802e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上图可以看到，ART 运行时堆划分为四个空间，分别是 Image Space、Zygote Space、Allocation Space 和 Large Object Space。其中，Image Space、Zygote Space、Allocation Space 是在地址上连续的空间，称为 Continuous Space，而 Large Object Space 是一些离散地址的集合，用来分配一些大对象，称为 Discontinuous Space。

在 Image Space 和 Zygote Space 之间，隔着一段用来映射 system@framework@boot.art@classes.oat 文件的内存。这部分内存是系统第一次启动时那些需要预加载的系统类对象翻译得到的，后续每次启动只要直接将其映射到内存区 Image Space 中就可以了，所以ART运行时第一次启动时会比较慢，但是以后启动实际上会更快。（M注：这部分内容由于我在 7.0 上验证时有点不一样就不展开了）

Zygote Space 和 Allocation Space 与 Dalvik 虚拟机垃圾收集机制中的 Zygote 堆和 Active 堆的作用是一样的。Zygote Space 在 Zygote 进程和应用程序进程之间共享的，而 Allocation Space 则是每个进程独占的。同样的，Zygote 进程一开始只有一个 Image Space 和一个 Zygote Space。在 Zygote 进程 fork 第一个子进程之前，就会把 Zygote Space 一分为二，原来的已经被使用的那部分堆还叫 Zygote Space，而未使用的那部分堆就叫 Allocation Space。以后的对象都在 Allocation Space 上分配。

与 Dalvik 虚拟机类似，在 ART 运行时中，主要用来分配对象的堆空间 Zygote Space和 Allocation Space 的底层使用的都是匿名共享内存，通过dlmalloc技术来尽量解决碎片问题。

### Memory 请求

![ART运行时为新创建对象分配内存的过程](https://upload-images.jianshu.io/upload_images/2946447-b4dd2b97633c11d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对比 Dalvik 虚拟机为新创建对象分配内存的过程，可以发现，ART 运行时和 Dalvik 虚拟机为新创建对象分配内存的过程几乎是一模一样的，它们的区别仅仅是在于垃圾收集的方式和策略不同。

正如前文所述，垃圾回收会对程序造成影响，因此在执行垃圾回收时，使用的力度要从小到大。

### Garbage Collection

4.4 的 ART GC 仍然是 Mark Sweep 算法。

![ART运行时的GC执行流程](https://upload-images.jianshu.io/upload_images/2946447-e76a65cf59fe55ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

图中可以看到，GC 有三类：

* FOR Alloc: 表示是在堆上分配对象时内存不足触发的 GC。

* Background: 表示并行 GC 时调用的后台 GC。

* EXPLICIT: 表示是应用程序调用 System.gc、VMRuntime.gc 接口或者收到 SIGUSR1 信号时触发的 GC。

并行GC和非并行GC的区别在于：

1. 非并行 GC 的标记阶段和回收阶段是在挂住所有的 ART 运行时线程的前提下进行的，因此，只需要执行一次标记即可。

2. 并行 GC 的标记阶段只锁住了 Java 堆，因此它不能阻止那些不是正在分配对象的 ART 运行时线程同时运行，而这些同进运行的 ART 运行时线程可能会引用了一些在之前的标记阶段没有被标记的对象。如果不对这些对象进行重新标记的话，那么就会导致它们被 GC 回收，造成错误。因此，与非并行 GC 相比，并行 GC 多了一个处理脏对象的阶段。所谓的脏对象就是我们前面说的在GC标记阶段同时运行的ART运行时线程访问或者修改过的对象。

3. 并行 GC 并不是自始至终都是并行的，例如，处理脏对象的阶段就是需要挂起除 GC 线程以外的其它 ART 运行时线程，这样才可以保证标记阶段可以结束。

与 Dalvik 虚拟机 GC 相比，ART 运行时 GC 的优势在于：

1. ART 运行时堆的划分和管理更细致，它分为 Image Space、Zygote Space、Allocation Space 和 Large Object Space 四个 Space，再加上一个 Allocation Stack。其中，Allocation Space 和 Large Object Space 和 Dalvik虚拟机的 Zygote 堆和 Active 堆作用是一样的，而其余的 Space 则有特别的作用，例如，Image Space的对象是永远不需要回收的。

2. ART 运行时的每一个 Space 都有不同的回收策略，ART 运行时根据这个特性提供了 Mark Sweep、Partial Mark Sweep 和 Sticky Mark Sweep 等三种回收力度不同的垃圾收集器。其中，Mark Sweep 的垃圾回收力度最大，它会同时回收 Zygote Space、Allocation Space 和 Large Object Space 的垃圾，Partial Mark Sweep 的垃圾回收力度居中，它只会同时回收 Allocation Space 和 Large Object Space 的垃圾，而 Sticky Mark Sweep 的垃圾回收力度最小，它只会回收 Allocation Stack 的垃圾，即上次 GC 以后分配出来的又不再使用了的对象。力度越大的垃圾收集器，回收垃圾时需要的时候也就越长。这样我们就可以在应用程序运行的过程中根据不同的情景使用不同的垃圾收集器，那就可以更有效地执行垃圾回收过程。

3. ART 运行时充分地利用了设备的 CPU 多核特性，在并行 GC 的执行过程中，将每一个并发阶段的工作划分成多个子任务，然后提交给一个线程池执行，这样就可以更高效率地完成整个 GC 过程，避免长时间对应用程序造成停顿。

到了 Android 5.0，ART 增加了对 Compacting GC 的支持，包括 Semi-Space（SS）、Generational Semi-Space（GSS）和 Mark-Compact （MC）三种。

>参考链接：
1. [ART运行时 Compacting GC 简要介绍和学习计划 ](http://blog.csdn.net/luoshengyang/article/details/44513977)
2. [ART运行时 Compacting GC 堆的创建过程](http://blog.csdn.net/luoshengyang/article/details/44789295)
3. [ART运行时 Compacting GC 的对象分配过程](http://blog.csdn.net/luoshengyang/article/details/44910271)
4. [ART运行时 Semi-Space（SS）和 Generational Semi-Space（GSS）GC 执行过程](http://blog.csdn.net/luoshengyang/article/details/45017207)
5. [ART运行时 Mark-Compact（ MC）GC 执行过程](http://blog.csdn.net/luoshengyang/article/details/45162589)
6. [ART运行时 Foreground GC 和 Background GC 切换过程分析](http://blog.csdn.net/luoshengyang/article/details/45301715)

所谓 Compacting GC，就是在进行 GC 的时候，同时对堆空间进行压缩，以消除碎片，因此它的堆空间利用率就更高。但是也正因为要对堆空间进行压缩，导致 Compacting GC 的 GC 效率不如 Mark-Sweep GC。不过，只要我们使用得到恰当，是能够同时发挥 Compacting GC 和 Mark-Sweep GC 的长处的。例如，当 Android 应用程序被激活在前台运行时，就使用 Mark-Sweep GC，而当 Android 应用程序回退到后台运行时，就使用 Compacting GC。

---

注：看到这部分相关的文章我已经放弃了…… 主要是自己对相关知识了解不多，准备后续看一下同事推荐的《深入理解Java虚拟机》相关内容再来看。

Android 发展到现在，针对内存管理的优化一直是不间断的，ART 的内存管理机制已经比较复杂了。简单说一下，在 Android 的 5.0 之后，Android 的内存模型是 Generational。将对象按照各个年代存放，比如新分配的对象属于 Young generation，当对象在对应的年代停留足够长的时间，就会移动到另一个年代区，而不同程度的 GC 就是针对各个年代的及其组合的对象的操作，并且通过判断前台后台程序来选择不同的 GC 算法以此来适应不同的情况，使得系统内存的管理更加的合理高效。

**如果有面试官问你内存管理机制的问题，请确认他问的是哪个版本的 Android**。

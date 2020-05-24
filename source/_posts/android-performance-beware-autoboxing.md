---
title: Android 性能优化——小心自动装箱（Autoboxing）
date: 2017-04-08 20:57:00
categories:
- 技术向
tags:
- Android
- 性能优化
---

>请保持淡定，分析代码，记住：性能很重要。

## 自动装箱引发的血案
有时性能瓶颈是由小问题累积到一起产生的。

一个典型的例子就是 Java 的自动装箱功能。自动拆装箱的目的就是自动地将基础类型与它们的对象版本相互转化，这样你就不用操心你代码中的这些转化了。例如 `Integer value = 0` 当中，将整型的 0 自动的转化为 Integer 的对象。

<!--more-->

Java 的基础类型及其大小如下表：

| 基本类型 | 存储大小 |
| :-- |:--|
| byte | 8 bits |
| short | 16 bits |
| int | 32 bits |
| long | 64 bits |
| float | 32 bits |
| double | 64 bits |
| char | 16 bits |
| boolean | 8 bits |

为了在泛型集合中使用基础类型的功能，Java 提供了对应的对象版本，提供了与基础整型相同的功能，但可以使用于泛型集合。

| 基本类型 | 对象 |
| :-- |:--|
| byte | java.lang.Byte |
| short | java.lang.Short |
| int | java.lang.Integer |
| long | java.lang.Long |
| float | java.lang.Float |
| double | java.lang.Double |
| char | java.lang.Character |
| boolean | java.lang.Boolean |


这种机制给代码编写带来了方便，举个例子：
```java
// Primitive version
int total = 0;
for (int i = 0; i < 100; i++) {
  total += i;
}

// Generic version
Integer total = 0;
for (int i = 0; i < 100; i++) {
  total += i;
}
```

当我们需要使用 total 时，使用第二个版本更方便，看起来你不需要写多余的代码就把事情完成了，事实上，第二个版本是这样子的：
```java
// Generic version
Integer total = 0;
for (int i = 0; i < 100; i++) {
  //total += i;
  // create new Integer()
  // push in new value
  // add to total
}
```
可以看到每次加上之前都要创建新的整数对象，将这个和第一个基础版本相比有着双重影响。

第一，这占用更多的内存，因为整形只有 4 字节，整数对象有 16 字节；

第二，创建对象需要耗费更多性能。不仅在循环中会出现这样的问题，当你在集合中使用基础类型时，也会出现这样的问题，特别地，对于 HashMap 这样的容器，只要你使用了基础类型，在进行插入、编辑或检索时就会产生一个基础类型和装箱对象。

为了避免 HashMap 的自动装箱行为，Android 系统提供了 SparseBoolMap，SparseIntMap，SparseLongMap，LongSparseMap 等容器，可减少运行时间开支，减少内存使用。它们与 HashMap 的简单对比如下：

| 容器 | 构造 |
| :-- |:--|
| HashMap | &lt;obj, obj&gt; |
| SparseBoolMap | &lt;bool, obj> |
| SparseIntMap | &lt;int, obj> |
| SparseLongMap | &lt;long, obj> |
| LongSparseMap | &lt;long, obj> |

## 确定自动装箱产生问题的位置
解决问题的前提是要先找到问题的所在，Android Studio 有两种工具可以帮你找到自动装箱产生问题的位置。

### 1) Allocation Tracker
如果你使用 Allocation Tracker(分配追踪器)发现同一个地方分配了大量整数对象 `java.lang.Integer` 时，要特别注意。

### 2) TraceView
在 TraceView 中，要注意任何大量调用 `java.lang.Integer.valueOf` 的地方。

以上这两个迹象都是自动封装将产生问题的信号。


正如本文所述，性能优化时有些问题是很容易发现和修复的，但是这些小问题会逐步积累，而不加检查时则可能造成例如响应迟钝这样的大问题。

>资料来源： Google 官方培训视频 [Android Performance Patterns: Beware Autoboxing](https://www.youtube.com/watch?v=dgzOwuoXVJU&list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE&index=29)

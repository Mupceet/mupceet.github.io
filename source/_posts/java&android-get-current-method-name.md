---
title: Java & Android 获取当前方法名
date: 2017-04-08 11:33:54
categories:
- 技术向
tags:
- Android
- Java
---

开发过程中，有时候需要获取当前正在执行的方法名，或者需要获取调用当前方法的方法名，可以通过 `Thread.currentThread().getStackTrace()` 来获取。

`Thread.currentThread().getStackTrace()` 返回的是一个 StackTraceElement 数组，内容为调用函数堆栈，并且以调用层级关系保存，显然，数组的第一个元素即 `s[0]` 就是获取这个数组的方法，因此，当前调用 `getStackTrace()` 的方法的方法名就是 `s[1]` 了。

<!--more-->

当然，这个是在 Java 环境下的该方法的层级计算方法，如果是在 Android 中，层级有点不一样了。我们看一下 Android 的 `getStackTrace()` 方法：
```Java
/**
 * Returns an array of {@link StackTraceElement} representing the current thread's stack.
 */
public StackTraceElement[] getStackTrace() {
    StackTraceElement ste[] = VMStack.getThreadStackTrace(this);
    return ste != null ? ste : EmptyArray.STACK_TRACE_ELEMENT;
}
```
它是 `VMStack.getThreadStackTrace(this)` 的封装，所以导致了层级会多一层。

举个例子，我在代码工具类中添加一个打印程序方法名的方法：
```Java
public static void recordMethodName(Class<?> callingClass) {
    String className = callingClass.getSimpleName();
    StackTraceElement[] s = Thread.currentThread().getStackTrace();
    String methodName = s[3].getMethodName();

    Log.i("MethodRecord", className + "." + methodName);
}
```

因为我需要获取调用 `recordMethodName` 的方法的方法名，在 `recordMethodName` 中使用 `Thread.currentThread().getStackTrace()` 返回数组 s。即调用关系如下：
```
 Fun --> recordMethodName() --> getStackTrace() --> getThreadStackTrace()
 s[3]    s[2]                   s[1]                s[0]
```

所以 `FunName = s[3].getMethodName()`。

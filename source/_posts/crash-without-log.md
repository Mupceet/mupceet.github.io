---
title: 诡异的崩溃：UncaughtExceptionHandler 与 RxJava 并发错误
date: 2018-04-12 23:20:33
categories:
- 技术向
tags:
- Android
- 坑
---

最近有几个 Bug 报了崩溃问题，但是奇怪的是 Log 中没有 `FATAL EXCEPTION` 相关的堆栈信息，经过查找，找到以下一句记录：

```
02-06 17:26:30.246   815   989 I ActivityManager: Killing 27917:com.xxx.weather/u0a83 (adj 800): crash
```

这个问题分成两个部分来解析，一是查看这个 crash Log 是如何打印出来的， 一是为什么没有打印出 crash 的堆栈信息。

## Crash Log 来龙去脉

从源码中搜索查看该 Log 在哪里打印，进行原因回溯。

在源码索引中搜索 `\"Killing`，从搜索结果中很容易定位到该 Log 所在的文件：
```java
// ProcessRecord.java
void kill(String reason, boolean noisy) {
    if (!killedByAm) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "kill");
        if (noisy) {
            Slog.i(TAG, "Killing " + toShortString() + " (adj " + setAdj + "): " + reason);
        }
        EventLog.writeEvent(EventLogTags.AM_KILL, userId, pid, processName, setAdj, reason);
        Process.killProcessQuiet(pid);
        ActivityManagerService.killProcessGroup(uid, pid);
        if (!persistent) {
            killed = true;
            killedByAm = true;
        }
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
    }
}
```

<!--more-->

根据打印的结果，可以看出调用该方法应该调用的是 `kill("crash", true)`，查看该方法的调用关系，可以定位到：

```java
ActivityManagerService#removeProcessLocked
AppErrors#handleAppCrashInActivityController
```

好吧……调用的情况还是有点多，但这两个方法实际上都被以下这个方法所调用：

```java
// AppErrors.java
void crashApplicationInner(ProcessRecord r, ApplicationErrorReport.CrashInfo crashInfo,
        int callingPid, int callingUid) {
    // 省略
    synchronized (mService) {
        /**
         * If crash is handled by instance of {@link android.app.IActivityController},
         * finish now and don't show the app error dialog.
         */
         // AppErrors#handleAppCrashInActivityController 在这里被调用，根据注释可以发现不是这里打的Log
        if (handleAppCrashInActivityController(r, crashInfo, shortMsg, longMsg, stackTrace,
                timeMillis, callingPid, callingUid)) {
            return;
        }

        // 省略
    synchronized (mService) {
        if (res == AppErrorDialog.MUTE) {
            stopReportingCrashesLocked(r);
        }
        if (res == AppErrorDialog.RESTART) {
            mService.removeProcessLocked(r, false, true, "crash");
            if (task != null) {
                try {
                    mService.startActivityFromRecents(task.taskId,
                            ActivityOptions.makeBasic().toBundle());
                } catch (IllegalArgumentException e) {
                    // Hmm, that didn't work, app might have crashed before creating a
                    // recents entry. Let's see if we have a safe-to-restart intent.
                    final Set<String> cats = task.intent.getCategories();
                    if (cats != null && cats.contains(Intent.CATEGORY_LAUNCHER)) {
                        mService.startActivityInPackage(task.mCallingUid,
                                task.mCallingPackage, task.intent, null, null, null, 0, 0,
                                ActivityOptions.makeBasic().toBundle(), task.userId, null,
                                "AppErrors");
                    }
                }
            }
        }
        // 最终可以看到，系统在这里打印了该 Log
        // mService.removeProcessLocked(r, false, false, "crash");
        if (res == AppErrorDialog.FORCE_QUIT) {
            long orig = Binder.clearCallingIdentity();
            try {
                // Kill it with fire!
                mService.mStackSupervisor.handleAppCrashLocked(r);
                if (!r.persistent) {
                    mService.removeProcessLocked(r, false, false, "crash");
                    mService.mStackSupervisor.resumeFocusedStackTopActivityLocked();
                }
            } finally {
                Binder.restoreCallingIdentity(orig);
            }
        }
        if (res == AppErrorDialog.FORCE_QUIT_AND_REPORT) {
            appErrorIntent = createAppErrorIntentLocked(r, timeMillis, crashInfo);
        }
        if (r != null && !r.isolated && res != AppErrorDialog.RESTART) {
            // XXX Can't keep track of crash time for isolated processes,
            // since they don't have a persistent identity.
            mProcessCrashTimes.put(r.info.processName, r.uid,
                    SystemClock.uptimeMillis());
        }
    }
	// 省略
}
```

继续向上查看调用关系：

```
crashApplicationInner <-- crashApplication <-- handleApplicationCrashInner <-- handleApplicationCrash
```

```java
// ActivityManagerService.java
public void handleApplicationCrash(IBinder app,
        ApplicationErrorReport.ParcelableCrashInfo crashInfo) {
    ProcessRecord r = findAppProcess(app, "Crash");
    final String processName = app == null ? "system_server"
            : (r == null ? "unknown" : r.processName);
    handleApplicationCrashInner("crash", r, processName, crashInfo);
}

void handleApplicationCrashInner(String eventType, ProcessRecord r, String processName,
        ApplicationErrorReport.CrashInfo crashInfo) {
    EventLog.writeEvent(EventLogTags.AM_CRASH, Binder.getCallingPid(),
            UserHandle.getUserId(Binder.getCallingUid()), processName,
            r == null ? -1 : r.info.flags,
            crashInfo.exceptionClassName,
            crashInfo.exceptionMessage,
            crashInfo.throwFileName,
            crashInfo.throwLineNumber);
    addErrorToDropBox(eventType, r, processName, null, null, null, null, null, crashInfo);
    mAppErrors.crashApplication(r, crashInfo);
}
```

如果你使用的是 `SDK` 查看的源码，到这里就无法跳转到其调用了，如果是使用 `AOSP` 源码查看的话还是可以的，没关系，转用源码索引去查查。可以定位到

```
handleApplicationCrash <-- KillApplicationHandler
```

具体如下：
```java
private static class KillApplicationHandler implements Thread.UncaughtExceptionHandler {
    public void uncaughtException(Thread t, Throwable e) {
        // 省略
        // Bring up crash dialog, wait for it to be dismissed
        ActivityManager.getService().handleApplicationCrash(
            mApplicationObject, new ApplicationErrorReport.ParcelableCrashInfo(e));
        // 省略
  }
}

private static class LoggingHandler implements Thread.UncaughtExceptionHandler {
    @Override
    public void uncaughtException(Thread t, Throwable e) {
        // Don't re-enter if KillApplicationHandler has already run
        if (mCrashing) return;
        if (mApplicationObject == null) {
            // The "FATAL EXCEPTION" string is still used on Android even though
            // apps can set a custom UncaughtExceptionHandler that renders uncaught
            // exceptions non-fatal.
            Clog_e(TAG, "*** FATAL EXCEPTION IN SYSTEM PROCESS: " + t.getName(), e);
        } else {
            StringBuilder message = new StringBuilder();
            // The "FATAL EXCEPTION" string is still used on Android even though
            // apps can set a custom UncaughtExceptionHandler that renders uncaught
            // exceptions non-fatal.
            message.append("FATAL EXCEPTION: ").append(t.getName()).append("\n");
            final String processName = ActivityThread.currentProcessName();
            if (processName != null) {
                message.append("Process: ").append(processName).append(", ");
            }
            message.append("PID: ").append(Process.myPid());
            Clog_e(TAG, message.toString(), e);
        }
    }
}
```

这两个 `UncaughtExceptionHandler` 的调用如下：

```java
protected static final void commonInit() {
    // 省略
    /*
    * set handlers; these apply to all threads in the VM. Apps can replace
    * the default handler, but not the pre handler.
    */
    Thread.setUncaughtExceptionPreHandler(new LoggingHandler());
    Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler());
    // 省略
}
```

系统在应用初始化的时候，给应用的主线程设置了两个 `UncaughtExceptionHandler`，其中 `LoggingHandler` 用来打印异常的 Log，也就是熟悉的崩溃异常 Log：“FATAL EXCEPTION:”开头的 Log；另一个 `KillApplicationHandler` 用来处理崩溃，通过这个进入 `AMS` 的异常处理流程，然后弹框，最后打印了一句开头的那一句 Crash 的 Log。

``Killing 27917:com.xxx.weather/u0a83 (adj 800): crash``

从名字也可以看出，一个 `UncaughtException` 会先由 `UncaughtExceptionPreHandler` 处理再由 `DefaultUncaughtExceptionHandler` 处理。如下所示：

```java
// Thread.java
public final void dispatchUncaughtException(Throwable e) {
    Thread.UncaughtExceptionHandler initialUeh =
            Thread.getUncaughtExceptionPreHandler();
    if (initialUeh != null) {
        try {
            initialUeh.uncaughtException(this, e);
        } catch (RuntimeException | Error ignored) {
            // Throwables thrown by the initial handler are ignored
        }
    }
    getUncaughtExceptionHandler().uncaughtException(this, e);
}
```

这里说明一下，`KillApplicationHandler` 是从 `API 26` 开始才有的，它将原先的 `API 25` 中的 `UncaughtExceptionHandler` 拆分成 `LoggingHandler` 和 `KillApplicationHandler` 两部分了，分工更明确。不少三方服务 `SDK` 都自定义了 `DefaultUncaughtExceptionHandler` 来拦截意外的异常进行记录、上报分析，所以将两者拆分能保留打印 `Log` 的功能，给开发者更好的调试体验。

看到这里，基本已经理清了这一条 Log 的来龙去脉了。

## Crash 的堆栈信息消失

经过以上的分析，这个问题就比较明确了。应用崩溃了，并且打印了相应的 Kill Log，所以异常被 `KillApplicationHandler` 进行了处理，但是又没有打印出 `FATAL EXCEPTION`，则说明异常没有被 `LoggingHandler` 进行处理，那么真相只有一个：这个异常不是真正的未捕获的异常。

具体而言，就是这个异常虽然造成了应用崩溃，但是它并不是应用没有处理的异常，实际上它是由应用自己 Catch 了之后主动调用系统的 `UncaughtExceptionHandler` 处理的。

查看日志可以看到以下内容：

```
W/System.err: io.reactivex.exceptions.UndeliverableException: java.util.concurrent.ExecutionException: com.android.volley.NoConnectionError: java.net.UnknownHostException: Unable to resolve host "api.accuweather.com": No address associated with hostname
W/System.err:     at io.reactivex.plugins.RxJavaPlugins.onError(RxJavaPlugins.java:349)
W/System.err:     at io.reactivex.internal.operators.observable.ObservableCreate$CreateEmitter.onError(ObservableCreate.java:74)
W/System.err:     at com.xxx.weather.data.http.RxVolleyRequest$5.subscribe(RxVolleyRequest.java:154)
W/System.err:     at io.reactivex.internal.operators.observable.ObservableCreate.subscribeActual(ObservableCreate.java:40)
W/System.err:     at io.reactivex.Observable.subscribe(Observable.java:10955)
W/System.err:     at io.reactivex.internal.operators.observable.ObservableSubscribeOn$SubscribeTask.run(ObservableSubscribeOn.java:96)
W/System.err:     at io.reactivex.Scheduler$DisposeTask.run(Scheduler.java:452)
W/System.err:     at io.reactivex.internal.schedulers.ScheduledRunnable.run(ScheduledRunnable.java:61)
W/System.err:     at io.reactivex.internal.schedulers.ScheduledRunnable.call(ScheduledRunnable.java:52)
W/System.err:     at java.util.concurrent.FutureTask.run(FutureTask.java:266)
W/System.err:     at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:301)
W/System.err:     at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1162)
W/System.err:     at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:636)
W/System.err:     at java.lang.Thread.run(Thread.java:764)
```

这种 `System.err` 大家应该都熟悉，它是 `ex.printStackTrace()` 打印出来的，也就是应用内部 Catch 了异常之后常见的打印堆栈的操作。

去查看 `RxJavaPlugins` 如下：
```java
public static void onError(@NonNull Throwable error) {
    Consumer<? super Throwable> f = errorHandler;

    if (error == null) {
        error = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
    } else {
        if (!isBug(error)) {
            error = new UndeliverableException(error);
        }
    }

    if (f != null) {
        try {
            f.accept(error);
            return;
        } catch (Throwable e) {
            // Exceptions.throwIfFatal(e); TODO decide
            e.printStackTrace(); // NOPMD
            uncaught(e);
        }
    }

    error.printStackTrace(); // NOPMD
    uncaught(error);
}

static void uncaught(@NonNull Throwable error) {
    Thread currentThread = Thread.currentThread();
    UncaughtExceptionHandler handler = currentThread.getUncaughtExceptionHandler();
    handler.uncaughtException(currentThread, error);
}
```

可以看到，如果 `RxJavaPlugins` 没有设置 `errorHandler`，那么，`RxJava` 就会打印堆栈，然后将该异常交给 `UncaughtExceptionHandler` 来处理。这就是测试提交 Log 上显示了应用崩溃，但应用并没有打印 `FATAL EXCEPTION` 的原因。


## 解决方案

### RxJavaPlugins.setErrorHandler

经过以上分析，如果我们设置了 `errorHandler`，那么这个异常就可以由应用进行处理，应用就不会崩溃了。恰好，它提供了设置的方法：

```java
public static void setErrorHandler(@Nullable Consumer<? super Throwable> handler) {
    if (lockdown) {
        throw new IllegalStateException("Plugins can't be changed anymore");
    }
    errorHandler = handler;
}
```

因此第一种解决方案就是设置该 `errorHandler`。

```java
static {
    RxJavaPlugins.setErrorHandler(new Consumer<Throwable>() {
        @Override
        public void accept(Throwable throwable) throws Exception {
            Log.e(TAG, "UncaughtException: accept");
        }
    });
}
```

使用静态代码块实现类加载时进行一次设置。

但这种方式只是避免了应用的崩溃，实际上，这个错误情况出现的根本原因没有解决。

### synchronized

再往上查看 `ObservableCreate$CreateEmitter.onError` 方法：

```java
@Override
public void onError(Throwable t) {
    if (!tryOnError(t)) {
        RxJavaPlugins.onError(t);
    }
}

@Override
public boolean tryOnError(Throwable t) {
    if (t == null) {
        t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
    }
    if (!isDisposed()) {
        try {
            observer.onError(t);
        } finally {
            dispose();
        }
        return true;
    }
    return false;
}
```

可以看到，它会调用 `RxJavaPlugins.onError` 的条件为当前的订阅关系已经被解除。

以下为项目中代码，可以看到在调用 `onError` 前已经对订阅关系进行了判断。（并没有什么卵用……）

```java
public static Observable<List<ConditionBean>> getConditions(final String cityKey) {
    return Observable.create(new ObservableOnSubscribe<List<ConditionBean>>() {
        @Override
        public void subscribe(ObservableEmitter<List<ConditionBean>> e) throws Exception {
            try {
                if (!e.isDisposed()) {
                    e.onNext(conditionRequest(cityKey));
                    e.onComplete();
                }
            } catch (Exception ex) {
                Log.e(TAG, "getConditions weather failed");
                Log.e("XXX", "getConditions");
                if (!e.isDisposed()) {
                    Log.e("XXX", "getConditions start");
                    e.onError(ex);什么意思呢，
                    Log.e("XXX", "getConditions end");
                }
            }
        }
    }).subscribeOn(Schedulers.io());
}

Observable
        .zip(RxVolleyRequest.getConditions(key),
                RxVolleyRequest.getHourlyWeathers(key),
                RxVolleyRequest.getDailyWeathers(key),
                (conditionBeans, hourlyWeatherBeans, dailyForecastBeans) -> {
                    // 省略
                })
        .subscribe(mWeatherDisposable);
```

在加入 `errorHandler` 后查看 Log，可以看到 zip 的 3 个请求几乎同时进入了 `e.onError` 方法，而订阅者实际上是同一个，导致某一个的 `tryOnError` 判断为已取消订阅，导致应用崩溃。

```
04-12 13:58:46.341 E/Weather/XXX: getConditions
04-12 13:58:46.341 E/Weather/XXX: getHourlyWeathers
04-12 13:58:46.341 E/Weather/XXX: getConditions start
04-12 13:58:46.341 E/Weather/XXX: getHourlyWeathers start
04-12 13:58:46.341 E/Weather/XXX: getConditions end
04-12 13:58:46.341 E/Weather/XXX: getDailyWeathers
04-12 13:58:46.341 E/Weather/XXX: getDailyWeathers start
04-12 13:58:46.343 E/Weather/XXX: getDailyWeathers end
04-12 13:58:46.343 E/Weather/XXX: UncaughtException: accept
04-12 13:58:46.343 E/Weather/XXX: getHourlyWeathers end
```

通过以上流程的梳理，可以说该问题根本原因是一个由于多个网络请求线程同时返回错误的并发问题。

因此，解决方案就是解决并发问题。这个不用多说，如下所示：

```java
synchronized (RxVolleyRequest.class) {
    Log.e("XXX", "getConditions");
    if (!e.isDisposed()) {
        Log.e("XXX", "getConditions start");
        e.onError(ex);
        Log.e("XXX", "getConditions end");
    }
}
```
这时再看看 Log 输出：

```
04-12 13:56:49.709 E/Weather/XXX: getDailyWeathers
04-12 13:56:49.709 E/Weather/XXX: getDailyWeathers start
04-12 13:56:49.711 E/Weather/XXX: getDailyWeathers end
04-12 13:56:49.712 E/Weather/XXX: getHourlyWeathers
04-12 13:56:49.712 E/Weather/XXX: getConditions

04-12 13:56:54.476 E/Weather/XXX: getConditions
04-12 13:56:54.476 E/Weather/XXX: getConditions start
04-12 13:56:54.479 E/Weather/XXX: getConditions end
04-12 13:56:54.479 E/Weather/XXX: getHourlyWeathers
04-12 13:56:54.479 E/Weather/XXX: getDailyWeathers
```

这样就从根本上解决了这个疑难问题。当然，这里要注意是否有线程阻塞的问题，项目里的相关代码不会发生主线程阻塞，所以最终解决方案就是如上所示。

## 其他说明

### Crash：偶现到必现
发现该问题过程没有上文说得那样轻松，原因在于测试提的 Bug 为一个偶现 Bug，一开始也没有很好的思路。后来在看代码时，我将觉得没什么作用的 Catch 的异常打印堆栈的语句删除，这才发现这个问题几乎是必现的。改动如下所示：

```java
// 原来的版本
try {
    // 省略
} catch (Exception ex) {
    Log.e(TAG, "getConditions weather failed");
    ex.printStackTrace();
    if (!e.isDisposed()) {
        e.onError(ex);
    }
}

// 新thsg
try {
    // 省略
} catch (Exception ex) {
    Log.e(TAG, "getConditions weather failed");
    if (!e.isDisposed()) {
        e.onError(ex);
    }
}
```

现在回过头看，是因为 `ex.printStackTrace()` 是一个相对耗时的操作，导致上述的并发竞争条件不容易出现，而去除之后就大概率可以出现了。

### onError/onComplete 顺序

学习 `RxJava` 过程中大家可能看到过这样的说明：

1. `onError` 和 `onComplete` 是互斥的，订阅者只会接收到其中一个。
2. 尽管如此，上游仍然可以先调用 `onError` 再调用 `onComplete`，只是下游收不到 `onComplete` 而已；
3. 但上游不能先调用 `onComplete` 再调用 `onError`，否则应用会崩溃。

这一说法相信通过上面的源码可以看出一点原因来，这里再梳理一下。

```java
@Override
public void onError(Throwable t) {
    if (!tryOnError(t)) {
        RxJavaPlugins.onError(t);
    }
}
@Override
public boolean tryOnError(Throwable t) {
    if (t == null) {
        t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
    }
    if (!isDisposed()) {
        try {
            observer.onError(t);
        } finally {
            dispose();
        }
        return true;
    }
    return false;
}
@Override
public void onComplete() {
    if (!isDisposed()) {
        try {
            observer.onComplete();
        } finally {
            dispose();
        }
    }
}
```

可以看到 `onError` 和 `onComplete` 都会取消订阅关系，这也就是说法 1 的原因。
说法 2 是因为虽然 `onError` 把订阅关系取消了，但再调用 `onComplete` 只是不执行操作而已，所以不会导致崩溃。
说法 3 则是因为 `onComplete` 把订阅关系取消后，再调用 `onError` 则会将异常抛到 `RxJavaPlugins` 来处理，而如上文所说，它的默认将这个异常交给应用默认的 `UncaughtExceptionHandler` 来处理，也就是导致应用崩溃。

## 感想

分析的过程中要大胆假设，代码就在那里，不会欺骗你。我在分析过程中，分析到系统打印出 Kill Log 但没有打印 FATAL EXCEPTION Log 时，就差点分析不下去了，一方面是对相关的 `UncaughtExceptionHandler` 不了解，另一方面是没想到为什么一个奇怪的异常交给应用处理后怎么只打印了其中一部分 Log。没有假设就没办法求证，浪费了不少时间。

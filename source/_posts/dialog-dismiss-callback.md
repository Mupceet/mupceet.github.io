---
title: Dialog 三种退出方式的回调分析
date: 2017-06-03 20:30:33
categories:
- 技术向
tags:
- Android
---

大家都知道监听 Dialog 消失事件常常是要重写 `onDismiss()` 或者  `onCancle()` 方法，有时候为了让 Dialog 主动消失，我们会调用 Dialog 的 `dismiss()` 和 `cancle()` 方法。而一个 Dialog 的消失有三种方式可以设置：

 1. 默认的 Cancel 按键：`builder.setNegativeButton(android.R.string.cancel,  null);`
 2. 设置点击外面退出：`builder.create().setCanceledOnTouchOutside(true);`
 3. 点击 Back 键

但是这三种方法是怎样让 Dialog 消失的？这个问题的答案通过查看相关源码来一探究竟。

<!--more-->

## setNegativeButton(android.R.string.cancel,  null)
查看 Dialog 的 Click 的响应事件可以看到：
``` java
private final View.OnClickListener mButtonHandler = new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        final Message m;
        if (v == mButtonPositive && mButtonPositiveMessage != null) {
            m = Message.obtain(mButtonPositiveMessage);
        } else if (v == mButtonNegative && mButtonNegativeMessage != null) {
            m = Message.obtain(mButtonNegativeMessage);
        } else if (v == mButtonNeutral && mButtonNeutralMessage != null) {
            m = Message.obtain(mButtonNeutralMessage);
        } else {
            m = null;
        }

        if (m != null) {
            m.sendToTarget();
        }

        // Post a message so we dismiss after the above handlers are executed
        mHandler.obtainMessage(ButtonHandler.MSG_DISMISS_DIALOG, mDialogInterface)
                .sendToTarget();
    }
};
```
由于默认的方式没有设置 mButtonNegativeMessage，所以默认的 Cancel 按键执行的是
``` java
mHandler.obtainMessage(ButtonHandler.MSG_DISMISS_DIALOG, mDialogInterface)
        .sendToTarget();
```
从对应的 Handler 可以看到，这个 case 下，所执行的是 dismiss() 方法。
``` java
case MSG_DISMISS_DIALOG:
                    ((DialogInterface) msg.obj).dismiss();
```
## setCanceledOnTouchOutside
当设置为 true 的时候可以看到这个 Dialog 就被设置为 cancelable 的。
``` java
public void setCanceledOnTouchOutside(boolean cancel) {
    if (cancel && !mCancelable) {
        mCancelable = true;
    }

    mWindow.setCloseOnTouchOutside(cancel);
}
```
可以看到 Dialog 范围外的 Touch 事件是由 Window 判断的。
``` java
public void setCloseOnTouchOutside(boolean close) {
    mCloseOnTouchOutside = close;
    mSetCloseOnTouchOutside = true;
}
```
通过查看 mCloseOnTouchOutside 的引用可以找到：
``` java
public boolean shouldCloseOnTouch(Context context, MotionEvent event) {
    if (mCloseOnTouchOutside && event.getAction() == MotionEvent.ACTION_DOWN
            && isOutOfBounds(context, event) && peekDecorView() != null) {
        return true;
    }
    return false;
}
```
可以看到确实是由 Window 类响应了这个点击事件。查看该函数的引用则找到 Dialog 里的方法：
``` java
public boolean onTouchEvent(MotionEvent event) {
    if (mCancelable && mShowing && mWindow.shouldCloseOnTouch(mContext, event)) {
        cancel();
        return true;
    }

    return false;
}
```
很明显，`cancel()` 方法就是要找的答案。
``` java
public void cancel() {
    if (!mCanceled && mCancelMessage != null) {
        mCanceled = true;
        // Obtain a new message so this dialog can be re-used
        Message.obtain(mCancelMessage).sendToTarget();
    }
    dismiss();
}
```
可以看到这和直接调用 `dismiss()` 方法是类似的，但是是先发送了一个 cancle 消息再调用 `dismiss()`。

## 点击 Back 键
点击 Back 键就很简单，一目了然。
``` java
public void onBackPressed() {
    if (mCancelable) {
        cancel();
    }
}
```
同样的调用了 cancel 方法。

## 总结
`cancel()` 方法先发送 cancel 消息，再 `dismiss()`，而 `dismiss()` 就是发送消息通知 WindowManager 立即移除 View， 接着发送 dismiss 消息。

简单跳转之后可以看到，每一个 AlertDialog 都注册了 `cencel()` 和 `dismiss()` 的 Listener，这也就是我们常常重写 `onCencel()` 和 `onDismiss()` 这两个方法的作用。

通过查看以上三种方式的源码，可以发现：

1. 默认的 Cancel 按键：调用 dismiss 方法，即窗口消失、回调 onDismiss；
2. 设置点击外面退出：调用 cancle 方法，即回调 onCancle、窗口消失、回调 onDismiss；
3. 点击 Back 键：调用 cancle 方法，即回调 onCancle、窗口消失、回调 onDismiss；

因此要注意你需要复写的是哪个方法，如果使用默认的 Cancle 按键，复写 `onCancle()` 可是一点作用都没有呀。

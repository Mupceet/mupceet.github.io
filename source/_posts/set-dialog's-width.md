---
title: 设置 Dialog 的显示宽度
date: 2017-05-01 20:27:33
updated: 2017-07-09 00:14:33
categories:
- 技术向
tags:
- Android
---

在项目中——一个基于 AOSP 的联系人修改的应用使用弹出框的时候，发现弹出框的宽度是根据其中内容的多少来决定宽度的，对于一个应用而言，UI 希望可以使用统一宽度的弹出框以达到和谐统一。

要在项目里实现统一弹出框宽度的要求，由于当时对源码中使用的各种 Style 不是很熟悉，不太敢改相关的样式，想到的一种方法是使用代码指定所需要的宽度。

## 方法
使用以下设置显示属性的方法去控制Dialog的显示宽度。

<!--more-->

```java
public static void setWidth(int dialogWidth, Dialog dialog, Context context) {
    if (dialog == null || context == null || dialogWidth < 0 || dialogWidth > 360) {
        return;
    }
    DisplayMetrics dm = new DisplayMetrics();
    WindowManager wm = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    wm.getDefaultDisplay().getMetrics(dm);
    // 标准像素：屏幕宽度360dp
    // 若设置Dialog的宽度为 n dp，由于之前使用的 Dialog 样式中系统默认的属性中有个16dp的Padding ，所以实际显示为 (n - 32)dp
    // 所以：若设计稿为 m dp，则宽度需要设置为 (m + 32) / 360 * dm.widthPixels
    // 注：0.888888 --> m = 288 = 80% * 360 为当前样式系统默认下的最大宽度
    double ratio = 0.888888; // default value
    if (dialogWidth != 0) {
        ratio = (double) (dialogWidth + 32) / 360;
    }
    dialog.getWindow().setLayout((int) (dm.widthPixels * ratio), ViewGroup.LayoutParams.WRAP_CONTENT);
}

public static void setWidth(Dialog dialog, Context context) {
    setWidth(0, dialog, context);
}
```
## 调用方法的时机
举两个例子：
### Dialog
对于Dialog，在其show之后立刻设置其显示属性。
```java
mDialog = new AlertDialog.Builder(getActivity())
                .setIconAttribute(android.R.attr.alertDialogIcon)
                .setMessage(messageId)
                .setNegativeButton(android.R.string.cancel, null)
                .setPositiveButton(positiveButtonId,
                    new DialogInterface.OnClickListener() {
                        @Override
                        public void onClick(DialogInterface dialog, int whichButton) {
                            doDeleteContact(contactUri);
                        }
                    }
                )
                .create();

mDialog.setOnDismissListener(this);
mDialog.show();
// 在show方法之后设置其显示属性
DialogUtils.setWidth(mDialog, getActivity());
```

### DialogFragment
对于DialogFragment形式的Dialog，在onStart方法中设置显示属性。
```java
@Override
public void onStart() {
    super.onStart();
    DialogUtils.setWidth(this.getDialog(), getActivity());
}
```

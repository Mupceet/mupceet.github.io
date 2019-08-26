---
title: 图标颜色管理
date: 2017-08-13 20:18:33
categories:
- 小技巧
tags:
- Android
---

应用的图标可以分成两类，一类是小图标，一般带 ic 前缀，另一类就是相对比较大的。大图不用说，一般是 UI 给的什么样子就用什么样子的，对于小图，可能修改的时候只涉及到颜色的改变。如果改个颜色就让 UI 重新切图，这是没有必要的，白白浪费了沟通的成本。

一般来说，涉及到风格变化，如果是采用改变 icon 的颜色的方式，以下管理方法是合适的。

## Toobar 溢出菜单颜色

```xml
<item name="android:textColorSecondary">@color/color_clock_accent</item>
```
改变这个属性就可以改变那三个点的颜色。

<!--more-->

## 封装图片资源
### 例子
返回图标：ic_arrow_back_24dp.png

![back.png](http://upload-images.jianshu.io/upload_images/2946447-7002c0c714914082.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要改变成蓝色、后续可能改变成黄色（这里是个例子）

使用的时候，对其进行封装：
drawable 下创建 ic_arrow_back_24dp.xml(这个名字就是你在布局当中引入图片时想取的名字)
```xml
<bitmap xmlns:android="http://schemas.android.com/apk/res/android"
        android:src="@drawable/ic_arrow_back_24dp"
        android:autoMirrored="true"
        android:tint="@color/actionbar_icon_color" />
```
### 解析
可以看到，这里有几个属性经常用到。
* `src` 表示你引用的图片
* `autoMirrored` 表示在 RTL 布局下是否要镜像。在Google MD 规范中，有的需要镜像，有的不用，比如放大镜、打勾就不镜像。
* `tint` 表示你对该图片进行怎么样的着色，也就是在这里可以在 UI 要求改变颜色的时候在这里进行修改

这种管理方式带来的灵活性如下：

### 优点
* 在布局中不需要修改引用图片的名字，因为直接引用的名字是在 drawable 文件夹下的 xml 文件名。
* 修改颜色直接修改 tint 引用的颜色即可。

## 同一图标不同颜色
如果同一个图片在多个地方使用，可以在代码里面改变颜色：

```Java
// 这里要注意 mutate() 一下
Drawable iconCreateNewGroup = DrawableCompat.wrap(getResources().
                getDrawable(R.drawable.tp_ic_add_white_24dp)).mutate();
DrawableCompat.setTintList(iconCreateNewGroup,ColorStateList.valueOf(Color.GRAY));
createNewGroup.setImageDrawable(iconCreateNewGroup);
```

不同界面使用了同一个图片，但颜色不同，通过 `DrawableCompat.setTintList()` 方法可以改变图片的颜色。
但要注意从 res 获取图片的时候需要 `mutate()` 一下。我们先看一下这个函数：

```Java
/**
 * Make this drawable mutable. This operation cannot be reversed. A mutable
 * drawable is guaranteed to not share its state with any other drawable.
 * This is especially useful when you need to modify properties of drawables
 * loaded from resources. By default, all drawables instances loaded from
 * the same resource share a common state; if you modify the state of one
 * instance, all the other instances will receive the same modification.
 *
 * Calling this method on a mutable Drawable will have no effect.
 *
 * @return This drawable.
 * @see ConstantState
 * @see #getConstantState()
 */
public @NonNull Drawable mutate() {
    return this;
}
```
可以看到注释里写得很清楚，默认情况下，不同地方从 res 加载的同一个图片它们共享状态，如果你修改了其中一个的状态，那其他的也会相应体现出来。

有时引用同一图片改变状态是为了减少资源的重复，这时候就要注意调用 mutate() 方法，让它是可变的，相互之间的状态不互相影响。

### 管理 color.xml
这里需要补充一下对于 color.xml 中的颜色的管理。

如果是一个独立的 App，通常一个具有审美的 UI 来说，使用的颜色种类是非常有限的。因此采用以下组织方式：

```xml
<resources>
    <!--Rom2.1新增-->
    <color name="color_clock_accent">#FF9400</color>
    <color name="color_alert_primary">#F7B314</color>
    <color name="color_alert_snooze">#007AFF</color>
    <color name="color_alert_close">#FF5B5C</color>
    <color name="color_float_button_start">#FFB319</color>
    <!--Rom2.1新增-->
    <color name="transparent">#00000000</color>
    <color name="black">#000000</color>
    <color name="black_10">#19000000</color>
    <color name="black_12">#1f000000</color>
    <color name="black_20">#33000000</color>
    <color name="black_25">#40000000</color>
</resources>
```
即所谓的调色板模式，这样子修改起来比较不容易重复定义，重复定义带来的后果就是修改容易遗漏。不要采用以下方式：
```xml
<resources>
      <color name="button_foreground">#FFFFFF</color>
      <color name="button_background">#2A91BD</color>
      <color name="comment_background_inactive">#5F5F5F</color>
      <color name="comment_background_active">#939393</color>
      <!-- 这种定义一般会看到特别特别长的 color.xml 文件 -->
</resources>
```
这种与具体 button 相关的，甚至是不同 button 还是不同的颜色的，应该在 `style` 里面进行定义。

---
title: RecyclerView 拖动/滑动多选的实现（1）
date: 2017-07-29 17:12:33
categories:
- 技术向
tags:
- Android
---

## 为什么要做滑动多选？
废话啊，当然是因为 UE 说要做啦！

可以看到众多 ROM 的系统应用都实现了滑动多选的功能，例如三星的文件管理器，OPPO 的短信等等，不知道来源是不是 Google 相册。因为交互上与 Google 相册的策略都是一致的。

### Photos 的策略
这里是 Google Photos 的效果：

![Photos](http://upload-images.jianshu.io/upload_images/2946447-c0aca7c2f26e0bd0.gif?imageMogr2/auto-orient/strip)

<!--more-->

可以看到策略为：

* 长按时选择该条目，并进入多选模式
* 往外滑动时选择滑动过的条目
* 往回滑动时取消选择条目
* 多选模式单击时反选

前三条规则是不论原先条目的被选择状态是怎样的。

### UE 的策略
而根据 UE 的描述，我需要实现的多选功能中的多选可以表述为拖动多选和滑动多选两种情况，什么意思呢？见下面具体的策略。

* 长按时进入拖动多选模式，即手指不放手可进行拖动选择
* 手指抬起后，依然处于多选模式，此时叫做滑动多选，因为这时在指定的区域内滑动可选择
* 多选模式单击时反选

我们的应用有线性布局和网格布局的列表，在线性布局中才有两种选择模式，在网格布局中没有滑动多选模式。

两种模式的选择策略是一样的：
* 反选手指按下时的条目（称为第一条目）
* 往外滑动过的条目状态与第一条目改变后的状态一致
* 往回滑动时条目恢复原先的状态

当然，由于要长按才进入多选模式，所以拖动选择实际上与 Photos 的拖动选择效果是一样的，只是在滑动选择中与之效果不同。先上 GitHub 找一下相关的库。


首先呢，肯定是考虑基于 RecyclerView 实现的库， 因为在我们的几个应用中是使用 RecyclerView 实现了线性布局和网格布局的列表，因此搜索时找到了以下这几个库：

1. [afollestad/drag-select-recyclerview：1.1k ★ ](https://github.com/afollestad/drag-select-recyclerview)
2. [MFlisar/DragSelectRecyclerView：267 ★](https://github.com/MFlisar/DragSelectRecyclerView)
3. [weidongjian/AndroidDragSelect-SimulateGooglePhoto：19 ★](https://github.com/weidongjian/AndroidDragSelect-SimulateGooglePhoto)

这三个库之间的关系是：
* 方案一是鼻祖，而且从 Start 数量也可以看出来，让很多人受到启发，GitHub 上有基于它的自定义 RecyclerView 的想法自定义了 GridView 的实现。其选择策略与 Photos 相同，但是这个库最大的缺点就是耦合度太高，不适合集成，具体后面分析会说到；
* 方案三就是分析了方案一的缺点之后，给出了自己的基于 `OnItemTouchListener` 的实现方案，耦合度低，可以很容易集成进现有的项目当中。而且增加了动画的设置，其最终效果：选择策略与动画效果与 Photos 几乎一致。
* 方案二则是在方案三的基础上进行改进的，它们使用了相同的自动滚动的方案，但是选择策略更多样，更人性化，并对超出列表区域时是否自动滚动做了处理。

接下来，我们分别对这几个库进行分析、比较，最后完成符合我们要求的滑动多选的库。

## 方案一：drag-select-recyclerview
这里主要是看一下其设计的思路，所以只分析了自定义 RecyclerView 的部分，对于自定义 Adapter 的代码不做分析，只简单提一下。以下对其进行分析时会对代码顺序做出一定的调整以符合分析流程。

```Java
public class DragSelectRecyclerView extends RecyclerView {
```
可以看到，此库是基于自定义一个 RecyclerView 的想法去实现的。在此自定义 View 中，设置滚动区；处理触摸事件使得手指在滚动区时列表自动滚动；手指滑动过程中对经过的范围进行选择处理。

接下来我们逐步进行分析，在最后总结一下这个库的实现有哪些缺点。

### 滚动区的定义

先看一下滚动区的定义，自定义 RecyclerView 通过设置三个属性，然后进行计算确认滚动区。以下三个属性值的禁止状态用 -1 表示；
```Java
private int hotspotHeight;          // 滑动热区的高度，默认为 56dp
private int hotspotOffsetTop;       // 顶部的滑动热区距离控件顶部的高度，默认为 0
private int hotspotOffsetBottom;    // 底部的滑动热区距离控件底部的高度，默认为 0
```
```xml
<resources>
  <declare-styleable name="DragSelectRecyclerView">
    <!--滚动热区的高度-->
    <attr name="dsrv_autoScrollHotspotHeight" format="dimension"/>
    <!--是否禁止滚动-->
    <attr name="dsrv_autoScrollEnabled" format="boolean"/>
    <!--滚动热区上边距-->
    <attr name="dsrv_autoScrollHotspot_offsetTop" format="dimension"/>
    <!--滚动热区下边距-->
    <attr name="dsrv_autoScrollHotspot_offsetBottom" format="dimension"/>
  </declare-styleable>
</resources>
```
可以看到开放给 xml 设置的属性有高度、上边距、下边距，还有禁止滚动。当禁止滚动的时候将三个值置为 -1。

通过以上三个属性值，在 `onMeasure()` 中确定滚动区的几个有用的坐标值：
```Java
// 上滑动热区上边的上边距：坐标 = hotspotOffsetTop
private int hotspotTopBoundStart;   
// 上滑动热区下边的上边距：坐标 = hotspotOffsetTop + hotspotHeight
private int hotspotTopBoundEnd;     
// 下滑动热区上边的上边距：坐标 = (getMeasuredHeight() - hotspotHeight) - hotspotOffsetBottom
private int hotspotBottomBoundStart;
// 下滑动热区下边的上边距：坐标 = getMeasuredHeight() - hotspotOffsetBottom
private int hotspotBottomBoundEnd;
```

以下这张图可以直观查看这些变量的含义。

![方案一滚动区](http://upload-images.jianshu.io/upload_images/2946447-c336dd4d1dbb1478.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 如何自动滚动

先看一下手指滑动到滚动区域时，列表自动滚动是怎样做到的。

答案就是使用一个 Handler 每 25ms post Runnable 调用滚动的方法并更新滚动速度。

通过手指是在上部滚动区还是下部滚动区来决定滚动的方向，滚动的速度通过 `autoScrollVelocity` 这个变量来控制。
```Java
private int autoScrollVelocity;     // 自动滚动时的速度，这个速度随着与边距的距离大小而改变
private Runnable autoScrollRunnable =
        new Runnable() {
            @Override
            public void run() {
                if (autoScrollHandler == null) {
                    return;
                }
                if (inTopHotspot) {// 上滚动区
                    scrollBy(0, -autoScrollVelocity);
                    autoScrollHandler.postDelayed(this, AUTO_SCROLL_DELAY);
                } else if (inBottomHotspot) { // 下滚动区
                    scrollBy(0, autoScrollVelocity);
                    autoScrollHandler.postDelayed(this, AUTO_SCROLL_DELAY);
                }
            }
        };
```

### 如何选择条目

选择条目的更新主要就是通过以下 4 个变量记录手指的活动范围，在手指活动时记录、更新变量，然后对相应位置的条目进行选择操作。
```Java
private int lastDraggedIndex;       // 手指停下来的位置
private int initialSelection;       // 手指点击开始滑动的位置
private int minReached;             // 手指滑动过程中到过的最小下标
private int maxReached;             // 手指滑动过程中到过的最大下标
```
以上的位置值使用 `RecyclerView.NO_POSITION` 表示初始状态，后续在需要的时候要对其进行重置。

#### 记录起点

记录起点是通过调用激活拖动多选的方法 `setDragSelectActive(boolean active, int initialSelection)` 时进行记录的，这个方法供长按条目 `onLongClick()` 时调用，主要完成的功能：

* 选中长按的条目
* 记录此次长按拖动多选的起点

```Java
// 使用一个标志位开启滑动多选的功能
private boolean dragSelectActive; // 此值为真时，触摸事件的分发时才会进行处理
// 需要使用自定义 Adapter
private DragSelectRecyclerViewAdapter<?> adapter;

public boolean setDragSelectActive(boolean active, int initialSelection) {
    // 已经激活了直接返回
    if (active && dragSelectActive) {
        LOG("Drag selection is already active.");
        return false;
    }
    lastDraggedIndex = -1;
    minReached = -1;
    maxReached = -1;
    // 判断点击的位置是不是可选择的（Adapter）
    if (!adapter.isIndexSelectable(initialSelection)) {
        dragSelectActive = false;
        this.initialSelection = -1;
        lastDraggedIndex = -1;
        LOG("Index %d is not selectable.", initialSelection);
        return false;
    }
    // 选中长按的条目（Adapter）
    adapter.setSelected(initialSelection, true);
    dragSelectActive = active;
    // 记录此次拖动选择的起点
    this.initialSelection = initialSelection;
    lastDraggedIndex = initialSelection;
    if (fingerListener != null) {
        fingerListener.onDragSelectFingerAction(true);
    }
    LOG("Drag selection initialized, starting at index %d.", initialSelection);
    return true;
}
```

#### 处理触摸事件

接下来看看如何处理具体的触摸事件，可以看到自定义的 RecyclerView 是在触摸事件的分发 `dispatchTouchEvent()`  中对手指活动事件进行处理的。手指是否进入滚动区的判断、滚动速度的设定、以及经过了哪些条目的信息的更新都在这里进行处理。

主要流程为：
1. 只在拖动多选被激活时才进行处理
2. 处理抬起手指 `ACTION_UP` 与手指滑动 `ACTION_MOVE` 两个事件
   * 抬起手指，重置状态，移除滚动的 Callback
   * 手指滑动时判断是在哪个区域，进行相应的处理
3. 滚动时在触摸到的条目发生变化时会更新那 4 个位置信息，从而在 Adapter 中选中 initial 到 last 之间的条目，清除 min 到 max 之间除了 initial 到 last 条目之外的条目

先看一下位于哪个区域的判断与处理部分：
```Java
@Override
public boolean dispatchTouchEvent(MotionEvent e) {
    if (adapter.getItemCount() == 0) return super.dispatchTouchEvent(e);
    // 只在拖动多选被激活时才进行处理
    if (dragSelectActive) {
        // 获取触摸时对应的条目位置下标
        final int itemPosition = getItemPosition(e);
        // 抬起手指，重置状态，移除滚动的 Callback
        if (e.getAction() == MotionEvent.ACTION_UP) {
            dragSelectActive = false;
            inTopHotspot = false;
            inBottomHotspot = false;
            autoScrollHandler.removeCallbacks(autoScrollRunnable);
            if (fingerListener != null) {
                fingerListener.onDragSelectFingerAction(false);
            }
            return true;
        } else if (e.getAction() == MotionEvent.ACTION_MOVE) {
            // Check for auto-scroll hotspot
            if (hotspotHeight > -1) {
                // 滑动时判断是在哪个区域：分为三种，上部、下部、非滚动区
                // 以在上部为例
                if (e.getY() >= hotspotTopBoundStart && e.getY() <= hotspotTopBoundEnd) {
                    inBottomHotspot = false;
                    if (!inTopHotspot) {
                        // 进入上部滚动区时，移除原先的Runnable，重新Post
                        // 原因是滚动的触发需要延迟25ms
                        inTopHotspot = true;
                        LOG("Now in TOP hotspot");
                        autoScrollHandler.removeCallbacks(autoScrollRunnable);
                        autoScrollHandler.postDelayed(autoScrollRunnable, AUTO_SCROLL_DELAY);
                    }
                    // 根据手指与滚动区的边距设置滚动速度
                    final float simulatedFactor = hotspotTopBoundEnd - hotspotTopBoundStart;
                    final float simulatedY = e.getY() - hotspotTopBoundStart;
                    autoScrollVelocity = (int) (simulatedFactor - simulatedY) / 2;

                    LOG("Auto scroll velocity = %d", autoScrollVelocity);
                } else if (e.getY() >= hotspotBottomBoundStart
                    && e.getY() <= hotspotBottomBoundEnd) {
                    inTopHotspot = false;
                    if (!inBottomHotspot) {
                        inBottomHotspot = true;
                        LOG("Now in BOTTOM hotspot");
                        autoScrollHandler.removeCallbacks(autoScrollRunnable);
                        autoScrollHandler.postDelayed(autoScrollRunnable, AUTO_SCROLL_DELAY);
                    }

                    final float simulatedY = e.getY() + hotspotBottomBoundEnd;
                    final float simulatedFactor = hotspotBottomBoundStart + hotspotBottomBoundEnd;
                    autoScrollVelocity = (int) (simulatedY - simulatedFactor) / 2;

                    LOG("Auto scroll velocity = %d", autoScrollVelocity);
                } else if (inTopHotspot || inBottomHotspot) {
                    LOG("Left the hotspot");
                    autoScrollHandler.removeCallbacks(autoScrollRunnable);
                    inTopHotspot = false;
                    inBottomHotspot = false;
                }
            }
            // ...
            // 省略更新手指范围的代码，放到后文
            return true;
        }
    }
    return super.dispatchTouchEvent(e);
}
```
其中 `getItemPosition()` 是获取触摸时对应的条目位置的方法，这个方法主要两个功能：
* 判断一下此 RecyclerView 使用的 Adapter 是不是正确继承了自定义的 Adapter
* 前一个条件成立时，返回此时触摸事件对应的条目位置

为什么要判断是否正确继承自定义 Adapter 呢？这是因为方案一的写法需要拿到 ViewHolder，从而才得到得位置信息。
```Java
private int getItemPosition(MotionEvent e) {
    final View v = findChildViewUnder(e.getX(), e.getY());
    if (v == null) return NO_POSITION;
    if (v.getTag() == null || !(v.getTag() instanceof ViewHolder)) {
        throw new IllegalStateException(
                "Make sure your adapter makes a call to super.onBindViewHolder(), "
                + "and doesn't override itemView tags.");
    }
    final ViewHolder holder = (ViewHolder) v.getTag();
    return holder.getAdapterPosition();
}
```

实际上，这里可能更多的是因为此自定义 View 调用了相应的自定义 Adapter 中的方法，所以在这里对是否使用了相应的 Adapter 进行检查，否则是没有必要的，写成如下形式即可：

```Java
private int getItemPosition(MotionEvent e) {
    final View v = findChildViewUnder(e.getX(), e.getY());
    if (v == null) return NO_POSITION;
    return getChildAdapterPosition(v);
}
```

#### 选中滑过的条目
以下就是上面省略的更新手指选择范围的代码。在手指滑动到新的条目时进行变量的更新，在更新选择范围之后会通过 Adapter 对条目进行选择操作。
```Java
// 自动滚动时在条目发生变化时会更新那4个条目位置信息
// Drag selection logic
if (itemPosition != NO_POSITION
  && lastDraggedIndex != itemPosition) {
    lastDraggedIndex = itemPosition;
    if (minReached == -1) {
        minReached = lastDraggedIndex;
    }
    if (maxReached == -1) {
        maxReached = lastDraggedIndex;
    }
    if (lastDraggedIndex > maxReached) {
        maxReached = lastDraggedIndex;
    }
    if (lastDraggedIndex < minReached) {
        minReached = lastDraggedIndex;
    }
    if (adapter != null) {
        // 在adapter中选中 initial 到 last 的条目，清除选中 min 到 max 除了前面要选中条目之外的条目
        adapter.selectRange(initialSelection, lastDraggedIndex, minReached, maxReached);
    }
    if (initialSelection == lastDraggedIndex) {
        minReached = lastDraggedIndex;
        maxReached = lastDraggedIndex;
    }
}
```

### DragSelectRecyclerViewAdapter
可以看到在 DragSelectRecyclerView 需要与 DragSelectRecyclerViewAdapter 搭配使用。因为需要调用 Adapter 进行选择处理。当然了，这个可以通过回调抽出来。

Adapter 中需要实现、处理的是：

#### onBindViewHolder
将 VH 通过 Tag 设置到本身的 View 上。这样在 RecyclerView 中就可以通过 VH 获得在 Adapter 中的 position。
>注：如上面所说，这一步不是必要的。只是通过这样可以检查是否使用了自定义的 Adapter。
```Java
@CallSuper
@Override
public void onBindViewHolder(VH holder, int position) {
  holder.itemView.setTag(holder);
}
```

#### 选择的方法
主要有：

* `setSelected(int index, boolean selected)`：设置对应条目的选择状态
* `toggleSelected(int index)`：反选对应条目，并返回新状态
* `selectRange(int from, int to, int min, int max)`：将 from 到 to 的位置的状态保持一致，反选另外的。
* `selectAll()` `clearSelected()`：全选、取消选中条目

### 缺点

* 这种方法使用了自定义 RecyclerView 与 Adapter 并相互之间发生了耦合，使用时就需要更改原来的 RecyclerView 和 Adapter 的继承与代码，不优雅。
* 可以看到选择范围的更新是在手指滑动时进行的，所以手指在滚动区按住不动时列表发生滚动但没有选择上，而在手指动了之后才会正确选中。

其相互耦合的方式是导致它无法被采用的根本原因，必须考虑其他的实现方式。

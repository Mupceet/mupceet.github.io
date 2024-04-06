---
title: RecyclerView 拖动/滑动多选的实现（1）
date: 2017-07-29 17:12:33
categories:
- 技术向
tags:
- Android
---

## 为什么要做滑动选择？

废话，当然是因为 UE 说要做啦（误）事实上是想要使用最小的交互动作达到最大的效果，可以看到不少应用都实现了滑动选择的功能，例如三星、华为的文件管理器，OPPO 的短信， Google 相册、华为的图库等等，它们包括了竖向列表和网格列表。

### 我们想要的选择操作

根据需求，我们要实现的多选功能可以表述为`长按多选`和`触摸多选`两种情况，什么意思呢？

**长按多选**指的是长按动作会触发选择，不松手情况下手指滑动时选中条目；而**触摸多选**指的是在条目指定区域（通常是多选框所在的区域）内触摸会触发选择，不松手情况下手指滑动时选中条目。为了表述方便，我们将前者称为“拖动多选（`DragSelect`）”，后者称为“滑动多选（`SlideSelect`）”。在具体的产品策略中，两种选择模式可以同时存在：长按时进入拖动多选模式，滑动进行选择；手指抬起后，依然处于多选状态，此时进入滑动多选模式，从指定的区域内滑动可继续进行选择。

此外，应用中可选择的列表一般是线性布局和网格布局两种形式，而用户较难意识到网格布局的列表的某个区域可触发滑动选择，因此网格布局通常情况下没有滑动多选模式。

选择的动作中，可以拆解成三个环节：

1. 触发选择时，手指按下时的条目（该条目称为第一条目）的状态：**状态设为选中**或者**状态反选**；
2. 触发选择后，往外滑动时经过的条目的状态：**状态与第一条目的状态一致**；
3. 触发选择后，往回滑动时经过的条目的状态：**状态不变**、**状态与第一条目的状态相反**、或者**恢复之前状态**；

<!--more-->

### Google 相册的选择操作

以 Google Photos 为例，我们看下它的交互效果：

![Photos](http://upload-images.jianshu.io/upload_images/2946447-c0aca7c2f26e0bd0.gif?imageMogr2/auto-orient/strip)

可以看到策略为：

* 长按时触发选择，第一条目设置为选中；
* 往外滑动时经过的条目的状态设置为与第一条目的状态一致；
* 往回滑动时经过的条目的状态设置为与第一条目的状态相反；
* 选择模式下单击时反选。

先上 GitHub 找一下相关的库。首先肯定是考虑基于 RecyclerView 实现的库， 因为在我们的几个应用中是使用 RecyclerView 实现了线性布局和网格布局的列表，因此搜索找到了以下这几个库：

1. [afollestad/drag-select-recyclerview](https://github.com/afollestad/drag-select-recyclerview)
2. [weidongjian/AndroidDragSelect-SimulateGooglePhoto](https://github.com/weidongjian/AndroidDragSelect-SimulateGooglePhoto)
3. [MFlisar/DragSelectRecyclerView](https://github.com/MFlisar/DragSelectRecyclerView)

这三个库之间的关系：

* 方案一大概是最早实现拖动选择的效果，而且从 Start 数量也可以看出来，让很多人受到启发，GitHub 上有参考它自定义 RecyclerView 的想法实现了自定义了 GridView。这个库的选择策略与 Photos 相同，但是这个库最大的缺点就是耦合度太高，不适合集成；
* 方案二就是分析了方案一的缺点之后，给出了自己的基于 `OnItemTouchListener` 的实现方案，耦合度低，可以很容易集成进现有的项目当中，而且增加了动画的设置，其最终选择策略与动画效果与 Photos 几乎一致。
* 方案三则是在方案二的基础上进行改进的，它们使用了相同的自动滚动的方案，但是解耦出多样的选择策略，并对超出列表区域时是否自动滚动做了处理。
  
> afollestad/drag-select-recyclerview 这个库在 2018/7/23 进行了重构，也使用 `OnItemTouchListener` 方案，下面的分析是基于当时未重构的版本。

接下来，我们分别对这几个库进行分析、比较，最后完成符合我们要求的滑动多选的库。

## 方案一：drag-select-recyclerview

```Java
public class DragSelectRecyclerView extends RecyclerView {
```

可以看到，此库是基于自定义 RecyclerView 的想法去实现的，实现代码中的核心流程可以分为三部分。

1. 在此自定义 View 中，设置滚动区；
1. 手指滑动过程中对经过的范围进行选择处理。
1. 处理触摸事件使得手指在滚动区时列表自动滚动；

> 以下对其进行分析时会对代码顺序做出一定的调整以符合分析流程。

### 滚动区的定义

以下这张图可以直观查看滚动区的定义。

![方案一滚动区](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-area-option-one.png)

自定义 RecyclerView 通过设置三个属性，然后进行计算确认滚动区。

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

### 选择条目

更新选择条目主要就是通过以下 4 个变量记录手指的活动范围，在**手指活动时**记录、更新变量，然后对相应位置的条目进行选择操作。

> 注：这里的实现导致了一个 Bug，即手动完全不动时，变量不更新，导致没进行选择。

```Java
private int initialSelection;       // 手指点击开始滑动的位置
private int lastDraggedIndex;       // 手指停下来的位置
private int minReached;             // 手指滑动过程中到过的最小下标
private int maxReached;             // 手指滑动过程中到过的最大下标
```

以上的位置值使用 `RecyclerView.NO_POSITION` 表示初始状态，后续在需要的时候要对其进行重置。

#### 记录起点

`setDragSelectActive(boolean active, int initialSelection)` 这个 API 给到使用者在希望触发选择的动作例如长按条目时调用，选择的起点就是传入的参数。该 API 完成两件事情：

1. 选中长按的条目
1. 记录此次长按拖动多选的起点

```Java
// 使用一个标志位开启滑动多选的功能
private boolean dragSelectActive; // 此值为真时，触摸事件的分发时才会进行处理
// 需要使用自定义 Adapter，它的作用见下面注释
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
    // 1. 选中长按的条目（Adapter）
    adapter.setSelected(initialSelection, true);
    dragSelectActive = active;
    // 2. 记录此次拖动选择的起点
    this.initialSelection = initialSelection;
    lastDraggedIndex = initialSelection;
    if (fingerListener != null) {
        fingerListener.onDragSelectFingerAction(true);
    }
    LOG("Drag selection initialized, starting at index %d.", initialSelection);
    return true;
}
```

#### 处理滑动事件

在用户滑动时，需要判断触控点是否位于滚动区，如果位于滚动区则需要让列表自动滚动起来，并且距离滚动区开始滚动的边距越远滚动速度也越快。

```Java
@Override
public boolean dispatchTouchEvent(MotionEvent e) {
    // ...
    if (e.getAction() == MotionEvent.ACTION_MOVE) {
        // Check for auto-scroll hotspot
        if (hotspotHeight > -1) {
            // 滑动时判断是在哪个区域：分为三种，上滚动区、下滚动区、非滚动区
            // 以在上滚动区为例
            if (e.getY() >= hotspotTopBoundStart && e.getY() <= hotspotTopBoundEnd) {
                inBottomHotspot = false;
                if (!inTopHotspot) {
                    // 进入上滚动区时，启动自动滚动
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
                // ... 下滚动区类似
            } else if (inTopHotspot || inBottomHotspot) {
                // ... 停止滚动
            }
        }
        // ... 省略更新手指范围的代码，放到后文
        return true;
    }
    // ...
}
```

在滑动过程中，还需要关注经过的条目，更新选择范围从而在 Adapter 中选中 initial 到 last 之间的条目，清除 min 到 max 之间除了 initial 到 last 条目之外的条目。

```Java
@Override
public boolean dispatchTouchEvent(MotionEvent e) {
    // ...
    // 获取触摸时对应的条目位置下标
    final int itemPosition = getItemPosition(e);
    // ...
    if (e.getAction() == MotionEvent.ACTION_MOVE) {
        // ...
        // 自动滚动时在条目发生变化时会更新那4个条目位置信息
        if (itemPosition != NO_POSITION && lastDraggedIndex != itemPosition) {
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
                // 在 adapter 中选中 initial 到 last 的条目，清除选中 min 到 max 除了前面要选中条目之外的条目
                adapter.selectRange(initialSelection, lastDraggedIndex, minReached, maxReached);
            }
            if (initialSelection == lastDraggedIndex) {
                minReached = lastDraggedIndex;
                maxReached = lastDraggedIndex;
            }
        }
        return true;
    }
    // ...
}
```

其中 `getItemPosition()` 是获取触摸时对应的条目位置的方法：

```Java
private int getItemPosition(MotionEvent e) {
    final View v = findChildViewUnder(e.getX(), e.getY());
    if (v == null) return NO_POSITION;
    return getChildAdapterPosition(v);
}
```

### 列表自动滚动

手指滑动到滚动区域时列表自动滚动是怎样做到的？答案就是**使用一个 Handler 每 25ms post Runnable 调用滚动的方法并更新滚动速度。**

滚动的方向是通过手指是在上部滚动区还是下部滚动区来决定的，滚动的速度通过判断手指与滚动区边距的距离大小而改变。

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

### 缺点

方案一基于自定义 RecyclerView，如果项目中已经使用了自定义 RecyclerView，则难以接入。此外，该方案的核心逻辑是在事件分发的 `dispatchTouchEvent()` 方法进行处理。事实上，RecyclerView 本身是没有重写该方法的，该实现可能会破坏原有的一些机制或性质。

## 方案二： AndroidDragSelect

前文说到，方案二就是分析了方案一的缺点之后，给出了自己的基于 `OnItemTouchListener` 的实现方案，耦合度低，可以很容易集成进现有的项目当中。

OnItemTouchListener 本身就是 RecyclerView 提供的扩展条目触控操作的接口，典型的应用则是长按拖拽排序、滑动删除等功能，在查看方案三的源码之前，我们先来看一下 RecyclerView 中的这个 `OnItemTouchListener` 接口。

### OnItemTouchListener

从 `OnItemTouchListener` 的源码注释可以看出，它是在与 RecyclerView 同一视图层级上对事件进行处理的，也就是在分发给子 View 之前。

```Java
public static interface OnItemTouchListener {
    // public boolean onInterceptTouchEvent(MotionEvent e)
    public boolean onInterceptTouchEvent(RecyclerView rv, MotionEvent e);
    // public boolean onTouchEvent(MotionEvent e)
    public void onTouchEvent(RecyclerView rv, MotionEvent e);
    // public void requestDisallowInterceptTouchEvent(boolean disallowIntercept)
    public void onRequestDisallowInterceptTouchEvent(boolean disallowIntercept);
}
```

其中，添加的注释是 ViewGroup 中这三个方法的定义，回顾一下事件分发机制： `dispatchTouchEvent()` 用来进行事件的分发，而 `onInterceptTouchEvent()` 被它调用，用来进行判断是否进行拦截，真正地处理点击事件则是在 `onTouchEvent()` 当中。

可以看到除了 `onRequestDisallowInterceptTouchEvent()` 方法之外，其他两个都有一点小差别：

* `onInterceptTouchEvent()` 这个方法参数不一样
* `onTouchEvent()` 除了参数不一样，返回值也变了，变成了无返回值。
  
那么也就可以猜测，如果 `OnItemTouchListener` 处理了点击事件，就不会再交由父 View 再进行处理了。到底是不是这样子呢，我们通过 RecyclerView 的源码查看一下。

```Java
@Override
public boolean onInterceptTouchEvent(MotionEvent e) {
    // 省略代码……
    if (dispatchOnItemTouchIntercept(e)) {
        cancelTouch();
        return true;
    }
    // 省略代码……
}
private boolean findInterceptingOnItemTouchListener(MotionEvent e) {
    int action = e.getAction();
    final int listenerCount = mOnItemTouchListeners.size();
    for (int i = 0; i < listenerCount; i++) {
        final OnItemTouchListener listener = mOnItemTouchListeners.get(i);
        if (listener.onInterceptTouchEvent(this, e) && action != MotionEvent.ACTION_CANCEL) {
            mInterceptingOnItemTouchListener = listener;
            return true;
        }
    }
    return false;
}
```

再查看 `dispatchOnItemTouchIntercept()` 可以看到，如果添加的 `OnItemTouchListener` 它在 `onInterceptTouchEvent` 拦截了 `MotionEvent` 事件，那么就会返回 `true` ，此时 RecyclerView `onInterceptTouchEvent` 也返回 `true` 表明拦截了此次事件，由 RecyclerView 的 `onTouchEvent` 处理而不是由子 View 进行处理。

再去看看 RecyclerView 的 `onTouchEvent()` 方法，看是不是同样地把这个事件交由 OnItemTouchListener 来处理。

```Java
@Override
public boolean onTouchEvent(MotionEvent e) {
    // 省略代码……
    if (dispatchOnItemTouch(e)) {
        cancelTouch();
        return true;
    }
    // 省略代码……
}
```

与 `dispatchOnItemTouchIntercept()` 类似的，如果添加的 `OnItemTouchListener` 它拦截了 MotionEvent 事件，即在 `onInterceptTouchEvent` 返回 true， 那么此处就会固定返回 true，并且就由它在 `onTouchEvent()` 中进行处理。

这里再稍微看一下 `dispatchOnItemTouch()` 来解决一个实践中的小困惑： `OnItemTouchListener` 里在 `onInterceptTouchEvent()` 中对于 MotionEvent.ACTION_DOWN 无论是否返回 true，都不会在 `onTouchEvent` 里收到此 MotionEvent。

```Java
private boolean dispatchOnItemTouch(MotionEvent e) {
    if (mActiveOnItemTouchListener != null) {
        if (action == MotionEvent.ACTION_DOWN) {
            // Stale state from a previous gesture, we're starting a new one. Clear it.
            mActiveOnItemTouchListener = null;
        } else {
            mActiveOnItemTouchListener.onTouchEvent(this, e);
            if (action == MotionEvent.ACTION_CANCEL || action == MotionEvent.ACTION_UP) {
                // Clean up for the next gesture.
                mActiveOnItemTouchListener = null;
            }
            return true;
        }
    }
    // 省略代码……
}
```

可以看到 OnItemTouchListener 中的 `onTouchEvent()` 是无法接收到 MotionEvent.ACTION_DOWN 的。

方案二 [weidongjian/AndroidDragSelect-SimulateGooglePhoto](https://github.com/weidongjian/AndroidDragSelect-SimulateGooglePhoto) 是个 Demo，参数、设定比较粗糙，但其设计思路是很好的。而方案三 [MFlisar/DragSelectRecyclerView](https://github.com/MFlisar/DragSelectRecyclerView) 是在其基础上进行改进完成的，两者的共同点是 `OnItemTouchListener`，它们几乎是一样的，但它的代码更规范一点。所以下文以方案三代码来具体分析方案中的 `OnItemTouchListener。`

### 滚动区的定义

方案二的滚动区设定比较简单，我就直接上图了，其实这个图也不对，可能原作者是这样子想的，但是源码里的那个 mTopBound 设定得不对。

![方案二滚动区](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-area-option-two.png)

方案三与方案一的滚动区设定一模一样，只是名称改了一下。

![方案三滚动区](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-area-option-three.png)

### 自动滚动实现

方案一使用的是通过 Handler 的 postDelayed 方法的延时策略，可以在大约每 25ms 时滚动一下，这里使用大约就是因为 Handler 的调度也是需要时间的。在本方案中，使用 Scroller 来实现流畅地滚动，Scroller 的使用、讲解可以看《Android 开发艺术探索》及网上资料来学习。具体就见下面的代码：

```Java
public void startAutoScroll() {
    if (recyclerView == null) {
        return;
    }
    // 创建 Scroller
    if (scroller == null) {
        scroller = ScrollerCompat.create(recyclerView.getContext(),
                new LinearInterpolator());
    }
    if (scroller.isFinished()) {
        recyclerView.removeCallbacks(scrollRun);
        // 设置参数，这里只有100000是有意义的，它代表
        // 手指在滚动区完全静止不动时最多可持续滚动100s
        scroller.startScroll(0, scroller.getCurrY(), 0, 5000, 100000);
        ViewCompat.postOnAnimation(recyclerView, scrollRun);
    }
}

public void stopAutoScroll() {
    if (scroller != null && !scroller.isFinished()) {
        recyclerView.removeCallbacks(scrollRun);
        scroller.abortAnimation();
    }
}

private Runnable scrollRun = new Runnable() {
    @Override
    public void run() {
        if (scroller != null && scroller.computeScrollOffset()) {
            scrollBy(scrollDistance);
            ViewCompat.postOnAnimation(recyclerView, scrollRun);
        }
    }
};

private void scrollBy(int distance) {
    int scrollDistance;
    // 限制滚动速度
    if (distance > 0) {
        scrollDistance = Math.min(distance, MAX_SCROLL_DISTANCE);
    } else {
        scrollDistance = Math.max(distance, -MAX_SCROLL_DISTANCE);
    }
    recyclerView.scrollBy(0, scrollDistance);
    // 自动滚动时的选择范围的更新在这里，因为只在自动滚动时这两个才有合法值
    if (lastX != Float.MIN_VALUE && lastY != Float.MIN_VALUE) {
        updateSelectedRange(recyclerView, lastX, lastY);
    }
}
```

### 触摸事件的处理

#### onInterceptTouchEvent

首先是 `onInterceptTouchEvent()` 方法，简单来说，在这里判断一下滑动选择功能是否激活，只在激活时候才拦截触摸事件；事实上，由于长按才 active，所以拦截不到 MotionEvent.ACTION_DOWN 事件，而它将在长按之后处理接收到的第一个 MotionEvent.ACTION_MOVE 事件，在这里进行参数的初始化。后续再接收到的 MotionEvent 就全部都由 `onTouchEvent()` 方法来处理了。

```Java
@Override
public boolean onInterceptTouchEvent(RecyclerView rv, MotionEvent e) {
    if (!mIsActive || rv.getAdapter().getItemCount() == 0)
        return false;

    int action = MotionEventCompat.getActionMasked(e);
    switch (action) {
        // 事实上，由于长按才active，所以以下两个case是不会收到的
        case MotionEvent.ACTION_POINTER_DOWN:
        case MotionEvent.ACTION_DOWN:
            reset();
            break;
    }
    // 参数设定
    mRecyclerView = rv;
    int height = rv.getHeight();
    mTopBoundFrom = mTouchRegionTopOffset;
    mTopBoundTo = mTouchRegionTopOffset + mAutoScrollDistance;
    mBottomBoundFrom = height - mTouchRegionBottomOffset - mAutoScrollDistance;
    mBottomBoundTo = height - mTouchRegionBottomOffset;
    return true;
}
```

#### onTouchEvent

这里就是对 Move 事件进行自动滚动、更新选择范围的处理。

```Java
@Override
public void onTouchEvent(RecyclerView rv, MotionEvent e) {
    if (!mIsActive) {
        return;
    }

    int action = MotionEventCompat.getActionMasked(e);
    switch (action) {
        case MotionEvent.ACTION_MOVE:
            // 将此方法提前，因为查看方法可以知道它只处理滚动区内的事件，
            // 包括自动滚动、更新选择范围
            processAutoScroll(e);
            if (!mInTopSpot && !mInBottomSpot) {
                // 不在滚动区内的只要更新选择范围
                updateSelectedRange(rv, e);
            }
            break;
        case MotionEvent.ACTION_CANCEL:
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_POINTER_UP:
            // 退出时重置状态
            reset();
            break;
    }
}
```

自动滚动的实现方式上面已经提过了，这里的的自动滚动处理主要是解决三个问题：

1. 记录手指最后位置，以便在手指不动时还可以更新选择范围；
1. 手指是否在滚动区的判断，以及是否允许滚动区之上的滚动；
1. 根据手指在滚动区的位置更新“速度”值；

```Java
private void processAutoScroll(MotionEvent event) {
    int y = (int) event.getY();
    if (y >= mTopBoundFrom && y <= mTopBoundTo) {
        // 严格位于上滚动区内
        mLastX = event.getX();
        mLastY = event.getY();
        // 计算速度 = maxSpeed * (手指离上滚动区下边界的距离 / 上滚动区的高度）
        // 往上滚速度为负数
        mScrollSpeedFactor = (mTopBoundTo - y) / (float)mAutoScrollDistance;
        mScrollDistance = (int) (mMaxScrollDistance * mScrollSpeedFactor * -1f);
        if (!mInTopSpot) {
            mInTopSpot = true;
            startAutoScroll();
        }
    } else if (mScrollAboveTopRegion && y < mTopBoundFrom) {
        // 允许在上滚动区之上进行自动滚动
        mLastX = event.getX();
        mLastY = event.getY();
        mScrollDistance = mMaxScrollDistance * -1;
        if (!mInTopSpot) {
            mInTopSpot = true;
            startAutoScroll();
        }
    } else if (y >= mBottomBoundFrom && y <= mBottomBoundTo) {
        // ... 下滚动区类似
    } else if (mScrollBelowTopRegion && y > mBottomBoundTo) {
        // ... 下滚动区之下类似
    } else {
        // 不在滚动区内重置数据
        mInBottomSpot = false;
        mInTopSpot = false;
        mLastX = Float.MIN_VALUE;
        mLastY = Float.MIN_VALUE;
        stopAutoScroll();
    }
}
```

### 选择范围的更新与回调

上面看到，在自动滚动时进行选择范围的更新。先来简单看一下更新范围的更新方法：

```Java
private void updateSelectedRange(RecyclerView rv, float x, float y) {
    View child = rv.findChildViewUnder(x, y);
    if (child != null) {
        int position = rv.getChildAdapterPosition(child);
        if (position != RecyclerView.NO_POSITION && mEnd != position) {
            mEnd = position;
            // 在手指到达新的条目时再通知更新
            notifySelectRangeChange();
        }
    }
}
```

可见重点在于 `notifySelectRangeChange()` 方法。这段代码可以结合图来理解。

![坐标图.png](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-selection-range.png)

首先明确一些条件：

* 手指按下的地方为 start，手指当前的地方为 end。但它们的大小关系不定。
* start 与 end 之间的条目一定是被选中的。
* newStart 代表现在 start 与 end 两者中较小者，newEnd 代表较大者
* lastStart 和 lastEnd 与 newStart 和 newEnd 含义相同，但指的是未更新前的位置

事实上，如果是列表型，那么因为这个范围不会跳变，所以 lastStart 和 lastEnd 与 newStart 和 newEnd 只会相差 1。但如果是网格型列表，可以上下行滑动时范围就会跳变。

```Java
private void notifySelectRangeChange() {
    if (mSelectListener == null)
        return;
    if (mStart == RecyclerView.NO_POSITION || mEnd == RecyclerView.NO_POSITION)
        return;

    int newStart, newEnd;
    newStart = Math.min(mStart, mEnd);
    newEnd = Math.max(mStart, mEnd);
    if (mLastStart == RecyclerView.NO_POSITION || mLastEnd == RecyclerView.NO_POSITION) {
        if (newEnd - newStart == 1)
            mSelectListener.onSelectChange(newStart, newStart, true);
        else
            mSelectListener.onSelectChange(newStart, newEnd, true);
    } else {
        // 重点看这四句，对照着坐标图可以看懂的
        if (newStart > mLastStart)
            mSelectListener.onSelectChange(mLastStart, newStart - 1, false);
        else if (newStart < mLastStart)
            // 此条件下如图，应该把它们之间的选中。而lastStart之前已经选中了。
            mSelectListener.onSelectChange(newStart, mLastStart - 1, true);
        if (newEnd > mLastEnd)
            mSelectListener.onSelectChange(mLastEnd + 1, newEnd, true);
        else if (newEnd < mLastEnd)
            // 此条件下如图，应该把它们之间的取消选中。而lastEnd之前已经选中了也要取消。
            mSelectListener.onSelectChange(newEnd + 1, mLastEnd, false);
    }

    mLastStart = newStart;
    mLastEnd = newEnd;
}
```

那么这个范围就是通过回调来通知监听者的。

```Java
public interface OnDragSelectListener {
    /**
     * @param start      the newly (un)selected range start
     * @param end        the newly (un)selected range end
     * @param isSelected true, it range got selected, false if not
     */
    void onSelectChange(int start, int end, boolean isSelected);
}
```

在我的理解中 start 与 end 之间的条目的选中状态是指一种状态，它可以代表是选择条目的状态，也可以是不选择条目的状态，具体来说就是选中与未选中是两种状态，我们指定 true 代表某一种状态，从而使用 false 代表另一种状态，因此方法 `void onSelectChange(int start, int end, boolean isSelected)` 的参数 3 确切地说应该命名为 state。这样子再重新理解一下上面 `notifySelectRangeChange` 中的那重要的四句话就会明白它指的是：

* 指定 start 与 end 之间的条目的状态为 A
* 根据坐标图，将 newStart 和 newEnd 之间的状态也置为 A，另外的则更新状态为非 A

以上说的这个状态相关内容，如果不是太理解，可以看看我的实现方案，它是在对方案三进行再次修订而成的，对于此内容会有更好的理解。

方案一回调为 `selectRange(initialSelection, lastDraggedIndex, minReached, maxReached)` 有 4 个参数，相当于把方案二的 lastStart、lastEnd、newStart 和 newEnd 全部传回来。但实际上，传回之后也是采用同样的处理方式，因此将选择与反选的操作放到 OnItemTouchListener 里会更方便。

### 方案二的使用与效果

到目前为止，基于这一个单纯的回调，就可以完成 Google 的选择策略了。使用也非常地简单：

```Java
touchListener.setSelectListener(new DragSelectTouchListener.onSelectListener() {
    @Override
    public void onSelectChange(int start, int end, boolean isSelected) {
        //选择的范围回调
        adapter.selectRangeChange(start, end, isSelected);
        actionBar.setTitle(String.valueOf(adapter.getSelectedSize()) + " selected");
    }
});
rv.addOnItemTouchListener(touchListener);
```

相比方案一的是不是特别地简洁？

## 方案三：DragSelectRecyclerView

### 选择策略的扩展

之前提到，方案三是基于方案二进行扩展的，可以看到，在 `OnItemTouchListener` 这一块，两者其实几乎是一模一样的。而方案三一个很好的地方，就是在几乎不修改 DragSelectTouchListener 的前提下，对其选择功能进行了强大方便的扩展。下面我将从设计的思路出发，理一理是怎样完成的。

首先要清楚方案三扩展了哪些选择策略，总共有 4 种模式：

* Simple: 滑过时选中，往回滑时取消选中
* ToggleAndUndo: 滑过时反选，往回滑时恢复原状态
* FirstItemDependent: 反选按下的第一条目，滑过时与第一条目状态一致，往回滑时与第一条目状态相反
* FirstItemDependentToggleAndUndo: 反选按下的第一条目，滑过时与第一条目状态一致，往回滑时恢复原状态

<!--more-->

关于这 4 种模式的效果请看 GIF 图：

| Simple | ToggleAndUndo |
| :---: | :---: |
| ![Simple](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-policy-simple.gif) | ![ToggleAndUndo](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-policy-toggle-and-undo.gif) |
| FirstItemDependent | FirstItemDependentToggleAndUndo |
| ![FirstItemDependent](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-policy-first-item-dependent.gif) | ![FirstItemDependentToggleAndUndo](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-policy-first-item-dependent-toggle-and-undo.gif) |

第 1 种模式其实就是 Google Photos 的策略，而第 4 种策略与我们产品要求实现的交互基本相同（感动地哭出声……）。看了效果之后，我们再想想基于方案二的一个回调 `onSelectChange(int start, int end, boolean isSelected)` 能完成吗？

首先可以知道 Simple 模式是可以做到的，因为这个模式下除了位置信息之外无需另外的信息。而另外三种都无法做到，因为它们都需要按下时列表目前的状态信息：

* ToggleAndUndo 需要知道按下时，哪些条目已经被选择了，这样子才能恢复原状态；
* FirstItemDependent 需要知道按下时的条目的原状态，才能反选第一条目；
* FirstItemDependentToggleAndUndo 需要知道的就包括前两者的信息：哪些条目被选择了、第一条目的原状态（事实上，这一信息包含在前一个信息里）。

那么，很自然地，在按下时也需要一个回调，以此为入口获取所需要的信息。因此方案三先扩展了 `DragSelectTouchListener.OnDragSelectListener` 的接口，首先看一下对 `OnDragSelectListener` 这个接口的扩展：

```Java
public interface OnAdvancedDragSelectListener extends OnDragSelectListener{
   void onSelectionStarted(int start);
   void onSelectionFinished(int end);
}

// 增加了新接口后在原代码逻辑中增加调用逻辑
public void startDragSelection(int position){
   // 省略代码...
   if (mSelectListener != null && mSelectListener instanceof OnAdvancedDragSelectListener) {
       ((OnAdvancedDragSelectListener)mSelectListener).onSelectionStarted(position);
   }        
}

private void reset() {
   // 省略代码...
   if (mSelectListener != null && mSelectListener instanceof OnAdvancedDragSelectListener)
       ((OnAdvancedDragSelectListener) mSelectListener).onSelectionFinished(mEnd);
}
```

可以看到，只是继承增加了两个接口，分别在点击开始、结束时被调用。实现该接口获取点击时列表的状态信息了，也就可以通过这些信息实现扩展的选择策略。

### DragSelectionProcessor

方案三在扩展了 DragSelectTouchListener 后，将其实现封装了一层，把这 4 种模式放到一个控制器里：

```Java
public class DragSelectionProcessor implements DragSelectTouchListener.OnAdvancedDragSelectListener {}
```

看看它是怎么在按下时的回调 `onSelectionStarted()` 中获得信息的呢？

```Java
@Override
public void onSelectionStarted(int start) {
   mOriginalSelection = new HashSet<>();
   Set<Integer> selected = mSelectionHandler.getSelection();
   if (selected != null)
       mOriginalSelection.addAll(selected);
   mFirstWasSelected = mOriginalSelection.contains(start);
   // 省略代码...
}
```

从上面代码中可以看到，正是在开始选择的回调中获取了列表中已选择项的信息，而且这也是使用了一个接口 `ISelectionHandler` 来获取信息的：

```Java
public interface ISelectionHandler {
   Set<Integer> getSelection();
   void updateSelection(int start, int end, boolean isSelected, boolean calledFromOnStart);
   boolean isSelected(int index);
}
```

可以看到该接口还有两个回调函数，那另外两个方法是做什么的呢？看一下上面 `onSelectionStarted()` 省略的代码：

```Java
@Override
public void onSelectionStarted(int start) {
   // 省略代码...
   switch (mMode) {
       case Simple: {
           mSelectionHandler.updateSelection(start, start, true, true);
           break;
       }
       case ToggleAndUndo: {
           mSelectionHandler.updateSelection(start, start, !mOriginalSelection.contains(start), true);
           break;
       }
       case FirstItemDependent: {
           mSelectionHandler.updateSelection(start, start, !mFirstWasSelected, true);
           break;
       }
       case FirstItemDependentToggleAndUndo: {
           mSelectionHandler.updateSelection(start, start, !mFirstWasSelected, true);
           break;
       }
   }
}
```

也就是对于不同模式下，得到了第一个条目的信息，要更新第一条目的状态，比如说 FirstItemDependent 就是要反选第一条目，所以 `updateSelection()` 这个方法就是调用具体设置状态的方法的。而在 `onSelectChange()` 回调中，我们看到更新状态变成了另一个方法 `checkedUpdateSelection()` 。

```Java
@Override
public void onSelectChange(int start, int end, boolean isSelected) {
   switch (mMode) {
       case Simple: {
           if (mCheckSelectionState)
               checkedUpdateSelection(start, end, isSelected);
           else
               mSelectionHandler.updateSelection(start, end, isSelected, false);
           break;
       }
       case ToggleAndUndo: {
           for (int i = start; i <= end; i++)
               checkedUpdateSelection(i, i, isSelected ? !mOriginalSelection.contains(i) : mOriginalSelection.contains(i));
           break;
       }
       case FirstItemDependent: {
           checkedUpdateSelection(start, end, isSelected ? !mFirstWasSelected : mFirstWasSelected);
           break;
       }
       case FirstItemDependentToggleAndUndo: {
           for (int i = start; i <= end; i++)
               checkedUpdateSelection(i, i, isSelected ? !mFirstWasSelected : mOriginalSelection.contains(i));
           break;
       }
   }
}
```

```Java
private void checkedUpdateSelection(int start, int end, boolean newSelectionState) {
   if (mCheckSelectionState) {
       for (int i = start; i <= end; i++) {
           if (mSelectionHandler.isSelected(i) != newSelectionState)
               mSelectionHandler.updateSelection(i, i, newSelectionState, false);
       }
   } else
       mSelectionHandler.updateSelection(start, end, newSelectionState, false);
}
```

一下子就明白了，`isSelected()` 是获取某一条目的选择状态的，可以用来检测原来列表的状态的选项。也就是说，如果原列表的某条目的状态 `mSelectionHandler.isSelected(i)` 如果与新状态不同的话，才需要更新该条目的状态。这个的原因其实之前也说过了，对于相同的状态就不要调用 Adapter 的方法去重新设置了，这是一种浪费。

拿 FirstItemDependentToggleAndUndo 模式下的选择：`checkedUpdateSelection(i, i, isSelected ? !mFirstWasSelected : mOriginalSelection.contains(i))` 来理解一下。

`onSelectChange` 这一回调中，第三个参数 `isSelected` 意义为 true 时想要将 i 条目状态设置为选中，false 时想要将 i 条目状态设置为取消选中。对于 FirstItemDependentToggleAndUndo 模式来说，true 代表 i 条目要与 start 条目的现状态相同，所以是 `!mFirstWasSelected`，false 不代表与 start 条目相反，而是代表 i 条目要恢复到原来的状态，所以变成了 `mOriginalSelection.contains(i)`。这种设计真是妙极。

当然了，前面也说到，这个 DragSelectionProcessor 就是对 `DragSelectTouchListener.OnAdvancedDragSelectListener` 的扩展的一个封装，而 `DragSelectTouchListener.OnAdvancedDragSelectListener` 是对 `DragSelectTouchListener.OnDragSelectListener` 的扩展。因此，如果你只需要实现 Simple 模式也就是 Google Photos 的选择模式的话，直接实现 `DragSelectTouchListener.OnDragSelectListener` 就可以了。

```Java
onDragSelectionListener = new DragSelectTouchListener.OnDragSelectListener() {
   @Override
   public void onSelectChange(int start, int end, boolean isSelected) {
    // update your selection
    // range is inclusive start/end positions
   }
}
```

如果需要在点击开始与结束时做一些操作，只需要实现 `DragSelectTouchListener.OnAdvancedDragSelectListener`。

```Java
onDragSelectionListener = new DragSelectTouchListener.OnAdvancedDragSelectListener() {
  @Override
  public void onSelectChange(int start, int end, boolean isSelected) {
    // update your selection
    // range is inclusive start/end positions
  }

  @Override
  public void onSelectionStarted(int start) {
    // drag selection was started at index start
  }

  @Override
  public void onSelectionFinished(int end) {
    // drag selection was finished at index start
  }
};
```

而如果想要使用扩展出来的 3 种模式，可以基于 `OnAdvancedDragSelectListener` 自己进行实现，也可以直接使用封装好的 DragSelectionProcessor。

```Java
onDragSelectionListener = new DragSelectionProcessor(new DragSelectionProcessor.ISelectionHandler() {
 @Override
 public Set<Integer> getSelection() {
  // return a set of all currently selected indizes
  return selection;
 }

 @Override
 public boolean isSelected(int index) {
  // return the current selection state of the index
  return selected;
 }

 @Override
 public void updateSelection(int start, int end, boolean isSelected, boolean calledFromOnStart) {
  // update your selection
  // range is inclusive start/end positions
  // and the processor has already converted all events according to it'smode
 }
})
 // pass in one of the 4 modes, simple mode is selected by default otherwise
 .withMode(DragSelectionProcessor.Mode.FirstItemDependentToggleAndUndo);

mDragSelectTouchListener = new DragSelectTouchListener()
 // check region OnDragSelectListener for more infos
 .withSelectListener(onDragSelectionListener);
```

具体的代码以及使用示例请直接查看 [MFlisar/DragSelectRecyclerView](https://github.com/MFlisar/DragSelectRecyclerView) 的 README 文档。至此，GitHub 上的三个库都分析完毕了，DragSelectRecyclerView 是完整度最好的，但我觉得它还可以进一步改进，因此，接下来是时候来撸一个自已的支持网格列表及常规列表的拖动、滑动多选的库了。

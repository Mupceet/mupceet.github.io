---
title: RecyclerView 拖动/滑动多选的实现（2）
date: 2017-07-30 21:02:33
categories:
- 技术向
tags:
- Android
---

## 方案三： AndroidDragSelect

前文说到，方案三就是分析了方案一的缺点之后，给出了自己的基于 `OnItemTouchListener` 的实现方案，耦合度低，可以很容易集成进现有的项目当中。

从自定义 RecyclerView 的方案中可以看到，它是在事件分发的时候进行处理。事实上，在这个方法里做计算感觉上就有点不对，从源码来看，RecyclerView 本身是没有重写 `dispatchTouchEvent()` 方法的，而方案一通过重写此方法并在这里完成自动滚动的计算处理，显得有些重。

回顾一下事件分发机制，其中 `dispatchTouchEvent()` 用来进行事件的分发，`onInterceptTouchEvent()` 被前一个方法调用，用来进行判断是否进行拦截，真正地处理点击事件则是在 `onTouchEvent()` 当中。所以方案三就是利用了 RecyclerView 的 `OnItemTouchListener` 来对触摸事件进行拦截处理。

<!--more-->

在查看方案三的源码之前，我们先来看一下 RecyclerView 中的这个 OnItemTouchListener 接口：

### OnItemTouchListener
从源码注释可以看出，三个方法是在与 RecyclerView 同一视图层级上对事件进行处理的，也就是在分发给子 View 之前。
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
其中注释是 ViewGroup 中这三个方法的定义，可以看到除了 `onRequestDisallowInterceptTouchEvent()` 方法之外，其他两个都有一点小差别。

`onInterceptTouchEvent()` 这个方法参数不一样，`onTouchEvent()` 除了参数不一样，返回值也变了，变成了无返回值。那么也就可以猜测，如果 OnItemTouchListener 处理了点击事件，就不会再交由父 View 再进行处理了。到底是不是这样子呢，我们通过 RecyclerView 的源码查看一下。

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
```
再进一步查看 `dispatchOnItemTouchIntercept()` 可以看到，如果添加的 OnItemTouchListener 它拦截了 MotionEvent 事件，那么就返回 true，此时 RecyclerView 也返回 true 表明拦截了此次事件不再由子 View 进行处理。

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
与 `dispatchOnItemTouchIntercept()` 类似的，如果添加的 OnItemTouchListener 它拦截了 MotionEvent 事件，那么就由它在 `onTouchEvent()` 中进行处理。这里再稍微看一下 `dispatchOnItemTouch()` 来解决一个实践中的小困惑：OnItemTouchListener 里在 `onInterceptTouchEvent()` 中对于 MotionEvent.ACTION_DOWN 无论是否返回 true，都不会在 onTouchEvent 里收到此 MotionEvent。
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
可以看到 OnItemTouchListener 中的 `onInterceptTouchEvent()` 是无法接收到 MotionEvent.ACTION_DOWN 的。

接下来对方案三进行分析，在正式分析之前，先吐槽一句，大家还是比较喜欢 Start 可以直接拿来用的库，因为这个方案三 [weidongjian/AndroidDragSelect-SimulateGooglePhoto：19 ★](https://github.com/weidongjian/AndroidDragSelect-SimulateGooglePhoto) 是个 Demo，参数、设定比较粗糙，导致了星星好少，但其设计思路是很好的。而方案二 [MFlisar/DragSelectRecyclerView：267 ★](https://github.com/MFlisar/DragSelectRecyclerView) 就是在其基础上进行的改进，两者的共同点就是 OnItemTouchListener，它们几乎是一样的。而方案三的滑动多选也就只是通过这一个类来实现的，所以下文以方案二代码来具体分析，它的代码更规范一点，但是方案二代码里面大括号是放在行首以及 if 代码块没有大括号让我很难受……

### 滚动区的定义

方案三的滚动区设定比较简单，我就直接上图了，其实这个图也不对，可能原作者是这样子想的，但是源码里的那个 mTopBound 设定得不对。

![方案三滚动区](http://upload-images.jianshu.io/upload_images/2946447-82c78b8a3cc2ca4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

方案二与方案一的滚动区设定一模一样，只是名称改了一下。

![方案二滚动区](http://upload-images.jianshu.io/upload_images/2946447-38a17f80479dc7de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### 自动滚动实现
方案一使用的是一种通过 Handler 的 postDelayed 方法的延时策略，可以在大约每 25ms 时滚动一下，这里使用大约就是因为 Handler 的调度也是需要时间的。在本方案中，使用 Scroller 来实现流畅地滚动，Scroller 的使用、讲解可以看《Android 开发艺术探索》及网上资料来学习。具体就见下面的代码：

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
* 记录手指最后位置，以便在手指不动时还可以更新选择范围
* 手指是否在滚动区的判断，以及是否允许滚动区之上的滚动
* 根据手指在滚动区的位置更新“速度”值

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
        mLastX = event.getX();
        mLastY = event.getY();
        mScrollSpeedFactor = ((y - mBottomBoundFrom)) / (float)mAutoScrollDistance;
        mScrollDistance = (int) ((float) mMaxScrollDistance * mScrollSpeedFactor);
        if (!mInBottomSpot) {
            mInBottomSpot = true;
            startAutoScroll();
        }
    } else if (mScrollBelowTopRegion && y > mBottomBoundTo) {
        mLastX = event.getX();
        mLastY = event.getY();
        mScrollDistance = mMaxScrollDistance;
        if (!mInBottomSpot) {
            mInBottomSpot = true;
            startAutoScroll();
        }
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


![坐标图.png](http://upload-images.jianshu.io/upload_images/2946447-2ee151afe5b02b46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


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

以上说有这个状态相关内容，如果不是太理解，可以看看我的实现方案，它是在对方案二进行再次修订而成的，对于此内容会有更好的理解。

方案一回调为 `selectRange(initialSelection, lastDraggedIndex, minReached, maxReached)` 有 4 个参数，相当于把方案二的 lastStart、lastEnd、newStart 和 newEnd 全部传回来。但实际上，传回之后也是采用同样的处理方式，因此将选择与反选的操作放到 OnItemTouchListener 里会更方便。

### 方案三的使用与效果
到目前为止，基于这一个单纯的回调，就可以完成 Google 的选择策略了。实现也非常地简单：
```Java
touchListener.setSelectListener(new DragSelectTouchListener.onSelectListener() {
    @Override
    public void onSelectChange(int start, int end, boolean isSelected) {
        //选择的范围回调
        adapter.selectRangeChange(start, end, isSelected);
        actionBar.setTitle(String.valueOf(adapter.getSelectedSize()) + " selected");
    }
});
```

是不是特别地简洁？但是这里有两点要注意，
* 一个是由于回调 `onSelectChange()` 非常频繁，所以在 Adapter 里的相应的选择的方法 `selectRangeChange` 一定要注意判断一下条目的原先的状态，也就是说如果状态没有改变，那么就什么都不做，如果状态更改了，才去更新状态：`notifyItemChanged()`。
* 另一个是，由于 Item Change 时默认带着动画，所以在滚动时如果速度比较快、条目比较宽，就会看到明显的残影。如果没有自定义的动画可以采用以下方法去除默认的 Change 动画即可：
  ```Java
  ((SimpleItemAnimator)recyclerView.getItemAnimator()).setSupportsChangeAnimations(false);
  // 或者
  recyclerView.getItemAnimator().setChangeDuration(0);
  ```
  如果有动画的话可能不行，动画的内容我后续会进行实践，并且还会看看使用 OnItemTouchListener 实现的 Click 事件回调方案与此滑动方案的兼容性。大家可以自行测试、处理。

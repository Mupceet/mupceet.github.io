---
title: RecyclerView 拖动/滑动多选的实现（3）
date: 2017-07-31 22:55:33
categories:
- 技术向
tags:
- Android
---

## DragMultiSelect

设计目标是为了给 RecyclerView 提供便捷多选的功能，它具有以下特性：

1. 基于 OnItemTouchListener 实现，便于与项目中的 RecyclerView 集成；
1. 支持拖动多选与滑动多选；
1. 支持横向滚动与竖向滚动的列表；

DragMultiSelect 引入了一个滑动选择的模式，因此需要在方法三的实现上进一步扩展。我们逐一看下方案的具体实现。

### 滚动区的定义

DragMultiSelect 的滚动区的定义与方案三一致：

![DragMultiSelect 滚动区](https://raw.githubusercontent.com/Mupceet/article-piture/master/drag-area-option-three.png)

<!--more-->

### 自动滚动实现

DragMultiSelect 的自动滚动的实现与方案一类似，在每一次的动画回调中调用列表的滚动方法。

```java
private final Runnable mScrollRunnable = new Runnable() {
    @Override
    public void run() {
        final AutoScroller scroller = mScroller;
        if (!scroller.isScrolling()) {
            return;
        }

        scroller.computeScrollDelta();
        scrollBy(scroller.getDelta());
        ViewCompat.postOnAnimation(mRecyclerView, this);
    }
};
```

### 触摸事件的处理

DragMultiSelect 的选择模式的切换如图所示：

```txt
                       !autoChangeMode           +-------------------+     inactiveSelect()
          +------------------------------------> |                   | <--------------------+
          |                                      |      Normal       |                      |
          |        activeDragSelect(position)    |                   | activeSlideSelect()  |
          |      +------------------------------ |                   | ----------+          |
          |      v                               +-------------------+           v          |
 +-------------------+                              autoChangeMode     +-----------------------+
 | Drag From Normal  | ----------------------------------------------> |                       |
 +-------------------+                                                 |                       |
 |                   |                                                 |                       |
 |                   | activeDragSelect(position) && allowDragInSlide  |        Slide          |
 |                   | <---------------------------------------------- |                       |
 |  Drag From Slide  |                                                 |                       |
 |                   |                                                 |                       |
 |                   | ----------------------------------------------> |                       |
 +-------------------+                                                 +-----------------------+
```

正常情况下（Normal），可以通过设置，设置成长按选择之后抬手退出以及长按选择之后抬手进入滑动选择的情况，又通过 API 或者操作，进入 Slide 模式。而 Drag 模式细分为 Drag From Normal 和 Drag From Slide，分别对应于长按进入拖动选择以及滑动进入选择的情况。

在触摸事件的处理中，增加了滑动选择模式的识别，具体的改动见代码中的注释。

```Java
@Override
public boolean onInterceptTouchEvent(@NonNull RecyclerView rv, @NonNull MotionEvent e) {
    RecyclerView.Adapter<?> adapter = rv.getAdapter();
    if (adapter == null || adapter.getItemCount() == 0) {
        return false;
    }
    boolean intercept = false;
    int action = e.getAction();
    int actionMask = action & MotionEvent.ACTION_MASK;
    // It seems that it's unnecessary to process multiple pointers.
    switch (actionMask) {
        case MotionEvent.ACTION_DOWN:
            // call the selection start's callback before moving
            if (mSelectState == SELECT_STATE_SLIDE && isInSlideArea(e)) {
                mSlideStateStartPosition = getItemPosition(rv, e.getX(), e.getY());
                if (mSlideStateStartPosition != RecyclerView.NO_POSITION) {
                    intercept = true;
                    mCallback.onSelectStart(mSlideStateStartPosition);
                    mHaveCalledSelectStart = true;
                }
            }
            break;
        case MotionEvent.ACTION_MOVE:
            if (mSelectState == SELECT_STATE_DRAG_FROM_NORMAL
                    || mSelectState == SELECT_STATE_DRAG_FROM_SLIDE) {
                Logger.i("onInterceptTouchEvent: move in drag mode");
                intercept = true;
            }
            break;
        case MotionEvent.ACTION_UP:
            if (mSelectState == SELECT_STATE_DRAG_FROM_NORMAL
                    || mSelectState == SELECT_STATE_DRAG_FROM_SLIDE) {
                intercept = true;
            }
            // fall through
        case MotionEvent.ACTION_CANCEL:
            // finger is lifted before moving
            Logger.i("onInterceptTouchEvent: finger is lifted before moving");
            if (mSlideStateStartPosition != RecyclerView.NO_POSITION) {
                selectFinished(mSlideStateStartPosition);
                mSlideStateStartPosition = RecyclerView.NO_POSITION;
            }
            // selection has triggered
            if (mSelectionRecorder.startPosition() != RecyclerView.NO_POSITION) {
                selectFinished(mSelectionRecorder.endPosition());
            }
            break;
        default:
            // do nothing
    }
    // Intercept only when the selection is triggered
    if (intercept) {
        Logger.i("will intercept event");
    }
    return intercept;
}

@Override
public boolean onInterceptTouchEvent(RecyclerView rv, MotionEvent e) {
   if (rv.getAdapter().getItemCount() == 0) {
       return false;
   }
   init(rv);

   int action = e.getAction();
   int actionMask = action & MotionEvent.ACTION_MASK;

   switch (actionMask) {
       case MotionEvent.ACTION_DOWN:
           // 记录按下时的坐标，避免在滚动区触发反向滚动
           mActionDownY = e.getY();
           break;
       case MotionEvent.ACTION_MOVE:
           // 此标志位在长按时激活拖动多选时设置
           if (mIsDragActive) {
               return true;
           }
           // 开启滑动多选模式时，会在拖动结束时进入滑动多选
           // 并且在定义的滑动区域内的事件才会拦截处理
           if (mIsSlideActive && isInSlideArea(e)) {
               activeSlideSelect(getItemPosition(rv, e));
               return true;
           }
           break;
   }

   // 其他情况则不拦截
   return false;
}
```

其中 `init()` 方法为初始化具体参数：

```Java
private void init(RecyclerView rv) {
   if (mHasInit) {// 只初始化一次
       return;
   }
   if (mRecyclerView == null) {
       mRecyclerView = rv;
   }
   int rvHeight = rv.getHeight();
   if (mHotspotHeight == -1f) { // 未设置滚动区的高度，采用（RV高度×比例）
       mHotspotHeight = rvHeight * mHotspotHeightRatio;
   } else { // 表明设置了滚动区的大小
       if (mHotspotHeight >= rvHeight / 2) {
           mHotspotHeight = rvHeight / 2;
       }
   }
   mTopRegionFrom = mHotspotOffset;
   mTopRegionTo = mTopRegionFrom + mHotspotHeight;
   mBottomRegionTo = rvHeight - mHotspotOffset;
   mBottomRegionFrom = mBottomRegionTo - mHotspotHeight;

   mHasInit = true;
}
```

在 `onInterceptTouchEvent()` 中对是否拦截进行处理后，具体的事件处理如下：

```Java
@Override
public void onTouchEvent(RecyclerView rv, MotionEvent e) {
   if (!isActive()) {
       return;
   }
   int action = e.getAction();
   int actionMask = action & MotionEvent.ACTION_MASK;

   switch (actionMask) {
       case MotionEvent.ACTION_MOVE:
           processAutoScroll(e);
           if (!mIsInTopHotspot && !mIsInBottomHotspot) {
               updateSelectedRange(rv, e);
           }
           break;
       case MotionEvent.ACTION_CANCEL:
       case MotionEvent.ACTION_UP:
           // 抬起手指时分成两种情况
           if (!mIsDisableSlide) {// 允许滑动模式
               mIsDragActive = false;
               mIsSlideActive = true;// 转换为滑动模式
               selectFinished();
           } else {// 退出多选模式
               mIsDragActive = false;
               mIsSlideActive = false;
               selectFinished();
           }
           break;
   }
}
```

### 选择范围的更新与回调

DragMultiSelect 按下时激活多选模式，记录此时的位置，与方案三相比，这里增加了按下时对该位置进行一次选择的调用：

```Java
public void activeDragSelect(int position) {
   mStart = position;
   mEnd = position;
   mLastStart = position;
   mLastEnd = position;
   if (!mIsDragActive && !mIsSlideActive) {
       mIsDragActive = true;
   }
   if (mSelectListener != null) {
       if (mSelectListener instanceof OnAdvancedDragSelectListener) {
           ((OnAdvancedDragSelectListener) mSelectListener).onSelectionStarted(position);
       }
       // 增加一次主动调用，方案三中 onSelectionStarted 要对第一条目进行处理，
       // 实际上第一条目的处理与其他条目处理是一致的
       mSelectListener.onSelectChange(position, true);
   }
}
```

自动滚动的处理同样是在 `processAutoScroll()` 中，但是增加了一个判断，以避免在滚动区中激活选择模式时触发了反向滚动，以在上滚动区为例：

```Java
// y < mActionDownY 增加此判断，在上滚动区向下滑动时不会触发上滚
if (y > mTopRegionFrom && y < mTopRegionTo && y < mActionDownY) {
   mLastX = e.getX();
   mLastY = e.getY();
   float scrollDistanceFactor = (y - mTopRegionTo) / mHotspotHeight;
   mScrollDistance = (int)(mMaxScrollDistance * scrollDistanceFactor);
   if (!mIsInTopHotspot) {
       mIsInTopHotspot = true;
       startAutoScroll();
       // 但如果在上滚动区向上滑动时要正常触发，此时将此值更新
       // 可以正常触发滚动与速度的更新
       mActionDownY = mTopRegionTo;
   }
}
```

选择范围的更新同样是在 `notifySelectRangeChange()` 中，其中具体的更新在原来方案三的实现中为：

```Java
private void notifySelectRangeChange() {
   // 省略代码……
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

而在接口的实现中，同样要调用 Adapter 的 selectRange 方法。前文说过，这里的参数实际上就是指一个状态，而且对于选择范围里的每一条目都要回调一次，那么就没有必要将这个 start 与 end 传递出去。故在这里我把 `mSelectListener.onSelectChange(newEnd + 1, mLastEnd, false)` 改成 `mSelectListener.onSelectChange(i, newState)`，而这只是进行了一下转换：

```Java
private void selectChange(int start, int end, boolean newState) {
   for (int i = start; i <= end; i++) {
       mSelectListener.onSelectChange(i, newState);
   }
}
```

这样子实际上最终 OnItemTouchListener 要实现的接口为：

```Java
public interface OnDragSelectListener {
   /**
    * @param position 此条目状态将转变为 new state
    * @param newState 新的状态
    */
   void onSelectChange(int position, boolean newState);
}

public interface OnAdvancedDragSelectListener extends OnDragSelectListener {
   /**
    * @param start 选择开始于此
    */
   void onSelectionStarted(int start);

   /**
    * @param end 选择结束于此
    */
   void onSelectionFinished(int end);
}
```

### DragSelectionProcessor

在 OnItemTouchListener 中进行修改之后，实际上 DragSelectionProcessor 的实现也显得更简洁一些：

```Java
@Override
public void onSelectionStarted(int start) {
   mOriginalSelection = new HashSet<>();
   Set<Integer> selected = mSelectionHandler.getSelection();
   if (selected != null) {
       mOriginalSelection.addAll(selected);
   }
   mFirstWasSelected = mOriginalSelection.contains(start);

   if (mStartFinishedListener != null) {
       mStartFinishedListener.onSelectionStarted(start, mFirstWasSelected);
   }
   // 选择开始时的回调只专心于获取当前的状态即可，对 start 条目无须进行处理
}

@Override
public void onSelectionFinished(int end) {
   mOriginalSelection = null;

   if (mStartFinishedListener != null) {
       mStartFinishedListener.onSelectionFinished(end);
   }
}

@Override
public void onSelectChange(int position, boolean newState) {
   // 省略代码……
   // 此处的实现与方案三几乎一致，具体可以查看代码
}

private void checkedUpdateSelection(int position, boolean newState) {
   if (mCheckSelectionState) {
       if (mSelectionHandler.isSelected(position) != newState ) {
           mSelectionHandler.updateSelection(position, newState);
       }
   } else {
       // 同样地将 updateSelection(int start, int end, boolean isSelected, boolean calledFromOnStart) 换成 updateSelection(position, newState) 更加地直观
       mSelectionHandler.updateSelection(position, newState);
   }
}
```

也就是说对应的 `ISelectionHandler` 只修改了一下 `updateSelection()` 方法的参数。

```Java
public interface ISelectionHandler {
   Set<Integer> getSelection();
   void updateSelection(int start, int end, boolean isSelected, boolean calledFromOnStart);
   boolean isSelected(int index);
}
```

到此为此，一个使用 OnItemTouchListener 实现 RecyclerView 拖动/滑动多选的功能的库就完成了。使用时按需要直接复制一个或两个类就好了，我就不搞什么 compile 之类的了，因为这个库能直接复制使用、定制修改，没必要搞复杂。最终，在这里放一个实现后的效果图吧，主要是看看拖动模式与滑动模式。

![DragMultiSelect](http://upload-images.jianshu.io/upload_images/2946447-8be11829176d07ec.gif?imageMogr2/auto-orient/strip)

正如前文所说，网格布局下由于长按拖动之后是要进行上下滚动的，所以在网格布局下就不要开启滑动选择模式了。具体的代码与使用方法请直接查看 [Mupceet/DragMultiSelect](https://github.com/Mupceet/DragMultiSelect) 吧。

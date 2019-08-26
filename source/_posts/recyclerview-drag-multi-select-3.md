---
title: RecyclerView 拖动/滑动多选的实现（3）
date: 2017-07-31 22:55:33
categories:
- 技术向
tags:
- Android
---

## 方案二：DragSelectRecyclerView
### 扩展的选择策略
之前提到，方案二是基于方案三进行扩展的，可以看到，在 `OnItemTouchListener` 这一块，两者其实几乎是一模一样的。而方案二一个很好的地方，就是在几乎不修改 DragSelectTouchListener 的前提下，对其选择功能进行了强大方便的扩展。下面我将从设计的思路出发，理一理是怎样完成的。

首先要清楚方案二扩展了哪些选择策略，总共有 4 种模式：

- Simple: 滑过时选中，往回滑时取消选中
- ToggleAndUndo: 滑过时反选，往回滑时恢复原状态
- FirstItemDependent: 反选按下的第一条目，滑过时与第一条目状态一致，往回滑时与第一条目状态相反
- FirstItemDependentToggleAndUndo: 反选按下的第一条目，滑过时与第一条目状态一致，往回滑时恢复原状态

<!--more-->

关于这 4 种模式的效果请看 GIF 图：

| Simple | ToggleAndUndo |
| :---: | :---: |
| ![](http://upload-images.jianshu.io/upload_images/2946447-8d96895d3148a146.gif?imageMogr2/auto-orient/strip)  | ![](http://upload-images.jianshu.io/upload_images/2946447-ee38d0edb5a09acd.gif?imageMogr2/auto-orient/strip) |
| FirstItemDependent | FirstItemDependentToggleAndUndo |
| ![](http://upload-images.jianshu.io/upload_images/2946447-287bf777d67f9ecc.gif?imageMogr2/auto-orient/strip) | ![](http://upload-images.jianshu.io/upload_images/2946447-ed26e2b06d46abcd.gif?imageMogr2/auto-orient/strip) |

第 1 种模式其实就是 Google Photos 的策略，而第 4 种策略与我需要实现的基本相同（感动地哭出声……）。看了效果之后，我们再想想基于方案三的一个回调 `onSelectChange(int start, int end, boolean isSelected)` 能完成吗？

首先可以知道 Simple 模式是可以做到的，因为这个模式下除了位置信息之外无需另外的信息。而另外三种都无法做到，因为它们都需要按下时列表目前的状态信息：

- ToggleAndUndo 需要知道按下时，哪些条目已经被选择了，这样子才能恢复原状态；
- FirstItemDependent 需要知道按下时的条目的原状态，才能反选第一条目；
- FirstItemDependentToggleAndUndo 需要知道的就包括前两者的信息：哪些条目被选择了、第一条目的原状态（事实上，这一信息包含在前一个信息里）。

那么，很自然地，在按下时也需要一个回调，以此为入口获取所需要的信息了。因此方案二先扩展了 `DragSelectTouchListener.OnDragSelectListener` 的接口，首先看一下对 `OnDragSelectListener` 这个接口的扩展：

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
方案二在扩展了 DragSelectTouchListener 后，将其实现封装了一层，把这 4 种模式放到一个控制器里：

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

具体的代码以及使用示例请直接查看 [MFlisar/DragSelectRecyclerView](https://github.com/MFlisar/DragSelectRecyclerView) 的 README 文档。至此，GitHub 上的三个库都分析完毕了，DragSelectRecyclerView 是完整度最好的，接下来是时候来撸一个自已的支持网格列表及常规列表的拖动、滑动多选的库了。

## DragMultiSelectRecyclerView
### 滚动区的定义
DragMultiSelectRecyclerView 的滚动区的定义与方案二一致：

![DragMultiSelectRecyclerView 滚动区](http://upload-images.jianshu.io/upload_images/2946447-ccdc1cf6cabd4815.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 自动滚动实现
DragMultiSelectRecyclerView 的自动滚动的实现与方案二、三是完全一致的，这里就不赘述。

### 触摸事件的处理
增加了滑动多选模式，具体的改动见代码中的注释。
```Java
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
DragMultiSelectRecyclerView 按下时激活多选模式，记录此时的位置，与方案二相比，这里增加了按下时对该位置进行一次选择的调用：

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
       // 增加一次主动调用，方案二中 onSelectionStarted 要对第一条目进行处理，
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

选择范围的更新同样是在 `notifySelectRangeChange()` 中，其中具体的更新在原来方案二的实现中为：
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
   // 此处的实现与方案二几乎一致，具体可以查看代码
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

![DragMultiSelectRecyclerView](http://upload-images.jianshu.io/upload_images/2946447-8be11829176d07ec.gif?imageMogr2/auto-orient/strip)

正如前文所说，网格布局下由于长按拖动之后是要进行上下滚动的，所以在网格布局下就不要开启滑动选择模式了。具体的代码与使用方法请直接查看 [Mupceet/DragMultiSelectRecyclerView](https://github.com/Mupceet/DragMultiSelectRecyclerView) 吧。

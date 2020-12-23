---
title: Flutter 中的事件流和手势
date: 2020-08-28 22:37:33
categories:
- 技术向
tags:
- Android
- Flutter
---

## 事件的来源

我们知道，Android 系统中，控件对手势事件的处理在 View 的 `onTouchEvent` 方法中，Flutter 也不例外，从 io.flutter.view.FlutterView 的这个方法开始，大致流程为将事件从 Java 转 C++ 再转 Dart 进入框架的处理流程。这里简单画个图，不详细展开。

### Android -> Engine (Java -> C++)

![Android -> Engine](https://raw.githubusercontent.com/Mupceet/article-piture/master/flutter-inputevent-android-to-engine.svg)

<!--more-->

### Engine -> Flutter (C++ -> Dart)

![Engine -> Flutter](https://raw.githubusercontent.com/Mupceet/article-piture/master/flutter-inputevent-engine-to-flutter.svg)

事件从 android 传到 flutter 中执行了 5 次转换：

1. Android 中，从 MotionEvent 中取出事件，并保存在 ByteBuffer 中
1. Engine 中，将 ByteBuffer 转成 PointerDataPacket（类对象）
1. Engine 中，为了传递给 Dart，将 PointerDataPacket 转成 buffer
1. Dart 中，将 buffer 再转成 PointerDataPacket（类对象）
1. Dart 中，将 PointerData 转成 PointerEvent，供上层使用

## 手势框架流程

![手势框架流程](https://raw.githubusercontent.com/Mupceet/article-piture/master/flutter-gesture-framework-process.svg)

Flutter 框架中手势的事件源头在 gestures/binding.dart 里的 GestureBinding 类里，框架会把事件传给 `GestureBinding._handlePointerDataPacket` 方法，查看该方法：

```dart
void _handlePointerDataPacket(ui.PointerDataPacket packet) {
  // We convert pointer data to logical pixels so that e.g. the touch slop can be
  // defined in a device-independent manner.
  _pendingPointerEvents.addAll(PointerEventConverter.expand(packet.data, window.devicePixelRatio));
  if (!locked)
    _flushPointerEventQueue();
}

void _flushPointerEventQueue() {
  assert(!locked);
  while (_pendingPointerEvents.isNotEmpty)
    _handlePointerEvent(_pendingPointerEvents.removeFirst());
}

void _handlePointerEvent(PointerEvent event) {
  assert(!locked);
  HitTestResult hitTestResult;
  if (event is PointerDownEvent || event is PointerSignalEvent) {
    assert(!_hitTests.containsKey(event.pointer));
    hitTestResult = HitTestResult();
    hitTest(hitTestResult, event.position);
    if (event is PointerDownEvent) {
      _hitTests[event.pointer] = hitTestResult;
    }
    assert(() {
      if (debugPrintHitTestResults)
        debugPrint('$event: $hitTestResult');
      return true;
    }());
  } else if (event is PointerUpEvent || event is PointerCancelEvent) {
    hitTestResult = _hitTests.remove(event.pointer);
  } else if (event.down) {
    // Because events that occur with the pointer down (like
    // PointerMoveEvents) should be dispatched to the same place that their
    // initial PointerDownEvent was, we want to re-use the path we found when
    // the pointer went down, rather than do hit detection each time we get
    // such an event.
    hitTestResult = _hitTests[event.pointer];
  }
  assert(() {
    if (debugPrintMouseHoverEvents && event is PointerHoverEvent)
      debugPrint('$event');
    return true;
  }());
  if (hitTestResult != null ||
      event is PointerHoverEvent ||
      event is PointerAddedEvent ||
      event is PointerRemovedEvent) {
    dispatchEvent(event, hitTestResult);
  }
}
```

主要流程体现在 `_handlePointerEvent` 方法中。其中的重点为：

1. `HitTestResult` 对象：当事件为 PointerDownEvent 事件时，会新建一个 HitTestResult 对象，并通过 `hitTest` 方法更新其中的 `path` 变量，该变量即为后续分发处理事件时的控件列表，也就是事件传递所经过的的控件节点。如果是其他事件，就直接使用原来的 HitTestResult 进行事件分发。
1. `dispatchEvent` 事件分发：逐一调用 `HitTestResult` 的控件列表的 `handleEvent` 方法对事件进行处理。

### HitTestResult 的更新

上面提到创建的 `HitTestResult` 对象的更新在 `hitTest` 中，那具体来看看它是如何判断并更新的。

`hitTest` 是 `HitTestable` 抽象类的方法，而所有实现 `HitTestable` 的类有 **`GestureBinding` 和 `RendererBinding`**，首先看下哪个类混合了这两个类，我们可以找到实际的类为 `WidgetsFlutterBinding` 这个类。

```dart
class WidgetsFlutterBinding extends BindingBase with GestureBinding, SchedulerBinding, ServicesBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {

  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```

`WidgetsFlutterBinding` 这个类 mixin 了这两个类，而由于 mixin 语法规则的调用顺序，调用 `hitTest` 方法会先调用 `RndererBinding` 的 `hitTest` 方法。

#### RendererBinding#hitTest

```dart
void hitTest(HitTestResult result, Offset position) {
  assert(renderView != null);
  renderView.hitTest(result, position: position);
  super.hitTest(result, position);
}
```

这里先调用 renderView 的 hitTest 方法，然后再调用 GestureBinding 的 hitTest 方法。我们看下这个 renderView 是什么时候设置的，查看它初始化的地方：

```dart
void initRenderView() {
  assert(renderView == null);
  renderView = RenderView(configuration: createViewConfiguration(), window: window);
  renderView.prepareInitialFrame();
}

@override
void initInstances() {
  super.initInstances();
  // 省略
    initRenderView();
  // 省略
}
```

而 `initInstances()` 是在基类的构造方法中调用：

```dart
BindingBase() {
  // 省略
  initInstances();
  // 省略
}
```

刚才 `WidgetsFlutterBinding` 这个类中，我们看到 `ensureInitialized()` 这个方法，这个方法中调用了构造方法。而再往上看，就可以看到，这正是在应用运行时调用的 runApp 所调用的。当我们跑起整个 Flutter 应用时：

```dart
void main() {
  runApp(new MyApp());
}

void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

查看 `renderView` 的 `hitTest` 方法，它将调用 `child` 的 `hitTest` 方法尝试把子控件添加到 `HitTestResult` 中，然后把自己添加进去。

```dart
bool hitTest(HitTestResult result, { Offset position }) {
  if (child != null)
    child.hitTest(BoxHitTestResult.wrap(result), position: position);
  result.add(HitTestEntry(this));
  return true;
}
```

而查看如下所示的 `child` 的 `hitTest` 方法源码，它通过 `_size.contains` 判断自己是否属于响应区域，确认响应后执行 `hitTestChildren` 和 `hitTestSelf` ，尝试添加下级的 child ，然后尝试添加自己，这样的**递归调用就让我们自下而上的得到了一个 `HitTestResult` 的列表。**所以 HitTestResult 中的路径顺序为：
`叶子节点 --> 父节点 --> 根节点`。

```dart
bool hitTest(BoxHitTestResult result, { @required Offset position }) {
  // 省略
  if (_size.contains(position)) {
    if (hitTestChildren(result, position: position) || hitTestSelf(position)) {
      result.add(BoxHitTestEntry(this, position));
      return true;
    }
  }
  return false;
}
```

#### GestureBinding#hitTest

```dart
@override
void hitTest(HitTestResult result, Offset position) {
  result.add(HitTestEntry(this));
}
```

GestureBinding 中的处理非常简单，就是将自身添加到 HitTestResult 中。因此，最终 HitTestResult 的完整路径顺序为：

`叶子节点 --> 父节点 --> 根节点 --> GestureBinding`

举个例子如下，其结构和结果如图所示：

![控件树示例](https://raw.githubusercontent.com/Mupceet/article-piture/master/flutter-render-tree-example.webp)

此图中的列表结果省略了一个 `GestureBinding` 对象，它位于列表的末尾。

### 事件分发

更新好 HitTestResult 后，dispatchEvent 中将遍历调用 path 上的 HitTestEntry 的 handleEvent 方法。

```dart
void dispatchEvent(PointerEvent event, HitTestResult hitTestResult) {
  assert(!locked);
  // No hit test information implies that this is a hover or pointer
  // add/remove event.
  if (hitTestResult == null) {
    assert(event is PointerHoverEvent || event is PointerAddedEvent || event is PointerRemovedEvent);
    try {
      pointerRouter.route(event);
    } catch (exception, stack) {
      // 省略
    }
    return;
  }
  for (final HitTestEntry entry in hitTestResult.path) {
    try {
      entry.target.handleEvent(event.transformed(entry.transform), entry);
    } catch (exception, stack) {
      // 省略
    }
  }
}
```

由于这个 Path 是顺序是从叶子节点到 GestureBinding，所以这里会先把事件交给叶子控件先处理，直到最后把事件交给 GestureBinding 调用 handleEvent 方法。

#### Listener

叶子控件实际指的就是具体的控件的处理，大部分控件没有特别的处理，相当于只是提供了一个入口供各个控件方便地进行拦截处理。不过其中有一个控件是专门用来处理手势事件的，即带有 `RenderPointerListener` (RenderObject) 的 `Listener` (Widget) ，它会在 `handleEvent` 中对事件进行回调处理：

```dart
  @override
  void handleEvent(PointerEvent event, HitTestEntry entry) {
    assert(debugHandleEvent(event, entry));
    if (onPointerDown != null && event is PointerDownEvent)
      return onPointerDown(event);
    if (onPointerMove != null && event is PointerMoveEvent)
      return onPointerMove(event);
    if (onPointerUp != null && event is PointerUpEvent)
      return onPointerUp(event);
    if (onPointerCancel != null && event is PointerCancelEvent)
      return onPointerCancel(event);
    if (onPointerSignal != null && event is PointerSignalEvent)
      return onPointerSignal(event);
  }
```

其中这些回调方法都是创建 `Listener` 这个 Widget 时传入的。即如果要监听处理手势事件，只需要给 Child 套上一个 Listener，这就类似于 Android 中 View 的 setOnTouchListener。

#### GestureBinding

各控件处理完 handleEvent 之后，再来看下 GestureBinding 的 handleEvent 方法：

```dart
void handleEvent(PointerEvent event, HitTestEntry entry) {
  pointerRouter.route(event);
  if (event is PointerDownEvent) {
    gestureArena.close(event.pointer);
  } else if (event is PointerUpEvent) {
    gestureArena.sweep(event.pointer);
  } else if (event is PointerSignalEvent) {
    pointerSignalResolver.resolve(event);
  }
}
```

从名称可以知道，这里的重点有两个，一个是 PointerRouter ，另一个是 GestureArena。

我们查看 route 方法，可以发现，其中重点是遍历 _routeMap 并再次对事件进行分发。

```dart
void route(PointerEvent event) {
  final Map<PointerRoute, Matrix4> routes = _routeMap[event.pointer];
  final Map<PointerRoute, Matrix4> copiedGlobalRoutes = Map<PointerRoute, Matrix4>.from(_globalRoutes);
  if (routes != null) {
    _dispatchEventToRoutes(
      event,
      routes,
      Map<PointerRoute, Matrix4>.from(routes),
    );
  }
  _dispatchEventToRoutes(event, _globalRoutes, copiedGlobalRoutes);
}
```

我们需要先知道这个 _routeMap 是什么添加的。可以看到，它是通过 pointer_router.dart 的 addRoute 方法进行添加的：

```dart
void addRoute(int pointer, PointerRoute route, [Matrix4 transform]) {
  final Map<PointerRoute, Matrix4> routes = _routeMap.putIfAbsent(
    pointer,
    () => <PointerRoute, Matrix4>{},
  );
  assert(!routes.containsKey(route));
  routes[route] = transform;
}
```

一路往上查看调用链，可以看到是：

```dart
/// pointer_router.dart
pointerRouter.addRoute <--

/// recognizer.dart
GestureRecognizer.startTrackingPointer <--
GestureRecognizer.addAllowedPointer <--
GestureRecognizer.addPointer <--

/// gesture_detector.dart
RawGestureDetectorState._handlePointerDown

void _handlePointerDown(PointerDownEvent event) {
  assert(_recognizers != null);
  for (final GestureRecognizer recognizer in _recognizers.values)
    recognizer.addPointer(event);
}
```

有这样的一个控件 RawGestureDetectorState，它内部添加了一个 Listener，它利用 Listener 会优先处理 handleEvent 从而在 Listener 的 onPointerDown 中按顺序把一系统的 GestureRecognizer 添加到 Router 中去。

因此，GestureBinding 的 handleEvent 中通过路由将事件转发的是给到这样的一批 GestureRecognizer  对象。很显然，我们需要进一步查看这个 GestureRecognizer ，以及另一个概念 GestureArena。

## GestureDetector

我们根据 gesture_detector.dart 的名称，可以定位到这样一个特殊的控件：GestureDetector。

> The GestureDetector widget decides which gestures to attempt to recognize based on which of its callbacks are non-null.

根据文档所说 GestureDetector 控件可以检测手势，并且根据手势调起相应回调。

```dart
GestureDetector({
    Key key,
    this.child,
    this.onTapDown,
    this.onTapUp,
    this.onTap,
    this.onTapCancel,
    this.onDoubleTap,
    this.onLongPress,
    this.onLongPressUp,
    this.onLongPressDragStart,
    this.onLongPressDragUpdate,
    this.onLongPressDragUp,
    this.onVerticalDragDown,
    this.onVerticalDragStart,
    this.onVerticalDragUpdate,
    this.onVerticalDragEnd,
    this.onVerticalDragCancel,
    this.onHorizontalDragDown,
    this.onHorizontalDragStart,
    this.onHorizontalDragUpdate,
    this.onHorizontalDragEnd,
    this.onHorizontalDragCancel,
    this.onForcePressStart,
    this.onForcePressPeak,
    this.onForcePressUpdate,
    this.onForcePressEnd,
    this.onPanDown,
    this.onPanStart,
    this.onPanUpdate,
    this.onPanEnd,
    this.onPanCancel,
    this.onScaleStart,
    this.onScaleUpdate,
    this.onScaleEnd,
    this.behavior,
    this.excludeFromSemantics = false,
    this.dragStartBehavior = DragStartBehavior.down,
  }) 
```

GestureDector 支持了相当多的手势，那它是如何检测手势的呢？

### 手势识别器（GestureRecognizer）

查看GestureDector 的  build 方法：

```dart
Widget build(BuildContext context) {
  final Map<Type, GestureRecognizerFactory> gestures = <Type, GestureRecognizerFactory>{};
  // 根据监听的手势，按一定的顺序创建对应的
  // gestures Map

  return RawGestureDetector(
    gestures: gestures,
    behavior: behavior,
    excludeFromSemantics: excludeFromSemantics,
    child: child,
  );
}
```

可以看到 GestureDector 其实就是根据注册的回调，按一定的顺序创建对应的手势识别器，并全传递到 RawGestureDetector。而 RawGestureDetector 的 build 方法如下：

```dart
Widget build(BuildContext context) {
  Widget result = Listener(
    onPointerDown: _handlePointerDown,
    behavior: widget.behavior ?? _defaultBehavior,
    child: widget.child,
  );
  if (!widget.excludeFromSemantics)
    result = _GestureSemantics(
    child: result,
    assignSemantics: _updateSemanticsForRenderObject,
  );
  return result;
}
```

关键在于 _handlePointerDown 方法，这个上面已经提到过了：

```dart
void _handlePointerDown(PointerDownEvent event) {
  assert(_recognizers != null);
  for (GestureRecognizer recognizer in _recognizers.values)
    recognizer.addPointer(event);
}
```

它会遍历 _recognizers（手势识别器）调用 addPointer 方法，

```dart
void addPointer(PointerDownEvent event) {
  _pointerToKind[event.pointer] = event.kind;
  if (isPointerAllowed(event)) {
    addAllowedPointer(event);
  } else {
    handleNonAllowedPointer(event);
  }
}
```

我们看下识别器的继承图：

![识别器的继承图](https://raw.githubusercontent.com/Mupceet/article-piture/master/flutter-gesture-class-graph.svg)

几个常用的识别器都是继承自 PrimaryPointerGestureRecognizer ，查看它们的实现：

```dart
void addAllowedPointer(PointerDownEvent event) {
  startTrackingPointer(event.pointer, event.transform);
  if (state == GestureRecognizerState.ready) {
    state = GestureRecognizerState.possible;
    primaryPointer = event.pointer;
    initialPosition = OffsetPair(local: event.localPosition, global: event.position);
    if (deadline != null)
      _timer = Timer(deadline, () => didExceedDeadlineWithEvent(event));
  }
}
```

到这里先理一下流程：当确定 PointerDown 事件落在 GestureDector 控件下的子组件时，在 GestureDector 上注册的 GesutreRecognizer 就会追踪这个 pointer（就是我们的手指），先看 startTrackingPointer 方法：

```dart
void startTrackingPointer(int pointer, [Matrix4 transform]) {
  GestureBinding.instance.pointerRouter.addRoute(pointer, handleEvent, transform);
  _trackedPointers.add(pointer);
  assert(!_entries.containsValue(pointer));
  _entries[pointer] = _addPointerToArena(pointer);
}
```

到这里就在手指按下时把对应的识别器添加到 PointerRouter 中，后续 GestureBinding 的 handleEvent 的事件转发就会把事件传递给识别器进行处理。

这里还有一个 _addPointerToArena 方法，我们又要引入一个重要的概念：GestureArena。

### 手势竞技场（GestureArena）

查看 GestureRcognizer 的 _addPointerToArena 方法

```dart
GestureArenaEntry _addPointerToArena(int pointer) {
  if (_team != null)
    return _team.add(pointer, this);
  return GestureBinding.instance.gestureArena.add(pointer, this);
}
```

这里 GestureBinding.instance.gestureArena 其实就是 GestureArenaManager，它用来管理 GestureArena 。查看它的 add 方法：

```dart
GestureArenaEntry add(int pointer, GestureArenaMember member) {
  final _GestureArena state = _arenas.putIfAbsent(pointer, () {
    assert(_debugLogDiagnostic(pointer, '★ Opening new gesture arena.'));
    return _GestureArena();
  });
  state.add(member);
  assert(_debugLogDiagnostic(pointer, 'Adding: $member'));
  return GestureArenaEntry._(this, pointer, member);
}
```

这里就是新建了一个 GestureArenaEntry 对象，好吧，我们得整理一下他们的关系：

```dart
class GestureArenaManager {
    final Map<int, _GestureArena> _arenas = <int, _GestureArena>{};
}

class _GestureArena {
    final List<GestureArenaMember> members = <GestureArenaMember>[];
}

class OneSequenceGestureRecognizer extends GestureArenaMember {
    final Map<int, GestureArenaEntry> _entries = <int, GestureArenaEntry>{};
}
```

到这里，我们已经看到了 GestureDetector 中的识别器是在 PointerDown 事件来时，统一添加到竞技场中的。

再回到事件分发流程，再次列出GestureBinding 的 handleEvent 方法：

```dart
void handleEvent(PointerEvent event, HitTestEntry entry) {
  pointerRouter.route(event);
  if (event is PointerDownEvent) {
    gestureArena.close(event.pointer);
  } else if (event is PointerUpEvent) {
    gestureArena.sweep(event.pointer);
  } else if (event is PointerSignalEvent) {
    pointerSignalResolver.resolve(event);
  }
}
```

由于前面的流程，在 PointerDown 时，各个识别器已经添加到 Router 中，部分识别器本身并没有处理这个 PointerDown 事件，这里会再次把这个事件分发给识别器。

### 识别器与竞技场的关系

我们先放下识别器的具体的手势识别流程，先形象地描述下识别器和竞技场的概念与关系：

1. 首先我们有一批竞技选手（各种 Recognizer），我们也可能会有好几个竞技场地（_GestureArena），我们的场地管理员（GestureArenaManager）会根据 Pointer 的多少来构建场地，但是各个选手也要拿到每个竞技场的入场券（GestureArenaEntry）才能入场与其他选手一较高下。
1. 当我们的选手拿着对应的入场券进场（PointerDown）后，现在各个场地都聚集了一批选手。各个选手先进行准备动作（先处理 PointerDown 事件），然后竞技场地入口关闭。
1. 关闭竞技场的时候看一眼，是否仅有一个选手，如果是的话，那也就不要竞争了，就是你了。
1. 大家各凭本身竞争（各识别器分别处理各个事件），竞争过程中，如果有选手胜利（acceptGesture），则会通知其他选手退场（rejectGesture）。
1. 叮的一声（PointerUp 事件）竞技彻底结束，我们就要打扫竞技场，如果之前没有人胜利，那也要确定最终的胜者了。
1. 当选手申请退出时，在其退出后，会尝试找出是否场上仅剩一名选手了，如果仅剩一名选手，那他就是最终胜者。
1. 当选手申请胜利时，那大家都收到通知，有人赢了，那各自退场，由这个选手获取胜利。

从这个流程中，我们可以看到这个竞争是非常地绅士的，那就会有这样的问题：如果没有人申请胜利的话，那这个竞技场就不关闭了吗？大家都不用下班的吗？

根据上面提到的打扫竞技场的逻辑，如果 PointerUp 事件来临，就要确认出一个胜利者来了。

1. 如果竞技场之前有人举手（hold），那这时就给个面子，表示稍后再来打扫。
1. 否则的话，就宣布第一个选手为胜者，其他人只能失败了。

那个举手是怎么回事呢？例如一个 DoubleTap 和一个 Tap，DoubleTap 手势可以在处理抬手时请求竞技场 Hold 住，但是请求竞技场 hold 住的手势，必须之后主动请求竞技场 release，等 DoubleTap 手势决定是否是优胜还是自动退出，就可以知道 Tap 手势是否最终生效。

那这里就有一个前提，双击的识别器一定要先于单击识别器对事件进行处理，否则，他还没有举手，单击识别器就会先申明取得胜利了。

整个手势竞技场只是一个君子竞争的地方，是为了友好地通知各识别器，应该何时处理与其他识别器的关系。实际中，在处理事件的过程中，各识别器可能已经开始干活了。

此外，这里简单介绍一下 GestureRecognizer 的几个状态：

- ready 初始状态准备好识别手势
- possible 开始追踪 pointer，不停接收路由的事件，如果停止追踪，则吧状态转回 ready
- defunct 手势已经被决议（accepted 或者 rejected）

## 手势识别举例：Tap

以 TapGestureRecognizer 为例，它的父类是 PrimaryPointerGestureRecognizer，父类的 handleEvent 处理如下，即如果 Move 得超过了一定限度，就认为已经不是这类手势了。

```dart
void handleEvent(PointerEvent event) {
  assert(state != GestureRecognizerState.ready);
  if (state == GestureRecognizerState.possible && event.pointer == primaryPointer) {
    final bool isPreAcceptSlopPastTolerance =
      !_gestureAccepted &&
      preAcceptSlopTolerance != null &&
      _getGlobalDistance(event) > preAcceptSlopTolerance;
    final bool isPostAcceptSlopPastTolerance =
      _gestureAccepted &&
      postAcceptSlopTolerance != null &&
      _getGlobalDistance(event) > postAcceptSlopTolerance;

    if (event is PointerMoveEvent && (isPreAcceptSlopPastTolerance || isPostAcceptSlopPastTolerance)) {
      resolve(GestureDisposition.rejected);
      stopTrackingPointer(primaryPointer);
    } else {
      handlePrimaryPointer(event);
    }
  }
  stopTrackingIfPointerNoLongerDown(event);
}
```

```dart
void handlePrimaryPointer(PointerEvent event) {
  if (event is PointerUpEvent) {
    _up = event;
    _checkUp();
  } else if (event is PointerCancelEvent) {
    resolve(GestureDisposition.rejected);
    if (_sentTapDown) {
      _checkCancel(event, '');
    }
    _reset();
  } else if (event.buttons != _down.buttons) {
    resolve(GestureDisposition.rejected);
    stopTrackingPointer(primaryPointer);
  }
}
```

这里并不处理 Down 事件，那我们看下 Down 事件的回调是怎么来的？

我们可以看到 _checkDown 方法会调用 Down 事件的回调。

```dart
void _checkDown() {
  if (_sentTapDown) {
    return;
  }
  handleTapDown(down: _down);
  _sentTapDown = true;
}
```

因此，查看调用它的地方，就知道了。总共有两个地方，一个是超时后调用，另一个是在 acceptGesture 中。

```dart
@override
void didExceedDeadline() {
  _checkDown();
}

@override
void acceptGesture(int pointer) {
  super.acceptGesture(pointer);
  if (pointer == primaryPointer) {
    _checkDown();
    _wonArenaForPrimaryPointer = true;
    _checkUp();
  }
}
```

我们结合实际的场景来说明：

1. 没有其他识别器：前面提到，Down 时除了会把识别器添加到竞技场中，还会关闭竞技场，由于此时仅有一个识别器，那就会直接调用这个识别器的 acceptGesture 方法，也就是说，这种情况下，会直接回调 handleTapDown 方法。
1. 有其他识别器：仔细查看这里有个超时时间调用的方法，也就是说当我们按下的时间超过 100 毫秒 TapGestureRecognizer.didExceedDeadline 就会调用接着调起 _checkDown 方法，也就是说，有可能延迟 100 毫秒触发 onTapDown。
1. 如果在 100ms 内就抬手，就由前面所说根据打扫竞技场的规则由第一个识别器获得手势。由于添加手势识别器时，点击识别器总是第一个添加的，所以也会正常回调 onTapDown，并且马上回调 onTapUp。正常抬手就会在 handlePrimaryPointer 里回调 onTapUp。

## 参考文章

> 1. [Flutter 中的事件流和手势简析](https://segmentfault.com/a/1190000011555283)
> 1. [十三、全面深入触摸和滑动原理](https://wizardforcel.gitbooks.io/gsyflutterbook/content/Flutter-13.html)
> 1. [Flutter之事件处理](https://juejin.im/post/6844903919563309064)
> 1. [Flutter 事件分发](https://juejin.im/post/6844904083371851790)

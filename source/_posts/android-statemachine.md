---
title: Android StateMachine
categories:
  - 技术向
tags:
  - Android
date: 2019-09-22 19:07:33
updated: 2020-02-08 02:14:33
---
在过去 CPU 非常昂贵的时候，通常通过设定任务让其不间断执行以提高效率，而在今天几乎所有的计算机在大多数时间都是空闲的，运行的是事件驱动系统（Event Driven System）亦或叫响应式系统（Reactive System）。可以回忆下常听到的“中断（Interrupt）”一词。

响应式系统通常没有开始与结束的概念，而是始终运行着，只在处理事件（Event）时才获得控制权，实际执行的动作由事件与所处的状态（State）共同决定，而当前的状态与过去的事件顺序相关，因此，整个系统都是由事件驱动的。事件很大程度上以不可预测的顺序和时间到达，因此软件每次运行时调用的代码的路径很可能不同。

## 有限状态机

状态机模型明确地声明了事件的处理依赖于**事件**与**状态**。当然你也可以使用条件语句和标志实现状态记录与事件响应，但这其中的可维护性与复杂度可想而知。状态机通过将状态和行为封装在一起，解决庞大分支语句带来的阅读性差和不方便扩展的问题，降低程序的复杂性，提升灵活性。

我们所说的状态机实际上是指 FSM（Finite State Machine，有限状态机），它表示有限个状态以及在这些状态下执行动作和状态转移等行为的模型。FSM 可分为两种类型：

- Moore Machine：状态机的**输出只与当前状态相关**，与当前的输入无关
- Mealy Machine：状态机的**输出与当前状态和当前输入有关**

<!--more-->

举例说明 FSM 及其两种类型的区别。游戏中士兵的行为状态有`巡逻`、`追击敌人`、`攻击敌人`、`逃跑`等；响应的事件有`发现敌人`、`追到敌人`、`敌人逃跑`、`敌人死亡`、自己`血量不足`等。这些状态的转移及对事件的响应如下表所示。

| **当前状态** | **当前输入** | **转移状态** | **输出（Mealy）** | **输出（Moore）** |
| --- | --- | --- | --- | --- |
| 巡逻 | 发现敌人 | 追击敌人 | 增加功劳 10 | 体力值下降 5 |
| 追击敌人 | 追到敌人 | 攻击敌人 | 增加功劳 20 | 体力值下降 20 |
| 追击敌人 | 敌人死亡 | 巡逻 | 增加功劳 0 | 体力值下降 20 |
| 攻击敌人 | 敌人死亡 | 巡逻 | 获得金钱 100 | 敌方掉血 100 |
| 攻击敌人 | 血量不足 | 逃跑 | 扣除功劳 100 | 敌方掉血 100 |

从表格中可以看出，FSM 分类依据中的**输出**不是指某个状态的输出，而是**状态机系统对外的输出**。状态机本身状态的转移与状态机当前状态和当前输入都是相关的。

实际上，系统中常使用混合状态机，即状态机既包含了 Moore 状态机也包含了 Mealy 状态机。仅部分单一功能的设计采用单纯的状态机类型（如：检测 1001 字符串）。其中由于 Moore 状态机中状态的增加与事件处理的修改对其输出影响不大，所以它具备安全可扩展的特点，因此混合状态机需要尽可能地 Moore 化。

用代码实现的思路描述 FSM 则是：不同的状态下都有一个 handleEvent 的方法供调用处理输入的不同的 Event，运行时 State 将根据事件输入进行转换，由于所处状态的不同，导致了即使向系统输入同一事件，它表现出来的行为都不一样。这也正是状态模式（State Pattern）的含义。

## 分层状态机

FSM 存在一个问题，随着系统复杂度增加，状态机的管理将越来越繁复。例如，在上面游戏系统的例子中加入一个用户退出事件，就需要在每个状态中增加退出事件的处理逻辑，而且处理逻辑可能相同，都是保存当前相关数据这一逻辑。实际上，大部分现实系统都是复杂度足够高的响应式系统，仅采用 FSM 显得力不从心。

如何改进呢？我们可以站在巨人的肩膀上进行思考，回想一下所知道的 GUI 系统是怎么处理用户输入事件的。例如 Android 系统，它处理触摸事件使用的是事件分发机制，将事件分发到子控件处理，如果子控件未处理则事件流回到父控件，重复此流程直至最高处理级别。整个处理逻辑体现了一种分层的结构设计，事件始终被尝试处理着，该设计模式是典型的责任链模式（Interator Pattern）

将这种事件处理的分层结构引入 FSM 则产生了 HFSM（Hierarchical Finite State Machines，分层状态机）。典型结构如下图所示。

![HFSM](https://barrgroup.com/images/articles/IntroHierarchicalStateMachines01UmlNestedStates.gif#align=left&display=inline&height=135&originHeight=135&originWidth=600&status=done&width=600)

1. **嵌套**：如果系统处于状态 s11（称为子状态），它也隐式地处于状态 s1（父状态）中。 此状态机将使用状态 s11 尝试处理事件。如果状态 s11 没有规定如何处理事件，则事件将交由更高级别的父状态 s1 进行处理。
1. **复用**：在 s1 中增加另一嵌套状态 s12 如 (b) 所示，两个子状态之间可以通过父状态复用的方式处理同一事件。

HFSM 最重要的设计为父子状态的引入。新定义状态可**继承**父状态的行为逻辑，因此只需要定义与现有状态的差异来快速引入新的状态，并且可以**复写**对事件的处理来**覆盖或扩展**父状态的事件处理。

HFSM 另一重要设计是状态的进入（Enter）与退出（Exit）操作。Enter 与 Exit 操作与状态相关，而与其转移条件无关，该设计即 Moore 状态机的表现。通常 Enter 与 Exit 用于状态资源的初始化与释放，其执行顺序为**从父状态往下进行初始化，退出则以相反顺序执行**，这个设计可与**面向对象编程中的构造和销毁**对比。该设计的典型应用为状态间对系统稀有资源的复用，仅需要通过这两个方法进行使用与释放。

## Android HFSM 使用说明

Android 系统源码中提供了 HFSM 的实现，源码路径为：sources/android-[version]/com/android/internal/util

- IState.java
- State.java
- StateMachine.java

### IState

```java
public interface IState {
    static final boolean HANDLED = true;
    static final boolean NOT_HANDLED = false;

    void enter();
    void exit();
    boolean processMessage(Message msg);

    String getName(); // for debugging
}
```

这是 HFSM 中每个状态的典型接口：

- `processMessage` 方法：状态机中任一时刻只执行一个 `processMessage` 方法，因此不需要同步锁，但需要尽快完成处理，否则后续事件都不会被处理。
- `enter/exit` 方法：状态的进入与退出，分别用于执行该状态的初始化和清理。
- `getName` 方法：获取状态的名称用于调试。默认返回类名，如果状态会有多个实例，最好让 `getName` 返回实例名称。

### State

```java
public class State implements IState {
　　protected State() {}
　　public void enter() {}
　　public void exit() {}
　　public boolean processMessage(Message msg) {
        return false;
    }
    public String getName() {
        String name = getClass().getName();
        int lastDollar = name.lastIndexOf('$');
        return name.substring(lastDollar + 1);
    }
}
```

这个是 ISate 的默认实现：

- `processMessage` 方法：默认不处理任何消息。
- `getName` 方法：默认返回类名，如果状态会有多个实例，最好让 `getName` 返回实例名称。

一般情况下，StateMachine 中的状态选择继承 State 而不用直接实现接口。其实我觉得 State 类设计成抽象类更合适。

### StateMachine

首先我们看下 StateMachine 的构造函数中的初始化方法，它们最终调用了以下方法：

```java
private void initStateMachine(String name, Looper looper) {
    mName = name;
    mSmHandler = new SmHandler(looper, this);
}
```

可以看出，状态机内部有一个 Handler 来处理消息，这个 Handler 可以在创建时传入，也可以内部自动创建子线程来构建。因此，我们可以知道，状态机消息处理时是处在 Handler 所在的线程中，这一点在主动传入外部 Handler 时要特别注意，不要因为外部的消息处理阻塞了状态机的消息处理。

更进一步，我们看以下这些状态机的公共方法的处理，可以发现，所有真正的执行动作都是在其内部的 Handler 中，与调用方法的上下文无关。因此，如果清楚 StateMachine 处理消息的线程，就不用担心它的处理会阻塞调用线程。

```java
public class StateMachine {
    // 初始化
    public final void addState(State state) {}
    public final void addState(State state, State parent) {}
    public final void setInitialState(State initialState) {}
    public void start() {}

    // 发送消息
    public final Message obtainMessage() {} // 一系列同名方法
    public void sendMessage(Message msg) {} // 一系列同名方法
    public void sendMessageDelayed(Message msg, long delayMillis) {} // 一系列同名方法
    public final void deferMessage(Message msg) {}
    protected final void sendMessageAtFrontOfQueue(Message msg) {} // 一系列同名方法

    // 状态操作
    public final void transitionTo(IState destState) {}
    public final void removeState(State state) {}
    public final void transitionToHaltingState() {}

    // 退出
    public final void quit() {}
    public final void quitNow() {}

    // 其他
    public final Handler getHandler() {}
    public final IState getCurrentState() {}
    public final Message getCurrentMessage() {}
}
```

我们先把 public 方法进行分类，分别说明。

#### 初始化相关方法

创建状态机时，`addState` 用于构建层次结构，`setInitialState` 用于标识初始状态。构造完成后，调用 `start`，完成初始化并启动状态机。正如 HFSM 设计，进入操作将逐一调用状态链的 `enter` 方法直到进入初始状态，等待消息到来进行处理。

**注意：不要在调用 `start` 之前给状态机发送消息，否则会导致崩溃。**

#### 发送消息相关方法

状态机启动后，可通过 `obtainMessage` 方法创建 Message 再 `sendMessage` 到状态机中，或者直接 `sendMessage` 发送消息到状态机中，状态机将调用当前状态的 `processMessage` 进行处理。而正如 HFSM 设计，如果当前状态未能处理该消息（returen NOT_HANDLED），那么在它存在父状态时，则将该消息交由父状态进行处理。
另外，还可通过 `defferMessage` 发送一个延缓处理的消息，该消息将在**状态转移后立即处理**。因此，在发送新消息时要根据该消息是要在当前状态进行处理还是在下一状态才进行处理，选择对应的 `sendMessage` 或 `deferMessage` 方法。
如果在状态机内部，还可通过 `sendMessageAtFrontOfQueue` 将消息发送至队列前面以尽快进行处理。

#### 状态操作相关方法

当前状态处理消息时，根据逻辑设计可调用 `transitionTo` 进行状态的转换，调用该方法并不会立即进行状态的转换，而是在该消息处理完成后进行状态的转移。
可根据需要可调用 `removeState` 从状态机中移除不再需要的状态，该方法不能移除当前的状态链中的状态，也不能移除某个子状态的父状态，条件比较严格。
此外，状态机提供了一个 Halting 的暂停状态，可通过 `transitionToHaltingState` 方法进入，状态机处于该状态时，后续的消息都由 `haltedProcessMessage` 处理，后文再对此特殊状态进行补充说明。

#### 退出相关方法

当希望退出状态机时，可以调用 `quit` 或 `quitNow` 方法，状态机将转移到 Quitting 状态。它们的区别是前者会在状态机处理完消息队列中的所有消息再退出，后者在当前消息处理完成后直接退出，正如 HFSM 设计，退出操作将逐一调用状态链的 `exit` 方法。

#### 其他模板方法

前面对状态机的 public 方法的使用进行了说明，其实状态机还有一些有用的模板方法，可在实现状态机时有选择地进行重写。

```java
protected void onPreHandleMessage(Message msg) {}
protected void onPostHandleMessage(Message msg) {}
protected void unhandledMessage(Message msg) {}
protected void haltedProcessMessage(Message msg) {}
protected void onHalting() {}
protected void onQuitting() {}
```

对正常消息来说，在处理前后会分别调用 `onPreHandleMessage` 和 `onPostHandleMessage` 方法，重写 `onPreHandleMessage` 方法时切记**不要修改 Message 对象**，否则出问题很难定位。

如果子状态及其所有父状态都未处理消息，则将调用 `unhandledMessage` 为状态机提供最后一次处理消息的机会。

上文说到状态机提供了两个特殊的状态，分别是 Halting 和 Quitting。在进入 Quitting 状态后，会调用 `onQuitting` 方法，这一般用来通知状态机本身进行资源的释放。同样地，在进入 Halting 状态后，会调用 `onHalting` 方法使得状态机了解该事件，可以释放暂时不需要的资源，并且后续的消息都由 `haltedProcessMessage` 进行处理，子类可重写该方法以便根据特定消息从 Halting 状态转移恢复到正常状态。

#### 调试相关

状态机提供了一些调试配置选项，简单介绍一下。

```java
// 日志调试
public void setDbg(boolean dbg) {}
public boolean isDbg() {}
protected void log(String s) {} // 一系列 log 方法

protected String getWhatToString(int what) {}
public final void setLogOnlyTransitions(boolean enable) {}
protected boolean recordLogRec(Message msg) {}

public final String getName() {}
public final int getLogRecSize() {}
public final void setLogRecSize(int maxSize) {}
public final int getLogRecMaxSize() {}
public final int getLogRecCount() {}
public final LogRec getLogRec(int index) {}
public final Collection<LogRec> copyLogRecs() {}
public void addLogRec(String string) {}
public void dump(FileDescriptor fd, PrintWriter pw, String[] args) {}
protected String getLogRecString(Message msg) {}
```

在开发调试时，最好打开 Debug 开关， `setDbg` 为 true 之后，处理消息和状态转移时会在日志中打印相关信息。状态机内部可以直接调用 `log` 系列方法打印出相同 TAG 的日志，与其内部保持一致。实践中，记得在 `log` 调用前加上 `isDbg` 判断，从而在上线版本中 `setDbg` 为 false 关闭日志输出。

log 打印的日志信息比较简单，如果要查看消息更多的信息，可以调用 StateMachine 的 `toString` 方法，实际上它也是调用了 `dump` 方法对这些所谓的 LogRec 进行输出。LogRec 的记录默认记录 20 条所有消息，这个可以通过查看 LogRecMaxSize 为 20 以及 `recordLogRec` 方法默认对所有消息返回 true 知道。如果只记录状态转移的消息可以 `setLogOnlyTransitions(true)` 。但这些信息打印消息时只打印 what，不够直观，我们可以重写 `getWhatToString(int what)` 方法，把 what 转成容易理解的字符串。

### 实践经验

1. 要特别注意消息处理线程和其他业务的线程的关系。要注意耗时处理的影响以及适时退出状态机，回收资源。
1. 消息的前后关系要放在心上，消息和状态的对应关系也要注意。有时候要插入消息或清除旧消息，要注意这种场景下的消息队列的消息处理情况。

> 参考资料：
>
> 1. [Finite-state machine Wikipedia](https://en.wikipedia.org/wiki/Finite-state_machine)
> 1. [State Machines for Event-Driven Systems](https://barrgroup.com/Embedded-Systems/How-To/State-Machines-Event-Driven-Systems)
> 1. [Introduction to Hierarchical State Machines (HSMs)](https://barrgroup.com/Embedded-Systems/How-To/Introduction-Hierarchical-State-Machines)
> 1. [层次化状态机的讨论](http://www.aisharing.com/archives/393)
> 1. [关于 StateMachine 的那些事儿](https://blog.csdn.net/a8403025/article/details/80666056)

---
title: Activity 启动模式全解析
date: 2017-05-21 20:21:33
categories:
- 技术向
tags:
- Android
---

# Tasks(Tasks)与返回栈(Back Stack)
Tasks 的定义：执行特定作业时与用户交互的一系列 Activity。这些 Activity 按照各自的打开顺序排列在堆栈（即返回栈）中，堆栈顶部 Activity 即为可见的界面。堆栈中的 Activity **永远不会** 重新排列，仅推入（push）和弹出（pop）堆栈：由当前 Activity 启动时推入堆栈；用户使用 `Back` 按钮退出时弹出堆栈。当所有 Activity 均从堆栈中移除后，Tasks 即不复存在。

举个例子，Contacts 应用的 PeopleActivity 可以启动 QuickContactActivity 来查看联系人信息，而 QuickContactActivity 可以通过 Intent 启动 Email 应用的编写邮件的 Activity。虽然 Activity 来自不同的应用，但是它们都被保留在相同的 Tasks 中以保持无缝的用户体验：用户按 Back 键时，会逐个回退到上一个 Activity，仿佛这些 Activity 是属于同一个应用的。

<!--more-->

Tasks 作为一个整体而存在，简单来说就是在某个 Activity 按 Home 键之后，该 Activity 所在的 Tasks 就被移到后台，但 Tasks 的返回栈不变。切换回这个 Activity 后，按 Back 键可以发现回退到了堆栈的上一个 Activity。基于 Tasks 与其返回栈的关系，我们使用 Tasks 栈来表达其含义。

Activity 和 Tasks 的默认行为总结如下：

当 Activity A 启动 Activity B 时，
* Activity A 将会停止，但系统会保留其状态（例如，滚动位置和已输入表单中的文本）。
* 创建 Activity B 的实例并向其传送 Intent，其将被推入 Activity A 的 Tasks 栈中。

处于 Activity B 时按 Back 按钮
* 当前 Activity B 会从堆栈弹出并被销毁。销毁 Activity 时，系统不会保留该 Activity 的状态。
* 堆栈中的前一个 Activity A 恢复其状态，继续执行。

用户通过按 Home 按钮离开 Tasks 时，
* 当前 Activity 将停止且其 Tasks 会进入后台。
* 系统将保留 Tasks 中每个 Activity 的状态。

选择开始 Tasks 的启动器图标来恢复 Tasks，
* Tasks 将出现在前台并恢复执行堆栈顶部的 Activity。
* 如果用户长时间离开 Tasks，则系统会清除 Tasks 栈中除根 Activity 之外的所有 Activity。当用户再次返回到 Tasks 时，仅恢复根 Activity。

# 管理 Tasks
上面提到的默认行为适用于大多数的 Activity，不需要额外的设置即可满足应用的需要。但有时候需要打破这种默认的行为，比如，打开应用的第一个 Activity A，然后用户注册经过了一个或多个 Activity，注册完成后回到第一个 Activity A，如果按照默认的行为，堆栈中将会实例化两个 Activity A，而且这两个之间还包含了注册用的 Activity，这样子在用户 Back 键想退出应用的时候，就会跳转到不想要跳转到的注册用的 Activity，这就要求回到 Activity A 时清除注册用的 Activity。或者，开发者希望用户不论在哪个 Activity 切换到后台时，再开启应用时要从应用的第一个 Activity 开始。

这些不符合默认行为的操作可以通过指定 Manifest 文件中 &lt;activity> 元素中的属性和传递给 startActivity() 的 Intent 中的标志来完成设置。

> **注意：** 应用大多数情况下都不必改变 Activity 和 Tasks 的默认行为；如果确定必须修改默认行为，请测试使用 Back 按钮从其他 Activity 或从 Recent 界面回到该 Activity 时的行为，确认应用的行为是否有可能与用户的预期行为冲突。

在清单文件中声明 Activity 时，可以指定 Activity 在启动时应该如何与 Tasks 关联。可以使用的 &lt;activity> 属性有：

* launchMode
* taskAffinity
* allowTaskReparenting
* clearTaskOnLaunch
* alwaysRetainTaskState
* finishOnTaskLaunch

调用 startActivity() 时，可以在 Intent 中加入一个标志，用于声明新 Activity 如何（或是否）与当前 Tasks 关联。可以使用的 Intent 标志主要有：

* FLAG_ACTIVITY_NEW_TASK
* FLAG_ACTIVITY_CLEAR_TOP
* FLAG_ACTIVITY_SINGLE_TOP

如果 Activity A 启动 Activity B，则 Activity B 可以在其清单文件中定义它应该如何与当前 Tasks 关联，并且 Activity A 还可以请求 Activity B 应该如何与当前 Tasks 关联。如果这两个 Activity 均定义 Activity B 应该如何与 Tasks 关联，则 Activity A 的请求（如 Intent 中所定义）优先级要高于 Activity B 的请求（如其清单文件中所定义）。

我们将属性分为两组：

1. 改变 Tasks 启动 Activity 时的默认行为
  * launchMode
  * taskAffinity
  * allowTaskReparenting
2. 改变 Tasks 恢复 Activity 时的默认行为
  * clearTaskOnLaunch
  * alwaysRetainTaskState
  * finishOnTaskLaunch

## 改变启动时的默认行为
### launchMode 和 taskAffinity
launchMode 包括了四种属性值：`standard (默认模式)`、`singleTop`、`singleTask`、`singleInstance`。相应的，使用 `FLAG_ACTIVITY_SINGLE_TOP` 标志产生与 `singleTop` 相同的行为，而`FLAG_ACTIVITY_NEW_TASK` 与 `FLAG_ACTIVITY_CLEAR_TOP` 一般一起使用，产生与 `singleTask` 相同的行为，一般不单独使用，因为单独使用常常导致奇怪的效果。

taskAffinity 指示 Activity 优先属于哪个 Tasks，即它希望进入的 Tasks，而 Tasks 的 taskAffinity 实际上等于它的根 Activity 的 taskAffinity。默认情况下，同一应用中的所有 Activity 默认使用软件包名称彼此关联。在不同应用中定义的 Activity 可以共享关联，或者可为在同一应用中定义的 Activity 分配不同的 Tasks 关联。taskAffinity 仅在两种情况下起作用：一是 Activity 的 launchMode 为 singleTask（亦即以 FLAG_ACTIVITY_NEW_TASK 的 Intent 启动的应用）；一是 Activity 将其 allowTaskReparenting 属性设置为 "true"。这两种情况分别见 singleTask/allowTaskReparenting 部分的示例与说明。

以下我们将分别解释 launchMode 的行为及对其进行测试分析。测试所用的应用代码在[这里](https://github.com/Mupceet/ActivityTaskDemo)。

#### standard
系统在启动 Activity 的 Tasks 中创建被启用的 Activity 的新实例并向其传送 Intent。Activity 可以多次实例化，而每个实例可属于不同的 Tasks 或者一个 Tasks 可以拥有多个实例。


测试 1：

```
MainActivity --> StandardActivity --> StandardActivity --> StandardActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1189 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
StandardActivity TaskId: 1189 HashCode:64163597
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
StandardActivity TaskId: 1189 HashCode:60617326
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
StandardActivity TaskId: 1189 HashCode:227576965
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 由 A 启动则它与 A 存在于同一 Tasks 栈中 | TaskId 相同 |
| 每次启动 Activity 都会创建一个新的实例 | StandardActivity 创建时走 onCreate 方法，HashCode 不相同 |


#### singleTop
如果当前 Tasks 栈的栈顶为 Activity 的一个实例，则系统会通过调用该实例的 onNewIntent() 方法向其传送 Intent，而不是创建 Activity 的新实例。但如果栈顶的 Activity 并不是 Activity 的现有实例，Activity 的表现与 standard 模式相同。

>**注：** 为某个 Activity 创建新实例时，用户可以按 Back 按钮返回到前一个 Activity。 但是，当 Activity 的现有实例处理新 Intent 时，则在新 Intent 到达 onNewIntent() 之前，用户无法按 Back 按钮返回到 Activity 的状态。

测试 2：SingleTopActivity 启动后处于栈顶

```
MainActivity --> SingleTopActivity --> SingleTopActivity --> SingleTopActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1190 HashCode:242719467
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleTopActivity TaskId: 1190 HashCode:122301260
taskAffinity:com.mupceet.activitytaskdemo
************* onNewIntent *************
SingleTopActivity TaskId: 1190 HashCode:122301260
taskAffinity:com.mupceet.activitytaskdemo
************* onNewIntent *************
SingleTopActivity TaskId: 1190 HashCode:122301260
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 由 A 启动则它与 A 存在于同一 Tasks 栈中 | TaskId 相同 |
| 第一次启动 Activity 会创建一个实例 | 第一次启动 SingleTopActivity 时运行 onCreate 方法 |
| 多次启动 Activity （处于栈顶） 只会回调 onNewIntent 方法 | 启动 SingleTopActivity 后，再多次启动 SingleTopActivity，Intent 由 onNewIntent 处理，且 HashCode 相同 |

测试 3：SingleTopActivity 启动后不处于栈顶

```
MainActivity --> SingleTopActivity --> OtherTopActivity --> SingleTopActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1191 HashCode:145258514
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleTopActivity TaskId: 1191 HashCode:9399791
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
OtherTopActivity TaskId: 1191 HashCode:209949367
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleTopActivity TaskId: 1191 HashCode:48271010
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| 再次启动 Activity (不处于栈顶) 会创建一个新实例 | 第二次启动 SingleTopActivity，运行 onCreate 方法，且 HashCode 不相同 |


#### singleTask
系统不存在该 Tasks（taskAffinity 标识的 Tasks 名）则创建新 Tasks 并实例化 Activity 推入 Tasks 栈底。如果存在 Tasks，但没有实例，将实例化 Activity 并推入 Tasks 栈，但是，如果该 Activity 的一个实例已存在于 Tasks 中，则系统会通过调用现有实例的 onNewIntent() 方法向其传送 Intent，并且原来栈中处于 Activity 之上的被清理出栈。即 Tasks 栈中只能存在 Activity 的一个实例。

> **注：** 尽管 Activity 在新 Task 中启动，但是用户按 Back 按钮仍会返回到前一个 Activity。

测试 4：SingleTaskActivity 不指定 taskAffinity

```
MainActivity --> SingleTaskActivity --> OtherTaskActivity --> SingleTaskActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1192 HashCode:160080556
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleTaskActivity TaskId: 1192 HashCode:125240432
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
OtherTaskActivity TaskId: 1192 HashCode:35483478
taskAffinity:com.mupceet.activitytaskdemo
************* onNewIntent *************
SingleTaskActivity TaskId: 1192 HashCode:125240432
taskAffinity:com.mupceet.activitytaskdemo
```

```
Running activities (most recent first):
  TaskRecord{216074a #1218 A=com.mupceet.activitytaskdemo U=0 StackId=1 sz=2}
    Run #1: ActivityRecord{8ca4bea u0 com.mupceet.activitytaskdemo/.SingleTaskActivity t1218}
    Run #0: ActivityRecord{5a6a171 u0 com.mupceet.activitytaskdemo/.MainActivity t1218}
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 由 A 启动则它与 A 存在于同一 Tasks 栈中 | TaskId 相同 |
| 再次启动 Activity 会回调 onNewIntent 方法 | 再次启动 SingleTaskActivity，Intent 由 onNewIntent 处理，且 HashCode 相同 |
| 原来栈中处于 Activity 之上的被清理出栈 | 再次启动 SingleTaskActivity 后，栈中仅有两个 Activity |

测试 5：SingleTaskActivity 指定 taskAffinity

```
MainActivity --> SingleTaskActivityWithAffinity --> StandardActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1200 HashCode:176168109
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleTaskActivityWithAffinity TaskId: 1201 HashCode:95759438
taskAffinity:com.mupceet.task2
************* onCreate *************
StandardActivity TaskId: 1201 HashCode:233514981
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 启动到新的 Tasks 栈中 | MainActivity 与 SingleTaskActivityWithAffinity 的 TaskId 不相同 |
| Activity 启动 standard 模式的 Activity 处于同一 Tasks 栈 | 启动 StandardActivity，两者的 TaskId 相同 |

测试 6：SingleTaskActivity 和 OtherActivity 指定相同 taskAffinity

```
(TaskDemo) MainActivity --> SingleTaskActivityWithAffinity --> home 返回主页
(TaskDemo2) MainActivity --> OtherActivityWithAffinity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1202 HashCode:8042982
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleTaskActivityWithAffinity TaskId: 1203 HashCode:37289106
taskAffinity:com.mupceet.task2

************* onCreate *************
MainActivity TaskId: 1204 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo2
************* onCreate *************
OtherActivityWithActivity TaskId: 1203 HashCode:186496063
taskAffinity:com.mupceet.task2
```

```
Running activities (most recent first):
      TaskRecord{5d38658 #1203 A=com.mupceet.task2 U=0 StackId=1 sz=2}
        Run #3: ActivityRecord{f653f3 u0 com.mupceet.activitytaskdemo2/.OtherActivityWithActivity t1203}
      TaskRecord{532c417 #1204 A=com.mupceet.activitytaskdemo2 U=0 StackId=1 sz=1}
        Run #2: ActivityRecord{2a6ae8b u0 com.mupceet.activitytaskdemo2/.MainActivity t1204}
      TaskRecord{5d38658 #1203 A=com.mupceet.task2 U=0 StackId=1 sz=2}
        Run #1: ActivityRecord{6ebc71c u0 com.mupceet.activitytaskdemo/.SingleTaskActivityWithAffinity t1203}
      TaskRecord{43259ed #1202 A=com.mupceet.activitytaskdemo U=0 StackId=1 sz=1}
        Run #0: ActivityRecord{df97ca7 u0 com.mupceet.activitytaskdemo/.MainActivity t1202}
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 启动到新的 Tasks 栈中 | MainActivity 与 SingleTaskActivityWithAffinity 的 TaskId 不相同 |
| 具有相同 taskAffinity 的 Activity 启动后将处于同一 Tasks 栈 | SingleTaskActivityWithAffinity 与 OtherActivityWithActivity 的 TaskId 相同 |

注意，这里有个现象需要注意，即如果存在 Tasks，但没有实例，将实例化 Activity 并推入 Tasks 栈，这个 Tasks 栈就在前台运行，此时按 Back 键得直到此 Tasks 栈清空后才会回到启动该 Activity 的那个 Tasks 栈，而不是像平常一次返回就可以返回启动 Activity。如果不明白这段话，只要将测试 6 的步骤改动一下为：
```
(TaskDemo) MainActivity --> SingleTaskActivityWithAffinity --> StandardActivity --> StandardActivity --> home 返回主页
(TaskDemo2) MainActivity --> OtherActivityWithAffinity
```
再点击几次 Back 键就知道是什么意思

#### singleInstance
与 "singleTask" 类似，但系统不会将任何其他 Activity 启动到包含该 Activity 实例的 Tasks 栈中。即该 Activity 始终是其 Tasks 栈唯一仅有的成员；由此 Activity 启动的任何 Activity 均在其他的 Tasks 中打开。

测试 7：

```
MainActivity--> SingleInstanceActivity--> MainActivity--> SingleInstanceActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1194 HashCode:19028065
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleInstanceActivity TaskId: 1195 HashCode:109601484
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
MainActivity TaskId: 1194 HashCode:202770375
taskAffinity:com.mupceet.activitytaskdemo
************* onNewIntent *************
SingleInstanceActivity TaskId: 1195 HashCode:109601484
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 启动到新的 Tasks 栈中，即使是默认的 taskAffinity | MainActivity 与 SingleInstanceActivity 的 TaskId 不相同 |
| 除了第一次启动时创建实例，其它多次启动复用并调到前台 | SingleInstanceActivity 的 HashCode 相同，Intent 由 onNewIntent 处理 |
| 启动的 Activity 运行在其他 Tasks 栈中 | SingleInstanceActivity 与 MainActivity 的 TaskId 不相同，而且由于 MainActivity 是 standard 模式，启动时创建了新实例，栈中有两个 MainActivity 实例 |

测试 8：在测试 7 基础上改为调用 SingleTopActivity

```
MainActivity--> SingleTopActivity--> SingleInstanceActivity--> SingleTopActivity--> SingleInstanceActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1196 HashCode:141418922
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleTopActivity TaskId: 1196 HashCode:74869513
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleInstanceActivity TaskId: 1197 HashCode:193894545
taskAffinity:com.mupceet.activitytaskdemo
************* onNewIntent *************
SingleTopActivity TaskId: 1196 HashCode:74869513
taskAffinity:com.mupceet.activitytaskdemo
************* onNewIntent *************
SingleInstanceActivity TaskId: 1197 HashCode:193894545
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 启动到新的 Tasks 栈中，即使用默认的 taskAffinity | MainActivity 与 SingleInstanceActivity 的 TaskId 不相同 |
| 除了第一次启动时创建实例，其它多次启动复用并调到前台 | SingleInstanceActivity 的 HashCode 相同，Intent 由 onNewIntent 处理 |
| 启动的 Activity 运行在其他 Tasks 栈中 | 启动的 SingleTopActivity 的 HashCode 相同，因为 SingleTopActivity 是 singleTop 模式，启动时未创建新实例，Intent 由 onNewIntent 处理 |

测试 9：

```
MainActivity--> SingleInstanceActivity--> StandardActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1206 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleInstanceActivity TaskId: 1209 HashCode:213193939
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
StandardActivity TaskId: 1206 HashCode:213531793
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 启动到新的 Tasks 栈中，即使用默认的 taskAffinity | MainActivity 与 SingleInstanceActivity 的 TaskId 不相同 |
| 启动的 Activity 运行在其他 Tasks 栈中 | SingleInstanceActivity 与 StandardActivity 的 TaskId 不相同，但由于 StandardActivity 和 MainActivity 的 taskAffinity 相同，它们存在于同一个 Tasks 栈中 |

测试 10：

```
(TaskDemo) MainActivity --> SingleInstanceActivity --> home 返回主页
(TaskDemo2) MainActivity --> SingleInstanceActivity
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1206 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo
************* onCreate *************
SingleInstanceActivity TaskId: 1207 HashCode:64163597
taskAffinity:com.mupceet.activitytaskdemo


************* onCreate *************
MainActivity TaskId: 1208 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo2
************* onNewIntent *************
SingleInstanceActivity TaskId: 1207 HashCode:64163597
taskAffinity:com.mupceet.activitytaskdemo
```

解析：

| 结论 | 来源 |
| :-- |:--|
| Activity 在系统中仅会存在一个实例 | 不同应用启动的 SingleInstanceActivity 的 TaskId 相同，HashCode 相同 |

### allowTaskReparenting
在这种属性情况下，Activity 在其关联的 Tasks 转到前台时，将从原先被启动时推入的 Tasks 栈移动到与其具有关联的 Tasks 栈中。

测试 11：TaskDemo2 启动 TaskDemo 的 StandardActivity

```
(TaskDemo2) MainActivity --> ToDemoStandard --> home 返回主页
(TaskDemo) MainActivity(打开应用)
```

结果：
```
************* onCreate *************
MainActivity TaskId: 1220 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo2
************* onCreate *************
StandardActivity TaskId: 1220 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo

（打开 Demo 应用无输出）
```
打开 TaskDemo 前：
```
Running activities (most recent first):
  TaskRecord{f0c1e3 #1222 A=com.mupceet.activitytaskdemo2 U=0 StackId=1 sz=2}
    Run #1: ActivityRecord{a32ff5f u0 com.mupceet.activitytaskdemo/.StandardActivity t1222}
    Run #0: ActivityRecord{371f577 u0 com.mupceet.activitytaskdemo2/.MainActivity t1222}
```

打开 TaskDemo 后：
```
Running activities (most recent first):
  TaskRecord{18af934 #1223 A=com.mupceet.activitytaskdemo U=0 StackId=1 sz=2}
    Run #1: ActivityRecord{a32ff5f u0 com.mupceet.activitytaskdemo/.StandardActivity t1223}
  TaskRecord{f0c1e3 #1222 A=com.mupceet.activitytaskdemo2 U=0 StackId=1 sz=1}
    Run #0: ActivityRecord{371f577 u0 com.mupceet.activitytaskdemo2/.MainActivity t1222}
```

解析：

| 结论 | 来源 |
| :-- |:--|
| standard 模式的 Activity 被启动时将被推入启动的 Tasks 栈中 | Demo2 的 MainActivity  与其启动的 Demo 的 StandardActivity 的 TaskId 相同 |
| allowTaskReparenting 的 Activity 被其关联的 Tasks 启动时将被转移到关联的 Tasks 栈中 | Demo 启动的 StandardActivity 未重新实例化，而是在 Tasks 栈中移动 |

## 改变恢复时的默认行为
如上文默认行为所述，如果用户长时间离开 Tasks，则系统会清除 Tasks 栈中除根 Activity 之外的所有 Activity。当用户再次返回到 Tasks 时，仅恢复根 Activity。系统这样做的原因是，经过很长一段时间后，用户可能已经放弃之前执行的操作，返回到 Tasks 是要开始执行新的操作。要修改恢复 Activity Tasks 栈的行为可以使用以下三个属性。
### alwaysRetainTaskState
如果在 Tasks 的根 Activity 中将此属性设置为 "true"，则不会发生刚才所述的默认行为。即使在很长一段时间后，Tasks 仍将所有 Activity 保留在其堆栈中。

### clearTaskOnLaunch
如果在 Tasks 的根 Activity 中将此属性设置为 "true"，则每当用户离开 Tasks 然后点击图标返回时，系统都会将堆栈清除到只剩下根 Activity。即使只离开 Tasks 片刻时间，用户也始终会返回到 Tasks 的初始状态。

测试 12：点击属于 TaskDemo2 的 StandardActivity 图标
```
StandardActivity --> StandardActivity --> StandardActivity --> StandardActivity
```
结果：
```
************* onCreate *************
StandardActivity TaskId: 1229 HashCode:241879363
taskAffinity:com.mupceet.activitytaskdemo2
************* onCreate *************
StandardActivity TaskId: 1229 HashCode:235658412
taskAffinity:com.mupceet.activitytaskdemo2
************* onCreate *************
StandardActivity TaskId: 1229 HashCode:188962139
taskAffinity:com.mupceet.activitytaskdemo2
************* onCreate *************
StandardActivity TaskId: 1229 HashCode:34150381
taskAffinity:com.mupceet.activitytaskdemo2
```

```
Running activities (most recent first):
  TaskRecord{d2b8f13 #1229 A=com.mupceet.activitytaskdemo2 U=0 StackId=1 sz=4}
    Run #3: ActivityRecord{e71827f u0 com.mupceet.activitytaskdemo2/.StandardActivity t1229}
    Run #2: ActivityRecord{7a51fa1 u0 com.mupceet.activitytaskdemo2/.StandardActivity t1229}
    Run #1: ActivityRecord{ef87b33 u0 com.mupceet.activitytaskdemo2/.StandardActivity t1229}
    Run #0: ActivityRecord{638598e u0 com.mupceet.activitytaskdemo2/.StandardActivity t1229}
```
Home 回到主页后，再次点击桌面图标返回
```
Running activities (most recent first):
  TaskRecord{d2b8f13 #1229 A=com.mupceet.activitytaskdemo2 U=0 StackId=1 sz=1}
    Run #0: ActivityRecord{638598e u0 com.mupceet.activitytaskdemo2/.StandardActivity t1229}
```
可以看到 Tasks 栈中仅剩根 Activity 了，点击一次 Back 即可退出。

### finishOnTaskLaunch
该属性与 clearTaskOnLaunch 类似，但这个针对单个 Activity 而不是整个 Tasks。在 Tasks 转入后台再点击桌面图标返回时设置该属性为 "true" 的 Activity 将被销毁。

> 参考资料：
> * [Google API 指南](https://developer.android.google.cn/guide/components/activities/tasks-and-back-stack.html)
> * [Android 面试题-深刻剖析 activity 启动模式(一)](http://www.jianshu.com/p/b33fd8c550bf)
> * [Android 面试题-深刻剖析 activity 启动模式(二)](http://www.jianshu.com/p/e1ea9e542112)
> * [Android 面试题-深刻剖析 activity 启动模式(三)](http://www.jianshu.com/p/d13e3d552d4b)

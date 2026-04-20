# Activity 全解析

## 一、Activity 是什么？

Activity 是 Android 四大组件之一，代表一个"页面"。用户看到的每一个界面，背后都是一个 Activity（或者 Activity 里的 Fragment）。

---

## 二、生命周期

### 2.1 完整生命周期

````
onCreate() → onStart() → onResume()
    ↕ 用户可见、可交互
onPause() → onStop() → onDestroy()
````
| 回调 | 含义 | 典型操作 |
|------|------|---------|
| `onCreate()` | Activity 被创建 | setContentView、初始化数据、绑定 ViewModel |
| `onStart()` | 即将可见（但还不能交互） | 注册广播、开始动画 |
| `onResume()` | 可见且可交互（前台） | 恢复播放、开启相机 |
| `onPause()` | 即将失去焦点（部分可见） | 暂停播放、保存轻量数据 |
| `onStop()` | 完全不可见 | 释放重资源、注销广播 |
| `onDestroy()` | 即将销毁 | 释放所有资源 |

### 2.2 常见场景的生命周期变化

**从 A 跳转到 B：**
````
A.onPause() → B.onCreate() → B.onStart() → B.onResume() → A.onStop()
````
> 注意：A.onPause() 先执行，B 完全显示后才执行 A.onStop()。所以 onPause() 里不要做耗时操作，否则会影响 B 的启动速度。

**从 B 返回 A：**
````
B.onPause() → A.onRestart() → A.onStart() → A.onResume() → B.onStop() → B.onDestroy()
````
**按 Home 键：**
````
onPause() → onStop()
````
**从后台回到前台：**
````
onRestart() → onStart() → onResume()
````
**屏幕旋转（未配置 configChanges）：**
````
onPause() → onStop() → onSaveInstanceState() → onDestroy()
→ onCreate() → onStart() → onRestoreInstanceState() → onResume()
````
> Activity 会被销毁重建！这就是为什么旋转屏幕后数据会丢失。

**弹出 Dialog：**
````
什么都不会调用！
````
> Dialog 是 Window 层级的，不会触发 Activity 的 onPause()。除非弹出的是一个 DialogTheme 的 Activity。

### 2.3 onSaveInstanceState 与 onRestoreInstanceState

当系统可能杀死 Activity 时（旋转屏幕、内存不足、进入后台），会调用 `onSaveInstanceState()` 保存状态。

````java
@Override
protected void onSaveInstanceState(@NonNull Bundle outState) {
    super.onSaveInstanceState(outState);
    outState.putString("key", "value");
    outState.putInt("scroll_position", 100);
}

@Override
protected void onRestoreInstanceState(@NonNull Bundle savedInstanceState) {
    super.onRestoreInstanceState(savedInstanceState);
    String value = savedInstanceState.getString("key");
}
````
**调用时机：**
- `onSaveInstanceState()`：在 `onStop()` 之前（Android P 之后是 `onStop()` 之后）
- `onRestoreInstanceState()`：在 `onStart()` 之后
- 如果 Activity 是用户主动销毁的（按返回键），**不会调用** onSaveInstanceState

---

## 三、启动模式（LaunchMode）

### 3.1 四种启动模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `standard` | 每次启动都创建新实例，放入当前任务栈 | 默认模式，大多数页面 |
| `singleTop` | 如果栈顶已经是该 Activity，不创建新实例，调用 `onNewIntent()` | 通知跳转页、搜索页 |
| `singleTask` | 在目标任务栈中查找，如果已存在则清除其上方所有 Activity，调用 `onNewIntent()` | 首页、主页面 |
| `singleInstance` | 独占一个任务栈，全局唯一 | 来电界面、系统级页面 |

### 3.2 通俗理解

想象任务栈是一摞盘子：

**standard：** 每次都往上放一个新盘子，不管下面有没有一样的。
````
栈：[A] → 启动A → [A, A] → 启动A → [A, A, A]
````
**singleTop：** 如果最上面那个盘子就是要放的，就不放了，直接复用。
````
栈：[A, B] → 启动B → [A, B]（复用栈顶B，调用onNewIntent）
栈：[A, B] → 启动A → [A, B, A]（A不在栈顶，创建新实例）
````
**singleTask：** 在栈里找，找到了就把它上面的盘子全扔掉。
````
栈：[A, B, C] → 启动A → [A]（B、C被销毁，A调用onNewIntent）
````
**singleInstance：** 这个盘子太特殊了，必须单独放一摞。
````
栈1：[A, B]
栈2：[C]（singleInstance，独占）
````
### 3.3 taskAffinity

每个 Activity 都有一个 `taskAffinity` 属性，表示它"倾向于"属于哪个任务栈。

- 默认值是应用的包名（所以默认所有 Activity 在同一个栈）
- `singleTask` 会根据 taskAffinity 查找目标任务栈
- 配合 `FLAG_ACTIVITY_NEW_TASK` 使用

````xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTask"
    android:taskAffinity="com.example.main" />
````
### 3.4 常用 Intent Flags

| Flag | 效果 |
|------|------|
| `FLAG_ACTIVITY_NEW_TASK` | 在新的任务栈中启动（如果 taskAffinity 不同） |
| `FLAG_ACTIVITY_CLEAR_TOP` | 如果目标 Activity 已在栈中，清除其上方所有 Activity |
| `FLAG_ACTIVITY_SINGLE_TOP` | 等同于 singleTop |
| `FLAG_ACTIVITY_CLEAR_TASK` | 清空目标任务栈，配合 NEW_TASK 使用 |
| `FLAG_ACTIVITY_NO_HISTORY` | 该 Activity 不会留在栈中（离开就销毁） |

````java
// 常见组合：回到首页并清空栈
Intent intent = new Intent(this, MainActivity.class);
intent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP);
startActivity(intent);
````
---

## 四、Activity 的启动流程（源码级）

这是高级面试题，了解大致流程即可。

````
1. startActivity()
2. → Instrumentation.execStartActivity()
3. → ATMS（ActivityTaskManagerService，跨进程 Binder 调用）
4. → ActivityStarter.execute()（解析 Intent、处理启动模式）
5. → ActivityStack 管理任务栈
6. → 如果目标进程未启动 → Zygote fork 新进程
7. → ActivityThread.main()（新进程入口）
8. → ActivityThread.handleLaunchActivity()
9. → Instrumentation.callActivityOnCreate()
10. → Activity.onCreate()
````
**简化版本（面试够用）：**
````
App 进程 → Binder → system_server（ATMS）
→ 处理启动模式、任务栈
→ Binder 回调 → App 进程
→ ActivityThread → 反射创建 Activity → onCreate()
````
---

## 五、Activity 与 Window 的关系

````
Activity
  └── PhoneWindow（Window 的唯一实现）
        └── DecorView（顶层 View，FrameLayout）
              ├── TitleBar / ActionBar
              └── ContentView（你 setContentView 设置的布局）
````
- 每个 Activity 持有一个 `PhoneWindow`
- `setContentView()` 实际上是把你的布局添加到 `DecorView` 的 `ContentView` 区域
- `DecorView` 最终被添加到 `WindowManager`，由 `ViewRootImpl` 管理绘制

---

## 六、面试题库

### 基础题

**Q1：Activity 的生命周期有哪些？分别在什么时候调用？**
> onCreate → onStart → onResume → onPause → onStop → onDestroy。onCreate 初始化，onStart 可见但不可交互，onResume 前台可交互，onPause 部分可见失去焦点，onStop 完全不可见，onDestroy 销毁。

**Q2：从 A 跳转到 B，生命周期的调用顺序是什么？**
> A.onPause → B.onCreate → B.onStart → B.onResume → A.onStop。关键点：A.onPause 先于 B.onCreate，所以 onPause 不能做耗时操作。

**Q3：弹出 Dialog 会触发 Activity 的 onPause 吗？**
> 不会。Dialog 是 Window 层级的，不影响 Activity 生命周期。但如果弹出的是一个 Theme 为 Dialog 的 Activity，则会触发 onPause。

**Q4：onSaveInstanceState 什么时候调用？用户按返回键会调用吗？**
> 系统可能杀死 Activity 时调用（旋转屏幕、进入后台、内存不足）。用户主动按返回键不会调用，因为这是用户主动销毁，不需要保存状态。

### 启动模式题

**Q5：四种启动模式的区别？**
> standard 每次新建；singleTop 栈顶复用；singleTask 栈内复用（清除上方）；singleInstance 独占任务栈。

**Q6：singleTask 的 Activity 被复用时，会走哪些生命周期？**
> onNewIntent → onRestart → onStart → onResume。注意 onNewIntent 在 onResume 之前调用。

**Q7：FLAG_ACTIVITY_CLEAR_TOP 和 singleTask 有什么区别？**
> CLEAR_TOP 只是清除目标 Activity 上方的实例。如果目标是 standard 模式，会先销毁再重建（走 onCreate）。singleTask 则是直接复用（走 onNewIntent）。两者配合 FLAG_ACTIVITY_SINGLE_TOP 使用时效果等同于 singleTask。

**Q8：什么是 taskAffinity？有什么用？**
> taskAffinity 指定 Activity 倾向的任务栈，默认是包名。singleTask 模式下会根据 taskAffinity 查找或创建目标任务栈。配合 allowTaskReparenting 可以实现 Activity 在任务栈间迁移。

### 进阶题

**Q9：Activity 的启动流程？**
> App 调用 startActivity → 通过 Binder 到 system_server 的 ATMS → 处理启动模式和任务栈 → 如果目标进程不存在则通过 Zygote fork → 回调到 App 进程的 ActivityThread → 反射创建 Activity 实例 → 调用 onCreate。

**Q10：setContentView 做了什么？**
> 创建 DecorView（如果没有）→ 根据主题选择布局（有无 ActionBar）→ 找到 DecorView 中的 ContentView 容器 → 将我们的布局 inflate 并添加到 ContentView 中。

**Q11：Activity、Window、DecorView 的关系？**
> Activity 持有 PhoneWindow，PhoneWindow 持有 DecorView。DecorView 是顶层 View（FrameLayout），包含 TitleBar 和 ContentView。setContentView 的布局被添加到 ContentView 区域。

**Q12：如何避免屏幕旋转导致 Activity 重建？**
> 方案一：在 Manifest 中配置 `android:configChanges="orientation|screenSize"`，旋转时只回调 onConfigurationChanged 而不重建。方案二：使用 ViewModel 保存数据，重建后自动恢复。

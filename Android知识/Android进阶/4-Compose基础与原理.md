# Compose 基础与原理

## 一、什么是 Compose？

Jetpack Compose 是 Android 的**声明式 UI 框架**，用 Kotlin 代码直接描述 UI，替代传统的 XML 布局。

**命令式（传统 View）vs 声明式（Compose）：**

````kotlin
// 命令式：告诉系统"怎么做"
textView.text = "Hello"
textView.setTextColor(Color.RED)
if (isVisible) textView.visibility = View.VISIBLE
else textView.visibility = View.GONE

// 声明式：告诉系统"要什么"
@Composable
fun Greeting(name: String, isVisible: Boolean) {
    if (isVisible) {
        Text(text = "Hello $name", color = Color.Red)
    }
}
````
声明式的核心思想：**UI = f(State)**。UI 是状态的函数，状态变了 UI 自动更新。

---

## 二、核心概念

### 2.1 @Composable 函数

````kotlin
@Composable
fun UserCard(user: User) {
    Card(modifier = Modifier.padding(8.dp)) {
        Column {
            Text(text = user.name, style = MaterialTheme.typography.h6)
            Text(text = user.email, color = Color.Gray)
        }
    }
}
````
- `@Composable` 标记的函数不是普通函数，编译器会对它做特殊处理
- 不能在普通函数中调用 @Composable 函数
- 没有返回值（不返回 View 对象），而是"发射" UI 节点

### 2.2 State（状态）

````kotlin
@Composable
fun Counter() {
    // remember：在重组时保留值（不会被重置）
    // mutableStateOf：创建可观察的状态
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("点击了 $count 次")
    }
    // count 变化 → Compose 检测到状态变化 → 重新执行 Counter() → UI 更新
}
````
**remember vs rememberSaveable：**

| 方法 | 配置变更（旋转） | 进程被杀 |
|------|----------------|---------|
| `remember` | ❌ 丢失 | ❌ 丢失 |
| `rememberSaveable` | ✅ 保留 | ✅ 保留（存入 Bundle） |

````kotlin
// 旋转屏幕后还在
var text by rememberSaveable { mutableStateOf("") }
````
### 2.3 状态提升（State Hoisting）

把状态从子组件提升到父组件，子组件变成无状态的，更容易复用和测试。

````kotlin
// ❌ 状态在内部，不好复用
@Composable
fun MyTextField() {
    var text by remember { mutableStateOf("") }
    TextField(value = text, onValueChange = { text = it })
}

// ✅ 状态提升到外部
@Composable
fun MyTextField(value: String, onValueChange: (String) -> Unit) {
    TextField(value = value, onValueChange = onValueChange)
}

// 父组件管理状态
@Composable
fun Screen() {
    var name by remember { mutableStateOf("") }
    MyTextField(value = name, onValueChange = { name = it })
}
````
---

## 三、重组（Recomposition）

### 3.1 什么是重组？

当 State 变化时，Compose 会重新执行受影响的 @Composable 函数，这个过程叫**重组**。

````kotlin
@Composable
fun Screen() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("标题")           // count 变化时，这行不会重新执行
        Text("计数: $count")   // 这行会重新执行（读取了 count）
        Button(onClick = { count++ }) {
            Text("点击")
        }
    }
}
````
### 3.2 重组的特性

| 特性 | 说明 |
|------|------|
| 智能重组 | 只重新执行读取了变化状态的 Composable，没读取的跳过 |
| 可能乱序 | Compose 可能以任意顺序执行 Composable 函数 |
| 可能并行 | Compose 可能在多个线程上并行执行重组 |
| 可能频繁 | 动画期间每帧都可能重组 |
| 幂等性 | 同样的输入应该产生同样的 UI |

### 3.3 避免不必要的重组

````kotlin
// ❌ 每次重组都创建新的 lambda，导致子组件重组
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }
    Child(onClick = { count++ })  // 每次重组都是新的 lambda 对象
}

// ✅ 用 remember 缓存 lambda
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }
    val onClick = remember { { count++ } }
    Child(onClick = onClick)
}

// ✅ 使用 Stable/Immutable 注解标记数据类
@Stable  // 告诉 Compose 这个类的实例是稳定的
data class UiState(val name: String, val age: Int)
````
---

## 四、副作用 API（Side Effects）

Composable 函数应该是纯函数（无副作用），但实际开发中需要做一些"副作用"操作（网络请求、日志、导航等）。Compose 提供了专门的 API。

### 4.1 LaunchedEffect

在 Composable 进入组合时启动协程，离开时自动取消。key 变化时重新启动。

````kotlin
@Composable
fun Screen(userId: String) {
    var user by remember { mutableStateOf<User?>(null) }

    // userId 变化时重新加载
    LaunchedEffect(userId) {
        user = repository.getUser(userId)  // 挂起函数
    }

    user?.let { UserCard(it) }
}
````
### 4.2 DisposableEffect

需要清理资源时使用（类似 onDestroy）。

````kotlin
@Composable
fun LifecycleAware(lifecycleOwner: LifecycleOwner) {
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            // 处理生命周期事件
        }
        lifecycleOwner.lifecycle.addObserver(observer)

        onDispose {
            // 清理：离开组合时调用
            lifecycleOwner.lifecycle.removeObserver(observer)
        }
    }
}
````
### 4.3 SideEffect

每次重组成功后执行，用于将 Compose 状态同步到非 Compose 代码。

````kotlin
@Composable
fun Screen(analytics: Analytics) {
    SideEffect {
        // 每次重组后执行
        analytics.setScreenName("HomeScreen")
    }
}
````
### 4.4 对比

| API | 触发时机 | 有协程 | 有清理 | 典型场景 |
|-----|---------|--------|--------|---------|
| `LaunchedEffect` | 进入组合 / key 变化 | ✅ | 自动取消 | 网络请求、延迟操作 |
| `DisposableEffect` | 进入组合 / key 变化 | ❌ | onDispose | 注册/注销监听器 |
| `SideEffect` | 每次重组成功后 | ❌ | ❌ | 同步状态到外部 |
| `rememberCoroutineScope` | 手动启动 | ✅ | 离开组合时取消 | 事件回调中启动协程 |

---

## 五、与 View 体系互操作

### 5.1 在 Compose 中使用 View

````kotlin
@Composable
fun MapScreen() {
    AndroidView(
        factory = { context ->
            MapView(context).apply { onCreate(null) }
        },
        update = { mapView ->
            // 状态变化时更新 View
        }
    )
}
````
### 5.2 在 View 中使用 Compose

````kotlin
// 在 XML 布局中
<androidx.compose.ui.platform.ComposeView
    android:id="@+id/compose_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />

// Activity/Fragment 中
composeView.setContent {
    MaterialTheme {
        MyComposable()
    }
}
````
---

## 六、Compose 的编译原理（简要）

Compose 编译器插件在编译时对 @Composable 函数做了大量转换：

1. **添加 Composer 参数**：每个 @Composable 函数都会被注入一个 `$composer` 参数
2. **生成 Group**：用于标识 UI 树中的节点位置
3. **插入比较逻辑**：在函数入口处比较参数是否变化，没变就跳过（智能重组）
4. **Slot Table**：Compose 运行时用 Slot Table（Gap Buffer 数据结构）存储 UI 树的状态

````kotlin
// 你写的代码
@Composable
fun Greeting(name: String) {
    Text("Hello $name")
}

// 编译后大致变成（简化）
fun Greeting(name: String, $composer: Composer, $changed: Int) {
    $composer.startGroup(123)  // 开始一个 Group
    if ($changed and 0b0001 != 0 || !$composer.skipping) {
        // 参数变了或者不能跳过，执行函数体
        Text("Hello $name", $composer, ...)
    } else {
        $composer.skipToGroupEnd()  // 参数没变，跳过
    }
    $composer.endGroup()
}
````
---

## 七、面试题库

### Q1：Compose 和传统 View 体系的区别？

> 最根本的区别是编程范式——传统 View 是命令式的，你需要手动操作 View 对象来更新 UI（setText、setVisibility）；Compose 是声明式的，你只需要描述 UI 应该是什么样子，状态变化时框架自动更新。传统 View 基于继承体系（View → ViewGroup → 各种子类），Compose 基于函数组合（@Composable 函数嵌套调用）。性能上 Compose 跳过了 XML 解析和 inflate 的开销，重组时只更新变化的部分。但 Compose 目前的生态和成熟度还不如传统 View，很多第三方库还没有 Compose 版本。

### Q2：什么是重组？Compose 怎么知道哪些部分需要重组？

> 重组是 Compose 在状态变化时重新执行 @Composable 函数的过程。Compose 通过 Snapshot 系统追踪状态的读取——当一个 @Composable 函数在执行过程中读取了某个 State 对象，Compose 就会记录这个依赖关系。当该 State 的值变化时，Compose 知道哪些函数读取了它，只重新执行这些函数，其他没有读取变化状态的函数会被跳过。这就是"智能重组"。编译器插件在每个 @Composable 函数入口处插入了参数比较逻辑，如果所有参数都没变就直接跳过函数体。

### Q3：remember 和 rememberSaveable 的区别？

> remember 在重组时保留值，但配置变更（旋转屏幕）时会丢失，因为整个 Composition 会被销毁重建。rememberSaveable 会把值保存到 Bundle 中，配置变更甚至进程被杀后都能恢复。rememberSaveable 要求值是 Bundle 支持的类型，或者实现 Saver 接口自定义序列化。简单的 UI 状态（输入框内容、滚动位置）用 rememberSaveable，临时的计算结果或对象引用用 remember。

### Q4：什么是状态提升？为什么要这么做？

> 状态提升是把状态从子组件移到父组件，子组件通过参数接收状态和事件回调。这样做的好处是：子组件变成无状态的纯函数，更容易复用和测试；状态集中管理，数据流向清晰（单向数据流）；多个子组件可以共享同一个状态。Compose 推荐的模式是状态向上提升、事件向下传递。在实际项目中，UI 状态通常提升到 ViewModel 中，ViewModel 暴露 StateFlow，Composable 函数通过 collectAsState 收集。

### Q5：LaunchedEffect、DisposableEffect、SideEffect 分别什么时候用？

> LaunchedEffect 用于需要在协程中执行的副作用，比如网络请求、延迟操作，它在进入组合时启动协程，key 变化时取消旧的重新启动，离开组合时自动取消。DisposableEffect 用于需要清理的副作用，比如注册监听器，在 onDispose 中注销。SideEffect 用于每次重组成功后将 Compose 状态同步到非 Compose 代码，比如更新 Analytics。选择原则：需要协程用 LaunchedEffect，需要清理用 DisposableEffect，需要每次重组后同步用 SideEffect。

### Q6：Compose 的性能怎么优化？

> 几个关键点：一是避免不必要的重组，用 remember 缓存计算结果和 lambda，给数据类加 @Stable 或 @Immutable 注解让 Compose 知道它是稳定的可以跳过比较。二是使用 derivedStateOf 避免频繁重组，比如列表滚动时只在满足条件时才触发重组。三是 LazyColumn/LazyRow 中给 item 设置 key，帮助 Compose 识别列表项的身份，避免不必要的重新创建。四是用 Layout Inspector 和 Compose 编译器报告检查哪些函数被标记为不可跳过（unstable），针对性优化。

### Q7：Compose 编译器做了什么？

> Compose 编译器插件在编译时对 @Composable 函数做了几个关键转换：给每个函数注入 Composer 参数用于管理 UI 树；插入 Group 标记用于标识节点位置；在函数入口处插入参数比较逻辑实现智能跳过；生成 $changed 位掩码追踪参数变化状态。运行时 Compose 用 Slot Table（基于 Gap Buffer 数据结构）存储整棵 UI 树的状态，重组时通过比较 Slot Table 中的旧值和新值来决定是否需要更新。

### Q8：Compose 中怎么使用 ViewModel？

> 通过 viewModel() 函数获取 ViewModel 实例，用 collectAsState() 将 StateFlow 转为 Compose 的 State。

````kotlin
@Composable
fun Screen(viewModel: MyViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    when (uiState) {
        is Loading -> CircularProgressIndicator()
        is Success -> ContentList(uiState.data)
        is Error -> ErrorMessage(uiState.message)
    }
}
````
> viewModel() 内部使用 ViewModelProvider，作用域和传统方式一样。collectAsState 会在 Flow 发射新值时触发重组。

### Q9：Compose 和 View 怎么混合使用？

> 两个方向：在 Compose 中嵌入 View 用 AndroidView，传入 factory 创建 View 实例和 update 回调更新 View；在 View 中嵌入 Compose 用 ComposeView，调用 setContent 设置 Composable 内容。实际项目中通常是渐进式迁移——新页面用 Compose 写，旧页面保持 View，通过 ComposeView 在旧页面中逐步引入 Compose 组件。需要注意的是 AndroidView 中的 View 不参与 Compose 的重组机制，需要在 update 回调中手动更新。

### Q10：Compose 中的 Modifier 是什么？

> Modifier 是 Compose 中用于修饰和配置 Composable 的链式 API，类似于传统 View 的 LayoutParams + 各种属性的组合。它可以设置大小（size、fillMaxWidth）、间距（padding）、背景（background）、点击事件（clickable）、滚动（verticalScroll）等。Modifier 的调用顺序很重要——它是从外到内应用的，padding 在 background 之前和之后效果完全不同。Modifier 是不可变的，每次调用都返回新的 Modifier 对象，这保证了线程安全。

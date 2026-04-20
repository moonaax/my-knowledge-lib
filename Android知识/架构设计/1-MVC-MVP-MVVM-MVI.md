# MVC / MVP / MVVM / MVI

> Android 客户端架构演进全解析：从 MVC 到 MVI 的设计哲学与实战对比。
> 关联主题：[[1-Jetpack-Lifecycle与ViewModel]] · [[Android-DataBinding]] · [[Kotlin-协程与Flow]] · [[Jetpack-Compose]]

---

## 一、核心概念

### 1.1 四种架构的定义

| 架构 | 全称 | 核心思想 |
|------|------|----------|
| MVC | Model-View-Controller | 控制器接收输入，协调 Model 与 View |
| MVP | Model-View-Presenter | Presenter 作为中间人，View 与 Model 完全隔离 |
| MVVM | Model-View-ViewModel | 数据绑定驱动 UI 自动更新，ViewModel 暴露可观察状态 |
| MVI | Model-View-Intent | 单向数据流 + 不可变状态，Intent 描述用户意图 |

### 1.2 演进关系

```
MVC (1979, Smalltalk)
 │
 │  问题：View 与 Model 耦合，Controller 臃肿
 ▼
MVP (1990s, Taligent)
 │
 │  问题：Presenter 与 View 通过接口通信，大量模板代码
 ▼
MVVM (2005, Microsoft WPF)
 │
 │  问题：双向绑定调试困难，状态分散难以追踪
 ▼
MVI (2017, Hannes Dorfmann)
     单向数据流 + 不可变状态，函数式思想
```

演进的核心驱动力：
- **关注点分离**：让每一层职责更单一
- **可测试性**：业务逻辑脱离 Android 框架，方便单元测试
- **状态管理**：从分散状态到集中式不可变状态

---

## 二、原理与对比

### 2.1 MVC 数据流

```
         用户操作
            │
            ▼
     ┌─────────────┐
     │  Controller  │──── 更新 ────▶┌─────────┐
     └─────────────┘                │  Model  │
            ▲                       └────┬────┘
            │                            │
            │                        通知变化
            │                            │
            │                            ▼
            └──── 用户事件 ◀────┌──────────┐
                                │   View   │
                                └──────────┘
```

**职责划分：**
- Model：数据与业务逻辑
- View：UI 展示，直接观察 Model 变化
- Controller：接收用户输入，调用 Model

**Android 中的现实：** Activity/Fragment 同时承担 Controller 和 View 的角色，导致"God Activity"问题。

### 2.2 MVP 数据流

```
     ┌──────────┐   接口调用   ┌─────────────┐   数据请求   ┌─────────┐
     │   View   │◀────────────▶│  Presenter  │─────────────▶│  Model  │
     │(Activity)│  IView 接口  └──────┬──────┘              └────┬────┘
     └──────────┘                     ▲                          │
                                      │       回调返回数据        │
                                      └──────────────────────────┘
```

**职责划分：**
- Model：数据层（Repository / DataSource）
- View：纯 UI，通过 IView 接口被动接收更新
- Presenter：中间人，持有 View 接口引用，调用 Model 并回调 View

### 2.3 MVVM 数据流

```
     ┌──────────┐   观察数据    ┌──────────────┐   请求数据   ┌─────────┐
     │   View   │──────────────▶│  ViewModel   │─────────────▶│  Model  │
     │(Activity)│  LiveData /   └──────┬───────┘              └────┬────┘
     └─────┬────┘  StateFlow          ▲                           │
           │                          │        返回数据            │
           │   用户事件                └───────────────────────────┘
           └──────────▶ ViewModel
```

**职责划分：**
- Model：数据层
- View：观察 ViewModel 暴露的可观察数据，触发事件
- ViewModel：持有 UI 状态（[[1-Jetpack-Lifecycle与ViewModel]]），不持有 View 引用

### 2.4 MVI 数据流

```
     ┌──────────┐                ┌──────────────┐               ┌─────────┐
     │   View   │── Intent ────▶│  ViewModel   │── 请求数据 ──▶│  Model  │
     │(Activity)│               │  (Reducer)   │               └────┬────┘
     └─────┬────┘               └──────┬───────┘                    │
           ▲                           │                            │
           │                      新 State                     返回结果
           │                           │                            │
           └──── 渲染 State ◀──────────┴────────────────────────────┘

     单向循环：View → Intent → Reducer → State → View
```

**职责划分：**
- Model：这里的 Model 指整个 UI 状态（不可变 data class）
- View：发送 Intent，根据 State 渲染
- Intent：用户意图的密封类描述
- Reducer：纯函数，`(旧State, Intent) → 新State`

### 2.5 综合对比表

| 维度 | MVC | MVP | MVVM | MVI |
|------|-----|-----|------|-----|
| 数据流向 | 双向，混乱 | 双向，通过接口 | 单向观察 + 事件回调 | 严格单向循环 |
| View 与 Model 耦合 | 直接耦合 | 完全隔离 | 完全隔离 | 完全隔离 |
| 状态管理 | 分散 | 分散在 Presenter | 分散在多个 LiveData | 集中式单一 State |
| 可测试性 | 差 | 好（Presenter 可测） | 好（ViewModel 可测） | 最佳（纯函数 Reducer） |
| 模板代码量 | 少 | 多（接口定义） | 中等 | 中等 |
| 学习曲线 | 低 | 中 | 中 | 高 |
| 调试难度 | 高（流向不清） | 中 | 中（双向绑定时较难） | 低（状态可追溯） |
| 适用场景 | 简单页面/Demo | 中型项目 | 大部分 Android 项目 | 复杂状态交互页面 |
| Android 官方推荐 | ✗ | ✗ | ✓ (Jetpack) | ✓ (趋势) |

---

## 三、Android 实战

> 以下用同一个场景「加载用户列表」演示四种架构的 Kotlin 实现。

### 3.0 公共数据层

```kotlin
// --- 公共 Model 层 ---
data class User(val id: Int, val name: String)

class UserRepository {
    suspend fun getUsers(): Result<List<User>> = runCatching {
        // 模拟网络请求
        delay(1000)
        listOf(User(1, "Alice"), User(2, "Bob"))
    }
}
```

### 3.1 MVC 实现

```kotlin
// Controller + View 混在 Activity 中（典型 Android MVC）
class MvcActivity : AppCompatActivity() {

    private val repo = UserRepository()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_list)
        loadUsers()
    }

    private fun loadUsers() {
        showLoading(true)
        lifecycleScope.launch {
            repo.getUsers()
                .onSuccess { users ->
                    showLoading(false)
                    findViewById<RecyclerView>(R.id.recyclerView)
                        .adapter = UserAdapter(users)
                }
                .onFailure { e ->
                    showLoading(false)
                    Toast.makeText(this@MvcActivity, e.message, Toast.LENGTH_SHORT).show()
                }
        }
    }

    private fun showLoading(show: Boolean) {
        findViewById<ProgressBar>(R.id.progressBar).isVisible = show
    }
}
```

**问题一目了然：** Activity 同时负责 UI 渲染、生命周期管理、业务逻辑调用，无法单元测试。

### 3.2 MVP 实现

```kotlin
// --- Contract 定义 ---
interface UserListContract {
    interface View {
        fun showLoading(show: Boolean)
        fun showUsers(users: List<User>)
        fun showError(msg: String)
    }
    interface Presenter {
        fun loadUsers()
        fun detach()
    }
}

// --- Presenter ---
class UserListPresenter(
    private var view: UserListContract.View?,
    private val repo: UserRepository = UserRepository()
) : UserListContract.Presenter {

    private val scope = CoroutineScope(Dispatchers.Main + SupervisorJob())

    override fun loadUsers() {
        view?.showLoading(true)
        scope.launch {
            repo.getUsers()
                .onSuccess { view?.showUsers(it) }
                .onFailure { view?.showError(it.message ?: "未知错误") }
            view?.showLoading(false)
        }
    }

    override fun detach() {
        view = null
        scope.cancel()
    }
}

// --- View (Activity) ---
class MvpActivity : AppCompatActivity(), UserListContract.View {

    private lateinit var presenter: UserListContract.Presenter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_list)
        presenter = UserListPresenter(this)
        presenter.loadUsers()
    }

    override fun showLoading(show: Boolean) {
        findViewById<ProgressBar>(R.id.progressBar).isVisible = show
    }

    override fun showUsers(users: List<User>) {
        findViewById<RecyclerView>(R.id.recyclerView).adapter = UserAdapter(users)
    }

    override fun showError(msg: String) {
        Toast.makeText(this, msg, Toast.LENGTH_SHORT).show()
    }

    override fun onDestroy() {
        super.onDestroy()
        presenter.detach() // 防止内存泄漏
    }
}
```

**优点：** Presenter 可以用 Mock View 做单元测试。
**痛点：** 接口爆炸——每个页面至少一对 Contract 接口；Presenter 需要手动管理生命周期。

### 3.3 MVVM 实现

```kotlin
// --- ViewModel ---
class UserListViewModel(
    private val repo: UserRepository = UserRepository()
) : ViewModel() {

    private val _loading = MutableStateFlow(false)
    val loading: StateFlow<Boolean> = _loading.asStateFlow()

    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()

    private val _error = MutableSharedFlow<String>()
    val error: SharedFlow<String> = _error.asSharedFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _loading.value = true
            repo.getUsers()
                .onSuccess { _users.value = it }
                .onFailure { _error.emit(it.message ?: "未知错误") }
            _loading.value = false
        }
    }
}

// --- View (Activity) ---
class MvvmActivity : AppCompatActivity() {

    private val viewModel: UserListViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_list)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch {
                    viewModel.users.collect { users ->
                        findViewById<RecyclerView>(R.id.recyclerView)
                            .adapter = UserAdapter(users)
                    }
                }
                launch {
                    viewModel.loading.collect { show ->
                        findViewById<ProgressBar>(R.id.progressBar).isVisible = show
                    }
                }
                launch {
                    viewModel.error.collect { msg ->
                        Toast.makeText(this@MvvmActivity, msg, Toast.LENGTH_SHORT).show()
                    }
                }
            }
        }

        viewModel.loadUsers()
    }
}
```

**优点：** ViewModel 自动感知生命周期（[[1-Jetpack-Lifecycle与ViewModel]]），配置变更时数据不丢失，无需手动 detach。
**注意：** 状态分散在多个 StateFlow 中，复杂页面容易出现状态不一致。

### 3.4 MVI 实现

```kotlin
// --- 单一状态 + Intent ---
data class UserListState(
    val loading: Boolean = false,
    val users: List<User> = emptyList(),
    val error: String? = null
)

sealed class UserListIntent {
    object LoadUsers : UserListIntent()
    object DismissError : UserListIntent()
}

// --- ViewModel (Reducer 风格) ---
class UserListMviViewModel(
    private val repo: UserRepository = UserRepository()
) : ViewModel() {

    private val _state = MutableStateFlow(UserListState())
    val state: StateFlow<UserListState> = _state.asStateFlow()

    fun dispatch(intent: UserListIntent) {
        when (intent) {
            is UserListIntent.LoadUsers -> loadUsers()
            is UserListIntent.DismissError -> _state.update { it.copy(error = null) }
        }
    }

    private fun loadUsers() {
        viewModelScope.launch {
            _state.update { it.copy(loading = true, error = null) }
            repo.getUsers()
                .onSuccess { users ->
                    _state.update { it.copy(loading = false, users = users) }
                }
                .onFailure { e ->
                    _state.update { it.copy(loading = false, error = e.message) }
                }
        }
    }
}

// --- View (Activity) ---
class MviActivity : AppCompatActivity() {

    private val viewModel: UserListMviViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user_list)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.state.collect { state -> render(state) }
            }
        }

        viewModel.dispatch(UserListIntent.LoadUsers)
    }

    private fun render(state: UserListState) {
        findViewById<ProgressBar>(R.id.progressBar).isVisible = state.loading
        if (state.users.isNotEmpty()) {
            findViewById<RecyclerView>(R.id.recyclerView).adapter = UserAdapter(state.users)
        }
        state.error?.let {
            Toast.makeText(this, it, Toast.LENGTH_SHORT).show()
            viewModel.dispatch(UserListIntent.DismissError)
        }
    }
}
```

**优点：** 单一 State 源，任何时刻 UI 状态可预测、可回放；天然适配 [[Jetpack-Compose]]。
**注意：** 一次性事件（Toast / Navigation）需要额外处理，常见方案是 `SharedFlow` side-effect 通道。

### 3.5 Compose + MVI（现代推荐写法）

```kotlin
@Composable
fun UserListScreen(viewModel: UserListMviViewModel = viewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    LaunchedEffect(Unit) {
        viewModel.dispatch(UserListIntent.LoadUsers)
    }

    Box(modifier = Modifier.fillMaxSize()) {
        if (state.loading) {
            CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
        }
        LazyColumn {
            items(state.users) { user ->
                Text(text = user.name, modifier = Modifier.padding(16.dp))
            }
        }
        state.error?.let { error ->
            Snackbar { Text(error) }
        }
    }
}
```

MVI 的单一 State 与 Compose 的声明式 UI 是天然搭档——State 变化自动触发重组。

---

## 四、面试题

### Q1：MVC 在 Android 中为什么不好用？

**A：** 标准 MVC 中 Controller 是独立组件，但 Android 的 Activity/Fragment 天然承担了 View 的职责（`setContentView`、`findViewById`），同时又被当作 Controller 处理用户事件和业务逻辑。这导致：
1. Activity 代码膨胀（"God Activity"），单个文件动辄上千行
2. 业务逻辑与 Android 框架耦合，无法脱离 Instrumentation 做单元测试
3. View 可以直接访问 Model，数据流向混乱

### Q2：MVP 的 Presenter 如何避免内存泄漏？

**A：** Presenter 持有 View（Activity）的引用，如果 Activity 销毁后 Presenter 仍存活（如后台协程未取消），就会泄漏。常见方案：
1. 在 `onDestroy()` 中调用 `presenter.detach()` 置空 View 引用并取消协程
2. 使用弱引用 `WeakReference<IView>` 持有 View
3. 结合 [[1-Jetpack-Lifecycle与ViewModel]] 的 `LifecycleObserver`，让 Presenter 感知生命周期自动解绑

相比之下，MVVM 的 ViewModel 由 Jetpack 管理生命周期，天然不持有 View 引用，从根本上避免了这个问题。

### Q3：MVVM 中 ViewModel 可以持有 Activity 引用吗？

**A：** 绝对不可以。ViewModel 的生命周期长于 Activity（配置变更时 Activity 重建但 ViewModel 存活），持有 Activity 引用会导致内存泄漏。如果需要 Context，应使用 `AndroidViewModel` 持有 Application Context。这也是为什么 Google 推荐用 `StateFlow` / `LiveData` 做观察式通信，而非直接回调。

### Q4：MVVM 和 MVI 的核心区别是什么？

**A：**

| 维度 | MVVM | MVI |
|------|------|-----|
| 状态数量 | 多个可观察字段（多个 LiveData/StateFlow） | 单一不可变 State |
| 数据流 | View 观察多个流 + 调用 ViewModel 方法 | 严格单向：Intent → Reducer → State → View |
| 状态一致性 | 多个字段可能出现中间不一致状态 | 原子更新，任意时刻状态一致 |
| 调试 | 需要分别观察各个流 | 可以 log 每次 State 变化，支持时间旅行调试 |

简单说：MVI 是 MVVM 的"严格模式"，用更强的约束换取更好的可预测性。

### Q5：MVI 中如何处理一次性事件（Toast、导航）？

**A：** 一次性事件（Side Effect）不适合放在 State 中，因为 State 是持久的，屏幕旋转后会重新渲染导致重复触发。常见方案：

1. **独立 SharedFlow 通道：**
```kotlin
class MyViewModel : ViewModel() {
    private val _effect = MutableSharedFlow<SideEffect>()
    val effect: SharedFlow<SideEffect> = _effect.asSharedFlow()
}
```

2. **Channel + receiveAsFlow：** 保证事件不丢失
```kotlin
private val _effect = Channel<SideEffect>(Channel.BUFFERED)
val effect = _effect.receiveAsFlow()
```

3. 在 Compose 中用 `LaunchedEffect` 收集 effect 流，在 View 体系中用 `repeatOnLifecycle` 收集。

### Q6：Google 官方推荐哪种架构？

**A：** Google 官方架构指南（2023 年更新）推荐的是 **UDF（Unidirectional Data Flow，单向数据流）** 模式，本质上是 MVVM + MVI 思想的融合：
- 使用 `ViewModel` + `StateFlow` 暴露 UI State（MVVM 的基础设施）
- UI State 用不可变 data class 表示（MVI 的核心理念）
- View 通过调用 ViewModel 方法触发状态变更（类似 Intent 但不强制密封类）

架构分层：`UI Layer → Domain Layer（可选）→ Data Layer`，参考 [[Android-官方架构指南]]。

---

## 五、实战与踩坑

### 5.1 选型建议

```
项目复杂度低（工具类 App、单页面）
  → MVVM 即可，不需要过度设计

中等复杂度（常规业务 App）
  → MVVM + Repository 模式
  → 关键复杂页面局部使用 MVI

高复杂度（社交/电商/IM，多状态交互）
  → MVI 全面采用
  → 配合 Compose 效果最佳

遗留项目重构
  → 先从 MVC → MVP（成本最低）
  → 再逐步 MVP → MVVM（引入 ViewModel）
  → 最后关键模块 MVVM → MVI
```

### 5.2 常见误区

**误区一：MVVM 就是 DataBinding**
DataBinding 只是 MVVM 的一种实现手段，不是必须的。现代 Android 开发更推荐 `StateFlow` + `ViewBinding` 或直接使用 Compose，DataBinding 的 XML 表达式反而增加调试难度。

**误区二：MVP 已经过时，不需要了解**
MVP 的思想（接口隔离、依赖倒置）是理解后续架构的基础。很多大型项目仍在使用 MVP，面试中也是高频考点。Clean Architecture 中的 UseCase 层本质上也借鉴了 Presenter 的思路。

**误区三：MVI 中所有东西都放进 State**
一次性事件（导航、Toast、SnackBar）不应放入 State，否则会在配置变更时重复触发。应使用独立的 Side Effect 通道（见 Q5）。

**误区四：ViewModel 中可以直接操作 View**
ViewModel 不应持有任何 View / Activity / Fragment 的引用。数据传递必须通过可观察的数据流（LiveData / StateFlow）。违反这一原则会导致内存泄漏和测试困难。

**误区五：架构越新越好**
架构选型应基于团队能力和项目需求，而非追新。一个 3 人团队的小项目强行上 MVI + Clean Architecture 只会增加开发成本。MVVM 对于大多数 Android 项目已经足够。

### 5.3 架构演进实战路径

```
第一步：引入 ViewModel + LiveData/StateFlow
  → 解决 Activity 臃肿问题
  → 解决配置变更数据丢失问题

第二步：引入 Repository 模式
  → 统一数据源管理（网络 + 本地缓存）
  → 参考 [[Android-Repository模式]]

第三步：引入 UseCase / Interactor（可选）
  → 复杂业务逻辑抽离，ViewModel 保持精简
  → 参考 [[2-Clean-Architecture]]

第四步：关键页面升级 MVI
  → 复杂表单、多步骤流程、实时协作等场景
  → 引入 State + Intent + Reducer 三件套

第五步：迁移 Compose（可选）
  → Compose + MVI 是声明式 UI 的最佳实践
  → 参考 [[Jetpack-Compose]]
```

### 5.4 测试策略对比

| 架构 | 单元测试目标 | 测试难度 | 关键工具 |
|------|-------------|---------|---------|
| MVC | 几乎无法测试业务逻辑 | 极高 | Robolectric（勉强） |
| MVP | Presenter（Mock IView） | 中 | Mockito / MockK |
| MVVM | ViewModel（观察 StateFlow） | 低 | Turbine + Coroutines Test |
| MVI | Reducer 纯函数 + ViewModel | 最低 | 直接断言 State 变化 |

---

> 📌 **总结一句话：** MVC 是起点，MVP 解耦了 View，MVVM 用数据绑定消除了模板代码，MVI 用单向数据流解决了状态一致性。理解演进逻辑比死记代码更重要。

---

*最后更新：2026-04-15*

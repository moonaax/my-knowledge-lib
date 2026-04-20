# Jetpack - LiveData 与 Flow

## 一、LiveData

### 1.1 是什么？

LiveData 是一个**可观察的数据持有类**，具有生命周期感知能力。它只在观察者处于活跃状态（STARTED 或 RESUMED）时才通知数据变化。

````java
// ViewModel 中
public class MyViewModel extends ViewModel {
    private final MutableLiveData<String> _name = new MutableLiveData<>();
    public LiveData<String> getName() { return _name; }  // 对外暴露不可变的

    public void updateName(String newName) {
        _name.setValue(newName);        // 主线程
        _name.postValue(newName);       // 子线程
    }
}

// Activity 中观察
viewModel.getName().observe(this, name -> {
    textView.setText(name);  // 自动在主线程回调
});
````
### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| 生命周期感知 | 只在 STARTED/RESUMED 状态下通知观察者 |
| 自动取消订阅 | 观察者的 LifecycleOwner 销毁时自动移除 |
| 不会内存泄漏 | 因为自动取消订阅 |
| 不会因 Activity 停止而崩溃 | 不活跃的观察者不会收到通知 |
| 配置变更后自动获取最新值 | Activity 重建后重新观察，立即收到最新数据 |

### 1.3 原理

````java
// LiveData 核心源码简化
public abstract class LiveData<T> {
    private int mVersion = -1;  // 数据版本号
    private T mData;            // 存储的数据
    private Map<Observer, ObserverWrapper> mObservers;  // 观察者列表

    public void observe(LifecycleOwner owner, Observer<T> observer) {
        // 包装观察者，绑定生命周期
        LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
        mObservers.put(observer, wrapper);
        owner.getLifecycle().addObserver(wrapper);
    }

    protected void setValue(T value) {
        mVersion++;       // 版本号 +1
        mData = value;
        dispatchingValue(null);  // 通知所有活跃的观察者
    }

    private void considerNotify(ObserverWrapper observer) {
        // 1. 检查观察者是否活跃
        if (!observer.shouldBeActive()) return;
        // 2. 检查版本号（避免重复通知）
        if (observer.mLastVersion >= mVersion) return;
        // 3. 更新版本号并通知
        observer.mLastVersion = mVersion;
        observer.mObserver.onChanged(mData);
    }
}
````
**关键流程：**
1. `observe()` 时将 Observer 包装成 LifecycleBoundObserver，注册到 Lifecycle
2. `setValue()` 时版本号 +1，遍历所有观察者，只通知活跃的
3. 生命周期变化时（比如从 STOPPED → STARTED），检查是否有新数据需要通知

### 1.4 粘性事件问题

**什么是粘性事件？**

LiveData 在新观察者注册时，会立即把最新的值发送给它。这在某些场景下是个问题：

````java
// ViewModel
MutableLiveData<String> errorEvent = new MutableLiveData<>();

void doSomething() {
    // 操作失败，发送错误事件
    errorEvent.setValue("网络错误");
}

// Fragment A 观察并弹了 Toast
// 然后跳转到 Fragment B
// Fragment B 也观察了 errorEvent
// 问题：Fragment B 一注册就收到了"网络错误"，又弹了一次 Toast！
````
**原因：** 新观察者的 `mLastVersion = -1`，而 LiveData 的 `mVersion > -1`，所以 `considerNotify` 判断有新数据，立即通知。

**解决方案：**

````java
// 方案一：SingleLiveEvent（Google 官方示例）
public class SingleLiveEvent<T> extends MutableLiveData<T> {
    private final AtomicBoolean pending = new AtomicBoolean(false);

    @Override
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        super.observe(owner, t -> {
            if (pending.compareAndSet(true, false)) {
                observer.onChanged(t);
            }
        });
    }

    @Override
    public void setValue(T t) {
        pending.set(true);
        super.setValue(t);
    }
}

// 方案二：Event 包装类
public class Event<T> {
    private final T content;
    private boolean hasBeenHandled = false;

    public Event(T content) { this.content = content; }

    public T getContentIfNotHandled() {
        if (hasBeenHandled) return null;
        hasBeenHandled = true;
        return content;
    }
}

// 使用
MutableLiveData<Event<String>> errorEvent = new MutableLiveData<>();
errorEvent.setValue(new Event<>("网络错误"));

// 观察
viewModel.errorEvent.observe(this, event -> {
    String msg = event.getContentIfNotHandled();
    if (msg != null) {
        Toast.makeText(this, msg, Toast.LENGTH_SHORT).show();
    }
});

// 方案三：用 SharedFlow（推荐，见下文）
````
### 1.5 setValue vs postValue

| 方法 | 线程 | 时机 |
|------|------|------|
| `setValue()` | 必须主线程 | 同步立即生效 |
| `postValue()` | 任意线程 | 通过 Handler post 到主线程，异步生效 |

**postValue 的坑：** 短时间内多次调用 postValue，只有最后一次的值会被分发。

````java
// ❌ 中间的值会丢失
liveData.postValue("A");
liveData.postValue("B");
liveData.postValue("C");
// 观察者只收到 "C"

// 原因：postValue 内部用一个 pending 变量，
// 多次调用只是更新 pending 的值，只 post 一次 Runnable
````
### 1.6 Transformations

````java
// map：转换数据类型
LiveData<String> userName = Transformations.map(userLiveData, user ->
    user.getFirstName() + " " + user.getLastName()
);

// switchMap：根据输入切换数据源
LiveData<List<User>> searchResults = Transformations.switchMap(queryLiveData, query ->
    repository.search(query)  // 返回新的 LiveData
);

// MediatorLiveData：合并多个 LiveData
MediatorLiveData<String> mediator = new MediatorLiveData<>();
mediator.addSource(liveData1, value -> mediator.setValue("Source1: " + value));
mediator.addSource(liveData2, value -> mediator.setValue("Source2: " + value));
````
---

## 二、Flow

### 2.1 是什么？

Flow 是 Kotlin 协程提供的**异步数据流**，可以按顺序发射多个值（LiveData 只能持有一个值）。

````kotlin
// 创建 Flow
fun getNumbers(): Flow<Int> = flow {
    for (i in 1..5) {
        delay(1000)
        emit(i)  // 发射值
    }
}

// 收集 Flow
lifecycleScope.launch {
    getNumbers().collect { number ->
        println(number)  // 1, 2, 3, 4, 5（每秒一个）
    }
}
````
### 2.2 Flow vs LiveData

| 对比项 | LiveData | Flow |
|--------|----------|------|
| 平台 | Android 专属 | Kotlin 通用（跨平台） |
| 线程 | 只在主线程观察 | 可以在任意调度器上 |
| 操作符 | 少（map、switchMap） | 丰富（map、filter、flatMap、zip、combine...） |
| 生命周期感知 | 内置 | 需要配合 repeatOnLifecycle |
| 粘性 | 有（新观察者收到最新值） | StateFlow 有，SharedFlow 可配置 |
| 背压处理 | 无 | 有（buffer、conflate、collectLatest） |
| 适用场景 | 简单的 UI 状态 | 复杂的数据流处理 |

### 2.3 StateFlow

StateFlow 是 Flow 的一种，类似于 LiveData，持有一个当前值，新的收集者会立即收到最新值。

````kotlin
class MyViewModel : ViewModel() {
    // StateFlow 必须有初始值
    private val _uiState = MutableStateFlow(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val data = repository.getData()
                _uiState.value = UiState.Success(data)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}

sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<Item>) : UiState()
    data class Error(val message: String?) : UiState()
}
````
**StateFlow vs LiveData：**

| 对比项 | StateFlow | LiveData |
|--------|-----------|----------|
| 初始值 | 必须有 | 可以没有 |
| 空安全 | 不允许 null（除非声明为 nullable） | 允许 null |
| 相同值 | 不通知（distinctUntilChanged） | 每次 setValue 都通知 |
| 生命周期 | 需要手动处理 | 自动感知 |

### 2.4 SharedFlow

SharedFlow 是更灵活的热流，可以配置重放（replay）和缓冲。

````kotlin
class MyViewModel : ViewModel() {
    // 一次性事件（不重放，类似 SingleLiveEvent）
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()

    fun showToast(message: String) {
        viewModelScope.launch {
            _events.emit(UiEvent.ShowToast(message))
        }
    }
}

// 收集
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.ShowToast -> toast(event.message)
            }
        }
    }
}
````
**SharedFlow 的参数：**
````kotlin
MutableSharedFlow<T>(
    replay = 0,           // 新收集者能收到几个历史值（0 = 不重放 = 解决粘性问题）
    extraBufferCapacity = 0,  // 额外缓冲区大小
    onBufferOverflow = BufferOverflow.SUSPEND  // 缓冲区满时的策略
)
````
| replay | 行为 | 等价于 |
|--------|------|--------|
| 0 | 新收集者不收到历史值 | SingleLiveEvent |
| 1 | 新收集者收到最新一个值 | 类似 LiveData |
| N | 新收集者收到最近 N 个值 | — |

### 2.5 在 UI 中安全收集 Flow

````kotlin
// ❌ 错误：Activity 进入后台后 Flow 还在收集，浪费资源
lifecycleScope.launch {
    viewModel.uiState.collect { state ->
        updateUI(state)
    }
}

// ✅ 正确：配合 repeatOnLifecycle，只在 STARTED 及以上状态收集
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            updateUI(state)
        }
    }
}

// ✅ 简写（扩展函数）
viewModel.uiState
    .flowWithLifecycle(lifecycle, Lifecycle.State.STARTED)
    .onEach { state -> updateUI(state) }
    .launchIn(lifecycleScope)
````
### 2.6 常用操作符

````kotlin
// map：转换
flow.map { it.toString() }

// filter：过滤
flow.filter { it > 0 }

// flatMapLatest：切换到新流时取消旧流（类似 switchMap）
searchQuery.flatMapLatest { query ->
    repository.search(query)
}

// combine：合并多个 Flow
combine(flow1, flow2) { a, b -> "$a - $b" }

// debounce：防抖（搜索框常用）
searchQuery.debounce(300)  // 300ms 内没有新值才发射

// distinctUntilChanged：去重
flow.distinctUntilChanged()

// catch：异常处理
flow.catch { e -> emit(defaultValue) }

// flowOn：切换上游的调度器
flow.flowOn(Dispatchers.IO)

// collectLatest：新值来时取消旧的收集
flow.collectLatest { value ->
    // 如果新值来了，这里的协程会被取消
    val result = heavyComputation(value)
    updateUI(result)
}
````
### 2.7 冷流 vs 热流

| 对比项 | 冷流（Flow） | 热流（StateFlow/SharedFlow） |
|--------|-------------|---------------------------|
| 创建时机 | 被 collect 时才开始执行 | 创建时就存在 |
| 多个收集者 | 每个收集者独立执行一遍 | 共享同一个数据源 |
| 有无状态 | 无状态 | 有状态（持有当前值） |
| 类比 | 点播视频 | 直播 |

````kotlin
// 冷流：每次 collect 都重新执行
val coldFlow = flow {
    println("开始执行")
    emit(1)
}
coldFlow.collect { }  // 打印"开始执行"
coldFlow.collect { }  // 又打印"开始执行"

// 热流：只执行一次，多个收集者共享
val hotFlow = MutableStateFlow(0)
// 收集者1 和 收集者2 收到的是同一个流的数据
````
---

## 三、实际项目中的选择

````
简单 UI 状态（加载中/成功/失败）→ StateFlow
一次性事件（Toast、导航、弹窗）→ SharedFlow(replay=0)
简单场景 + 不想引入协程 → LiveData
复杂数据流（搜索、多数据源合并）→ Flow + 操作符
````
---

## 四、面试题库

### Q1：LiveData 的原理是什么？

> LiveData 内部维护了一个数据版本号 mVersion 和观察者列表。每次 setValue 时版本号 +1，然后遍历所有观察者，只通知处于活跃状态（STARTED 及以上）的观察者。观察者通过 LifecycleBoundObserver 绑定了 LifecycleOwner 的生命周期，当生命周期变为 DESTROYED 时自动移除观察者，避免内存泄漏。当观察者从非活跃变为活跃时（比如 Activity 从后台回到前台），会检查版本号，如果有新数据就立即通知，这就是为什么回到页面能看到最新数据的原因。

### Q2：LiveData 的粘性事件是什么？怎么解决？

> LiveData 在新观察者注册时会立即发送当前持有的最新值，这就是粘性事件。原因是新观察者的 lastVersion 初始为 -1，而 LiveData 的 mVersion 大于 -1，considerNotify 判断有新数据就通知了。对于一次性事件（Toast、导航）这是个问题。解决方案有三种：一是用 Event 包装类，通过 hasBeenHandled 标记确保只消费一次；二是用 SingleLiveEvent（Google 示例），通过 AtomicBoolean 控制；三是用 SharedFlow(replay=0) 替代，新收集者不会收到历史值，这是目前最推荐的方案。

### Q3：setValue 和 postValue 的区别？postValue 有什么坑？

> setValue 必须在主线程调用，同步立即生效。postValue 可以在任意线程调用，内部通过 Handler post 到主线程异步执行。postValue 的坑是短时间内多次调用只有最后一次生效——因为内部用一个 mPendingData 变量暂存值，多次调用只是更新这个变量，只 post 一次 Runnable 到主线程。所以如果需要每个值都被观察到，应该切到主线程用 setValue，或者用 Flow 的 emit。

### Q4：StateFlow 和 LiveData 的区别？什么时候用哪个？

> StateFlow 必须有初始值，LiveData 可以没有。StateFlow 默认 distinctUntilChanged，相同值不通知；LiveData 每次 setValue 都通知。StateFlow 是 Kotlin 协程的一部分，跨平台可用；LiveData 是 Android 专属。StateFlow 没有内置生命周期感知，需要配合 repeatOnLifecycle 使用。选择上，如果项目已经全面使用协程，推荐 StateFlow + SharedFlow 的组合；如果是简单项目或者团队对协程不熟悉，LiveData 完全够用。我个人倾向于 UI 状态用 StateFlow，一次性事件用 SharedFlow。

### Q5：SharedFlow 和 StateFlow 的区别？

> StateFlow 是 SharedFlow 的特化版本，等价于 SharedFlow(replay=1, onBufferOverflow=DROP_OLDEST) 加上 distinctUntilChanged。StateFlow 必须有初始值且始终持有一个当前值，适合表示 UI 状态。SharedFlow 更灵活，replay 可以设为 0（不重放，解决粘性问题），适合一次性事件。StateFlow 的 value 属性可以直接读取当前值，SharedFlow 没有这个属性。简单说，StateFlow 是"状态"，SharedFlow 是"事件"。

### Q6：Flow 的 collect 和 collectLatest 有什么区别？

> collect 会等当前值处理完才处理下一个值，如果处理耗时，新值会排队等待。collectLatest 在新值到来时会取消当前正在处理的协程，立即开始处理新值。典型场景是搜索框——用户快速输入时，collectLatest 会取消之前的搜索请求，只执行最新的。如果用 collect，每次输入都会发起搜索请求并等待完成，造成不必要的网络开销和 UI 延迟。

### Q7：repeatOnLifecycle 是什么？为什么需要它？

> repeatOnLifecycle 是一个挂起函数，它在指定的生命周期状态下启动协程块，在低于该状态时取消。比如 repeatOnLifecycle(STARTED) 会在 onStart 时启动 collect，在 onStop 时取消，下次 onStart 时重新启动。如果不用它，直接在 lifecycleScope.launch 中 collect，Activity 进入后台后 Flow 还在收集，浪费资源甚至可能更新已不可见的 UI。它本质上是把 Flow 的收集和生命周期绑定，实现了类似 LiveData 的生命周期感知能力。

### Q8：冷流和热流的区别？

> 冷流（Flow）是惰性的，只有被 collect 时才开始执行，每个收集者都会触发独立的执行。热流（StateFlow/SharedFlow）是主动的，不管有没有收集者都可以发射数据，多个收集者共享同一个数据源。类比的话，冷流像点播视频，每个人看到的是独立的播放；热流像直播，所有人看到的是同一个画面。在 ViewModel 中通常用热流持有 UI 状态，在 Repository 层用冷流封装数据请求。

### Q9：Flow 的 flowOn 和 collect 的调度器有什么关系？

> flowOn 改变的是上游（它之前的操作符）的执行调度器，不影响下游。collect 的调度器由启动协程的 scope 决定。比如 flow { emit(heavyWork()) }.flowOn(Dispatchers.IO).collect { updateUI(it) }，heavyWork 在 IO 线程执行，collect 的 lambda 在启动协程的线程（通常是 Main）执行。这和 RxJava 的 subscribeOn/observeOn 类似，但 flowOn 只影响上游，而且可以多次调用，每次只影响它和上一个 flowOn 之间的操作符。

### Q10：如何用 Flow 实现搜索防抖？

> 经典的搜索防抖实现：把 EditText 的输入转成 Flow，然后用 debounce + distinctUntilChanged + flatMapLatest 组合。debounce(300) 确保用户停止输入 300ms 后才发起搜索；distinctUntilChanged 过滤掉相同的搜索词；flatMapLatest 在新搜索词到来时取消上一次的搜索请求。

````kotlin
private val _query = MutableStateFlow("")

val searchResults = _query
    .debounce(300)
    .distinctUntilChanged()
    .filter { it.isNotBlank() }
    .flatMapLatest { query ->
        repository.search(query)
            .catch { emit(emptyList()) }
    }
    .stateIn(viewModelScope, SharingStarted.Lazily, emptyList())
````
> 这比用 Handler.postDelayed 实现的防抖更优雅，而且自动处理了取消和异常。stateIn 把冷流转成 StateFlow，避免每次 collect 都重新执行。

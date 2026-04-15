# Jetpack - Lifecycle 与 ViewModel

## 一、Lifecycle

### 1.1 是什么？

Lifecycle 是 Jetpack 提供的生命周期感知组件。它让你的类能够**自动感知** Activity/Fragment 的生命周期变化，不需要在 Activity 的每个生命周期回调里手动调用。

**没有 Lifecycle 之前：**
```java
// 到处都是手动管理
public class MyActivity extends AppCompatActivity {
    private MyLocationManager locationManager = new MyLocationManager();

    @Override
    protected void onStart() {
        super.onStart();
        locationManager.start();  // 手动启动
    }

    @Override
    protected void onStop() {
        super.onStop();
        locationManager.stop();  // 手动停止
    }
}
```

**有了 Lifecycle 之后：**
```java
// LocationManager 自己管理生命周期
public class MyLocationManager implements DefaultLifecycleObserver {
    public MyLocationManager(Lifecycle lifecycle) {
        lifecycle.addObserver(this);
    }

    @Override
    public void onStart(@NonNull LifecycleOwner owner) {
        start();  // 自动启动
    }

    @Override
    public void onStop(@NonNull LifecycleOwner owner) {
        stop();  // 自动停止
    }
}

// Activity 只需要一行
public class MyActivity extends AppCompatActivity {
    private MyLocationManager locationManager = new MyLocationManager(getLifecycle());
}
```

### 1.2 核心概念

| 概念 | 说明 |
|------|------|
| `LifecycleOwner` | 生命周期的拥有者（Activity、Fragment 都实现了这个接口） |
| `Lifecycle` | 生命周期对象，从 LifecycleOwner 获取 |
| `LifecycleObserver` | 生命周期观察者，注册后自动收到生命周期回调 |
| `State` | 生命周期状态：INITIALIZED、CREATED、STARTED、RESUMED、DESTROYED |
| `Event` | 生命周期事件：ON_CREATE、ON_START、ON_RESUME、ON_PAUSE、ON_STOP、ON_DESTROY |

**State 和 Event 的关系：**
```
    INITIALIZED ──ON_CREATE──→ CREATED ──ON_START──→ STARTED ──ON_RESUME──→ RESUMED
                                  ↑                     ↑                      │
                              ON_STOP              ON_PAUSE                ON_PAUSE
                                  │                     │                      │
                              CREATED ←──ON_STOP──── STARTED ←──ON_PAUSE──── RESUMED
                                  │
                              ON_DESTROY
                                  │
                              DESTROYED
```

### 1.3 原理

**Activity 的 Lifecycle 实现：**

ComponentActivity（AppCompatActivity 的父类）实现了 LifecycleOwner 接口。它通过一个**无 UI 的 Fragment**（ReportFragment）来监听生命周期事件。

```java
// ComponentActivity.java
public class ComponentActivity extends Activity implements LifecycleOwner {
    private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}

// ReportFragment 在 Activity 的 onCreate 中被添加
// 它的生命周期回调中会通知 LifecycleRegistry 状态变化
```

**LifecycleRegistry：**
- Lifecycle 的实现类，维护当前状态和观察者列表
- 状态变化时遍历观察者，按顺序分发事件
- 新添加的观察者会收到"追赶事件"（比如在 onResume 中添加观察者，会依次收到 ON_CREATE、ON_START、ON_RESUME）

### 1.4 自定义 LifecycleOwner

```java
// 比如让一个 Service 具有生命周期感知能力
public class MyService extends Service implements LifecycleOwner {
    private final LifecycleRegistry lifecycleRegistry = new LifecycleRegistry(this);

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return lifecycleRegistry;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        lifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_DESTROY);
    }
}
```

---

## 二、ViewModel

### 2.1 是什么？

ViewModel 是专门用来**存储和管理 UI 相关数据**的类。它的核心特性：

1. **配置变更（如旋转屏幕）时不会被销毁**
2. 生命周期比 Activity/Fragment 更长
3. 不持有 View 或 Activity 的引用（避免内存泄漏）

```java
public class MyViewModel extends ViewModel {
    private final MutableLiveData<List<User>> _users = new MutableLiveData<>();
    public LiveData<List<User>> getUsers() { return _users; }

    public void loadUsers() {
        // 在子线程加载数据
        Executors.newSingleThreadExecutor().execute(() -> {
            List<User> users = repository.getUsers();
            _users.postValue(users);
        });
    }

    @Override
    protected void onCleared() {
        // ViewModel 被销毁时调用，释放资源
    }
}

// Activity 中使用
public class MyActivity extends AppCompatActivity {
    private MyViewModel viewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        viewModel = new ViewModelProvider(this).get(MyViewModel.class);
        viewModel.getUsers().observe(this, users -> {
            // 更新 UI
        });
        viewModel.loadUsers();
    }
}
```

### 2.2 为什么旋转屏幕后 ViewModel 还在？

这是面试高频题，必须搞清楚。

```
旋转屏幕前：
Activity A（实例1）──持有──→ ViewModelStore ──持有──→ MyViewModel

旋转屏幕：Activity 销毁重建
Activity A（实例1）→ onDestroy()（但 ViewModelStore 没有被清除！）
Activity A（实例2）→ onCreate()（从 ViewModelStore 中取回 MyViewModel）

用户按返回键退出：
Activity A → onDestroy()（ViewModelStore 被清除，ViewModel.onCleared() 被调用）
```

**关键：ViewModelStore 存在哪里？**

在 `ComponentActivity` 中，ViewModelStore 通过 `NonConfigurationInstances` 在配置变更时保留：

```java
// Activity 销毁前
onRetainNonConfigurationInstance() {
    // 把 ViewModelStore 保存到 NonConfigurationInstances 中
    return new NonConfigurationInstances(viewModelStore);
}

// Activity 重建后
onCreate() {
    // 从 NonConfigurationInstances 中恢复 ViewModelStore
    NonConfigurationInstances nc = getLastNonConfigurationInstance();
    if (nc != null) {
        viewModelStore = nc.viewModelStore;  // 拿回来了！
    }
}
```

`NonConfigurationInstances` 是 ActivityThread 在销毁 Activity 时特意保留的对象，不会因为配置变更而丢失。

### 2.3 ViewModel 的作用域

| 获取方式 | 作用域 | 使用场景 |
|---------|--------|---------|
| `by viewModels()` | 当前 Activity/Fragment | 页面私有数据 |
| `by activityViewModels()` | 宿主 Activity | Fragment 之间共享数据 |
| `by navGraphViewModels(R.id.nav_graph)` | Navigation 图 | 导航流程中共享数据 |

```java
// Fragment 中
public class FragmentA extends Fragment {
    // 私有 ViewModel（FragmentA 独有）
    private PrivateViewModel privateVM;

    // 共享 ViewModel（和同一 Activity 下的其他 Fragment 共享）
    private SharedViewModel sharedVM;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        privateVM = new ViewModelProvider(this).get(PrivateViewModel.class);
        sharedVM = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
    }
}
```

### 2.4 SavedStateHandle

ViewModel 在配置变更时存活，但**进程被杀死后会丢失**。SavedStateHandle 解决了这个问题。

```java
public class MyViewModel extends ViewModel {
    private final SavedStateHandle savedStateHandle;

    public MyViewModel(SavedStateHandle savedStateHandle) {
        this.savedStateHandle = savedStateHandle;
    }

    // 自动保存和恢复，即使进程被杀
    public LiveData<String> getSearchQuery() {
        return savedStateHandle.getLiveData("query");
    }

    public void setQuery(String query) {
        savedStateHandle.set("query", query);
    }
}
```

**SavedStateHandle vs onSaveInstanceState：**

| 对比项 | onSaveInstanceState | SavedStateHandle |
|--------|-------------------|-----------------|
| 存储位置 | Activity 的 Bundle | ViewModel 中 |
| 数据类型 | Bundle 支持的类型 | 同 Bundle |
| 大小限制 | 约 500KB | 同 Bundle |
| 使用方式 | Activity/Fragment 中手动保存恢复 | ViewModel 中自动管理 |

### 2.5 ViewModel 中不能持有 Context

```java
// ❌ 错误：持有 Activity 引用会内存泄漏
public class MyViewModel extends ViewModel {
    private final Activity activity; // 不要这样做！
}

// ✅ 如果需要 Context，用 AndroidViewModel
public class MyViewModel extends AndroidViewModel {
    public MyViewModel(@NonNull Application application) {
        super(application);
    }

    // getApplication() 返回 Application Context，不会泄漏
    public void doSomething() {
        Context context = getApplication();
    }
}
```

---

## 三、面试题库

### Q1：Lifecycle 的原理是什么？

> Lifecycle 的核心是观察者模式。ComponentActivity 实现了 LifecycleOwner 接口，内部持有一个 LifecycleRegistry 对象来管理生命周期状态和观察者列表。生命周期事件的感知通过 ReportFragment 实现——它是一个无 UI 的 Fragment，被添加到 Activity 中，利用 Fragment 的生命周期回调来感知 Activity 的状态变化，然后通知 LifecycleRegistry 更新状态并分发事件给所有观察者。新注册的观察者会收到追赶事件，比如在 RESUMED 状态下注册的观察者会依次收到 ON_CREATE、ON_START、ON_RESUME，确保状态同步。

### Q2：ViewModel 为什么能在屏幕旋转后存活？

> 关键在于 ViewModelStore 的保存机制。Activity 在配置变更导致的销毁前，会调用 onRetainNonConfigurationInstance() 把 ViewModelStore 保存到 NonConfigurationInstances 对象中。这个对象由 ActivityThread 持有，不会因为配置变更而销毁。Activity 重建后在 onCreate 中通过 getLastNonConfigurationInstance() 取回 ViewModelStore，从中获取之前的 ViewModel 实例。所以 ViewModel 实例自始至终没有被销毁，只是 Activity 的壳换了。但如果是用户主动退出（按返回键）或系统杀死进程，ViewModelStore 会被清除，ViewModel.onCleared() 会被调用。

### Q3：ViewModel 的 onCleared 什么时候调用？

> 当 ViewModel 的宿主（Activity/Fragment）真正被销毁且不会重建时调用。具体来说，Activity 的 onDestroy 中会判断 isChangingConfigurations——如果是配置变更导致的销毁，不清除 ViewModel；如果是 finish 或系统回收导致的销毁，才调用 ViewModelStore.clear()，进而触发每个 ViewModel 的 onCleared。在 onCleared 中应该取消协程、关闭流、释放资源。

### Q4：ViewModel 为什么不能持有 Activity 或 View 的引用？

> 因为 ViewModel 的生命周期比 Activity 长。屏幕旋转时旧 Activity 被销毁，但 ViewModel 还在。如果 ViewModel 持有旧 Activity 的引用，旧 Activity 就无法被 GC 回收，造成内存泄漏。同理也不能持有 View、Fragment 或任何间接引用 Activity 的对象。如果确实需要 Context，用 AndroidViewModel 获取 Application Context，因为 Application 的生命周期和进程一样长，不存在泄漏问题。

### Q5：viewModels() 和 activityViewModels() 的区别？

> viewModels() 的作用域是当前 Fragment，每个 Fragment 实例有自己独立的 ViewModel。activityViewModels() 的作用域是宿主 Activity，同一个 Activity 下的所有 Fragment 共享同一个 ViewModel 实例。实际项目中，Fragment 之间需要通信时用 activityViewModels 共享 ViewModel 是最推荐的方式，比接口回调和 EventBus 都更优雅，而且天然具有生命周期安全性。

### Q6：SavedStateHandle 是什么？和 onSaveInstanceState 有什么区别？

> SavedStateHandle 是 ViewModel 中用于保存和恢复状态的工具，底层也是基于 Bundle 机制。它解决的问题是：ViewModel 在配置变更时存活，但进程被杀后会丢失。SavedStateHandle 的数据会随 Activity 的 savedInstanceState 一起保存到磁盘，进程恢复后自动还原。和 onSaveInstanceState 的区别主要在使用位置——SavedStateHandle 在 ViewModel 中使用，数据管理更集中；onSaveInstanceState 在 Activity/Fragment 中使用。两者底层共享同一个 Bundle，大小限制也一样（约 500KB）。

### Q7：如何自定义 LifecycleOwner？

> 实现 LifecycleOwner 接口，内部创建 LifecycleRegistry 实例，在合适的时机调用 handleLifecycleEvent 分发生命周期事件。典型场景是让 Service 具有生命周期感知能力，在 onCreate 中分发 ON_CREATE，在 onDestroy 中分发 ON_DESTROY。自定义 View 也可以实现 LifecycleOwner，在 onAttachedToWindow 和 onDetachedFromWindow 中分发事件。这样依赖生命周期的组件（如 LiveData、协程 scope）就能在自定义组件中正常工作。

### Q8：ViewModel 内部怎么启动协程？viewModelScope 是什么？

> viewModelScope 是 ViewModel 的扩展属性，返回一个 CoroutineScope，绑定了 Dispatchers.Main.immediate 调度器和 SupervisorJob。它的关键特性是在 ViewModel.onCleared() 时自动取消所有协程，避免泄漏。使用方式是在 ViewModel 中直接 viewModelScope.launch { } 启动协程。底层实现是通过 ViewModel 的 closeables 列表，onCleared 时遍历关闭所有 Closeable，viewModelScope 的 Job 就是其中之一。

### Q9：多个 Fragment 共享 ViewModel 时，数据是怎么同步的？

> 多个 Fragment 通过 activityViewModels() 获取的是同一个 ViewModel 实例（因为 ViewModelStore 是 Activity 级别的）。数据同步通过 LiveData 或 StateFlow 实现——一个 Fragment 修改 ViewModel 中的 LiveData 值，其他正在观察这个 LiveData 的 Fragment 会自动收到通知并更新 UI。这是观察者模式的典型应用。需要注意的是，只有处于 STARTED 及以上状态的 Fragment 才会收到 LiveData 的更新，不可见的 Fragment 不会收到，等它重新可见时会收到最新值。

### Q10：ViewModel 和 onRetainNonConfigurationInstance 的关系？

> ViewModel 的存活机制就是建立在 onRetainNonConfigurationInstance 之上的。这是 Activity 提供的一个方法，允许在配置变更时保留一个任意对象。ComponentActivity 重写了这个方法，把 ViewModelStore 作为 NonConfigurationInstances 的一部分保存下来。Activity 重建后通过 getLastNonConfigurationInstance 取回。所以 ViewModel 并不是什么黑魔法，它就是利用了 Activity 框架本身提供的配置变更数据保留机制，只是做了更好的封装和生命周期管理。

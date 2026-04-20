# LeakCanary

> LeakCanary 是 Square 开源的 Android [[内存泄漏]]自动检测库，能在开发阶段自动发现 Activity、Fragment 等对象的泄漏，并给出完整的 GC Root 引用链，是 [[2-内存优化]] 的核心工具。

---

## 一、核心概念

### 1.1 LeakCanary 的作用

LeakCanary 解决的核心问题：**Android 应用中对象该被回收却未被回收（内存泄漏）**。

典型泄漏场景：
- Activity/Fragment 销毁后仍被静态变量、单例、Handler、匿名内部类持有
- View 被长生命周期对象引用
- 未反注册的 Listener / Callback

LeakCanary 的价值：
1. **自动检测**：无需手动触发，Activity/Fragment 销毁时自动监控
2. **精准定位**：给出从泄漏对象到 GC Root 的完整引用链
3. **零侵入**：Debug 包只需添加依赖，无需写任何初始化代码

### 1.2 WeakReference + ReferenceQueue 检测原理

这是 LeakCanary 最核心的检测机制，理解它需要先理解 [[Java 引用类型]]：

```java
// 核心原理演示
ReferenceQueue<Object> queue = new ReferenceQueue<>();
// 用 WeakReference 包装被监控对象，并关联 ReferenceQueue
WeakReference<Activity> weakRef = new WeakReference<>(activity, queue);

// 当 activity 只剩 WeakReference 引用时，GC 后：
// weakRef 会被加入 queue 中
// 如果 GC 后 queue 中没有 weakRef —— 说明 activity 还被强引用持有 —— 泄漏！
```

**检测流程**：

```
对象销毁 (onDestroy)
    │
    ▼
用 WeakReference + ReferenceQueue 包装该对象
    │
    ▼
等待 5 秒（给 GC 充足时间）
    │
    ▼
检查 ReferenceQueue 中是否出现该 WeakReference
    │
    ├── 出现 → 对象已被回收，无泄漏 ✅
    │
    └── 未出现 → 手动触发 GC，再检查一次
                    │
                    ├── 出现 → 无泄漏 ✅
                    └── 仍未出现 → 判定为泄漏 🔴 → dump heap
```

### 1.3 泄漏对象的 GC Root 路径

当判定泄漏后，LeakCanary 会 dump 堆内存（`.hprof` 文件），然后用 Shark 库分析出从 **GC Root** 到泄漏对象的最短引用链。

常见的 GC Root 类型：
| GC Root 类型 | 说明 |
|---|---|
| 静态变量 | `static` 字段持有的引用 |
| 线程栈变量 | 活跃线程的局部变量 |
| JNI 引用 | Native 层持有的全局引用 |
| Monitor 锁 | `synchronized` 持有的对象 |

LeakCanary 输出示例：

```
┬───
│ GC Root: Thread object
│
├─ com.example.MyThread instance
│    Leaking: NO (Thread is alive)
│    ↓ MyThread.callback
│
├─ com.example.MyCallback instance
│    Leaking: UNKNOWN
│    ↓ MyCallback.activity
│
╰→ com.example.MainActivity instance
│    Leaking: YES (Activity#onDestroy() was called)
```

这条链清晰地告诉我们：`MyThread` → `MyCallback` → `MainActivity`，修复方向是断开 callback 对 Activity 的强引用。

---

## 二、原理与源码

### 2.1 ObjectWatcher 监控流程

`ObjectWatcher` 是 LeakCanary 的核心监控器，负责跟踪所有需要被回收的对象。

```java
// ObjectWatcher 核心逻辑（简化版）
public class ObjectWatcher {

    // 存储所有被监控对象的 key → KeyedWeakReference 映射
    private final Map<String, KeyedWeakReference> watchedObjects = new ConcurrentHashMap<>();
    private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

    /**
     * 监控一个预期会被回收的对象
     */
    public void expectWeaklyReachable(Object watchedObject, String description) {
        String key = UUID.randomUUID().toString();
        KeyedWeakReference reference = new KeyedWeakReference(
            watchedObject, key, description, queue
        );
        watchedObjects.put(key, reference);

        // 5 秒后检查是否已回收
        executor.execute(() -> {
            removeWeaklyReachableObjects(); // 先清理已回收的
            if (watchedObjects.containsKey(key)) {
                // 仍在 map 中 → 可能泄漏 → 触发 GC 再检查
                GcTrigger.DEFAULT.runGc();
                removeWeaklyReachableObjects();
                if (watchedObjects.containsKey(key)) {
                    // 确认泄漏，通知 OnObjectRetainedListener
                    onObjectRetainedListeners.forEach(
                        listener -> listener.onObjectRetained()
                    );
                }
            }
        });
    }

    /**
     * 从 ReferenceQueue 中取出已回收的引用，从 map 中移除
     */
    private void removeWeaklyReachableObjects() {
        KeyedWeakReference ref;
        while ((ref = (KeyedWeakReference) queue.poll()) != null) {
            watchedObjects.remove(ref.key);
        }
    }
}
```

`KeyedWeakReference` 继承自 `WeakReference`，额外携带了 `key`（唯一标识）和 `description`（描述信息，如 Activity 类名）。

### 2.2 AppWatcher 自动安装（ContentProvider）

LeakCanary 2.x 实现了**零代码初始化**，原理是利用 [[ContentProvider]] 的生命周期特性：

```
Application.attachBaseContext()
    │
    ▼
ContentProvider.onCreate()   ← LeakCanary 在这里自动初始化
    │
    ▼
Application.onCreate()
```

源码关键路径：

```xml
<!-- leakcanary-object-watcher-android 的 AndroidManifest.xml -->
<provider
    android:name="leakcanary.internal.MainProcessAppWatcherInstaller"
    android:authorities="${applicationId}.leakcanary-installer"
    android:exported="false" />
```

```java
// MainProcessAppWatcherInstaller.java（简化）
public final class MainProcessAppWatcherInstaller extends ContentProvider {
    @Override
    public boolean onCreate() {
        Application app = (Application) getContext().getApplicationContext();
        AppWatcher.INSTANCE.manualInstall(app);
        return true;
    }
    // query/insert/update/delete 全部空实现
}
```

`AppWatcher.manualInstall()` 内部注册了四类默认 Watcher：

```java
// AppWatcher 默认安装的监控器
List<InstallableWatcher> watchers = Arrays.asList(
    new ActivityWatcher(application, objectWatcher),      // 监控 Activity
    new FragmentAndViewModelWatcher(application, objectWatcher), // 监控 Fragment + ViewModel
    new RootViewWatcher(objectWatcher),                   // 监控 RootView
    new ServiceWatcher(objectWatcher)                     // 监控 Service
);
```

以 `ActivityWatcher` 为例：

```java
public class ActivityWatcher {
    void install() {
        application.registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
            @Override
            public void onActivityDestroyed(Activity activity) {
                // Activity 销毁时，交给 ObjectWatcher 监控
                objectWatcher.expectWeaklyReachable(
                    activity,
                    activity.getClass().getName() + " received Activity#onDestroy()"
                );
            }
        });
    }
}
```

### 2.3 HeapAnalyzer 堆分析（Shark 库）

当确认有对象泄漏后，LeakCanary 会：

1. **Dump Heap**：调用 `Debug.dumpHprofData(path)` 生成 `.hprof` 文件
2. **分析 Heap**：使用 **Shark**（Square 自研的 hprof 分析库）解析

Shark 的分析流程：

```
.hprof 文件
    │
    ▼
HprofReader 解析二进制格式
    │
    ▼
构建对象图 (ObjectGraph)
    │
    ▼
找到所有 KeyedWeakReference 实例
    │
    ▼
过滤出 referent 不为 null 的（泄漏对象）
    │
    ▼
从 GC Root 出发，BFS 找到到泄漏对象的最短路径
    │
    ▼
生成 LeakTrace（泄漏引用链）
```

Shark 相比 HAHA 库（LeakCanary 1.x 使用）的优势：
- 纯 Kotlin 实现，无需依赖 `android.os` 包
- 内存占用更低（流式解析，不需要将整个 hprof 加载到内存）
- 解析速度提升约 6 倍

### 2.4 泄漏判定算法

LeakCanary 对引用链上的每个节点进行 **Leaking 状态判定**：

| 状态 | 含义 | 判定依据 |
|---|---|---|
| `Leaking: YES` | 确认泄漏 | Activity 已 onDestroy、Fragment 已 onDestroy |
| `Leaking: NO` | 确认未泄漏 | Application 实例、正在运行的 Thread |
| `Leaking: UNKNOWN` | 无法确定 | 普通对象，无法判断生命周期 |

算法会在引用链上找到 **Leaking: NO → Leaking: YES 的边界**，这个边界就是泄漏的根因所在。

```
├─ android.app.Application instance
│    Leaking: NO (Application is never leaking)
│    ↓ Application.sManager          ← 泄漏根因在这附近
├─ com.example.Manager instance
│    Leaking: UNKNOWN
│    ↓ Manager.mActivity
╰→ com.example.MainActivity instance
│    Leaking: YES (Activity#onDestroy() was called)
```

---

## 三、实战

### 3.1 自定义配置

**基础依赖**（仅 Debug 包生效）：

```groovy
dependencies {
    debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.14'
}
```

**自定义 ObjectWatcher 配置**：

```java
// 在 Application.onCreate() 中配置
LeakCanary.setConfig(
    LeakCanary.getConfig().newBuilder()
        .retainedVisibleThreshold(3)       // 保留对象达到 3 个时才触发 heap dump
        .dumpHeapWhenDebugging(false)       // 调试时不 dump（避免卡顿）
        .referenceMatchers(                 // 自定义已知泄漏过滤
            AndroidReferenceMatchers.getAppDefaults()
        )
        .build()
);
```

**禁用自动安装，改为手动控制**：

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="leakcanary.internal.MainProcessAppWatcherInstaller"
    android:authorities="${applicationId}.leakcanary-installer"
    android:enabled="false" />
```

```java
// 手动安装
public class MyApp extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        AppWatcher.INSTANCE.manualInstall(this);
    }
}
```

**监控自定义对象**：

```java
// 监控任意对象的回收情况
public class MyPool {
    public void release(PooledObject obj) {
        pool.remove(obj);
        // 期望 obj 被回收，如果没有则报泄漏
        AppWatcher.INSTANCE.getObjectWatcher()
            .expectWeaklyReachable(obj, "PooledObject was released");
    }
}
```

### 3.2 线上使用方案（只监控不弹通知）

LeakCanary 默认只用于 Debug 包。线上环境需要定制方案：

**方案一：使用 leakcanary-android-process（独立进程分析）**

```groovy
// 线上只引入 watcher，不引入 UI 和自动 dump
releaseImplementation 'com.squareup.leakcanary:leakcanary-object-watcher-android:2.14'
```

```java
// 自定义 OnObjectRetainedListener，上报而非弹通知
public class LeakUploader implements OnObjectRetainedListener {

    @Override
    public void onObjectRetained() {
        // 获取当前保留对象数量
        int count = AppWatcher.INSTANCE.getObjectWatcher()
            .getRetainedObjectCount();

        // 上报到自己的监控平台
        MonitorSDK.report("leak_retained_count", count);

        // 可选：达到阈值时手动 dump
        if (count >= 5) {
            String path = "/data/data/" + context.getPackageName() + "/files/leak.hprof";
            Debug.dumpHprofData(path);
            // 异步上传 hprof 到服务端分析
            uploadHprof(path);
        }
    }
}
```

**方案二：配合 [[Shark]] 在服务端分析**

```java
// 服务端使用 Shark CLI 分析上传的 hprof
// 依赖：shark-cli
// 命令：shark-cli analyze-heap leak.hprof
```

线上方案注意事项：
- hprof 文件通常 50-200MB，需压缩后上传，注意流量消耗
- dump heap 会 freeze 主线程 5-15 秒，建议在后台空闲时触发
- 采样率控制：不要对所有用户开启，建议 1%-5% 灰度

### 3.3 与 CI 集成

LeakCanary 提供了 `leakcanary-android-instrumentation` 模块，可在 UI 测试中自动检测泄漏：

```groovy
androidTestImplementation 'com.squareup.leakcanary:leakcanary-android-instrumentation:2.14'
```

```java
// 自定义 Test Runner
public class LeakCanaryTestRunner extends AndroidJUnitRunner {
    @Override
    public void finish(int resultCode, Bundle results) {
        // 测试结束时检查泄漏
        FailTestOnLeakRunListener.failOnLeak(results);
        super.finish(resultCode, results);
    }
}
```

```groovy
// build.gradle
android {
    defaultConfig {
        testInstrumentationRunner "com.example.LeakCanaryTestRunner"
    }
}
```

CI 流水线集成：

```yaml
# .github/workflows/leak-check.yml
- name: Run instrumentation tests with leak detection
  run: ./gradlew connectedDebugAndroidTest
- name: Upload leak traces
  if: failure()
  uses: actions/upload-artifact@v3
  with:
    name: leak-traces
    path: app/build/outputs/androidTest-results/
```

---

## 四、面试题

### Q1：LeakCanary 的核心检测原理是什么？

**A**：基于 `WeakReference` + `ReferenceQueue` 机制。当被监控对象（如 Activity）销毁时，LeakCanary 用 `WeakReference` 包装它并关联一个 `ReferenceQueue`。等待 5 秒后检查 `ReferenceQueue`，如果 `WeakReference` 没有入队，说明对象仍被强引用持有，手动触发一次 GC 后再检查，仍未入队则判定为泄漏，随后 dump heap 并用 Shark 分析引用链。

### Q2：LeakCanary 2.x 为什么不需要在 Application 中初始化？

**A**：利用了 `ContentProvider` 的生命周期特性。LeakCanary 在 `AndroidManifest.xml` 中注册了 `MainProcessAppWatcherInstaller`（继承 ContentProvider），其 `onCreate()` 在 `Application.onCreate()` 之前被系统调用，在这里完成自动初始化。这是 Android 库实现零代码初始化的常见技巧，[[Jetpack]] 的 App Startup 也是基于类似原理。

### Q3：LeakCanary 如何区分「真正泄漏」和「GC 还没来得及回收」？

**A**：两步确认机制。第一步等待 5 秒（`watchDurationMillis` 可配置），给系统 GC 充足时间。第二步如果仍未回收，主动调用 `Runtime.getRuntime().gc()` 触发 GC，再等待后检查。只有两次检查都未回收才判定为泄漏。这样可以有效避免误报。

### Q4：LeakCanary 对性能有什么影响？能用在线上吗？

**A**：主要性能影响在两个阶段：
- **监控阶段**：影响极小，只是创建 WeakReference 和定时检查，几乎无开销
- **Dump Heap 阶段**：会 freeze 整个进程 5-15 秒，且生成的 hprof 文件很大（50-200MB）

因此默认只在 Debug 包使用。线上可以只引入 `leakcanary-object-watcher-android`（只监控不 dump），统计泄漏对象数量上报，必要时在后台空闲期采样 dump。

### Q5：Shark 和 HAHA 库有什么区别？为什么 LeakCanary 2.x 换用 Shark？

**A**：HAHA 是 LeakCanary 1.x 使用的 hprof 分析库，基于 `perflib`（Android Studio 的堆分析库），需要将整个 hprof 加载到内存，内存占用大且速度慢。Shark 是 Square 用 Kotlin 重写的分析库，采用流式解析，内存占用降低 10 倍以上，解析速度提升约 6 倍，且不依赖 Android SDK，可以在纯 JVM 环境运行（支持服务端分析）。

### Q6：如何处理 LeakCanary 报出的系统级泄漏（Android Framework 自身的泄漏）？

**A**：Android 系统本身存在一些已知泄漏（如某些版本的 `InputMethodManager` 持有 Activity 引用）。LeakCanary 内置了 `AndroidReferenceMatchers`，维护了一份已知系统泄漏列表，默认会过滤这些泄漏。如果遇到新的系统泄漏，可以通过自定义 `ReferenceMatcher` 添加到过滤列表中：

```java
List<ReferenceMatcher> matchers = new ArrayList<>(
    AndroidReferenceMatchers.getAppDefaults()
);
matchers.add(new IgnoredReferenceMatcher(
    new LibraryLeakReferenceMatcher(
        new ReferencePattern.InstanceFieldPattern(
            "android.widget.Toast", "mTN"
        )
    )
));
LeakCanary.setConfig(
    LeakCanary.getConfig().newBuilder()
        .referenceMatchers(matchers)
        .build()
);
```

---

## 五、实战与踩坑

### 踩坑 1：LeakCanary 报泄漏但代码看起来没问题

**现象**：LeakCanary 报 Activity 泄漏，但检查代码没有明显的引用持有。

**原因**：常见于 `Handler` + `postDelayed` 场景：

```java
public class MainActivity extends Activity {
    private Handler handler = new Handler();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 匿名 Runnable 隐式持有 MainActivity.this
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                updateUI();
            }
        }, 30000); // 30 秒延迟
    }
    // Activity 销毁后，MessageQueue 中的 Message 仍持有 Runnable → Activity
}
```

**修复**：

```java
// 方案 1：onDestroy 中移除所有回调
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null);
}

// 方案 2：使用静态内部类 + WeakReference
private static class SafeRunnable implements Runnable {
    private final WeakReference<MainActivity> activityRef;

    SafeRunnable(MainActivity activity) {
        this.activityRef = new WeakReference<>(activity);
    }

    @Override
    public void run() {
        MainActivity activity = activityRef.get();
        if (activity != null) {
            activity.updateUI();
        }
    }
}
```

### 踩坑 2：Fragment 泄漏 — View 引用未清理

**现象**：Fragment 的 View 泄漏，引用链指向 Fragment 中的成员变量。

```java
public class MyFragment extends Fragment {
    private RecyclerView recyclerView; // 持有 View 引用

    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_my, container, false);
        recyclerView = view.findViewById(R.id.recycler_view);
        return view;
    }
    // onDestroyView 后 View 被销毁，但 recyclerView 字段仍持有引用
}
```

**修复**：

```java
@Override
public void onDestroyView() {
    super.onDestroyView();
    recyclerView = null; // 清理 View 引用
}
```

> 这也是为什么 [[ViewBinding]] 官方推荐在 `onDestroyView` 中将 binding 置为 null。

### 踩坑 3：单例持有 Context 导致泄漏

```java
// ❌ 错误写法
public class AppManager {
    private static AppManager instance;
    private Context context;

    public static AppManager getInstance(Context context) {
        if (instance == null) {
            instance = new AppManager();
            instance.context = context; // 如果传入 Activity，则泄漏
        }
        return instance;
    }
}

// ✅ 修复：使用 ApplicationContext
public static AppManager getInstance(Context context) {
    if (instance == null) {
        instance = new AppManager();
        instance.context = context.getApplicationContext();
    }
    return instance;
}
```

### 踩坑 4：LeakCanary 与 OkHttp/Retrofit 的误报

**现象**：LeakCanary 报 `ConnectionPool` 或 `Dispatcher` 持有 Activity。

**原因**：通常是在 Activity 中创建了 OkHttpClient 实例，或者回调中持有 Activity 引用。

```java
// ❌ 在 Activity 中创建 Client
public class MainActivity extends Activity {
    private OkHttpClient client = new OkHttpClient();

    void loadData() {
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onResponse(Call call, Response response) {
                // 匿名内部类持有 MainActivity.this
                updateUI(response);
            }
        });
    }
}
```

**修复**：OkHttpClient 应全局单例，回调中使用弱引用或在 `onDestroy` 中取消请求：

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    // 取消所有关联请求
    for (Call call : GlobalClient.get().dispatcher().runningCalls()) {
        if (call.request().tag() == this) {
            call.cancel();
        }
    }
}
```

### 踩坑 5：多进程场景下 LeakCanary 重复初始化

**现象**：多进程 App 中，每个进程都会触发 ContentProvider 初始化，导致非主进程也运行 LeakCanary。

**修复**：

```java
// 禁用自动安装，手动控制
// AndroidManifest.xml 中 disable ContentProvider
// 然后在 Application 中判断进程
public class MyApp extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        if (isMainProcess()) {
            AppWatcher.INSTANCE.manualInstall(this);
        }
    }

    private boolean isMainProcess() {
        int pid = android.os.Process.myPid();
        ActivityManager am = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
        for (ActivityManager.RunningAppProcessInfo info : am.getRunningAppProcesses()) {
            if (info.pid == pid) {
                return getPackageName().equals(info.processName);
            }
        }
        return false;
    }
}
```

---

## 相关链接

- [[2-内存优化]]
- [[Java 引用类型]]
- [[Android 性能优化]]
- [[Jetpack]]
- [[OkHttp]]
- [LeakCanary 官方文档](https://square.github.io/leakcanary/)
- [Shark GitHub](https://github.com/square/leakcanary/tree/main/shark)

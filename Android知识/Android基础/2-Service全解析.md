# Service 全解析

## 一、Service 是什么？

Service 是 Android 四大组件之一，用于在**后台执行长时间运行的操作**，没有用户界面。

常见使用场景：
- 后台播放音乐
- 后台下载文件
- 后台上传数据
- 与服务端保持长连接

> **重要误区：Service 默认运行在主线程！** 它不会自动创建新线程。如果在 Service 里做耗时操作（网络请求、IO），会阻塞主线程导致 ANR。

---

## 二、两种启动方式

### 2.1 startService（启动式）

调用 `startService()` 启动，Service 会一直运行，直到自己调用 `stopSelf()` 或外部调用 `stopService()`。

**生命周期：**
```
startService() → onCreate() → onStartCommand() → [运行中] → stopService()/stopSelf() → onDestroy()
```

- `onCreate()`：只在第一次创建时调用
- `onStartCommand()`：每次 startService 都会调用
- 多次 startService 只会触发多次 onStartCommand，不会多次 onCreate

```java
public class MyService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
        // 初始化，只调用一次
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 每次 startService 都会调用
        // 在这里开启子线程做耗时操作
        new Thread(() -> {
            // 执行任务...
            stopSelf(); // 任务完成后停止自己
        }).start();
        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null; // 启动式不需要绑定
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        // 释放资源
    }
}
```

**onStartCommand 的返回值：**

| 返回值 | 含义 |
|--------|------|
| `START_NOT_STICKY` | 被杀后不重建。适合可以重新发起的任务（如上传） |
| `START_STICKY` | 被杀后重建，但 Intent 为 null。适合音乐播放器这类需要持续运行的 |
| `START_REDELIVER_INTENT` | 被杀后重建，并重新传递最后一个 Intent。适合必须完成的任务（如下载） |

### 2.2 bindService（绑定式）

调用 `bindService()` 绑定，Service 的生命周期与绑定者（Activity/Fragment）关联。所有绑定者都解绑后，Service 自动销毁。

**生命周期：**
```
bindService() → onCreate() → onBind() → [绑定中，客户端通过 IBinder 通信] → onUnbind() → onDestroy()
```

```java
// Service 端
public class MyBindService extends Service {
    private final IBinder binder = new LocalBinder();

    public class LocalBinder extends Binder {
        MyBindService getService() {
            return MyBindService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder; // 返回 Binder 给客户端
    }

    // 提供给客户端调用的方法
    public String getData() {
        return "Hello from Service";
    }
}

// Activity 端
public class MyActivity extends AppCompatActivity {
    private MyBindService service;
    private boolean bound = false;

    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder binder) {
            MyBindService.LocalBinder localBinder = (MyBindService.LocalBinder) binder;
            service = localBinder.getService();
            bound = true;
            // 现在可以调用 service.getData()
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            bound = false;
        }
    };

    @Override
    protected void onStart() {
        super.onStart();
        Intent intent = new Intent(this, MyBindService.class);
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (bound) {
            unbindService(connection);
            bound = false;
        }
    }
}
```

### 2.3 混合启动

同一个 Service 可以既被 start 又被 bind。此时必须同时 stopService + 所有客户端 unbind 后才会销毁。

```
startService() → bindService() → unbindService() → stopService() → onDestroy()
```

---

## 三、前台服务（Foreground Service）

从 Android 8.0 开始，后台 Service 会被系统限制（几分钟后可能被杀）。如果需要长时间运行，必须使用前台服务。

前台服务必须显示一个**通知**，让用户知道有服务在运行。

```java
public class ForegroundService extends Service {
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 创建通知渠道（Android 8.0+）
        NotificationChannel channel = new NotificationChannel(
            "service_channel", "前台服务", NotificationManager.IMPORTANCE_LOW);
        NotificationManager manager = getSystemService(NotificationManager.class);
        manager.createNotificationChannel(channel);

        // 创建通知
        Notification notification = new NotificationCompat.Builder(this, "service_channel")
            .setContentTitle("服务运行中")
            .setContentText("正在后台处理任务...")
            .setSmallIcon(R.drawable.ic_notification)
            .build();

        // 启动前台服务
        startForeground(1, notification);

        return START_STICKY;
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

**Android 各版本对前台服务的限制：**

| 版本 | 限制 |
|------|------|
| Android 8.0 | 后台应用不能 startService，需要 startForegroundService + 5秒内调用 startForeground |
| Android 9.0 | 需要声明 `FOREGROUND_SERVICE` 权限 |
| Android 10 | 前台服务需要声明 `foregroundServiceType`（location、camera、microphone 等） |
| Android 12 | 后台启动前台服务受限，推荐用 WorkManager |
| Android 14 | 必须声明具体的 `foregroundServiceType`，不能为空 |

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />

<service
    android:name=".ForegroundService"
    android:foregroundServiceType="location" />
```

---

## 四、IntentService（已废弃但面试常问）

IntentService 是 Service 的子类，自带一个工作线程，任务执行完自动停止。

```java
public class MyIntentService extends IntentService {
    public MyIntentService() {
        super("MyIntentService");
    }

    @Override
    protected void onHandleIntent(@Nullable Intent intent) {
        // 这个方法在子线程执行！
        // 多个任务会排队串行执行
        String action = intent.getAction();
        // 处理任务...
    }
    // 所有任务处理完后自动 stopSelf
}
```

**IntentService 的原理：**
- 内部创建了一个 `HandlerThread`（带 Looper 的工作线程）
- 通过 Handler 将任务发送到工作线程的消息队列
- 任务串行执行，一个完成后才执行下一个
- 所有任务完成后自动调用 `stopSelf()`

**为什么被废弃？**
- Android 8.0+ 后台限制，IntentService 可能被系统杀死
- 推荐使用 `JobIntentService`（也废弃了）或 `WorkManager`

---

## 五、JobScheduler 与 WorkManager

### JobScheduler（API 21+）

系统级的任务调度器，可以设置执行条件（网络、充电、空闲等）。

```java
JobScheduler scheduler = (JobScheduler) getSystemService(JOB_SCHEDULER_SERVICE);
JobInfo job = new JobInfo.Builder(1, new ComponentName(this, MyJobService.class))
    .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED) // 需要 WiFi
    .setRequiresCharging(true)  // 需要充电
    .setPeriodic(15 * 60 * 1000) // 最少 15 分钟间隔
    .build();
scheduler.schedule(job);
```

### WorkManager（推荐）

Jetpack 组件，底层根据 API 版本自动选择 JobScheduler / AlarmManager / BroadcastReceiver。

```java
OneTimeWorkRequest request = new OneTimeWorkRequest.Builder(MyWorker.class)
    .setConstraints(
        new Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .build();

WorkManager.getInstance(context).enqueue(request);
```

**Service vs JobScheduler vs WorkManager：**

| 特性 | Service | JobScheduler | WorkManager |
|------|---------|-------------|-------------|
| 立即执行 | ✅ | ❌（系统调度） | ❌（系统调度） |
| 条件约束 | ❌ | ✅ | ✅ |
| 存活保证 | ❌（可能被杀） | ✅（系统保证） | ✅（系统保证） |
| 持久化 | ❌ | ❌ | ✅（重启后恢复） |
| 链式任务 | ❌ | ❌ | ✅ |
| 最低 API | 1 | 21 | 14 |

---

## 六、Service 与线程的区别

这是面试必问题。

| 对比项 | Service | Thread |
|--------|---------|--------|
| 运行线程 | 默认主线程 | 独立线程 |
| 生命周期 | 受系统管理，有完整生命周期 | 不受系统管理 |
| 优先级 | 比纯后台进程高，不容易被杀 | 依附于所在进程 |
| 跨组件通信 | 支持（bindService） | 不方便 |
| 适用场景 | 需要长期运行、跨组件、需要系统保活 | 单纯的异步任务 |

**什么时候用 Service？**
- 需要在后台持续运行（音乐播放、位置追踪）
- 需要跨组件通信（多个 Activity 共享数据）
- 需要提升进程优先级（前台服务）

**什么时候用 Thread？**
- 简单的异步任务（网络请求、IO 操作）
- 不需要跨组件
- 配合 Handler、协程、RxJava 使用

---

## 七、面试题库

### 基础题

**Q1：Service 的两种启动方式有什么区别？**
> startService：独立运行，需要手动 stopService/stopSelf 停止。bindService：与绑定者生命周期关联，所有绑定者解绑后自动销毁。可以混合使用，此时需要同时 stop + unbind 才会销毁。

**Q2：Service 运行在哪个线程？**
> 默认运行在主线程。如果需要做耗时操作，必须自己创建子线程。这是一个常见误区，很多人以为 Service 自带后台线程。

**Q3：onStartCommand 的返回值有什么区别？**
> START_NOT_STICKY：被杀后不重建。START_STICKY：被杀后重建但 Intent 为 null。START_REDELIVER_INTENT：被杀后重建并重传 Intent。根据任务是否需要重试来选择。

**Q4：Service 和 Thread 的区别？**
> Service 是四大组件，运行在主线程，有生命周期，受系统管理，优先级较高。Thread 是独立线程，不受系统管理。Service 适合需要长期运行和跨组件通信的场景，Thread 适合简单异步任务。

### 前台服务题

**Q5：什么是前台服务？为什么需要它？**
> 前台服务会显示一个通知，告诉用户有服务在运行。Android 8.0+ 后台 Service 会被系统限制，几分钟后可能被杀。前台服务优先级高，不容易被杀，适合音乐播放、导航等场景。

**Q6：Android 8.0 对 Service 有什么限制？**
> 后台应用不能直接 startService（会抛 IllegalStateException）。需要用 startForegroundService 启动，并在 5 秒内调用 startForeground 显示通知，否则系统会 ANR。

**Q7：Android 12+ 对前台服务有什么新限制？**
> 后台启动前台服务受限（除非有特定豁免条件，如收到高优先级 FCM）。推荐使用 WorkManager 处理后台任务。Android 14 要求必须声明具体的 foregroundServiceType。

### 进阶题

**Q8：IntentService 的原理是什么？为什么被废弃？**
> 内部创建 HandlerThread（带 Looper 的工作线程），通过 Handler 将任务串行发送到工作线程执行，完成后自动 stopSelf。被废弃是因为 Android 8.0+ 后台限制导致它可能被杀，推荐用 WorkManager。

**Q9：bindService 的通信原理？**
> 同进程：通过 Binder 的子类直接获取 Service 引用，调用其公开方法。跨进程：通过 AIDL 生成的 Stub/Proxy，底层走 Binder IPC 机制。

**Q10：如何保活 Service？**
> 1. 前台服务（最推荐）。2. onStartCommand 返回 START_STICKY。3. onDestroy 中重新启动（不推荐）。4. 双进程守护（不推荐，高版本失效）。5. WorkManager（系统保证执行）。注意：过度保活会影响用户体验和电量，应该优先考虑 WorkManager。

**Q11：多次调用 startService 会怎样？**
> onCreate 只调用一次，onStartCommand 每次都会调用。Service 实例只有一个，不会创建多个。stopService 一次就能停止，不需要调用多次。

**Q12：Service 如何与 Activity 通信？**
> 1. bindService + Binder（最直接）。2. 广播（BroadcastReceiver）。3. EventBus / LiveData。4. Messenger（基于 Handler 的跨进程通信）。5. AIDL（跨进程，性能最好）。

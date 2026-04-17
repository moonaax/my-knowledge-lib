# BroadcastReceiver 与 ContentProvider

## 一、BroadcastReceiver（广播接收器）

### 1.1 是什么？

BroadcastReceiver 是 Android 的"广播系统"，类似于一个全局的事件总线。一个组件发送广播，其他注册了对应广播的组件都能收到。

常见系统广播：
- 网络变化：`android.net.conn.CONNECTIVITY_CHANGE`
- 电量变化：`Intent.ACTION_BATTERY_CHANGED`
- 开机完成：`Intent.ACTION_BOOT_COMPLETED`
- 应用安装/卸载：`Intent.ACTION_PACKAGE_ADDED`

### 1.2 广播的类型

| 类型 | 特点 |
|------|------|
| **普通广播（Normal）** | 异步发送，所有接收器几乎同时收到，无法拦截 |
| **有序广播（Ordered）** | 按优先级依次传递，高优先级可以修改数据或拦截（abortBroadcast） |
| **本地广播（Local）** | 只在 App 内部传播，更安全更高效（已废弃，推荐 LiveData/Flow） |
| **粘性广播（Sticky）** | 发送后一直存在，新注册的接收器也能收到（已废弃） |

```java
// 发送普通广播
Intent intent = new Intent("com.example.MY_ACTION");
intent.putExtra("data", "hello");
sendBroadcast(intent);

// 发送有序广播
sendOrderedBroadcast(intent, null);
```

### 1.3 注册方式

**静态注册（AndroidManifest）：**
```xml
<receiver android:name=".MyReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="com.example.MY_ACTION" />
    </intent-filter>
</receiver>
```

**动态注册（代码中）：**
```java
MyReceiver receiver = new MyReceiver();
IntentFilter filter = new IntentFilter("com.example.MY_ACTION");
registerReceiver(receiver, filter);

// 不用时必须注销，否则内存泄漏
unregisterReceiver(receiver);
```

**两种方式的区别：**

| 对比项 | 静态注册 | 动态注册 |
|--------|---------|---------|
| 注册时机 | 安装时 | 代码执行时 |
| 生命周期 | 常驻（App 未启动也能收到） | 跟随注册者（Activity/Service） |
| Android 8.0+ 限制 | 大部分隐式广播不再支持静态注册 | 不受限制 |
| 适用场景 | 开机广播等必须常驻的 | 大多数场景 |

### 1.4 Android 8.0+ 的广播限制

从 Android 8.0 开始，为了省电和优化性能：
- **静态注册的隐式广播大部分失效**（系统不再发送给静态注册的接收器）
- 少数例外：`BOOT_COMPLETED`、`LOCALE_CHANGED` 等
- 解决方案：改用动态注册，或使用 `setPackage()` 指定目标包名变成显式广播

### 1.5 BroadcastReceiver 的 onReceive

```java
public class MyReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        // 运行在主线程！不能做耗时操作
        // 执行时间超过 10 秒会 ANR
        String data = intent.getStringExtra("data");

        // 如果需要做耗时操作，启动 Service 或用 goAsync()
        PendingResult result = goAsync();
        new Thread(() -> {
            // 耗时操作...
            result.finish(); // 必须调用
        }).start();
    }
}
```

---

## 二、ContentProvider（内容提供者）

### 2.1 是什么？

ContentProvider 是 Android 四大组件中最特殊的一个，它的作用是**在不同应用之间共享数据**。

你手机上的通讯录、相册、短信，其他 App 能读取到，就是因为系统提供了对应的 ContentProvider。

### 2.2 核心概念

**URI（统一资源标识符）：**
```
content://com.example.provider/users/1
  │          │                  │     │
  │          │                  │     └── ID（可选，指定某条记录）
  │          │                  └── 表名/路径
  │          └── authority（唯一标识，通常是包名+provider）
  └── scheme（固定为 content://）
```

**CRUD 操作：**
```java
// 查询通讯录
Cursor cursor = getContentResolver().query(
    ContactsContract.Contacts.CONTENT_URI,  // URI
    null,    // 查询列（null = 全部）
    null,    // WHERE 条件
    null,    // WHERE 参数
    null     // 排序
);

// 插入
ContentValues values = new ContentValues();
values.put("name", "张三");
values.put("phone", "13800000000");
Uri uri = getContentResolver().insert(MyProvider.CONTENT_URI, values);

// 更新
getContentResolver().update(uri, values, "id=?", new String[]{"1"});

// 删除
getContentResolver().delete(uri, "id=?", new String[]{"1"});
```

### 2.3 自定义 ContentProvider

```java
public class MyProvider extends ContentProvider {
    public static final Uri CONTENT_URI =
        Uri.parse("content://com.example.provider/users");

    private SQLiteDatabase db;

    @Override
    public boolean onCreate() {
        // 初始化数据库，运行在主线程
        db = new MyDbHelper(getContext()).getWritableDatabase();
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection, String selection,
                        String[] selectionArgs, String sortOrder) {
        return db.query("users", projection, selection, selectionArgs,
                        null, null, sortOrder);
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        long id = db.insert("users", null, values);
        getContext().getContentResolver().notifyChange(uri, null);
        return ContentUris.withAppendedId(uri, id);
    }

    @Override
    public int update(Uri uri, ContentValues values, String selection,
                      String[] selectionArgs) {
        int count = db.update("users", values, selection, selectionArgs);
        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }

    @Override
    public int delete(Uri uri, String selection, String[] selectionArgs) {
        int count = db.delete("users", selection, selectionArgs);
        getContext().getContentResolver().notifyChange(uri, null);
        return count;
    }

    @Override
    public String getType(Uri uri) {
        return "vnd.android.cursor.dir/vnd.com.example.users";
    }
}
```

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name=".MyProvider"
    android:authorities="com.example.provider"
    android:exported="true"
    android:permission="com.example.READ_DATA" />
```

### 2.4 ContentProvider 的特点

- **跨进程通信**：底层基于 Binder 机制，天然支持跨进程
- **统一接口**：不管底层是 SQLite、文件还是网络，对外都是 CRUD 接口
- **权限控制**：可以设置读写权限，精确控制数据访问
- **数据变化通知**：通过 `ContentObserver` 监听数据变化

```java
// 监听数据变化
getContentResolver().registerContentObserver(
    MyProvider.CONTENT_URI,
    true,  // notifyForDescendants
    new ContentObserver(new Handler()) {
        @Override
        public void onChange(boolean selfChange) {
            // 数据发生变化
        }
    }
);
```

### 2.5 ContentProvider 的启动时机

**ContentProvider 在 Application.onCreate() 之前初始化！**

```
Application.attachBaseContext()
    ↓
ContentProvider.onCreate()  ← 先于 Application.onCreate
    ↓
Application.onCreate()
```

这个特性被很多库利用来做**无侵入初始化**（比如 Firebase、LeakCanary），在 ContentProvider 的 onCreate 中完成 SDK 初始化，App 开发者不需要手动调用 init。

但这也是一个**性能隐患**：太多 ContentProvider 会拖慢 App 启动速度。Jetpack 的 `App Startup` 库就是为了解决这个问题，用一个 ContentProvider 统一管理多个库的初始化。

---

## 三、面试题库

### BroadcastReceiver 题

**Q1：广播的注册方式有哪些？区别是什么？**
> 静态注册（Manifest）和动态注册（代码）。静态注册常驻，App 未启动也能收到；动态注册跟随注册者生命周期。Android 8.0+ 静态注册的隐式广播大部分失效。

**Q2：有序广播和普通广播的区别？**
> 普通广播异步发送，接收器几乎同时收到，无法拦截。有序广播按优先级依次传递，高优先级可以修改数据或调用 abortBroadcast 拦截。

**Q3：本地广播（LocalBroadcast）为什么被废弃了？**
> 因为它本质是一个进程内的观察者模式，用 LiveData 或 Flow 可以更好地实现，而且有生命周期感知，不会内存泄漏。

**Q4：BroadcastReceiver 的 onReceive 能做耗时操作吗？**
> 不能，运行在主线程，超过 10 秒会 ANR。如果需要耗时操作，可以用 goAsync() 获取 PendingResult 在子线程处理，或者启动 Service/WorkManager。

**Q5：Android 8.0 对广播有什么限制？**
> 静态注册的隐式广播大部分不再生效（为了省电）。解决方案：改用动态注册，或用 setPackage 指定目标包名变成显式广播。

### ContentProvider 题

**Q6：ContentProvider 的作用是什么？底层原理？**
> 用于跨应用/跨进程共享数据，提供统一的 CRUD 接口。底层基于 Binder 机制实现跨进程通信。

**Q7：ContentProvider 的 onCreate 在什么时候调用？**
> 在 Application.onCreate 之前调用。很多第三方库利用这个特性做无侵入初始化（如 Firebase）。但过多 ContentProvider 会拖慢启动速度，Jetpack App Startup 就是为了解决这个问题。

**Q8：ContentProvider、ContentResolver、ContentObserver 的关系？**
> ContentProvider 提供数据。ContentResolver 是客户端用来访问 ContentProvider 的工具（通过 URI 定位）。ContentObserver 用来监听数据变化（观察者模式）。

**Q9：为什么说 ContentProvider 是天然的跨进程方案？**
> 因为它底层基于 Binder，系统会自动处理跨进程通信。调用方通过 ContentResolver 发起请求，系统通过 Binder 转发到目标进程的 ContentProvider，开发者不需要手动处理 IPC。

**Q10：ContentProvider 和直接用 AIDL 有什么区别？**
> ContentProvider 是更高层的封装，提供标准的 CRUD 接口，适合数据共享场景。AIDL 更灵活，可以定义任意接口，适合复杂的跨进程调用。ContentProvider 底层也是用 Binder（类似 AIDL）实现的。

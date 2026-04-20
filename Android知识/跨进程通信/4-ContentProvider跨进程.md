# ContentProvider 跨进程通信

> ContentProvider 是 Android 四大组件之一，专为**跨进程数据共享**设计，基于 [[1-Binder机制原理]] 实现 IPC，对外暴露统一 URI 接口。与 [[3-BroadcastReceiver与ContentProvider]] 配合可构建完整的跨进程数据通知体系。

---

## 一、核心概念

### 1.1 ContentProvider 的角色

ContentProvider 在 Android 组件体系中扮演**数据中间层**的角色：

- **统一数据访问接口**：无论底层是 SQLite、文件还是网络数据，对外暴露标准 CRUD 接口
- **跨进程数据共享**：基于 Binder 机制，天然支持 IPC
- **权限控制**：通过 `readPermission`/`writePermission` 实现细粒度访问控制
- **数据变更通知**：配合 ContentObserver 实现观察者模式

### 1.2 URI 体系

ContentProvider 使用 URI 标识数据：

````
content://com.example.provider/user/123
|scheme| |---authority---| |path| |id|
````
各部分含义：

| 组成部分 | 说明 | 示例 |
|---------|------|------|
| scheme | 固定为 `content://` | `content://` |
| authority | Provider 的唯一标识，通常为包名 + provider | `com.example.provider` |
| path | 数据表名或资源路径 | `/user` |
| id | 可选，特定记录的 ID | `/123` |

URI 匹配使用 `UriMatcher`：

````java
public class MyProvider extends ContentProvider {

    private static final String AUTHORITY = "com.example.provider";
    private static final int CODE_USER_LIST = 1;
    private static final int CODE_USER_ITEM = 2;

    private static final UriMatcher uriMatcher = new UriMatcher(UriMatcher.NO_MATCH);
    static {
        // content://com.example.provider/user -> 匹配用户列表
        uriMatcher.addURI(AUTHORITY, "user", CODE_USER_LIST);
        // content://com.example.provider/user/# -> 匹配单个用户，# 为数字通配符
        uriMatcher.addURI(AUTHORITY, "user/#", CODE_USER_ITEM);
    }
}
````
### 1.3 CRUD 操作

ContentProvider 定义了 6 个核心抽象方法：

| 方法 | 作用 | 返回值 |
|------|------|--------|
| `onCreate()` | 初始化 | Boolean |
| `query()` | 查询数据 | Cursor? |
| `insert()` | 插入数据 | 新记录的 Uri |
| `update()` | 更新数据 | 受影响行数 |
| `delete()` | 删除数据 | 受影响行数 |
| `getType()` | 返回 MIME 类型 | String? |

### 1.4 ContentResolver：客户端代理

调用方不直接与 ContentProvider 交互，而是通过 `ContentResolver` 作为代理：

````java
// 查询
Cursor cursor = contentResolver.query(
    ContactsContract.Contacts.CONTENT_URI,
    new String[]{ContactsContract.Contacts.DISPLAY_NAME}, null, null, null
);
if (cursor != null) {
    try {
        while (cursor.moveToNext()) Log.d("Contact", cursor.getString(0));
    } finally {
        cursor.close();
    }
}

// 插入
ContentValues values = new ContentValues();
values.put("name", "张三");
values.put("age", 25);
contentResolver.insert(Uri.parse("content://com.example.provider/user"), values);
````
`ContentResolver` 的核心作用：根据 URI 的 authority 找到对应 Provider → 通过 Binder 跨进程调用 → 管理 ContentObserver 注册与通知。

### 1.5 ContentObserver：数据变更监听

````java
// 注册观察者
ContentObserver observer = new ContentObserver(new Handler(Looper.getMainLooper())) {
    @Override
    public void onChange(boolean selfChange, Uri uri) {
        Log.d("Observer", "数据变更: " + uri);
    }
};
contentResolver.registerContentObserver(
    Uri.parse("content://com.example.provider/user"),
    true, observer // notifyForDescendants=true 监听子路径
);

// Provider 端在 insert/update/delete 中通知变更
if (getContext() != null) {
    getContext().getContentResolver().notifyChange(newUri, null);
}

// 不再需要时取消注册
contentResolver.unregisterContentObserver(observer);
````
---

## 二、原理与源码

### 2.1 ContentProvider 的启动时机

ContentProvider 的启动时机非常早，**比 Application.onCreate() 还早**。

进程启动时序：

````
Application.<init>() → attachBaseContext() → ContentProvider.onCreate() ⚠️ → Application.onCreate() → Activity/Service
````
源码验证（`ActivityThread.handleBindApplication`）：

````java
// frameworks/base/core/java/android/app/ActivityThread.java
private void handleBindApplication(AppBindData data) {
    // 1. 创建 Application 对象
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    // 2. 安装所有 ContentProvider（在 Application.onCreate 之前！）
    if (!data.restrictedBackupMode && !ArrayUtils.isEmpty(data.providers)) {
        installContentProviders(app, data.providers);
        // 内部：逐个反射创建 Provider → 调用 onCreate → 批量 publishContentProviders 到 AMS
    }
    // 3. 最后才调用 Application.onCreate
    mInstrumentation.callApplicationOnCreate(app);
}
````
### 2.2 AMS 中 ContentProvider 的发布与获取流程

#### 发布流程（Provider 所在进程）

````
App 进程启动 → handleBindApplication() → installContentProviders()
  → installProvider(): 反射创建 Provider → attachInfo() → onCreate()
  → AMS.publishContentProviders(): 通过 Binder 注册到 AMS 的 ProviderMap
````
AMS 端通过 `ProviderMap` 按 authority 索引所有已发布的 `ContentProviderRecord`，其中包含 `IContentProvider`（Binder 代理）、所属进程信息、以及所有客户端连接。

#### 获取流程（调用方进程）

````
ContentResolver.query(uri)
  → acquireProvider(authority)
    → 先查本地缓存 mProviderMap
    → 未命中 → AMS.getContentProvider(authority)
      → 若 Provider 进程未启动 → 启动进程 → 等待发布
      → 返回 ContentProviderHolder (含 IContentProvider Binder)
    → installProvider() 安装到本地缓存
  → IContentProvider.query()  // 跨进程 Binder 调用
````
### 2.3 Binder 调用链

ContentProvider 的跨进程调用基于 [[1-Binder机制原理]]，完整调用链如下：

````
调用方进程                              Provider 进程
ContentResolver.query()
  → ContentProviderProxy.query()
    ═══ Binder transact ═══>          ContentProviderNative.onTransact()
                                        → Transport.query()
                                        → ContentProvider.query() // 开发者实现
    <═══ CursorWindow (Ashmem) ═══    返回 Cursor 数据
````
核心类：`ContentProviderProxy`（客户端代理）→ Binder transact → `ContentProviderNative`（服务端桩）→ `Transport` → 开发者实现的 `ContentProvider`。

> **注意**：query 返回的 Cursor 数据通过 `CursorWindow`（基于 Ashmem 匿名共享内存）传输，避免了大量数据的 Binder 拷贝。



---

## 三、实战

### 3.1 自定义 ContentProvider 实现跨进程数据共享

#### Step 1：定义数据库 Helper

````java
public class UserDbHelper extends SQLiteOpenHelper {

    public UserDbHelper(Context context) {
        super(context, "user.db", null, 1);
    }

    @Override
    public void onCreate(SQLiteDatabase db) {
        db.execSQL("CREATE TABLE user ("
            + "_id INTEGER PRIMARY KEY AUTOINCREMENT,"
            + "name TEXT NOT NULL,"
            + "age INTEGER DEFAULT 0)");
    }

    @Override
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        db.execSQL("DROP TABLE IF EXISTS user");
        onCreate(db);
    }
}
````
#### Step 2：实现 ContentProvider

````java
public class UserProvider extends ContentProvider {

    public static final String AUTHORITY = "com.example.app.provider";
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTHORITY + "/user");
    private static final int CODE_LIST = 1;
    private static final int CODE_ITEM = 2;
    private static final UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH);
    static {
        matcher.addURI(AUTHORITY, "user", CODE_LIST);
        matcher.addURI(AUTHORITY, "user/#", CODE_ITEM);
    }

    private UserDbHelper dbHelper;

    @Override
    public boolean onCreate() {
        dbHelper = new UserDbHelper(getContext());
        return true;
    }

    @Override
    public Cursor query(Uri uri, String[] projection,
                        String selection, String[] selectionArgs, String sortOrder) {
        SQLiteDatabase db = dbHelper.getReadableDatabase();
        Cursor cursor;
        switch (matcher.match(uri)) {
            case CODE_LIST:
                cursor = db.query("user", projection, selection, selectionArgs, null, null, sortOrder);
                break;
            case CODE_ITEM:
                cursor = db.query("user", projection, "_id=?",
                    new String[]{String.valueOf(ContentUris.parseId(uri))}, null, null, null);
                break;
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
        if (cursor != null && getContext() != null) {
            cursor.setNotificationUri(getContext().getContentResolver(), uri);
        }
        return cursor;
    }

    @Override
    public Uri insert(Uri uri, ContentValues values) {
        long id = dbHelper.getWritableDatabase().insert("user", null, values);
        Uri newUri = ContentUris.withAppendedId(CONTENT_URI, id);
        if (getContext() != null) {
            getContext().getContentResolver().notifyChange(newUri, null);
        }
        return newUri;
    }

    // update / delete 结构类似 query，根据 CODE_LIST / CODE_ITEM 分发
    @Override
    public int update(Uri uri, ContentValues values, String s, String[] a) {
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        int count;
        switch (matcher.match(uri)) {
            case CODE_LIST:
                count = db.update("user", values, s, a);
                break;
            case CODE_ITEM:
                count = db.update("user", values, "_id=?",
                    new String[]{String.valueOf(ContentUris.parseId(uri))});
                break;
            default:
                count = 0;
        }
        if (count > 0 && getContext() != null) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }

    @Override
    public int delete(Uri uri, String s, String[] a) {
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        int count;
        switch (matcher.match(uri)) {
            case CODE_LIST:
                count = db.delete("user", s, a);
                break;
            case CODE_ITEM:
                count = db.delete("user", "_id=?",
                    new String[]{String.valueOf(ContentUris.parseId(uri))});
                break;
            default:
                count = 0;
        }
        if (count > 0 && getContext() != null) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }

    @Override
    public String getType(Uri uri) {
        switch (matcher.match(uri)) {
            case CODE_LIST:
                return "vnd.android.cursor.dir/vnd." + AUTHORITY + ".user";
            case CODE_ITEM:
                return "vnd.android.cursor.item/vnd." + AUTHORITY + ".user";
            default:
                throw new IllegalArgumentException("Unknown URI: " + uri);
        }
    }
}
````
#### Step 3：AndroidManifest 注册

````xml
<provider
    android:name=".UserProvider"
    android:authorities="com.example.app.provider"
    android:exported="true"
    android:readPermission="com.example.app.READ_DATA"
    android:writePermission="com.example.app.WRITE_DATA" />
````
#### Step 4：其他进程/应用调用

````java
Uri uri = Uri.parse("content://com.example.app.provider/user");

// 插入
ContentValues values = new ContentValues();
values.put("name", "李四");
values.put("age", 30);
contentResolver.insert(uri, values);

// 查询
Cursor cursor = contentResolver.query(uri, null, null, null, "name ASC");
if (cursor != null) {
    try {
        while (cursor.moveToNext()) {
            String name = cursor.getString(cursor.getColumnIndexOrThrow("name"));
            int age = cursor.getInt(cursor.getColumnIndexOrThrow("age"));
            Log.d("User", "name=" + name + ", age=" + age);
        }
    } finally {
        cursor.close();
    }
}
````
### 3.2 FileProvider 的使用

FileProvider 是 ContentProvider 的子类，用于安全共享应用私有文件。Android 7.0+ 直接使用 `file://` URI 会抛出 `FileUriExposedException`。

#### 配置 FileProvider

````xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
````
````xml
<!-- res/xml/file_paths.xml -->
<paths>
    <files-path name="internal_images" path="images/" />
    <external-path name="external" path="." />
    <cache-path name="cache" path="images/" />
    <external-cache-path name="external_cache" path="." />
</paths>
````
#### 使用 FileProvider 分享文件

````java
public void shareFile(Context context, File file) {
    Uri uri = FileProvider.getUriForFile(
        context,
        context.getPackageName() + ".fileprovider",
        file
    );
    Intent intent = new Intent(Intent.ACTION_SEND);
    intent.setType("image/*");
    intent.putExtra(Intent.EXTRA_STREAM, uri);
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    context.startActivity(Intent.createChooser(intent, "分享图片"));
}
````
#### 使用 FileProvider 拍照

````java
// 创建临时文件并获取 content:// URI
File photoFile = new File(getFilesDir(), "images/photo_" + System.currentTimeMillis() + ".jpg");
if (photoFile.getParentFile() != null) photoFile.getParentFile().mkdirs();
Uri photoUri = FileProvider.getUriForFile(this, getPackageName() + ".fileprovider", photoFile);

// 使用 ActivityResult API 启动相机
ActivityResultLauncher<Uri> takePicture = registerForActivityResult(
    new ActivityResultContracts.TakePicture(), success -> {
        if (success) imageView.setImageURI(photoUri);
    }
);
takePicture.launch(photoUri);
````
---

## 四、面试题

### Q1：ContentProvider 的 onCreate 和 Application 的 onCreate 谁先执行？为什么？

**A**：ContentProvider 的 `onCreate()` 先执行。在 `ActivityThread.handleBindApplication()` 中，先创建 Application 并调用 `attachBaseContext()`，然后 `installContentProviders()` 逐个调用每个 Provider 的 `onCreate()`，最后才调用 `Application.onCreate()`。这保证了 Application 初始化时所有 Provider 已就绪。许多第三方库（Firebase、Jetpack App Startup）利用此特性实现无侵入式初始化。

---

### Q2：ContentProvider 的 CRUD 方法运行在哪个线程？是否线程安全？

**A**：`onCreate()` 运行在主线程；CRUD 方法在跨进程调用时运行在 **Binder 线程池**中，同进程调用时运行在调用者线程。CRUD 方法本身不是线程安全的，需要开发者自行保证。使用 SQLiteDatabase 时其内部锁可保证基本并发安全，但复合操作仍需事务或 `synchronized`。

---

### Q3：ContentProvider 和 AIDL 相比，各自适用什么场景？

**A**：

| 维度 | ContentProvider | AIDL |
|------|----------------|------|
| 数据模型 | 结构化数据（表/行/列） | 任意数据和方法 |
| 适用场景 | 数据共享 | 复杂业务逻辑 IPC |
| 复杂度 | 低，标准 API | 高，需定义接口 |
| 数据通知 | 内置 ContentObserver | 需自行实现 |

简单来说：**共享数据用 ContentProvider，调用方法用 AIDL**。

---

### Q4：为什么 Android 7.0 之后必须使用 FileProvider？

**A**：Android 7.0（API 24）起禁止通过 `file://` URI 向其他应用暴露文件，违反会抛出 `FileUriExposedException`。`file://` 直接暴露文件绝对路径存在安全隐患，且接收方可能无文件系统权限。FileProvider 通过 `content://` URI 提供临时、可控的访问权限（`FLAG_GRANT_READ_URI_PERMISSION`），权限精确到单个文件，Intent 结束后自动回收。

---

### Q5：ContentProvider 的 call() 方法有什么用？

**A**：`call()` 是 ContentProvider 的通用扩展方法：`fun call(method: String, arg: String?, extras: Bundle?): Bundle?`。不受 CRUD 模型限制，参数和返回值都是 Bundle，适合跨进程 RPC 风格调用（获取配置、触发同步等）。

---

### Q6：多个 ContentProvider 的 onCreate 执行顺序是怎样的？能否控制？

**A**：按 AndroidManifest 中声明的顺序在主线程串行执行，无法通过 API 直接控制。可通过调整 `<provider>` 声明顺序间接控制。Jetpack App Startup 通过 `InitializationProvider` + 依赖图管理初始化顺序，将多个库合并到一个 ContentProvider 中以减少启动耗时。每个 Provider 的 `onCreate()` 都会增加冷启动时间，应避免耗时操作。



---

## 五、实战与踩坑

### 5.1 ContentProvider 启动耗时影响冷启动

**问题**：每个 ContentProvider 的 `onCreate()` 在主线程串行执行，大量第三方库各自注册 Provider 会显著增加冷启动时间。

**排查方法**：

````java
public class MyApp extends Application {
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        Log.d("Startup", "attachBaseContext: " + SystemClock.elapsedRealtime());
    }

    @Override
    public void onCreate() {
        super.onCreate();
        // 与 attachBaseContext 的时间差 = 所有 ContentProvider.onCreate() 总耗时
        Log.d("Startup", "Application.onCreate: " + SystemClock.elapsedRealtime());
    }
}
````
**优化方案**：

1. **Jetpack App Startup 合并初始化**：实现 `Initializer` 接口，通过依赖图管理顺序，同时在 Manifest 中用 `tools:node="remove"` 禁用库自带的 Provider

````xml
<provider
    android:name="androidx.work.impl.WorkManagerInitializer"
    android:authorities="${applicationId}.workmanager-init"
    tools:node="remove" />
````
2. **延迟初始化**：非必要的 Provider 使用懒加载
3. **减少 Provider 数量**：审查依赖库，移除不必要的 ContentProvider

### 5.2 多进程并发安全

**问题**：CRUD 方法在跨进程调用时运行在 Binder 线程池，多客户端并发调用会产生线程安全问题。

**方案一：依赖 SQLiteDatabase 的内置锁**

SQLiteDatabase 在 WAL 模式下支持并发读，写操作通过内部锁串行化：

````java
@Override
public boolean onCreate() {
    dbHelper = new UserDbHelper(getContext());
    // 启用 WAL 模式，提升并发读性能
    dbHelper.getWritableDatabase().enableWriteAheadLogging();
    return true;
}
````
**方案二：使用 synchronized 保护复合操作**

````java
private final Object lock = new Object();

@Override
public Uri insert(Uri uri, ContentValues values) {
    synchronized (lock) {
        SQLiteDatabase db = dbHelper.getWritableDatabase();
        db.beginTransaction();
        try {
            // 先检查再插入，保证原子性
            Cursor cursor = db.query("user", null, "name=?",
                new String[]{values.getAsString("name")}, null, null, null);
            boolean exists = false;
            if (cursor != null) {
                exists = cursor.getCount() > 0;
                cursor.close();
            }
            if (exists) throw new IllegalStateException("用户已存在");
            long id = db.insert("user", null, values);
            db.setTransactionSuccessful();
            return ContentUris.withAppendedId(CONTENT_URI, id);
        } finally {
            db.endTransaction();
        }
    }
}
````
**方案三：使用 Room + ContentProvider** — Room 天然线程安全，Dao 方法可返回 `Cursor` 供 ContentProvider 使用。

### 5.3 call() 方法扩展：实现跨进程 RPC

`call()` 方法突破了 CRUD 的限制，可以实现灵活的跨进程调用：

#### Provider 端实现

````java
public class ConfigProvider extends ContentProvider {

    public static final String AUTHORITY = "com.example.app.config";

    @Override
    public Bundle call(String method, String arg, Bundle extras) {
        switch (method) {
            case "getConfig": {
                Bundle result = new Bundle();
                result.putString("value", getContext().getSharedPreferences("config", Context.MODE_PRIVATE)
                    .getString(arg, ""));
                return result;
            }
            case "clearCache": {
                Bundle result = new Bundle();
                result.putBoolean("success", deleteDir(getContext().getCacheDir()));
                return result;
            }
            case "syncData": {
                Bundle result = new Bundle();
                result.putBoolean("success", true);
                return result;
            }
            default:
                throw new IllegalArgumentException("Unknown method: " + method);
        }
    }

    private boolean deleteDir(File dir) {
        if (dir != null && dir.isDirectory()) {
            for (File child : dir.listFiles()) {
                deleteDir(child);
            }
        }
        return dir != null && dir.delete();
    }

    @Override
    public boolean onCreate() { return true; }
    @Override
    public Cursor query(Uri u, String[] p, String s, String[] a, String o) { return null; }
    @Override
    public Uri insert(Uri uri, ContentValues values) { return null; }
    @Override
    public int update(Uri u, ContentValues v, String s, String[] a) { return 0; }
    @Override
    public int delete(Uri uri, String s, String[] a) { return 0; }
    @Override
    public String getType(Uri uri) { return null; }
}
````
#### 调用方使用

````java
Uri configUri = Uri.parse("content://com.example.app.config");

// 获取配置
Bundle result = contentResolver.call(configUri, "getConfig", "theme_mode", null);
Log.d("Config", "theme_mode = " + (result != null ? result.getString("value") : ""));

// 触发同步
Bundle extras = new Bundle();
extras.putString("type", "incremental");
contentResolver.call(configUri, "syncData", null, extras);
````
### 5.4 其他常见踩坑

**踩坑 1：Cursor 未关闭导致内存泄漏** — 始终使用 `cursor?.use { }` 自动关闭。

**踩坑 2：Android 12+ exported 必须显式声明** — `targetSdk >= 31` 时 `<provider>` 必须显式声明 `android:exported`，否则安装失败。

**踩坑 3：notifyChange 未调用** — Provider 的 insert/update/delete 中必须调用 `context?.contentResolver?.notifyChange(uri, null)`，否则 ContentObserver 收不到通知。

---

## 总结

| 特性 | 说明 |
|------|------|
| 本质 | 基于 [[1-Binder机制原理]] 的 IPC 数据共享组件 |
| 启动时机 | 早于 `Application.onCreate()` |
| 线程模型 | onCreate 主线程；CRUD 在 Binder 线程池 |
| 数据传输 | CursorWindow（Ashmem 共享内存） |
| 扩展 | `call()` 支持自定义 RPC |

> 参见：[[1-Binder机制原理]]、[[3-BroadcastReceiver与ContentProvider]]

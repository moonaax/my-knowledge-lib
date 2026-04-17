# AIDL 使用与实践

> AIDL（Android Interface Definition Language）是 Android 提供的跨进程通信接口定义语言，底层基于 [[Binder机制原理]]，是实现 IPC 最常用、最灵活的方式之一。

---

## 一、核心概念

### 1.1 AIDL 是什么

AIDL 是一种 **接口定义语言**，用于定义客户端与服务端之间跨进程通信的接口契约。编译器会根据 `.aidl` 文件自动生成 Java/Kotlin 代码，包含 `Stub`（服务端骨架）和 `Proxy`（客户端代理）两个核心类，开发者无需手动编写 [[Binder机制原理]] 的底层 transact/onTransact 逻辑。

**核心流程：**

```
.aidl 文件 → 编译器生成 Stub/Proxy → 服务端实现 Stub → 客户端通过 Proxy 调用
```

### 1.2 支持的数据类型

| 类型 | 说明 |
|------|------|
| 基本类型 | `int`, `long`, `float`, `double`, `boolean`, `char`, `byte` |
| `String` / `CharSequence` | 直接支持 |
| `List` | 元素必须是 AIDL 支持的类型，实际接收端为 `ArrayList` |
| `Map` | Key/Value 必须是 AIDL 支持的类型，实际接收端为 `HashMap` |
| `Parcelable` | 需要额外声明 `.aidl` 文件 |
| `IBinder` | 可传递 Binder 对象引用 |
| 其他 AIDL 接口 | 可作为参数或返回值传递 |

> ⚠️ 不支持 `Serializable`，所有自定义对象必须实现 `Parcelable`。

### 1.3 方向标记：in / out / inout

AIDL 中非基本类型参数必须标记数据流向：

```aidl
interface IBookManager {
    void addBook(in Book book);       // 客户端 → 服务端（只读）
    void getBook(out Book book);      // 服务端 → 客户端（只写）
    void updateBook(inout Book book); // 双向传输
}
```

| 标记 | 含义 | 性能 |
|------|------|------|
| `in` | 数据从客户端传到服务端，服务端修改不影响客户端 | 最优，只序列化一次 |
| `out` | 数据从服务端传回客户端，客户端传入的值服务端收不到 | 只反序列化一次 |
| `inout` | 双向传输，客户端和服务端都能读写 | 最差，序列化 + 反序列化各一次 |

> 💡 基本类型默认 `in` 且不能更改；`String` 和 `CharSequence` 也默认 `in`。实际开发中优先使用 `in`，避免不必要的 `inout` 带来的性能开销。

### 1.4 Parcelable 传输

自定义对象跨进程传输需要两步：

**第一步：实现 Parcelable**

```java
// Book.java
import android.os.Parcel;
import android.os.Parcelable;

public class Book implements Parcelable {
    private final int id;
    private final String name;

    public Book(int id, String name) {
        this.id = id;
        this.name = name;
    }

    protected Book(Parcel parcel) {
        id = parcel.readInt();
        name = parcel.readString() != null ? parcel.readString() : "";
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(id);
        dest.writeString(name);
    }

    @Override
    public int describeContents() { return 0; }

    /**
     * out / inout 方向标记时需要此方法，AIDL 生成代码会调用
     */
    public void readFromParcel(Parcel parcel) {
        // 不可变字段，实际需要用可变字段
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel parcel) { return new Book(parcel); }
        @Override
        public Book[] newArray(int size) { return new Book[size]; }
    };

    public int getId() { return id; }
    public String getName() { return name; }
}
```

**第二步：声明 AIDL 文件**

```aidl
// Book.aidl — 包名必须与 Book.kt 一致
package com.example.aidl;
parcelable Book;
```

> ⚠️ 踩坑：`Book.aidl` 的包路径必须和 `Book.kt` 完全一致，否则编译报错。

---

## 二、原理与源码

AIDL 的本质是 [[Binder机制原理]] 的上层封装。编译器根据 `.aidl` 文件生成一个继承自 `IInterface` 的接口，内部包含 `Stub` 和 `Proxy` 两个关键类。

### 2.1 生成代码整体结构

以如下 AIDL 为例：

```aidl
// IBookManager.aidl
package com.example.aidl;
import com.example.aidl.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

编译后生成的 Java 代码结构（简化）：

```java
public interface IBookManager extends IInterface {

    // Stub — 服务端骨架，运行在服务端进程
    public static abstract class Stub extends Binder implements IBookManager {

        private static final String DESCRIPTOR = "com.example.aidl.IBookManager";
        static final int TRANSACTION_getBookList = IBinder.FIRST_CALL_TRANSACTION + 0;
        static final int TRANSACTION_addBook = IBinder.FIRST_CALL_TRANSACTION + 1;

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * 核心方法：判断本地还是远程
         */
        public static IBookManager asInterface(IBinder obj) {
            if (obj == null) return null;
            // 查找本地接口 — 同进程直接返回 Stub 本身
            IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (iin != null && iin instanceof IBookManager) {
                return (IBookManager) iin;
            }
            // 不同进程 — 返回 Proxy 代理
            return new Proxy(obj);
        }

        @Override
        public IBinder asBinder() { return this; }

        /**
         * 服务端分发逻辑：根据 code 调用对应方法
         */
        @Override
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags)
                throws RemoteException {
            switch (code) {
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    List<Book> result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    Book book = Book.CREATOR.createFromParcel(data);
                    this.addBook(book);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        // Proxy — 客户端代理，运行在客户端进程
        private static class Proxy implements IBookManager {
            private IBinder mRemote;

            Proxy(IBinder remote) { mRemote = remote; }

            @Override
            public IBinder asBinder() { return mRemote; }

            @Override
            public List<Book> getBookList() throws RemoteException {
                Parcel data = Parcel.obtain();
                Parcel reply = Parcel.obtain();
                try {
                    data.writeInterfaceToken(DESCRIPTOR);
                    // 通过 Binder 驱动发起跨进程调用
                    mRemote.transact(TRANSACTION_getBookList, data, reply, 0);
                    reply.readException();
                    return reply.createTypedArrayList(Book.CREATOR);
                } finally {
                    data.recycle();
                    reply.recycle();
                }
            }

            @Override
            public void addBook(Book book) throws RemoteException {
                Parcel data = Parcel.obtain();
                Parcel reply = Parcel.obtain();
                try {
                    data.writeInterfaceToken(DESCRIPTOR);
                    book.writeToParcel(data, 0);
                    mRemote.transact(TRANSACTION_addBook, data, reply, 0);
                    reply.readException();
                } finally {
                    data.recycle();
                    reply.recycle();
                }
            }
        }
    }
}
```

### 2.2 asInterface 的本地/远程判断

```
asInterface(IBinder obj)
    ├── obj.queryLocalInterface(DESCRIPTOR)
    │     ├── 非 null → 同进程，直接返回 Stub（无 IPC 开销）
    │     └── null    → 跨进程，返回 Proxy（走 Binder 驱动）
```

关键点：
- `Stub` 构造函数中调用了 `attachInterface(this, DESCRIPTOR)`，将自身注册为本地接口
- `queryLocalInterface` 在同进程时能找到这个注册，直接返回 Stub 实例
- 跨进程时 `IBinder` 实际是 `BinderProxy`，`queryLocalInterface` 返回 null

### 2.3 onTransact 分发逻辑

`onTransact` 运行在 **服务端的 Binder 线程池** 中（非主线程），流程：

```
客户端 Proxy.transact(code, data, reply, flags)
    → Binder 驱动传输
        → 服务端 Stub.onTransact(code, data, reply, flags)
            → switch(code) 分发到具体方法
                → 将结果写入 reply
                    → Binder 驱动回传
                        → 客户端从 reply 读取结果
```

> 💡 `transact` 默认是 **同步阻塞** 的，客户端线程会挂起直到服务端返回。如果设置 `flags = FLAG_ONEWAY`，则为异步调用，客户端不等待返回。

---

## 三、实战

### 3.1 完整 AIDL 跨进程通信示例

#### 3.1.1 定义 AIDL 接口

```aidl
// src/main/aidl/com/example/aidl/Book.aidl
package com.example.aidl;
parcelable Book;
```

```aidl
// src/main/aidl/com/example/aidl/IBookManager.aidl
package com.example.aidl;
import com.example.aidl.Book;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
}
```

#### 3.1.2 服务端实现

```java
// BookManagerService.java
public class BookManagerService extends Service {

    private final List<Book> bookList = new ArrayList<>(Arrays.asList(
        new Book(1, "Android 开发艺术探索"),
        new Book(2, "Kotlin 实战")
    ));

    private final IBookManager.Stub binder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() {
            return bookList;
        }

        @Override
        public void addBook(Book book) {
            bookList.add(book);
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

AndroidManifest.xml 中声明为独立进程：

```xml
<service
    android:name=".BookManagerService"
    android:process=":remote"
    android:exported="false" />
```

#### 3.1.3 客户端绑定

```java
// ClientActivity.java
public class ClientActivity extends AppCompatActivity {

    private IBookManager bookManager;

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // asInterface 自动判断同进程/跨进程
            bookManager = IBookManager.Stub.asInterface(service);
            try {
                List<Book> books = bookManager.getBookList();
                Log.d("AIDL", "书籍列表: " + books);
                bookManager.addBook(new Book(3, "深入理解 JVM"));
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            bookManager = null;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(connection);
        super.onDestroy();
    }
}
```

### 3.2 回调接口与 RemoteCallbackList

当服务端需要主动通知客户端时（如新书到货），需要定义回调接口。

#### 3.2.1 定义回调 AIDL

```aidl
// IOnNewBookArrivedListener.aidl
package com.example.aidl;
import com.example.aidl.Book;

interface IOnNewBookArrivedListener {
    void onNewBookArrived(in Book book);
}
```

扩展主接口：

```aidl
// IBookManager.aidl
package com.example.aidl;
import com.example.aidl.Book;
import com.example.aidl.IOnNewBookArrivedListener;

interface IBookManager {
    List<Book> getBookList();
    void addBook(in Book book);
    void registerListener(IOnNewBookArrivedListener listener);
    void unregisterListener(IOnNewBookArrivedListener listener);
}
```

#### 3.2.2 服务端使用 RemoteCallbackList

> ⚠️ 跨进程传递的对象经过序列化/反序列化后不再是同一个对象，普通 `List` 无法正确 `remove`。`RemoteCallbackList` 内部通过 `IBinder` 作为 key 来识别同一个客户端回调。

```java
public class BookManagerService extends Service {

    private final List<Book> bookList = new ArrayList<>();
    // 使用 RemoteCallbackList 管理跨进程回调
    private final RemoteCallbackList<IOnNewBookArrivedListener> listeners = new RemoteCallbackList<>();

    private final IBookManager.Stub binder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() { return bookList; }

        @Override
        public void addBook(Book book) {
            bookList.add(book);
            notifyNewBook(book);
        }

        @Override
        public void registerListener(IOnNewBookArrivedListener listener) {
            listeners.register(listener);
        }

        @Override
        public void unregisterListener(IOnNewBookArrivedListener listener) {
            listeners.unregister(listener);
        }
    };

    /**
     * 遍历通知所有已注册的客户端回调
     * beginBroadcast / finishBroadcast 必须配对使用
     */
    private void notifyNewBook(Book book) {
        int count = listeners.beginBroadcast();
        for (int i = 0; i < count; i++) {
            try {
                listeners.getBroadcastItem(i).onNewBookArrived(book);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        listeners.finishBroadcast();
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

#### 3.2.3 客户端注册回调

```java
public class ClientActivity extends AppCompatActivity {

    private IBookManager bookManager;

    private final IOnNewBookArrivedListener.Stub bookArrivedListener = new IOnNewBookArrivedListener.Stub() {
        @Override
        public void onNewBookArrived(Book book) {
            // ⚠️ 此回调运行在 Binder 线程池，更新 UI 需切换主线程
            runOnUiThread(() -> {
                Log.d("AIDL", "新书到货: " + book);
            });
        }
    };

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            bookManager = IBookManager.Stub.asInterface(service);
            try {
                bookManager.registerListener(bookArrivedListener);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            bookManager = null;
        }
    };

    @Override
    protected void onDestroy() {
        try {
            if (bookManager != null) {
                bookManager.unregisterListener(bookArrivedListener);
            }
        } catch (RemoteException e) {
            e.printStackTrace();
        }
        unbindService(connection);
        super.onDestroy();
    }
}
```

---

## 四、面试题

### Q1：AIDL 和 Messenger 有什么区别？

**A：** Messenger 底层也是 AIDL，但它将所有请求串行化到一个 Handler 中处理，适合轻量级、无需并发的场景。AIDL 支持多线程并发调用，适合需要高性能、多方法的复杂 IPC 场景。

| 对比项 | Messenger | AIDL |
|--------|-----------|------|
| 并发 | 串行处理 | 多线程并发 |
| 接口定义 | 基于 Message | 自定义接口方法 |
| 复杂度 | 简单 | 较复杂 |
| 适用场景 | 简单消息传递 | 复杂业务接口 |

### Q2：AIDL 方法调用是同步还是异步的？

**A：** 默认是 **同步阻塞** 的——客户端调用 `transact()` 后线程挂起，直到服务端 `onTransact()` 执行完毕并写入 reply。因此不能在客户端主线程调用耗时的 AIDL 方法，否则会 ANR。

如果在 AIDL 方法前加 `oneway` 关键字，则该方法变为异步调用，客户端不等待返回：

```aidl
oneway void asyncMethod(in Book book);
```

> `oneway` 方法不能有返回值，也不能有 `out`/`inout` 参数。

### Q3：为什么不能用普通 List 管理跨进程回调？

**A：** 跨进程传递的对象经过 Parcel 序列化/反序列化后会生成新的对象实例，客户端注册和反注册时传入的 listener 在服务端看来是两个不同的对象，`List.remove()` 无法匹配。

`RemoteCallbackList` 内部使用 `IBinder`（即 `listener.asBinder()`）作为 key 存储在 `ArrayMap` 中。同一个客户端 Binder 代理对应同一个 `IBinder`，因此能正确识别和移除。此外它还能自动处理客户端进程死亡的情况（通过 `DeathRecipient`）。

### Q4：AIDL 中 in、out、inout 的区别和性能影响？

**A：**
- `in`：数据从客户端序列化传到服务端，服务端的修改不会回传。性能最好。
- `out`：客户端不传数据，服务端写入后回传给客户端。
- `inout`：双向传输，序列化两次（去一次、回一次），性能最差。

实际开发中绝大多数场景用 `in` 即可，只有确实需要服务端修改并回传时才用 `inout`。

### Q5：客户端调用 AIDL 方法时，服务端在哪个线程执行？

**A：** 服务端的 AIDL 方法运行在 **Binder 线程池** 中，不是主线程。这意味着：
- 服务端方法实现必须是 **线程安全** 的
- 如果需要操作 UI 或访问主线程资源，需要通过 Handler 切换
- Binder 线程池默认最大 16 个线程（可通过 `ProcessState::setThreadPoolMaxThreadCount` 调整）

### Q6：AIDL 调用过程中服务端抛出异常，客户端会怎样？

**A：** 服务端在 `onTransact` 中抛出的异常会通过 `reply.writeException()` 序列化传回客户端，客户端在 `reply.readException()` 时会重新抛出。支持传递的异常类型包括：`SecurityException`、`BadParcelableException`、`IllegalArgumentException`、`NullPointerException`、`IllegalStateException` 等。

如果是未捕获的 `RuntimeException`，会导致服务端进程崩溃，客户端收到 `DeadObjectException`。

---

## 五、实战与踩坑

### 5.1 线程安全

AIDL 方法在服务端的 Binder 线程池中并发执行，**必须保证线程安全**：

```java
public class BookManagerService extends Service {

    // ✅ 使用 CopyOnWriteArrayList 保证线程安全
    private final CopyOnWriteArrayList<Book> bookList = new CopyOnWriteArrayList<>();

    private final IBookManager.Stub binder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() {
            return new ArrayList<>(bookList);
        }

        @Override
        public void addBook(Book book) {
            bookList.add(book);
        }
    };

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

常见线程安全策略：

| 方案 | 适用场景 |
|------|---------|
| `CopyOnWriteArrayList` | 读多写少的列表 |
| `ConcurrentHashMap` | 键值对存储 |
| `synchronized` | 需要原子操作的复合逻辑 |
| `AtomicInteger` 等原子类 | 简单计数器 |

### 5.2 权限校验

服务端应校验客户端身份，防止未授权的进程调用：

**方式一：自定义 Permission**

```xml
<!-- AndroidManifest.xml -->
<permission
    android:name="com.example.permission.ACCESS_BOOK_SERVICE"
    android:protectionLevel="signature" />
```

```java
private final IBookManager.Stub binder = new IBookManager.Stub() {

    @Override
    public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        // 校验自定义权限
        int check = checkCallingPermission("com.example.permission.ACCESS_BOOK_SERVICE");
        if (check == PackageManager.PERMISSION_DENIED) {
            Log.w("AIDL", "权限校验失败，uid=" + Binder.getCallingUid());
            return false;
        }
        return super.onTransact(code, data, reply, flags);
    }

    @Override
    public List<Book> getBookList() { return new ArrayList<>(bookList); }
    @Override
    public void addBook(Book book) { bookList.add(book); }
};
```

**方式二：校验调用方包名**

```java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    int callingUid = Binder.getCallingUid();
    String[] packages = getPackageManager().getPackagesForUid(callingUid);
    if (packages == null) return false;
    boolean allowed = false;
    for (String pkg : packages) {
        if (pkg.startsWith("com.example")) {
            allowed = true;
            break;
        }
    }
    if (!allowed) return false;
    return super.onTransact(code, data, reply, flags);
}
```

> 💡 `Binder.getCallingUid()` 和 `Binder.getCallingPid()` 只在 `onTransact` 中有效，在其他地方调用返回的是当前进程的 uid/pid。

### 5.3 大数据传输

[[Binder机制原理]] 中 Binder 事务缓冲区大小约为 **1MB**（所有进行中的事务共享），单次传输数据过大会抛出 `TransactionTooLargeException`。

**解决方案一：分页查询**

```aidl
interface IBookManager {
    List<Book> getBookList(int offset, int limit);
}
```

```java
@Override
public List<Book> getBookList(int offset, int limit) {
    int end = Math.min(offset + limit, bookList.size());
    if (offset < bookList.size()) {
        return new ArrayList<>(bookList.subList(offset, end));
    } else {
        return new ArrayList<>();
    }
}
```

**解决方案二：使用 ParcelFileDescriptor 传输大文件**

```aidl
interface IFileTransfer {
    ParcelFileDescriptor openFile(String path);
}
```

```java
// 服务端
@Override
public ParcelFileDescriptor openFile(String path) throws RemoteException {
    File file = new File(path);
    try {
        return ParcelFileDescriptor.open(file, ParcelFileDescriptor.MODE_READ_ONLY);
    } catch (FileNotFoundException e) {
        throw new RemoteException(e.getMessage());
    }
}

// 客户端
ParcelFileDescriptor pfd = fileTransfer.openFile("/data/large_file.dat");
InputStream inputStream = new ParcelFileDescriptor.AutoCloseInputStream(pfd);
try {
    // 读取大文件内容
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = inputStream.read(buffer)) != -1) {
        // 处理数据
    }
} finally {
    inputStream.close();
}
```

**解决方案三：共享内存（MemoryFile / SharedMemory）**

适用于需要频繁读写大块数据的场景（如音视频帧传输）：

```java
// 服务端：创建共享内存并写入数据
SharedMemory sharedMemory = SharedMemory.create("shared_data", dataSize);
ByteBuffer buffer = sharedMemory.mapReadWrite();
buffer.put(largeByteArray);
SharedMemory.unmap(buffer);

// 通过 AIDL 传递 SharedMemory（实现了 Parcelable）
```

### 5.4 其他常见踩坑

**1. 死亡代理 DeathRecipient**

客户端应监听服务端进程死亡，及时重连：

```java
private final IBinder.DeathRecipient deathRecipient = () -> {
    bookManager = null;
    // 重新绑定服务
    bindService(intent, connection, Context.BIND_AUTO_CREATE);
};

@Override
public void onServiceConnected(ComponentName name, IBinder service) {
    try {
        service.linkToDeath(deathRecipient, 0);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
    bookManager = IBookManager.Stub.asInterface(service);
}
```

**2. oneway 的限制**

```aidl
// oneway 修饰的方法：异步调用，客户端不阻塞
oneway interface IAsyncCallback {
    void onResult(in String data);
}
```

- `oneway` 方法不能有返回值（必须 `void`）
- 不能有 `out` / `inout` 参数
- 异常不会传回客户端
- 多个 `oneway` 调用按顺序到达服务端（同一个 Binder 引用上串行）

**3. AIDL 文件修改后需要 Clean + Rebuild**

AIDL 编译器有时不会检测到增量变化，修改 `.aidl` 文件后建议执行 `Build → Clean Project → Rebuild Project`，避免生成代码不一致导致的运行时崩溃。

---

> 📌 **总结**：AIDL 是 Android 跨进程通信的核心工具，理解其背后的 [[Binder机制原理]] 是关键。实际开发中需要特别注意线程安全、权限校验和数据大小限制，合理选择 `in`/`out`/`inout` 方向标记以优化性能。

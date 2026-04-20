# Messenger 跨进程通信

> Messenger 是 Android 提供的基于 [[2-AIDL使用与实践]] 的轻量级跨进程通信方案，底层依赖 [[1-Binder机制原理]]，通过 [[6-Handler消息机制]] 串行处理消息。

---

## 一、核心概念

### 1.1 Messenger 是什么

Messenger 是 Android 对 AIDL 的一层轻量封装，本质上是一个可以跨进程传递的 `IMessenger` 接口引用。它允许不同进程之间通过 `Message` 对象进行通信，而无需手动编写 `.aidl` 文件。

核心设计思路：

- 服务端创建一个 `Handler`，用它构造一个 `Messenger`
- 客户端通过 `bindService` 拿到服务端的 `IBinder`，再用它构造一个 `Messenger`
- 客户端通过 `Messenger.send(Message)` 向服务端发送消息
- 服务端的 `Handler` 收到并处理消息

````
客户端进程                          服务端进程
┌──────────┐   Binder IPC    ┌──────────────┐
│ Messenger│ ──────────────> │   Handler    │
│ (Proxy)  │   send(msg)    │ handleMessage│
└──────────┘                 └──────────────┘
````
### 1.2 Messenger vs AIDL

| 对比维度 | Messenger | AIDL |
|---------|-----------|------|
| 底层实现 | 基于 AIDL（IMessenger.aidl） | 直接定义 .aidl 接口 |
| 线程模型 | **串行**——消息排队由 Handler 逐个处理 | **并行**——Binder 线程池并发处理 |
| 数据载体 | `Message`（what/arg1/arg2/Bundle/obj） | 自定义方法参数，支持 in/out/inout |
| 复杂度 | 低，无需写 .aidl 文件 | 高，需定义接口、处理线程安全 |
| 返回值 | 无直接返回值，需通过 `replyTo` 回传 | 方法可直接返回值 |
| 性能 | 串行处理，高并发场景下有瓶颈 | 并发处理，吞吐量更高 |

### 1.3 适用场景

- **低频、轻量的跨进程通信**：如主进程向后台服务发送配置更新
- **不需要并发处理**：消息可以排队串行处理的场景
- **快速原型开发**：不想写 AIDL 文件，快速实现 IPC
- **单向或简单双向通信**：通过 `replyTo` 实现双向

不适用场景：
- 高并发请求（如多客户端同时调用）
- 需要复杂接口定义（多方法、自定义 Parcelable 参数）
- 需要同步返回值

---

## 二、原理与源码

### 2.1 IMessenger.aidl

Messenger 的底层是系统预定义的 AIDL 接口，位于 `frameworks/base/core/java/android/os/IMessenger.aidl`：

````java
// IMessenger.aidl（系统源码）
package android.os;

import android.os.Message;

/** @hide */
oneway interface IMessenger {
    void send(in Message msg);
}
````
关键点：
- `oneway` 修饰符表示调用是**异步**的，客户端调用 `send()` 后不会阻塞等待
- 参数是 `in Message`，即 Message 对象会被序列化传递到服务端进程

### 2.2 Messenger 源码分析

````java
// android.os.Messenger 核心源码
public final class Messenger implements Parcelable {
    private final IMessenger mTarget;

    // 服务端使用：用 Handler 构造
    public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }

    // 客户端使用：用 IBinder 构造
    public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }

    // 发送消息——本质是 AIDL 的 Binder 调用
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }

    // 获取底层 Binder，用于 onBind() 返回
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
}
````
### 2.3 Handler 中的 IMessenger 实现

`Handler.getIMessenger()` 返回的是一个内部类：

````java
// Handler 内部
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}

private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
````
这里的 `MessengerImpl` 继承自 `IMessenger.Stub`（即 Binder 的服务端实现），当客户端调用 `send()` 时：

1. 客户端的 `Messenger.send()` → `IMessenger.Stub.Proxy.send()`（跨进程 Binder 调用）
2. 服务端的 `MessengerImpl.send()` 被触发
3. 内部调用 `Handler.sendMessage(msg)` 将消息投递到 [[6-Handler消息机制]] 的 MessageQueue
4. Handler 在其 Looper 线程中串行处理消息

### 2.4 完整调用链

````
客户端 Messenger.send(msg)
  → IMessenger.Stub.Proxy.send(msg)     // 客户端代理
    → Binder Driver (内核)                // 跨进程传输
      → IMessenger.Stub.onTransact()     // 服务端 Binder 线程
        → MessengerImpl.send(msg)
          → Handler.sendMessage(msg)     // 投递到 MessageQueue
            → Handler.handleMessage(msg) // Looper 线程串行处理
````
---

## 三、实战：完整的 Messenger 双向通信示例

### 3.1 服务端 Service

````java
// MessengerService.java
public class MessengerService extends Service {

    public static final int MSG_SAY_HELLO = 1;
    public static final int MSG_REPLY = 2;

    // 服务端 Handler，处理客户端发来的消息
    private final Handler handler = new Handler(Looper.getMainLooper()) {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_SAY_HELLO:
                    String data = msg.getData().getString("content");
                    Log.d("MessengerService", "收到客户端消息: " + data);

                    // 通过 replyTo 回复客户端
                    if (msg.replyTo != null) {
                        Message reply = Message.obtain(null, MSG_REPLY);
                        Bundle bundle = new Bundle();
                        bundle.putString("reply", "服务端已收到: " + data);
                        reply.setData(bundle);
                        try {
                            msg.replyTo.send(reply);
                        } catch (RemoteException e) {
                            e.printStackTrace();
                        }
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    };

    // 用 Handler 构造 Messenger
    private final Messenger messenger = new Messenger(handler);

    @Override
    public IBinder onBind(Intent intent) {
        // 返回 Messenger 内部的 Binder
        return messenger.getBinder();
    }
}
````
### 3.2 AndroidManifest 配置

````xml
<service
    android:name=".MessengerService"
    android:process=":remote"
    android:exported="false" />
````
`android:process=":remote"` 使 Service 运行在独立进程中。

### 3.3 客户端 Activity

````java
// MessengerActivity.java
public class MessengerActivity extends AppCompatActivity {

    private Messenger serverMessenger;
    private boolean isBound = false;

    // 客户端 Handler，接收服务端的回复
    private final Handler clientHandler = new Handler(Looper.getMainLooper()) {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MessengerService.MSG_REPLY:
                    String reply = msg.getData().getString("reply");
                    Log.d("MessengerClient", "收到服务端回复: " + reply);
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    };

    // 客户端 Messenger，用于接收回复
    private final Messenger clientMessenger = new Messenger(clientHandler);

    private final ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 用服务端返回的 IBinder 构造 Messenger
            serverMessenger = new Messenger(service);
            isBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            serverMessenger = null;
            isBound = false;
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 绑定服务
        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }

    // 发送消息到服务端
    public void sendMessage() {
        if (!isBound) return;
        Message msg = Message.obtain(null, MessengerService.MSG_SAY_HELLO);
        Bundle bundle = new Bundle();
        bundle.putString("content", "Hello from client!");
        msg.setData(bundle);
        // 关键：设置 replyTo，服务端通过它回复
        msg.replyTo = clientMessenger;
        try {
            serverMessenger.send(msg);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (isBound) {
            unbindService(connection);
            isBound = false;
        }
    }
}
````
### 3.4 双向通信流程

````
1. 客户端 bindService() → 服务端 onBind() 返回 Messenger.binder
2. 客户端 onServiceConnected() 拿到 IBinder，构造 serverMessenger
3. 客户端构造 Message，设置 replyTo = clientMessenger
4. 客户端调用 serverMessenger.send(msg) → 跨进程到服务端
5. 服务端 Handler.handleMessage() 处理消息
6. 服务端通过 msg.replyTo 拿到客户端的 Messenger
7. 服务端调用 clientMessenger.send(reply) → 跨进程回到客户端
8. 客户端 clientHandler.handleMessage() 收到回复
````
---

## 四、面试题

### Q1：Messenger 的底层实现原理是什么？

Messenger 底层基于 AIDL。系统预定义了 `IMessenger.aidl` 接口，其中只有一个 `oneway void send(in Message msg)` 方法。Messenger 构造时，服务端通过 `Handler` 内部的 `MessengerImpl`（继承 `IMessenger.Stub`）提供 Binder 实体；客户端通过 `IMessenger.Stub.asInterface(binder)` 获取代理。调用 `send()` 本质是一次 [[1-Binder机制原理]] 的跨进程调用，消息到达服务端后通过 `Handler.sendMessage()` 投递到 MessageQueue 串行处理。

### Q2：Messenger 为什么是串行的？能否改成并行？

因为服务端的 `MessengerImpl.send()` 内部调用的是 `Handler.sendMessage()`，消息被投递到 Handler 关联的 Looper 的 MessageQueue 中，由 Looper 逐条取出处理。即使多个客户端同时发送消息，也会在 MessageQueue 中排队。

如果需要并行处理，应直接使用 [[2-AIDL使用与实践]]，AIDL 的方法调用在 Binder 线程池中并发执行。Messenger 本身无法改成并行，这是其设计决定的。

### Q3：Messenger 如何实现双向通信？

通过 `Message.replyTo` 字段。客户端发送消息时，将自己的 `Messenger` 设置到 `msg.replyTo`。服务端收到消息后，从 `msg.replyTo` 取出客户端的 Messenger，调用其 `send()` 方法即可将回复发送回客户端。`replyTo` 字段是 `Messenger` 类型，本身实现了 `Parcelable`，可以跨进程传递。

### Q4：Messenger 传递数据有什么限制？

- `Message.obj` 在跨进程时**不能**传递自定义对象（仅限系统 Parcelable），应使用 `Message.setData(Bundle)` 传递数据
- Bundle 中的数据必须是可序列化的（基本类型、String、Parcelable、Serializable 等）
- 受 Binder 事务缓冲区限制（通常 1MB），单次传输数据不能过大
- 不支持直接传递文件描述符（需要 AIDL 的 `ParcelFileDescriptor`）

### Q5：Messenger 和 BroadcastReceiver 跨进程通信有什么区别？

| 维度 | Messenger | BroadcastReceiver |
|------|-----------|-------------------|
| 通信模式 | 点对点，需要 bindService | 广播，一对多 |
| 实时性 | 高，直接 Binder 调用 | 低，经过 AMS 中转 |
| 双向通信 | 支持（replyTo） | 不直接支持 |
| 安全性 | 较高，基于 Service 绑定 | 较低，需要权限控制 |
| 数据量 | Bundle，受 Binder 限制 | Intent extras，同样受限 |

### Q6：oneway 修饰符对 Messenger 有什么影响？

`IMessenger.aidl` 中的 `send()` 方法被 `oneway` 修饰，意味着：
- 客户端调用 `send()` 后**立即返回**，不会阻塞等待服务端处理完成
- 如果服务端处理抛出异常，客户端**无法感知**
- 多次调用 `send()` 时，消息按发送顺序到达服务端（oneway 保证同一 Binder 引用的调用顺序）
- 这也是 Messenger 没有返回值的根本原因——异步调用无法同步返回结果

---

## 五、实战与踩坑

### 5.1 Messenger 的局限性

**1. 无法定义接口方法**

Messenger 只有一个 `send(Message)` 方法，所有业务逻辑都要通过 `msg.what` 区分，当消息类型增多时代码可读性差：

````java
// 随着业务增长，Handler 变成巨大的 switch 分支
@Override
public void handleMessage(Message msg) {
    switch (msg.what) {
        case 1: handleLogin(msg); break;
        case 2: handleLogout(msg); break;
        case 3: handleSync(msg); break;
        case 4: handleConfig(msg); break;
        // ... 越来越多
    }
}
````
此时应考虑迁移到 [[2-AIDL使用与实践]]。

**2. 无同步返回值**

由于 `oneway` 特性，`send()` 是异步的，无法直接获取返回值。所有"返回"都需要通过 `replyTo` 异步回调，增加了代码复杂度。

**3. 串行处理瓶颈**

所有消息在 Handler 线程串行处理，如果某条消息处理耗时较长，后续消息会被阻塞。解决方案：

````java
@Override
public void handleMessage(Message msg) {
    switch (msg.what) {
        case MSG_HEAVY_TASK:
            final Messenger replyTo = msg.replyTo;
            // 将耗时操作转移到子线程
            new Thread(() -> {
                String result = doHeavyWork();
                if (replyTo != null) {
                    Message reply = Message.obtain(null, MSG_RESULT);
                    Bundle bundle = new Bundle();
                    bundle.putString("result", result);
                    reply.setData(bundle);
                    try {
                        replyTo.send(reply);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
            break;
    }
}
````
### 5.2 Message.replyTo 机制详解

`replyTo` 是 `Message` 类中的一个 `Messenger` 类型字段：

````java
// Message.java
public Messenger replyTo;
````
它能跨进程传递的原因：
- `Messenger` 实现了 `Parcelable`
- `Message` 在序列化时会将 `replyTo` 一并写入 `Parcel`
- 服务端反序列化后得到的 `replyTo` 是客户端 Messenger 的 Binder 代理

**踩坑：replyTo 为 null**

如果客户端忘记设置 `replyTo`，服务端取到的就是 `null`，必须做空判断：

````java
if (msg.replyTo != null) {
    try {
        msg.replyTo.send(responseMsg);
    } catch (RemoteException e) {
        e.printStackTrace();
    }
} else {
    Log.w(TAG, "客户端未设置 replyTo，无法回复");
}
````
### 5.3 常见踩坑汇总

**1. Message.obj 跨进程崩溃**

````java
// ❌ 错误：跨进程时 obj 不能是自定义对象
Message msg1 = Message.obtain();
msg1.obj = new MyData("hello");  // 跨进程会崩溃

// ✅ 正确：使用 Bundle
Message msg2 = Message.obtain();
Bundle bundle = new Bundle();
bundle.putParcelable("data", new MyData("hello"));
msg2.setData(bundle);
````
`Message.obj` 在同进程内可以传递任意对象，但跨进程时只支持系统框架内的 Parcelable 对象。

**2. 忘记 unbindService 导致泄漏**

````java
// 必须在 onDestroy 中解绑
@Override
protected void onDestroy() {
    super.onDestroy();
    if (isBound) {
        unbindService(connection);
        isBound = false;
    }
}
````
**3. Handler 内存泄漏**

匿名内部类 Handler 持有外部类引用，可能导致 Activity/Service 泄漏。推荐使用静态内部类 + 弱引用：

````java
public class MessengerService extends Service {

    private static class IncomingHandler extends Handler {
        private final WeakReference<MessengerService> serviceRef;

        IncomingHandler(MessengerService service) {
            super(Looper.getMainLooper());
            serviceRef = new WeakReference<>(service);
        }

        @Override
        public void handleMessage(Message msg) {
            MessengerService service = serviceRef.get();
            if (service != null) {
                service.handleIncomingMessage(msg);
            } else {
                Log.w("Handler", "Service already destroyed");
            }
        }
    }

    private final Handler handler = new IncomingHandler(this);
    private final Messenger messenger = new Messenger(handler);

    private void handleIncomingMessage(Message msg) {
        // 实际处理逻辑
    }

    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
````
**4. 服务端进程被杀后的 DeadObjectException**

当服务端进程意外终止时，客户端调用 `send()` 会抛出 `RemoteException`（通常是 `DeadObjectException`）：

````java
public void sendMessage() {
    try {
        if (serverMessenger != null) {
            serverMessenger.send(msg);
        }
    } catch (DeadObjectException e) {
        Log.e(TAG, "服务端进程已死亡，尝试重新绑定");
        rebindService();
    } catch (RemoteException e) {
        e.printStackTrace();
    }
}
````
### 5.4 Messenger vs AIDL 选型建议

````
需要跨进程通信？
  ├── 通信频率低、逻辑简单 → Messenger ✅
  ├── 需要并发处理多客户端 → AIDL ✅
  ├── 需要同步返回值 → AIDL ✅
  ├── 接口方法 > 3 个 → AIDL ✅
  └── 快速原型验证 → Messenger ✅
````
---

> 相关主题：[[1-Binder机制原理]] · [[2-AIDL使用与实践]] · [[6-Handler消息机制]]

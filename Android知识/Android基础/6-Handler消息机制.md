# Handler 消息机制

## 一、为什么需要 Handler？

Android 有一条铁律：**只有主线程（UI 线程）才能更新 UI**。

但是网络请求、数据库操作这些耗时任务必须在子线程做（否则 ANR）。那子线程拿到数据后怎么更新 UI？答案就是 Handler 消息机制——它是 Android 线程间通信的核心方案。

````java
// 经典场景：子线程获取数据 → 主线程更新 UI
Handler handler = new Handler(Looper.getMainLooper());

new Thread(() -> {
    String data = fetchDataFromNetwork();  // 子线程耗时操作
    handler.post(() -> {
        textView.setText(data);  // 切回主线程更新 UI
    });
}).start();
````
---

## 二、四大核心角色

````
┌─────────────────────────────────────────────────┐
│                    Thread                        │
│                                                  │
│   ┌──────────┐    ┌──────────────────────────┐  │
│   │  Handler  │───→│     MessageQueue         │  │
│   │ (发送消息) │    │  ┌───┬───┬───┬───┬───┐  │  │
│   └──────────┘    │  │ M │ M │ M │ M │ M │  │  │
│        ↑          │  └───┴───┴───┴───┴───┘  │  │
│        │          └──────────────────────────┘  │
│        │                     ↓                   │
│        │              ┌──────────┐               │
│        └──────────────│  Looper  │               │
│         (回调处理)     │ (循环取消息)│               │
│                       └──────────┘               │
└─────────────────────────────────────────────────┘
````
### 2.1 Message（消息）

Message 是消息的载体，包含：

| 字段 | 含义 |
|------|------|
| `what` | 消息标识（int），用来区分不同类型的消息 |
| `arg1, arg2` | 简单的 int 数据 |
| `obj` | 携带的对象数据 |
| `target` | 处理这条消息的 Handler |
| `when` | 消息的执行时间（绝对时间） |
| `next` | 下一条消息（链表结构） |

**Message 的复用池：**

Message 内部维护了一个**链表结构的对象池**（最大 50 个），避免频繁创建对象。

````java
// ✅ 推荐：从复用池获取
Message msg = Message.obtain();
// 或
Message msg = handler.obtainMessage();

// ❌ 不推荐：直接 new
Message msg = new Message();
````
`obtain()` 从池中取一个复用的 Message，用完后 `recycle()` 放回池中。这是一个典型的**享元模式**。

### 2.2 MessageQueue（消息队列）

MessageQueue 并不是一个真正的队列（Queue），而是一个**按时间排序的单链表**。

核心方法：
- `enqueueMessage()`：插入消息（按 when 时间排序插入链表）
- `next()`：取出下一条消息（如果没有到期的消息，会阻塞）

````
链表结构（按 when 排序）：
head → [when=100] → [when=200] → [when=300] → null
````
**next() 方法的阻塞机制：**

````java
// 简化版 next() 逻辑
Message next() {
    for (;;) {  // 死循环
        nativePollOnce(ptr, nextPollTimeoutMillis);  // 阻塞等待
        // 检查是否有到期的消息
        if (msg != null && now >= msg.when) {
            return msg;  // 返回消息
        } else {
            nextPollTimeoutMillis = msg.when - now;  // 计算等待时间
        }
    }
}
````
`nativePollOnce` 是 native 方法，底层用 Linux 的 **epoll 机制**实现阻塞等待，不会消耗 CPU。

### 2.3 Looper（循环器）

Looper 的职责很简单：**不断从 MessageQueue 中取消息，交给对应的 Handler 处理**。

````java
// Looper.loop() 简化版
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;

    for (;;) {  // 死循环
        Message msg = queue.next();  // 取消息（可能阻塞）
        if (msg == null) {
            return;  // 消息为 null 说明 Looper 退出了
        }
        msg.target.dispatchMessage(msg);  // 交给 Handler 处理
        msg.recycleUnchecked();  // 回收 Message
    }
}
````
**每个线程最多只有一个 Looper：**

````java
// Looper 通过 ThreadLocal 保证线程唯一
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<>();

private static void prepare() {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper());
}
````
**主线程的 Looper 在哪里创建的？**

在 `ActivityThread.main()` 中：

````java
// App 启动的入口
public static void main(String[] args) {
    Looper.prepareMainLooper();  // 创建主线程 Looper
    ActivityThread thread = new ActivityThread();
    thread.attach(false);
    Looper.loop();  // 开始循环（永远不会退出）
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
````
### 2.4 Handler（处理器）

Handler 负责发送消息和处理消息。

**发送消息：**
````java
// 发送即时消息
handler.sendMessage(msg);
handler.sendEmptyMessage(what);

// 发送延迟消息
handler.sendMessageDelayed(msg, delayMillis);
handler.postDelayed(runnable, delayMillis);

// 发送定时消息
handler.sendMessageAtTime(msg, uptimeMillis);

// post 系列（本质是把 Runnable 包装成 Message）
handler.post(runnable);
handler.postDelayed(runnable, delayMillis);
````
**处理消息的优先级：**

````java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        // 优先级1：Message 自带的 callback（post 系列）
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            // 优先级2：Handler 构造时传入的 Callback
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 优先级3：Handler 子类重写的 handleMessage
        handleMessage(msg);
    }
}
````
---

## 三、ThreadLocal

ThreadLocal 是实现"每个线程一个 Looper"的关键。它为每个线程维护一份独立的变量副本。

````java
// 简单理解
ThreadLocal<String> threadLocal = new ThreadLocal<>();

new Thread(() -> {
    threadLocal.set("Thread A");
    System.out.println(threadLocal.get());  // "Thread A"
}).start();

new Thread(() -> {
    threadLocal.set("Thread B");
    System.out.println(threadLocal.get());  // "Thread B"
}).start();
````
**原理：**
- 每个 Thread 内部有一个 `ThreadLocalMap`
- `ThreadLocal.set(value)` → 把 value 存到当前线程的 ThreadLocalMap 中
- `ThreadLocal.get()` → 从当前线程的 ThreadLocalMap 中取值
- key 是 ThreadLocal 对象本身（弱引用），value 是存储的值

````
Thread A 的 ThreadLocalMap:
  { ThreadLocal@123 → "Thread A" }

Thread B 的 ThreadLocalMap:
  { ThreadLocal@123 → "Thread B" }
````
**内存泄漏风险：**
- key 是弱引用，ThreadLocal 对象被回收后 key 变成 null
- 但 value 是强引用，不会被回收 → 内存泄漏
- 解决：用完后调用 `threadLocal.remove()`

---

## 四、同步屏障（Sync Barrier）

同步屏障是 MessageQueue 中的一种特殊机制，用于**优先处理异步消息**。

### 4.1 同步消息 vs 异步消息

默认情况下，所有消息都是同步消息，按顺序执行。异步消息可以"插队"——当设置了同步屏障后，同步消息会被阻塞，只有异步消息能被取出执行。

````java
// 设置异步消息
Message msg = Message.obtain();
msg.setAsynchronous(true);  // 标记为异步消息
handler.sendMessage(msg);
````
### 4.2 同步屏障的工作原理

````
正常情况：
[同步1] → [同步2] → [异步1] → [同步3]
按顺序执行：同步1 → 同步2 → 异步1 → 同步3

设置同步屏障后：
[屏障] → [同步1] → [同步2] → [异步1] → [同步3]
跳过同步消息，只执行异步消息：异步1
同步1、同步2、同步3 被阻塞
````
### 4.3 应用场景

**View 的绘制就用了同步屏障！**

````java
// ViewRootImpl.scheduleTraversals()
void scheduleTraversals() {
    // 1. 设置同步屏障，阻塞普通消息
    mTraversalBarrier = mHandler.getLooper().getQueue()
        .postSyncBarrier();

    // 2. 发送异步消息（绘制任务）
    mChoreographer.postCallback(
        Choreographer.CALLBACK_TRAVERSAL,
        mTraversalRunnable, null);
}

void doTraversal() {
    // 3. 移除同步屏障
    mHandler.getLooper().getQueue()
        .removeSyncBarrier(mTraversalBarrier);

    // 4. 执行绘制
    performTraversals();
}
````
这样做的目的是：**确保 UI 绘制消息优先执行**，不会被其他消息（比如你 post 的 Runnable）延迟，保证 16ms 内完成一帧的绘制。

---

## 五、IdleHandler

IdleHandler 是 MessageQueue 提供的一个回调接口，当消息队列**空闲时**（没有消息要处理或者下一条消息还没到时间）会被调用。

````java
Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override
    public boolean queueIdle() {
        // 消息队列空闲时执行
        // 适合做一些不紧急的初始化工作
        doSomeLazyInit();

        return false;  // false：执行一次后移除
                        // true：每次空闲都执行
    }
});
````
**使用场景：**
- 延迟初始化（启动优化：把非必要的初始化放到空闲时执行）
- GC 回调（ActivityThread 中用 IdleHandler 触发 GC）
- 页面绘制完成后的操作（比 onWindowFocusChanged 更精确）

---

## 六、主线程为什么不会因为 Looper.loop() 的死循环而 ANR？

这是面试超高频题。

**先理解 ANR 是什么：**
- ANR 不是因为"主线程在循环"，而是因为"主线程在规定时间内没有处理完某个事件"
- 比如：触摸事件 5 秒内没响应、BroadcastReceiver 10 秒内没执行完

**Looper.loop() 的死循环不会 ANR 的原因：**

1. `MessageQueue.next()` 在没有消息时会调用 `nativePollOnce()` **阻塞**，底层用 epoll 机制，线程会休眠，**不消耗 CPU**
2. 当有新消息到来时（比如触摸事件、Activity 生命周期回调），通过 `nativeWake()` 唤醒线程
3. Android 的所有事件（触摸、绘制、生命周期）都是通过 Handler 消息驱动的
4. ANR 是因为某条消息的处理时间太长，阻塞了后续消息的处理，而不是因为循环本身

````
正常情况：
loop() → 取消息 → 处理（1ms）→ 取消息 → 处理（2ms）→ 取消息 → 休眠...

ANR 情况：
loop() → 取消息 → 处理（耗时 6 秒！）→ 后面的触摸事件等不到处理 → ANR
````
---

## 七、Handler 内存泄漏

### 7.1 为什么会泄漏？

````java
// ❌ 经典泄漏写法
public class MyActivity extends Activity {
    private Handler handler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            // 匿名内部类持有外部类（Activity）的引用
            textView.setText("hello");
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        handler.postDelayed(() -> {
            // 这个 Runnable 也持有 Activity 引用
        }, 60000);  // 延迟 60 秒
    }
}
````
**泄漏链：**
````
主线程 Looper → MessageQueue → Message → Handler → Activity
````
Activity 关闭后，如果 MessageQueue 中还有未处理的消息引用着 Handler，而 Handler 又引用着 Activity，Activity 就无法被 GC 回收。

### 7.2 解决方案

````java
// ✅ 方案一：静态内部类 + 弱引用
public class MyActivity extends AppCompatActivity {

    private static class MyHandler extends Handler {
        private final WeakReference<MyActivity> activityRef;

        MyHandler(MyActivity activity) {
            super(Looper.getMainLooper());
            activityRef = new WeakReference<>(activity);
        }

        @Override
        public void handleMessage(@NonNull Message msg) {
            MyActivity activity = activityRef.get();
            if (activity == null) return;
            activity.textView.setText("hello");
        }
    }

    private final MyHandler handler = new MyHandler(this);

    @Override
    protected void onDestroy() {
        super.onDestroy();
        handler.removeCallbacksAndMessages(null);  // 清除所有消息
    }
}

// ✅ 方案二：在 onDestroy 中移除所有消息（最简单）
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null);
}
````
---

## 八、HandlerThread

HandlerThread 是一个自带 Looper 的线程，方便在子线程中使用 Handler。

````java
// 创建一个带消息循环的后台线程
HandlerThread handlerThread = new HandlerThread("background-thread");
handlerThread.start();

// 在这个线程上创建 Handler
Handler backgroundHandler = new Handler(handlerThread.getLooper());

// 发送任务到后台线程执行
backgroundHandler.post(() -> {
    // 这里运行在 handlerThread 线程
    String data = loadFromDatabase();

    // 切回主线程更新 UI
    mainHandler.post(() -> {
        textView.setText(data);
    });
});

// 不用时退出
handlerThread.quitSafely();
````
**HandlerThread vs 普通 Thread：**
- 普通 Thread 执行完就结束了，不能复用
- HandlerThread 有 Looper，可以持续接收和处理消息，适合需要串行执行多个任务的场景

---

## 九、面试题库

### Q1：描述一下 Handler 消息机制的整体流程？

> Handler 消息机制由四个核心组件构成：Handler、Message、MessageQueue、Looper。整体流程是这样的：Handler 通过 sendMessage 或 post 将 Message 发送到 MessageQueue，MessageQueue 是一个按执行时间排序的单链表，enqueueMessage 时按 when 字段插入到合适的位置。Looper 在 loop 方法中死循环调用 MessageQueue.next() 取消息，next 方法在没有消息时通过 nativePollOnce 阻塞等待（底层 epoll 机制），有消息到来时被 nativeWake 唤醒。取到消息后调用 msg.target.dispatchMessage 交给对应的 Handler 处理。处理完后 Message 被回收到对象池中复用。

### Q2：为什么主线程的 Looper.loop() 死循环不会导致 ANR？

> 这个问题很多人理解错了。ANR 的本质不是"主线程在循环"，而是"某个事件在规定时间内没有被处理完"。Looper.loop() 的死循环在没有消息时会通过 nativePollOnce 进入休眠状态，底层用的是 Linux 的 epoll 机制，线程挂起不消耗 CPU。当有新消息到来（触摸事件、生命周期回调等）时通过 nativeWake 唤醒。Android 所有的事件驱动都是基于这个消息循环的，Activity 的生命周期回调、View 的绘制、触摸事件分发，全都是通过 Handler 消息来驱动的。ANR 发生在某条消息的处理时间过长，导致后续的输入事件或广播得不到及时处理。

### Q3：Handler 的 post 和 sendMessage 有什么区别？

> 本质上没有区别。post(Runnable) 内部会把 Runnable 包装成 Message 的 callback 字段，然后调用 sendMessageDelayed。区别在于处理时的优先级：dispatchMessage 中先检查 msg.callback（post 的 Runnable），再检查 Handler 构造时传入的 Callback，最后才是 handleMessage。所以 post 的 Runnable 优先级最高。实际开发中，简单的线程切换用 post 更方便，需要区分消息类型时用 sendMessage + what。

### Q4：MessageQueue 的 next() 方法是怎么阻塞的？

> next() 内部是一个 for 死循环。每次循环先调用 nativePollOnce(ptr, timeout) 阻塞等待，timeout 为 -1 表示无限等待，为 0 表示不等待，正数表示等待指定毫秒。唤醒后检查链表头部的消息，如果 when 时间已到就取出返回；如果还没到就计算差值作为下次 nativePollOnce 的 timeout；如果队列为空就传 -1 无限等待。底层用的是 Linux epoll 机制，通过 eventfd 实现线程间的唤醒通知，休眠时不占用 CPU 资源。

### Q5：什么是同步屏障？有什么用？

> 同步屏障是 MessageQueue 中的一种特殊机制。当设置同步屏障后，MessageQueue 的 next() 方法会跳过所有同步消息，只取出异步消息执行。最典型的应用是 View 的绘制流程——ViewRootImpl.scheduleTraversals() 会先设置同步屏障，然后通过 Choreographer 发送异步的绘制消息。这样即使消息队列中有大量普通消息排队，绘制消息也能优先执行，保证 UI 的流畅性。绘制完成后在 doTraversal 中移除同步屏障，普通消息恢复执行。这也是为什么我们 post 的 Runnable 不会影响 UI 绘制帧率的原因。

### Q6：Handler 为什么会导致内存泄漏？怎么解决？

> 匿名内部类或非静态内部类的 Handler 会隐式持有外部 Activity 的引用。如果 Handler 发送了延迟消息，消息在 MessageQueue 中排队时引用链是：主线程 Looper → MessageQueue → Message → Handler → Activity。Activity 关闭后因为这条引用链无法被 GC 回收。解决方案有三种：一是用静态内部类 + WeakReference 持有 Activity；二是在 onDestroy 中调用 handler.removeCallbacksAndMessages(null) 清除所有待处理消息；三是用 Lifecycle 感知的方案比如 lifecycleScope，Activity 销毁时自动取消。实际项目中我推荐第三种，最简洁也最不容易遗漏。

### Q7：ThreadLocal 的原理？在 Handler 机制中起什么作用？

> ThreadLocal 为每个线程维护一份独立的变量副本。原理是每个 Thread 对象内部有一个 ThreadLocalMap，ThreadLocal.set() 把值存到当前线程的 map 中，get() 从当前线程的 map 中取。在 Handler 机制中，Looper 通过 ThreadLocal 保证每个线程最多只有一个 Looper 实例。Looper.prepare() 时检查当前线程的 ThreadLocal 是否已有值，有就抛异常，没有就创建新 Looper 并存入。Looper.myLooper() 就是从 ThreadLocal 中获取当前线程的 Looper。需要注意 ThreadLocal 有内存泄漏风险，因为 key 是弱引用但 value 是强引用，用完要调用 remove()。

### Q8：IdleHandler 是什么？有什么应用场景？

> IdleHandler 是 MessageQueue 提供的回调接口，在消息队列空闲时（没有待处理消息或下一条消息还没到执行时间）被调用。queueIdle() 返回 false 表示执行一次后移除，返回 true 表示每次空闲都执行。实际应用场景：一是启动优化，把非关键的初始化任务放到 IdleHandler 中，等首屏渲染完成后再执行；二是 Activity 的 GC 回调，ActivityThread 中用 IdleHandler 在空闲时触发 GC；三是精确判断页面绘制完成，比 onWindowFocusChanged 更准确。但要注意如果消息队列一直不空闲，IdleHandler 可能长时间得不到执行。

### Q9：Message 的复用机制是怎么实现的？

> Message 内部维护了一个静态的单链表对象池，最大容量 50 个。Message.obtain() 从池头取一个复用的 Message 对象，如果池为空才 new 一个新的。Message 被处理完后调用 recycleUnchecked() 清空所有字段并放回池头。这是享元模式的典型应用，避免了高频创建 Message 对象带来的内存抖动和 GC 压力。所以开发中应该始终用 Message.obtain() 或 handler.obtainMessage() 获取 Message，不要直接 new。

### Q10：子线程中怎么使用 Handler？HandlerThread 有什么用？

> 子线程默认没有 Looper，直接创建 Handler 会抛异常。需要先调用 Looper.prepare() 创建 Looper，再创建 Handler，最后调用 Looper.loop() 开始消息循环。但这样比较繁琐，Android 提供了 HandlerThread 封装了这些步骤。HandlerThread 是一个自带 Looper 的线程，start 后可以直接用它的 looper 创建 Handler。它适合需要在后台串行处理多个任务的场景，比如数据库操作队列、日志写入队列。和普通线程的区别是它有消息循环可以持续接收任务，不会执行完一个就退出。不用时要调用 quitSafely() 退出循环释放资源。

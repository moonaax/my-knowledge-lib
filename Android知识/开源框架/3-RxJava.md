---
aliases: [RxJava, ReactiveX, 响应式编程]
tags: [开源框架, 响应式, Android, Java]
created: 2026-04-16
---

# RxJava

> RxJava 是 ReactiveX 在 JVM 上的实现，基于 [[4-响应式架构]] 思想，通过可观察序列组合异步与事件驱动程序。它是 Android 开发中最流行的异步框架，也是 [[1-OkHttp与Retrofit]] 网络层的核心搭档。

---

## 一、核心概念

### 1.1 观察者模式

RxJava 的根基是**观察者模式（Observer Pattern）**的扩展。传统观察者模式中，Subject 维护一组 Observer，状态变化时通知所有观察者。RxJava 在此基础上增加了三个关键能力：

- **完成信号**：`onComplete()` 表示数据流结束
- **错误信号**：`onError(Throwable)` 表示数据流异常终止
- **组合与变换**：通过操作符对数据流进行链式变换

事件契约遵循如下协议：

```
onSubscribe -> (onNext)* -> (onComplete | onError)?
```

即：订阅后可收到零到多个 `onNext`，最终以 `onComplete` 或 `onError` 结束（二者互斥且只触发一次）。

```java
Observable.create(emitter -> {
    emitter.onNext("A");
    emitter.onNext("B");
    emitter.onComplete(); // 流结束
}).subscribe(
    item -> System.out.println("收到: " + item),
    error -> System.err.println("错误: " + error),
    () -> System.out.println("完成")
);
```

### 1.2 五种被观察者类型

| 类型 | 发射元素数量 | 是否支持背压 | 典型场景 |
|------|------------|------------|---------|
| **Observable** | 0..N | ❌ | UI 事件、少量数据流 |
| **Flowable** | 0..N | ✅ | 大量数据、文件读取、数据库查询 |
| **Single** | 恰好 1 个 | ❌ | 网络请求、数据库单条查询 |
| **Maybe** | 0 或 1 个 | ❌ | 查询可能为空的缓存 |
| **Completable** | 0 个（仅完成/错误） | ❌ | 写入操作、删除操作 |

```java
// Single：网络请求返回单个结果
Single<User> userSingle = apiService.getUser(userId);

// Maybe：缓存可能命中也可能为空
Maybe<User> cachedUser = Maybe.fromCallable(() -> cache.get(userId));

// Completable：只关心成功或失败
Completable deleteTask = Completable.fromAction(() -> db.delete(userId));

// Flowable：处理大量数据，支持背压
Flowable.range(1, 10000)
    .onBackpressureBuffer()
    .observeOn(Schedulers.computation())
    .subscribe(i -> process(i));
```

### 1.3 调度器 Scheduler

Scheduler 是 RxJava 的线程调度抽象，决定代码在哪个线程执行：

| Scheduler | 底层实现 | 适用场景 |
|-----------|---------|---------|
| `Schedulers.io()` | 缓存线程池（可无限增长） | 网络请求、文件 IO、数据库操作 |
| `Schedulers.computation()` | 固定线程池（CPU 核心数） | CPU 密集型计算 |
| `Schedulers.newThread()` | 每次创建新线程 | 不推荐频繁使用 |
| `Schedulers.single()` | 单线程 | 顺序执行任务 |
| `Schedulers.trampoline()` | 当前线程排队执行 | 测试、递归调度 |
| `AndroidSchedulers.mainThread()` | Android 主线程 Handler | UI 更新（来自 RxAndroid） |

核心用法——`subscribeOn` 指定上游线程，`observeOn` 切换下游线程：

```java
Observable.fromCallable(() -> fetchDataFromNetwork())
    .subscribeOn(Schedulers.io())         // 上游在 IO 线程执行
    .observeOn(AndroidSchedulers.mainThread()) // 下游切回主线程
    .subscribe(data -> updateUI(data));
```

### 1.4 背压 Backpressure

当生产者速度远大于消费者速度时，就会产生**背压**问题。`Observable` 不处理背压，数据堆积会导致 OOM；`Flowable` 提供了背压策略：

| 策略 | 说明 |
|------|------|
| `BackpressureStrategy.BUFFER` | 无限缓冲（仍可能 OOM） |
| `BackpressureStrategy.DROP` | 丢弃消费不过来的数据 |
| `BackpressureStrategy.LATEST` | 只保留最新的一条 |
| `BackpressureStrategy.ERROR` | 抛出 MissingBackpressureException |
| `BackpressureStrategy.MISSING` | 不做任何处理，交给下游操作符 |

```java
Flowable.create(emitter -> {
    for (int i = 0; i < 1000000; i++) {
        if (emitter.isCancelled()) return;
        emitter.onNext(i);
    }
    emitter.onComplete();
}, BackpressureStrategy.DROP)
    .observeOn(Schedulers.computation(), false, 128)
    .subscribe(i -> process(i));
```


---

## 二、原理与源码

### 2.1 订阅流程源码（subscribeActual）

RxJava 的每个被观察者类型都是抽象类，核心方法是 `subscribeActual(Observer)`。以 `Observable.create()` 为例：

```java
// 用户调用 subscribe() 时，实际执行链路：
// Observable.subscribe(observer)
//   -> Observable.subscribeActual(observer)  // 抽象方法，由子类实现

// ObservableCreate 的实现：
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        // 1. 创建发射器，包装 observer
        CreateEmitter<T> parent = new CreateEmitter<>(observer);
        // 2. 回调 onSubscribe，传递 Disposable
        observer.onSubscribe(parent);
        // 3. 触发用户定义的数据发射逻辑
        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            parent.onError(ex);
        }
    }
}
```

关键点：`subscribe()` 是模板方法，内部做插件 hook、空检查等通用逻辑后委托给 `subscribeActual()`。每个操作符都会生成新的 `Observable` 子类，各自实现该方法。

### 2.2 操作符链式调用原理

每个操作符调用都会创建一个新的 `Observable` 包装上游，形成**装饰器链**：

```java
Observable.just("hello")       // ObservableJust
    .map(s -> s.length())      // ObservableMap(upstream=ObservableJust)
    .filter(len -> len > 3)    // ObservableFilter(upstream=ObservableMap)
    .subscribe(observer);      // 触发从下游到上游的逆向订阅
```

这形成了 RxJava 的**双向链**结构：订阅时**自下而上**逆向传播（Filter → Map → Just），数据发射时**自上而下**正向传递（Just → Map → Filter → Observer）。

### 2.3 线程切换原理（subscribeOn / observeOn）

**subscribeOn** 影响的是上游的 `subscribeActual` 在哪个线程执行：

```java
// ObservableSubscribeOn 核心逻辑：
@Override
public void subscribeActual(Observer<? super T> observer) {
    SubscribeOnObserver<T> parent = new SubscribeOnObserver<>(observer);
    observer.onSubscribe(parent);
    // 将上游的订阅动作调度到指定 Scheduler
    parent.setDisposable(
        scheduler.scheduleDirect(() -> source.subscribe(parent))
    );
}
```

因此 `subscribeOn` 只对它上方的操作符生效，且**多次调用只有第一次有效**（因为最上游的 subscribeOn 最终决定了源头的执行线程）。

**observeOn** 影响的是下游 `onNext/onError/onComplete` 在哪个线程执行：

```java
// ObservableObserveOn 核心逻辑：
// 内部维护一个队列，上游数据先入队
// 然后通过 Scheduler.Worker 调度，在目标线程逐个取出并传递给下游
@Override
public void onNext(T t) {
    if (!checkTerminated(done, queue.isEmpty(), a)) {
        queue.offer(t);  // 数据入队
    }
    schedule();           // 调度到目标线程执行 drainNormal()
}
```

`observeOn` 每次调用都会生效，可以多次切换下游线程。

### 2.4 Disposable 取消机制

`Disposable` 是 RxJava 的取消/释放接口：

```java
public interface Disposable {
    void dispose();      // 取消订阅，释放资源
    boolean isDisposed(); // 是否已取消
}
```

在 `CreateEmitter` 中，`dispose()` 会设置原子标志位，后续的 `onNext` 会检查该标志并跳过发射：

```java
// CreateEmitter 中的 onNext：
@Override
public void onNext(T t) {
    if (!isDisposed()) {  // 检查是否已取消
        observer.onNext(t);
    }
}
```

实际开发中用 `CompositeDisposable` 统一管理：

```java
CompositeDisposable compositeDisposable = new CompositeDisposable();

Disposable d1 = observable1.subscribe(data -> { /* ... */ });
Disposable d2 = observable2.subscribe(data -> { /* ... */ });

compositeDisposable.add(d1);
compositeDisposable.add(d2);

// Activity/Fragment onDestroy 时统一取消
compositeDisposable.clear(); // clear 可复用，dispose 不可
```


---

## 三、常用操作符

### 3.1 变换操作符

#### map — 一对一变换

将每个元素通过函数映射为另一个元素：

```java
Observable.just(1, 2, 3)
    .map(i -> "Item-" + i)
    .subscribe(s -> System.out.println(s));
// 输出: Item-1, Item-2, Item-3
```

#### flatMap — 一对多变换（不保序）

将每个元素映射为一个新的 `Observable`，然后将所有结果**合并**（交错发射，不保证顺序）：

```java
Observable.just(1, 2, 3)
    .flatMap(i -> Observable.just(i * 10, i * 100)
        .subscribeOn(Schedulers.io()))
    .subscribe(v -> System.out.println(v));
// 输出顺序不确定，如: 10, 200, 100, 30, 20, 300
```

#### concatMap — 一对多变换（保序）

与 `flatMap` 类似，但严格按照上游顺序**串行**订阅内部 Observable，保证输出顺序：

```java
Observable.just(1, 2, 3)
    .concatMap(i -> Observable.just(i * 10, i * 100)
        .delay(100, TimeUnit.MILLISECONDS))
    .subscribe(v -> System.out.println(v));
// 输出: 10, 100, 20, 200, 30, 300（严格有序）
```

#### switchMap — 只保留最新

每当上游发射新元素时，**取消**前一个内部 Observable 的订阅，只保留最新的。适合搜索联想场景：

```java
// 搜索框输入联想：用户快速输入时只保留最后一次请求
searchObservable
    .debounce(300, TimeUnit.MILLISECONDS)
    .switchMap(keyword -> apiService.search(keyword)
        .subscribeOn(Schedulers.io()))
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(results -> showResults(results));
```

### 3.2 组合操作符

#### zip — 配对组合

将多个 Observable 的元素按顺序一一配对，任一源完成则整体完成：

```java
Observable<String> names = Observable.just("张三", "李四", "王五");
Observable<Integer> ages = Observable.just(25, 30, 28);

Observable.zip(names, ages, (name, age) -> name + ":" + age)
    .subscribe(s -> System.out.println(s));
// 输出: 张三:25, 李四:30, 王五:28
```

#### combineLatest — 最新组合

任一源发射新元素时，取各源的**最新值**进行组合。适合表单校验：

```java
Observable<String> username = usernameEditText.textChanges();
Observable<String> password = passwordEditText.textChanges();

Observable.combineLatest(username, password,
    (u, p) -> u.length() >= 4 && p.length() >= 6)
    .subscribe(valid -> loginButton.setEnabled(valid));
```

### 3.3 过滤操作符

#### debounce — 防抖

元素发射后等待指定时间，期间无新元素则发射，否则重新计时：

```java
RxTextView.textChanges(searchEditText)
    .debounce(400, TimeUnit.MILLISECONDS)
    .filter(text -> text.length() > 0)
    .distinctUntilChanged()
    .switchMap(keyword -> search(keyword))
    .subscribe(results -> display(results));
```

#### throttleFirst / throttleLast — 节流

- `throttleFirst`：时间窗口内只取**第一个**（防重复点击）
- `throttleLast`（即 `sample`）：取时间窗口内**最后一个**

```java
// 防止按钮重复点击
RxView.clicks(submitButton)
    .throttleFirst(1, TimeUnit.SECONDS)
    .subscribe(click -> doSubmit());
```

### 3.4 错误处理操作符

#### retry — 重试

发生错误时重新订阅上游，可指定次数或条件：

```java
apiService.getData()
    .subscribeOn(Schedulers.io())
    .retry(3) // 最多重试 3 次
    .subscribe(data -> handleData(data), error -> showFinalError(error));
```

更高级的指数退避重试见面试题 Q6。

### 3.5 操作符选择速查

| 需求 | 推荐操作符 |
|------|-----------|
| 同步一对一变换 | `map` |
| 异步一对多（不关心顺序） | `flatMap` |
| 异步一对多（保序） | `concatMap` |
| 只关心最新结果 | `switchMap` |
| 多源配对 | `zip` |
| 多源取最新 | `combineLatest` |
| 搜索防抖 | `debounce` |
| 防重复点击 | `throttleFirst` |
| 失败重试 | `retry` / `retryWhen` |


---

## 四、面试题

### Q1：Observable 和 Flowable 的区别是什么？什么时候用 Flowable？

**A：** 核心区别在于**背压支持**。`Observable` 不处理背压，上游过快会导致 OOM。`Flowable` 实现了 Reactive Streams 规范，内置背压策略，下游通过 `request(n)` 主动拉取数据。

- Flowable 场景：上游数据量大且不可控（大文件、数据库游标）、上下游速度差异明显
- Observable 场景：数据量小（< 1000）、UI 事件流、不需要背压的同步场景

### Q2：subscribeOn 和 observeOn 的区别？多次调用分别是什么效果？

**A：**
- `subscribeOn` 决定**上游数据源**在哪个线程执行（影响 `subscribeActual` 的执行线程）。多次调用只有**离源头最近的（即第一个）生效**，因为订阅是从下游向上游传播的，最终源头的订阅线程由最上游的 `subscribeOn` 决定。
- `observeOn` 决定**下游操作符**在哪个线程执行（通过队列 + Worker 调度实现线程切换）。每次调用都会生效，可以在链路中多次切换线程。

```java
Observable.create(emitter -> {
        Log.d("TAG", "发射线程: " + Thread.currentThread().getName());
        emitter.onNext(1);
        emitter.onComplete();
    })
    .subscribeOn(Schedulers.io())          // ✅ 生效：源头在 IO 线程
    .subscribeOn(Schedulers.computation()) // ❌ 无效：被上面的覆盖
    .observeOn(Schedulers.computation())   // ✅ 切到计算线程
    .map(i -> i * 2)                       // 在 computation 线程执行
    .observeOn(AndroidSchedulers.mainThread()) // ✅ 切到主线程
    .subscribe(v -> updateUI(v));          // 在主线程执行
```

### Q3：RxJava 如何避免内存泄漏？

**A：** RxJava 订阅会持有 Observer 的引用，如果 Observer 是 Activity/Fragment 的匿名内部类，就会间接持有外部类引用，导致内存泄漏。解决方案：

1. **CompositeDisposable**：手动在 `onDestroy` 中调用 `clear()`
2. **AutoDispose**（推荐）：绑定生命周期自动取消
3. **RxLifecycle**：通过 `compose` 绑定生命周期

详见下方「实战与踩坑」章节。

### Q4：flatMap、concatMap、switchMap 三者的区别？

**A：**

| 操作符 | 并发订阅 | 输出顺序 | 取消前一个 |
|--------|---------|---------|-----------|
| `flatMap` | ✅ 并发 | ❌ 交错（不保序） | ❌ |
| `concatMap` | ❌ 串行 | ✅ 严格有序 | ❌ |
| `switchMap` | ✅ 并发 | 只保留最新 | ✅ 取消前一个 |

- `flatMap`：适合不关心顺序的并发请求（如批量图片加载）
- `concatMap`：适合需要保序的场景（如分页加载）
- `switchMap`：适合只关心最新结果的场景（如搜索联想，用户快速输入时取消旧请求）

### Q5：RxJava 的 onError 和 onComplete 可以同时触发吗？

**A：** 不可以。`onComplete` 和 `onError` 是**终止事件**，二者互斥且只能触发一次。触发后续的事件都会被忽略。

若在已终止/已 dispose 的流上触发 `onError`，RxJava 会将异常交给 `RxJavaPlugins.onError`，可能导致崩溃。建议设置全局兜底：

```java
RxJavaPlugins.setErrorHandler(e -> {
    if (e instanceof UndeliverableException) {
        Log.w("RxJava", "Undeliverable exception", e.getCause());
    }
});
```

### Q6：如何实现 RxJava 的指数退避重试？

**A：** 使用 `retryWhen` 操作符，结合 `zipWith` 和 `timer` 实现：

```java
apiService.request()
    .subscribeOn(Schedulers.io())
    .retryWhen(errors -> errors
        .zipWith(Observable.range(1, 3), (error, retryCount) -> {
            if (retryCount >= 3) throw new RuntimeException(error);
            return retryCount;
        })
        .flatMap(retryCount -> {
            long delay = (long) Math.pow(2, retryCount); // 2s, 4s, 8s
            return Observable.timer(delay, TimeUnit.SECONDS);
        }))
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(
        data -> onSuccess(data),
        error -> onFinalError(error)
    );
```

原理：`retryWhen` 接收错误流，发射元素时触发重试，终止时停止。通过 `zipWith(range)` 限制次数，`timer` 实现延迟。

---

## 五、实战与踩坑

### 5.1 内存泄漏

RxJava 在 Android 中最常见的问题。订阅持有 Observer 引用 → 匿名内部类持有 Activity 引用 → Activity 无法 GC。

#### 方案一：CompositeDisposable（手动管理）

```java
public class UserActivity extends AppCompatActivity {
    private final CompositeDisposable disposables = new CompositeDisposable();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        disposables.add(
            apiService.getUser()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(this::showUser, this::showError)
        );
    }

    @Override
    protected void onDestroy() {
        disposables.clear();
        super.onDestroy();
    }
}
```

#### 方案二：AutoDispose（推荐，自动绑定生命周期）

AutoDispose 由 Uber 开源，与 [[Android Jetpack]] 的 Lifecycle 无缝集成：

```java
Observable.interval(1, TimeUnit.SECONDS)
    .as(AutoDispose.autoDisposable(
        AndroidLifecycleScopeProvider.from(this))) // this = LifecycleOwner
    .subscribe(tick -> updateTimer(tick));
// Activity onDestroy 时自动取消，无需手动管理
```

#### 方案三：RxLifecycle（较早方案）

```java
apiService.getUser()
    .compose(bindToLifecycle()) // Activity 需继承 RxAppCompatActivity
    .subscribe(user -> showUser(user));
```

> ⚠️ RxLifecycle 已不再积极维护，新项目推荐使用 AutoDispose。

### 5.2 错误处理链断裂

RxJava 中一旦 `onError` 触发，整个链路就会终止。如果没有提供 `onError` 回调，会抛出 `OnErrorNotImplementedException` 导致崩溃。

#### 坑 1：忘记处理错误

```java
// ❌ 危险：网络异常会直接崩溃
observable.subscribe(data -> showData(data));

// ✅ 始终提供 onError
observable.subscribe(
    data -> showData(data),
    error -> showError(error)
);
```

#### 坑 2：onError 后流终止，后续事件丢失

```java
// ❌ 第一次错误后整个流终止，后续点击无响应
RxView.clicks(button)
    .flatMap(click -> apiService.submit()
        .subscribeOn(Schedulers.io()))
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(result -> showSuccess(), error -> showError(error));

// ✅ 将错误控制在内部 Observable，不影响外部流
RxView.clicks(button)
    .throttleFirst(1, TimeUnit.SECONDS)
    .flatMap(click -> apiService.submit()
        .subscribeOn(Schedulers.io())
        .onErrorReturn(e -> Result.error(e))) // 错误转为正常数据
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(result -> {
        if (result.isSuccess()) showSuccess();
        else showError(result.getError());
    });
```

#### 坑 3：全局未处理异常

当 `onError` 在已 dispose 的订阅上触发时，异常无处投递。默认行为在 Android 上会导致崩溃，需在 `Application.onCreate` 中设置全局兜底：

```java
RxJavaPlugins.setErrorHandler(throwable -> {
    Throwable e = (throwable instanceof UndeliverableException)
        ? throwable.getCause() : throwable;
    if (e instanceof IOException || e instanceof InterruptedException) {
        Log.w("RxGlobal", "Ignored", e);
        return;
    }
    Thread.currentThread().getUncaughtExceptionHandler()
        .uncaughtException(Thread.currentThread(), e);
});
```

### 5.3 subscribeOn 多次调用的坑

很多开发者误以为多次 `subscribeOn` 可以多次切换上游线程，实际上**只有第一次（最上游的）生效**：

```java
Observable.create(emitter -> {
        // 实际在 IO 线程执行，而非 computation
        Log.d("Thread", Thread.currentThread().getName());
        emitter.onNext("data");
        emitter.onComplete();
    })
    .subscribeOn(Schedulers.io())          // ✅ 第一个，生效
    .map(s -> {
        // 仍在 IO 线程（subscribeOn 影响的是整个上游链）
        Log.d("Thread", "map: " + Thread.currentThread().getName());
        return s.toUpperCase();
    })
    .subscribeOn(Schedulers.computation()) // ❌ 第二个，无效
    .subscribe(s -> Log.d("Thread", "subscribe: " + s));
```

**原理分析**：订阅从下游向上游传播，第一个 `subscribeOn(io)` 最终决定了源头线程。详细原理见面试题 Q2。

**正确做法**：需要在链路中间切换线程时使用 `observeOn`：

```java
Observable.create(emitter -> {
        emitter.onNext("data");
        emitter.onComplete();
    })
    .subscribeOn(Schedulers.io())              // 源头在 IO 线程
    .observeOn(Schedulers.computation())       // 切到 computation
    .map(s -> heavyComputation(s))             // 在 computation 执行
    .observeOn(AndroidSchedulers.mainThread()) // 切到主线程
    .subscribe(result -> updateUI(result));    // 在主线程执行
```

### 5.4 其他常见陷阱

#### Disposable 未正确管理导致空指针

```java
// ❌ Activity 已销毁，回调中访问 View 导致 NPE
apiService.getUser()
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(user -> {
        textView.setText(user.getName()); // Activity 已销毁，textView 为 null
    });
```

#### interval 未取消导致持续执行

`Observable.interval` 是无限流，必须保存 `Disposable` 并在适当时机（如 `onDestroy`）调用 `dispose()`，否则会一直执行导致泄漏。

---

## 参考与关联

- [[4-响应式架构]] — 响应式编程理论基础
- [[1-OkHttp与Retrofit]] — RxJava 与 Retrofit 配合使用
- [[Android Jetpack]] — Lifecycle 与 AutoDispose 集成
- [[Kotlin协程]] — RxJava 的现代替代方案
- 官方文档：[ReactiveX](http://reactivex.io/)
- GitHub：[RxJava](https://github.com/ReactiveX/RxJava)

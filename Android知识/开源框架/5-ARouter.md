# ARouter 路由框架

> ARouter 是阿里巴巴开源的 Android 路由框架，用于解决组件化开发中模块间通信、页面跳转、服务发现等核心问题。

相关主题：[[2-组件间通信]] [[5-注解处理器APT与KSP]] [[Android组件化架构]]

---

## 一、核心概念

### 1.1 为什么需要路由框架

在组件化架构下，各业务模块相互隔离、互不依赖，存在以下痛点：

- **类引用不可达**：模块 A 无法直接引用模块 B 的 Activity 类，显式 Intent 失效
- **隐式 Intent 维护成本高**：需声明大量 `intent-filter`，缺乏编译期校验
- **服务调用困难**：模块间缺少统一的服务发现机制
- **缺乏统一拦截**：登录校验、降级策略等横切逻辑散落各处

路由框架的本质是 **URL → 目标组件** 的映射表，通过字符串路径解耦模块间的直接依赖。参见 [[2-组件间通信]]。

### 1.2 核心功能

| 功能 | 说明 |
|------|------|
| **页面跳转** | 通过路径字符串跳转 Activity/Fragment，支持参数、动画、ForResult |
| **服务发现** | 基于 `IProvider` 接口实现跨模块服务调用 |
| **拦截器** | 全局拦截器链，可实现登录校验、埋点等横切逻辑 |
| **降级策略** | 路由未命中时触发降级回调，可跳转 H5 或错误页 |

### 1.3 注解体系

#### @Route — 标记路由目标

````java
@Route(path = "/order/detail", group = "order", name = "订单详情页")
public class OrderDetailActivity extends AppCompatActivity { }

@Route(path = "/account/service")
public class AccountServiceImpl implements IAccountService { }
````
- `path`：必填，以 `/` 开头且至少两级（`/group/name`）
- `group`：可选，默认取 path 第一段
- `name`：可选，用于生成路由文档

#### @Interceptor — 标记拦截器

````java
@Interceptor(priority = 5, name = "登录拦截器")
public class LoginInterceptor implements IInterceptor {
    @Override
    public void init(Context context) { }

    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
        if (needLogin(postcard) && !UserManager.isLogin()) {
            callback.onInterrupt(new RuntimeException("need login"));
        } else {
            callback.onContinue(postcard);
        }
    }
}
````
- `priority`：数值越小越先执行

#### @Autowired — 自动注入参数

````java
@Route(path = "/order/detail")
public class OrderDetailActivity extends AppCompatActivity {
    @Autowired
    String orderId;

    @Autowired(name = "amount", required = true)
    double orderAmount;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ARouter.getInstance().inject(this); // 触发注入
    }
}
````
编译时生成 `XXX$$ARouter$$Autowired` 辅助类，运行时通过 `inject()` 调用，无反射开销。

---

## 二、原理与源码

### 2.1 APT 编译时生成路由表

ARouter 使用 [[5-注解处理器APT与KSP]] 在编译期扫描 `@Route` 注解，为每个模块生成路由注册类：

````java
// 自动生成 — 分组路由表
public class ARouter$$Group$$order implements IRouteGroup {
    @Override
    public void loadInto(Map<String, RouteMeta> atlas) {
        atlas.put("/order/detail",
            RouteMeta.build(RouteType.ACTIVITY, OrderDetailActivity.class,
                "/order/detail", "order", null, -1, -2147483648));
    }
}

// 自动生成 — 分组索引
public class ARouter$$Root$$order implements IRouteRoot {
    @Override
    public void loadInto(Map<String, Class<? extends IRouteGroup>> routes) {
        routes.put("order", ARouter$$Group$$order.class);
    }
}
````
初始化流程：

````
ARouter.init(application)
  └─ LogisticsCenter.init()
       ├─ 扫描 dex 找到 com.alibaba.android.arouter.routes 包下所有类
       │   （或通过 arouter-register 插件编译期注入，跳过运行时扫描）
       ├─ IRouteRoot → Warehouse.groupsIndex
       ├─ IInterceptorGroup → Warehouse.interceptorsIndex
       └─ IProviderGroup → Warehouse.providersIndex
````
### 2.2 分组懒加载机制

ARouter 不会在初始化时加载所有路由，而是 **按组懒加载**：

````java
// LogisticsCenter.completion() 简化源码
public synchronized static void completion(Postcard postcard) {
    RouteMeta routeMeta = Warehouse.routes.get(postcard.getPath());
    if (routeMeta == null) {
        // 路由表中没有 → 按组加载
        Class<? extends IRouteGroup> groupMeta =
            Warehouse.groupsIndex.get(postcard.getGroup());
        if (groupMeta == null) {
            throw new NoRouteFoundException("no route: " + postcard.getPath());
        }
        // 实例化分组类，加载该组所有路由
        groupMeta.getConstructor().newInstance().loadInto(Warehouse.routes);
        Warehouse.groupsIndex.remove(postcard.getGroup()); // 避免重复加载
        completion(postcard); // 递归再次查找
    } else {
        postcard.setDestination(routeMeta.getDestination());
        postcard.setType(routeMeta.getType());
        // ...填充 Postcard
    }
}
````
100 个路由分 10 组，初始化只加载 10 个索引，首次访问某组时才加载该组全部路由。

### 2.3 拦截器责任链源码

````java
// InterceptorServiceImpl.java 简化
@Route(path = "/arouter/service/interceptor")
public class InterceptorServiceImpl implements InterceptorService {
    @Override
    public void doInterceptions(Postcard postcard, InterceptorCallback callback) {
        if (Warehouse.interceptors.size() > 0) {
            LogisticsCenter.executor.execute(() -> {
                CancelableCountDownLatch counter =
                    new CancelableCountDownLatch(Warehouse.interceptors.size());
                _execute(0, counter, postcard);
                counter.await(postcard.getTimeout(), TimeUnit.SECONDS);
                if (counter.getCount() > 0) {
                    callback.onInterrupt(new TimeoutException("timeout"));
                } else if (postcard.getTag() instanceof Throwable) {
                    callback.onInterrupt((Throwable) postcard.getTag());
                } else {
                    callback.onContinue(postcard);
                }
            });
        } else {
            callback.onContinue(postcard);
        }
    }

    private static void _execute(int index,
            CancelableCountDownLatch counter, Postcard postcard) {
        if (index < Warehouse.interceptors.size()) {
            Warehouse.interceptors.get(index).process(postcard, new InterceptorCallback() {
                @Override
                public void onContinue(Postcard p) {
                    counter.countDown();
                    _execute(index + 1, counter, postcard);
                }
                @Override
                public void onInterrupt(Throwable e) {
                    postcard.setTag(e);
                    counter.cancel();
                }
            });
        }
    }
}
````
关键点：拦截器在子线程执行，`CancelableCountDownLatch` 实现超时控制（默认 300s），`onInterrupt()` 可中断整条链。

### 2.4 IProvider 服务发现源码

````java
// LogisticsCenter.completion() 中对 PROVIDER 类型的处理
case PROVIDER:
    Class<? extends IProvider> providerClass =
        (Class<? extends IProvider>) routeMeta.getDestination();
    IProvider instance = Warehouse.providers.get(providerClass);
    if (instance == null) {
        IProvider provider = providerClass.getConstructor().newInstance();
        provider.init(mContext);
        Warehouse.providers.put(providerClass, provider); // 单例缓存
        instance = provider;
    }
    postcard.setProvider(instance);
    postcard.greenChannel(); // Provider 默认跳过拦截器
    break;
````
`IProvider` 实例默认单例缓存，且自动走绿色通道（跳过拦截器）。

---

## 三、实战

### 3.1 基础配置

````groovy
// 各模块 build.gradle
dependencies {
    implementation 'com.alibaba:arouter-api:1.5.2'
    annotationProcessor 'com.alibaba:arouter-compiler:1.5.2'
}

android {
    defaultConfig {
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [AROUTER_MODULE_NAME: project.getName()]
            }
        }
    }
}
````
````java
// Application 初始化
public class App extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        if (BuildConfig.DEBUG) {
            ARouter.openLog();
            ARouter.openDebug();
        }
        ARouter.init(this);
    }
}
````
### 3.2 页面跳转与参数传递

````java
// 基础跳转
ARouter.getInstance().build("/order/detail").navigation();

// 携带参数
ARouter.getInstance()
    .build("/order/detail")
    .withString("orderId", "20260416001")
    .withDouble("amount", 99.9)
    .withParcelable("address", addressObj)
    .navigation();

// startActivityForResult
ARouter.getInstance().build("/order/edit").navigation(this, REQUEST_CODE);

// 导航回调
ARouter.getInstance().build("/order/detail").navigation(this, new NavigationCallback() {
    @Override public void onFound(Postcard postcard) { }
    @Override public void onLost(Postcard postcard) { /* 降级处理 */ }
    @Override public void onArrival(Postcard postcard) { }
    @Override public void onInterrupt(Postcard postcard) { }
});
````
### 3.3 获取 Fragment

````java
Fragment fragment = (Fragment) ARouter.getInstance()
    .build("/order/list_fragment")
    .withString("type", "pending")
    .navigation();
getSupportFragmentManager().beginTransaction()
    .replace(R.id.container, fragment).commit();
````
### 3.4 跨模块服务调用

````java
// Step 1: 公共模块定义接口
public interface IAccountService extends IProvider {
    boolean isLogin();
    String getUserId();
}

// Step 2: 业务模块实现
@Route(path = "/account/service")
public class AccountServiceImpl implements IAccountService {
    @Override public void init(Context context) { }
    @Override public boolean isLogin() { return AccountManager.getInstance().isLogin(); }
    @Override public String getUserId() { return AccountManager.getInstance().getUserId(); }
}

// Step 3: 其他模块调用
IAccountService service = ARouter.getInstance().navigation(IAccountService.class);
if (service.isLogin()) {
    loadOrders(service.getUserId());
}

// 也可通过字段注入
@Autowired
IAccountService accountService;
````
### 3.5 拦截器实战

````java
@Interceptor(priority = 5, name = "登录拦截器")
public class LoginInterceptor implements IInterceptor {
    @Override
    public void init(Context context) { }

    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
        if ((postcard.getExtra() & RouteExtra.LOGIN_REQUIRED) != 0) {
            if (ARouter.getInstance().navigation(IAccountService.class).isLogin()) {
                callback.onContinue(postcard);
            } else {
                ARouter.getInstance().build("/account/login")
                    .withString("redirect", postcard.getPath()).navigation();
                callback.onInterrupt(new RuntimeException("need login"));
            }
        } else {
            callback.onContinue(postcard);
        }
    }
}

// 路由声明时标记需要登录
@Route(path = "/order/create", extras = RouteExtra.LOGIN_REQUIRED)
public class CreateOrderActivity extends AppCompatActivity { }
````
### 3.6 全局降级

````java
@Route(path = "/service/degrade")
public class DegradeServiceImpl implements DegradeService {
    @Override
    public void onLost(Context context, Postcard postcard) {
        ARouter.getInstance().build("/webview/common")
            .withString("url", "https://m.example.com" + postcard.getPath())
            .navigation();
    }
    @Override public void init(Context context) { }
}
````
---

## 四、面试题

### Q1：ARouter 的路由表何时生成？运行时如何加载？

**A**：路由表在 **编译期** 由 APT 生成。每个模块生成 `ARouter$$Root$$模块名`（分组索引）和 `ARouter$$Group$$分组名`（具体路由）。

运行时分两步：
1. `ARouter.init()` 扫描 dex 找到所有生成类，将分组索引加载到 `Warehouse.groupsIndex`
2. 首次导航到某路径时，从 `groupsIndex` 找到对应 `IRouteGroup`，实例化并加载该组路由到 `Warehouse.routes`，然后移除该分组索引

使用 `arouter-register` 插件可通过 ASM 字节码注入替代运行时 dex 扫描，提升初始化速度。

### Q2：拦截器在哪个线程执行？如何超时控制？

**A**：拦截器在 **子线程** 执行。通过 `CancelableCountDownLatch` 实现超时：初始 count 为拦截器数量，`onContinue()` 时 countDown，`onInterrupt()` 时 cancel，主流程 `await(300s)` 等待。超时后回调 `onInterrupt()`。拦截器中弹 Dialog 需手动切主线程。

### Q3：IProvider 是单例还是多例？

**A**：**单例**。首次获取时反射创建并调用 `init()`，缓存在 `Warehouse.providers`（key 为 Class），后续直接返回缓存。生命周期与进程一致，无显式 destroy 回调。

### Q4：@Autowired 的实现原理？

**A**：编译期 APT 为包含 `@Autowired` 的类生成 `XXX$$ARouter$$Autowired` 辅助类（实现 `ISyringe` 接口），其 `inject()` 方法通过 `getIntent().getExtras()` 取值赋给目标字段。运行时 `ARouter.getInstance().inject(this)` 找到辅助类并调用，无反射开销。支持 `required` 校验和自定义对象（需配合 `SerializationService`）。参见 [[5-注解处理器APT与KSP]]。

### Q5：greenChannel() 是什么？

**A**：绿色通道，调用后该次导航跳过所有拦截器。`IProvider` 默认走绿色通道。适用于登录页、WebView 等不需要拦截的页面。原理是 `Postcard` 内部标志位，拦截链执行前检查。

### Q6：多模块项目中 init() 只需调用一次吗？

**A**：是的。每个模块编译时独立生成路由类，包名统一为 `com.alibaba.android.arouter.routes`，打包时合并到同一 APK。`init()` 扫描该包名下所有类自动汇总。关键：每个模块必须配置唯一的 `AROUTER_MODULE_NAME`，否则类名冲突导致编译失败。

---

## 五、实战与踩坑

### 5.1 Gradle 8.x / AGP 8.x 兼容

`arouter-register` 插件基于 Transform API，**AGP 8.0 已移除 Transform API**。

| 方案 | 做法 | 缺点 |
|------|------|------|
| 移除 register 插件 | 回退到运行时扫描 dex | Debug 初始化耗时增加 200-500ms |
| 社区 fork 版本 | 使用适配 AGP 8.x 的 register 插件 | 非官方维护 |
| 迁移框架 | 迁移到 TheRouter | 有一定迁移成本 |

### 5.2 ARouter vs TheRouter vs WMRouter

| 维度 | ARouter | TheRouter | WMRouter |
|------|---------|-----------|----------|
| 维护方 | 阿里巴巴 | 货拉拉 | 美团 |
| 维护状态 | 基本停更 | 活跃维护 | 低频维护 |
| AGP 8.x | ❌ 需适配 | ✅ 原生支持 | ❌ 需适配 |
| KSP 支持 | ❌ 仅 APT | ✅ 支持 | ❌ 仅 APT |
| 动态路由 | ❌ | ✅ 远程下发 | ❌ |
| 拦截器 | 全局链 | 全局 + FlowTask | UriInterceptor |
| Kotlin 友好 | 一般 | 优秀 | 一般 |

### 5.3 从 ARouter 迁移到 TheRouter

核心变更：

````java
// 注解：@Route(path = "/order/detail") → @Route(path = "therouter://order/detail")
// 跳转：ARouter.getInstance().build("/x").navigation() → TheRouter.build("therouter://x").navigation()
// 服务：IProvider + @Route → @ServiceProvider，获取用 TheRouter.get(IXService.class)
````
````groovy
// 依赖替换
// arouter-api → cn.therouter:router:1.2.2
// arouter-compiler → cn.therouter:apt:1.2.2（或 ksp）
````
迁移步骤：引入 TheRouter → 新页面用 TheRouter → 两框架短期共存 → 逐步替换旧模块 → 移除 ARouter。

### 5.4 常见踩坑

| 问题 | 原因与解决 |
|------|-----------|
| 多模块编译失败 | 每个模块必须配置唯一 `AROUTER_MODULE_NAME` |
| Instant Run 路由丢失 | Debug 模式开启 `ARouter.openDebug()` |
| path 只有一级报错 | path 必须至少两级，第一级作为 group 名 |
| 混淆后路由失效 | 添加 `-keep public class com.alibaba.android.arouter.routes.**{*;}` 等规则 |
| Fragment 不支持 ForResult | 通过 ViewModel / EventBus 传递结果 |
| 自定义对象传递失败 | 需实现 `SerializationService`（如 Gson 适配） |

---

## 参考

- [ARouter GitHub](https://github.com/alibaba/ARouter)
- [TheRouter 官方文档](https://therouter.cn/)
- [[2-组件间通信]]
- [[5-注解处理器APT与KSP]]
- [[Android组件化架构]]

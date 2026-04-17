# OkHttp 与 Retrofit

> Android 面试高频主题笔记，涵盖核心原理、源码分析、高级用法及实战踩坑。
> 相关：[[Android 网络编程]]、[[协程与 Flow]]、[[设计模式]]、[[Kotlin 基础]]

---

## 一、核心概念

### 1.1 OkHttp 概述

OkHttp 是 Square 开源的高性能 HTTP 客户端，Android 4.4 起 `HttpURLConnection` 底层已替换为 OkHttp。核心优势：HTTP/2 多路复用、连接池复用、透明 GZIP、响应缓存、拦截器链。

### 1.2 拦截器链（Interceptor Chain）

采用 [[设计模式#责任链模式]]，内置五大拦截器（按执行顺序）：

| 拦截器 | 职责 |
|---|---|
| `RetryAndFollowUpInterceptor` | 重试与重定向 |
| `BridgeInterceptor` | 补全请求头 |
| `CacheInterceptor` | 缓存读写 |
| `ConnectInterceptor` | 建立/复用连接 |
| `CallServerInterceptor` | 发送请求并读取响应 |

```java
OkHttpClient client = new OkHttpClient.Builder()
    .addInterceptor(appInterceptor)            // 应用拦截器：最先执行，只调用一次
    .addNetworkInterceptor(networkInterceptor) // 网络拦截器：连接建立后执行
    .build();
```

**应用拦截器 vs 网络拦截器**：
- 应用拦截器：不关心重定向，始终只调用一次；可短路不调用 `chain.proceed()`
- 网络拦截器：能获取最终请求（含补全头部）；重定向时会多次调用

### 1.3 连接池（ConnectionPool）

通过 `ConnectionPool` 复用连接，避免重复 TCP/TLS 握手：

```java
ConnectionPool pool = new ConnectionPool(5, 5, TimeUnit.MINUTES);
```

- 使用 `Deque<RealConnection>` 存储，后台线程定期清理空闲超时连接
- HTTP/2 支持单连接多路复用

### 1.4 缓存机制

遵循 HTTP 缓存规范（`Cache-Control`、`ETag`、`Last-Modified`），由 `CacheStrategy` 决定策略：

```java
OkHttpClient client = new OkHttpClient.Builder()
    .cache(new Cache(new File(context.getCacheDir(), "http_cache"), 10L * 1024 * 1024))
    .build();
```

### 1.5 Retrofit 概述

基于 OkHttp 的 RESTful 封装层，**接口 + 注解描述请求，动态代理生成实现**：

```kotlin
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
}

val api = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(ApiService::class.java)
```

### 1.6 动态代理

`create()` 使用 [[Java 基础#动态代理]] (`Proxy.newProxyInstance`) 运行时生成接口实现，调用方法时由 `InvocationHandler.invoke()` 完成注解解析和请求构建。

### 1.7 注解体系

| 类别 | 注解 |
|---|---|
| 请求方法 | `@GET` `@POST` `@PUT` `@DELETE` `@PATCH` |
| URL 处理 | `@Path` `@Query` `@QueryMap` `@Url` |
| 请求体 | `@Body` `@Field` `@FieldMap` `@Part` |
| 请求头 | `@Header` `@HeaderMap` `@Headers` |

### 1.8 CallAdapter

将 `Call<T>` 适配为其他类型。Retrofit 2.6+ 内置 `suspend` 支持，无需额外 Adapter。

---

## 二、原理与源码

### 2.1 OkHttp 责任链源码

```java
// RealCall.java 简化
Response getResponseWithInterceptorChain() throws IOException {
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());              // 应用拦截器
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.cache()));
    interceptors.add(ConnectInterceptor.INSTANCE);
    if (!forWebSocket) interceptors.addAll(client.networkInterceptors()); // 网络拦截器
    interceptors.add(new CallServerInterceptor(forWebSocket));

    RealInterceptorChain chain = new RealInterceptorChain(interceptors, 0, originalRequest);
    return chain.proceed(originalRequest);
}

// RealInterceptorChain.java 简化
@Override
public Response proceed(Request request) throws IOException {
    RealInterceptorChain next = copy(index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    return interceptor.intercept(next); // 当前拦截器内部调用 next.proceed()
}
```

请求从外向内传递，响应从内向外返回，如同递归调用栈。

### 2.2 Retrofit create() 动态代理

```java
// Retrofit.java 简化
public <T> T create(final Class<T> service) {
    return (T) Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] { service },
        (proxy, method, args) -> {
            if (method.getDeclaringClass() == Object.class)
                return method.invoke(this, args);
            return loadServiceMethod(method).invoke(args);
        }
    );
}
```

`loadServiceMethod()` 流程：查 ConcurrentHashMap 缓存 → 未命中则 `ServiceMethod.parseAnnotations()` → 生成 `RequestFactory` → 查找 `CallAdapter` + `Converter` → 构建 `HttpServiceMethod`。

### 2.3 请求流程图

```
┌──────────────────────────────────────────────┐
│                 Retrofit 层                   │
│  ApiService.getUser("123")                   │
│       ▼                                      │
│  Proxy.invoke() → loadServiceMethod()        │
│       ▼                                      │
│  HttpServiceMethod.invoke()                  │
│       ▼                                      │
│  CallAdapter.adapt(OkHttpCall)               │
│       ▼                                      │
│  OkHttpCall.execute() / enqueue()            │
└──────┬───────────────────────────────────────┘
       ▼
┌──────────────────────────────────────────────┐
│                 OkHttp 层                     │
│  getResponseWithInterceptorChain()           │
│       ▼                                      │
│  应用拦截器 → RetryAndFollowUp               │
│       ▼                                      │
│  Bridge → Cache → Connect                    │
│       ▼                                      │
│  网络拦截器 → CallServer                      │
│       ▼                                      │
│  Server Response                             │
└──────────────────────────────────────────────┘
```

### 2.4 suspend 函数处理

Retrofit 检测到 `suspend`（最后参数为 `Continuation`）时，内部通过 `suspendCancellableCoroutine` 将回调转协程：

```kotlin
suspend fun <T> Call<T>.await(): T = suspendCancellableCoroutine { cont ->
    cont.invokeOnCancellation { cancel() }
    enqueue(object : Callback<T> {
        override fun onResponse(call: Call<T>, response: Response<T>) {
            if (response.isSuccessful) cont.resume(response.body()!!)
            else cont.resumeWithException(HttpException(response))
        }
        override fun onFailure(call: Call<T>, t: Throwable) {
            cont.resumeWithException(t)
        }
    })
}
```

---

## 三、高级用法

### 3.1 自定义拦截器

#### Token 注入

```java
public class AuthInterceptor implements Interceptor {
    private final Supplier<String> tokenProvider;

    public AuthInterceptor(Supplier<String> tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        String token = tokenProvider.get();
        if (token == null) return chain.proceed(chain.request());
        return chain.proceed(
            chain.request().newBuilder().header("Authorization", "Bearer " + token).build()
        );
    }
}
```

#### Token 过期自动刷新

```java
public class TokenRefreshInterceptor implements Interceptor {
    private final TokenManager tokenManager;

    public TokenRefreshInterceptor(TokenManager tokenManager) {
        this.tokenManager = tokenManager;
    }

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = chain.proceed(
            request.newBuilder().header("Authorization", "Bearer " + tokenManager.getAccessToken()).build()
        );
        if (response.code() == 401) {
            response.close();
            synchronized (this) {
                String newToken = tokenManager.refreshTokenSync();
                return chain.proceed(
                    request.newBuilder().header("Authorization", "Bearer " + newToken).build()
                );
            }
        }
        return response;
    }
}
```

### 3.2 自定义 Converter（自动解包）

```java
// 服务端统一格式 {"code":0, "msg":"ok", "data":{...}}
public class UnwrapConverterFactory extends Converter.Factory {
    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(
            Type type, Annotation[] annotations, Retrofit retrofit) {
        Type wrappedType = TypeToken.getParameterized(ApiResponse.class, type).getType();
        Converter<ResponseBody, ApiResponse<Object>> delegate =
            retrofit.nextResponseBodyConverter(this, wrappedType, annotations);
        return (Converter<ResponseBody, Object>) body -> {
            ApiResponse<Object> resp = delegate.convert(body);
            if (resp != null && resp.getCode() == 0) return resp.getData();
            else throw new ApiException(
                resp != null ? resp.getCode() : -1,
                resp != null ? resp.getMsg() : "Unknown");
        };
    }
}
// 添加顺序：UnwrapConverterFactory 在 GsonConverterFactory 之前
```

### 3.3 协程 + Flow 集成

```kotlin
// ViewModel 中使用
class UserViewModel(private val api: ApiService) : ViewModel() {
    private val _user = MutableStateFlow<UiState<User>>(UiState.Loading)
    val user: StateFlow<UiState<User>> = _user.asStateFlow()

    fun loadUser(id: String) = viewModelScope.launch {
        _user.value = try { UiState.Success(api.getUser(id)) }
        catch (e: Exception) { UiState.Error(e.message ?: "Unknown") }
    }
}

// 通用 Flow 封装
fun <T> apiFlow(block: suspend () -> T): Flow<UiState<T>> = flow {
    emit(UiState.Loading)
    emit(UiState.Success(block()))
}.catch { emit(UiState.Error(it.message ?: "Unknown")) }.flowOn(Dispatchers.IO)
```

---

## 四、面试题

### Q1：应用拦截器和网络拦截器的区别？

**A**：应用拦截器在最外层，只执行一次，可短路；网络拦截器在 `ConnectInterceptor` 之后，能获取最终请求，重定向时多次执行。

### Q2：Retrofit 动态代理的实现原理？

**A**：`create()` 调用 `Proxy.newProxyInstance()` 生成代理对象。方法调用时触发 `InvocationHandler.invoke()` → `loadServiceMethod()` 解析注解（有 ConcurrentHashMap 缓存）→ 生成 `RequestFactory` → 查找 `CallAdapter` + `Converter` → 构建 `OkHttpCall` 执行请求。

### Q3：OkHttp 连接池如何工作？

**A**：`ConnectionPool` 用 `Deque<RealConnection>` 存储连接，请求时按 host/port/TLS 匹配复用。后台清理线程回收空闲超时（默认 5 分钟）或超出上限（默认 5 个）的连接。HTTP/2 单连接多路复用。

### Q4：Converter 和 CallAdapter 的查找机制？

**A**：都采用工厂模式 + 顺序查找。遍历 Factory 列表，第一个返回非 null 的胜出。添加顺序很重要，自定义 Factory 放在内置之前。`nextResponseBodyConverter()` 可跳过当前 Factory 实现委托。

### Q5：OkHttp 如何取消请求？

**A**：`Call.cancel()` 设置标志位 → 排队中直接移除；执行中关闭 Socket 触发 IOException；`RetryAndFollowUpInterceptor` 每次 proceed 前检查标志。协程场景下 `suspendCancellableCoroutine` 的 `invokeOnCancellation` 自动调用 `call.cancel()`。

### Q6：OkHttp 和 Retrofit 的线程模型？

**A**：OkHttp 同步 `execute()` 在调用线程；异步 `enqueue()` 由 `Dispatcher` 线程池执行（默认 maxRequests=64，maxRequestsPerHost=5）。Retrofit 的 `suspend` 函数内部用 `enqueue()` + 协程挂起，自动切回调用者 Dispatcher。

---

## 五、实战与踩坑

### 5.1 超时配置

```java
OkHttpClient client = new OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .writeTimeout(30, TimeUnit.SECONDS)
    .callTimeout(60, TimeUnit.SECONDS) // 覆盖整个请求生命周期
    .build();
```

踩坑：`callTimeout` 含重试重定向总时间；上传大文件需加大 `writeTimeout`。

### 5.2 证书锁定

```java
OkHttpClient client = new OkHttpClient.Builder()
    .certificatePinner(new CertificatePinner.Builder()
        .add("api.example.com", "sha256/AAAA...=") // 主证书
        .add("api.example.com", "sha256/BBBB...=") // 备用证书
        .build())
    .build();
```

踩坑：证书更换后公钥哈希不匹配会断网，**务必添加备用 Pin**。建议配合 [[Android 安全]] Network Security Config。

### 5.3 日志拦截器

```java
HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
logging.setLevel(BuildConfig.DEBUG ? Level.BODY : Level.NONE);
```

踩坑：`BODY` 级别会将响应体全部读入内存，大文件会 OOM；生产环境设 `NONE` 防泄露敏感数据。推荐 [[Android 调试工具#Chucker]]。

### 5.4 多 BaseUrl 方案

**方案一**：多个 Retrofit 实例（简单直接）

```java
public class ApiManager {
    public static final MainApi mainApi = createRetrofit("https://api.example.com/").create(MainApi.class);
    public static final UploadApi uploadApi = createRetrofit("https://upload.example.com/").create(UploadApi.class);
}
```

**方案二**：`@Url` 注解传完整 URL

```kotlin
@GET
suspend fun getData(@Url fullUrl: String): ResponseBody
```

**方案三**：拦截器动态替换（灵活）

```java
public class DynamicBaseUrlInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Invocation invocation = request.tag(Invocation.class);
        BaseUrl annotation = invocation != null
            ? invocation.method().getAnnotation(BaseUrl.class) : null;
        if (annotation != null) {
            HttpUrl newUrl = request.url().newBuilder()
                .host(HttpUrl.parse(annotation.value()).host()).build();
            return chain.proceed(request.newBuilder().url(newUrl).build());
        }
        return chain.proceed(request);
    }
}
```

### 5.5 其他踩坑

- **ResponseBody 只能消费一次**：`response.body?.string()` 读取后流关闭，需提前缓存
- **返回类型选择**：`suspend fun getUser(): User` 非 2xx 抛 HttpException；`suspend fun getUser(): Response<User>` 可手动判断状态码
- **并发限制**：默认 maxRequests=64、maxRequestsPerHost=5，高并发需调整 `Dispatcher`

---

## 参考

- [OkHttp 官方文档](https://square.github.io/okhttp/) · [Retrofit 官方文档](https://square.github.io/retrofit/)
- [[Android 网络编程]] · [[协程与 Flow]] · [[设计模式#责任链模式]] · [[Android 安全]]

> 最后更新：2026-04-15

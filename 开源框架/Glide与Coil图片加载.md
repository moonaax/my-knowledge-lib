# Glide 与 Coil 图片加载

> 本文系统梳理 Android 两大主流图片加载框架 **Glide** 与 **Coil** 的核心原理、源码分析、高级用法及面试高频题。代码以 Java 为主，Coil 部分使用 Kotlin。
> 相关笔记：[[内存优化]] · [[OkHttp与Retrofit]] · [[Bitmap与图片压缩]] · [[Android生命周期]]

---

## 一、核心概念

### 1.1 图片加载三级缓存

图片加载框架普遍采用 **内存缓存 → 磁盘缓存 → 网络** 的三级缓存策略，目标是在速度与资源消耗之间取得平衡。

```
请求图片 URL
    │
    ▼
┌──────────────┐  命中   ┌────────────┐
│  内存缓存     │───────→│  直接显示   │
│ (ActiveRes + │        └────────────┘
│  LruCache)   │
└──────┬───────┘
       │ 未命中
       ▼
┌──────────────┐  命中   ┌────────────┐
│  磁盘缓存     │───────→│ 解码后显示  │
│ (DiskLruCache)│       └────────────┘
└──────┬───────┘
       │ 未命中
       ▼
┌──────────────┐        ┌────────────┐
│  网络请求     │───────→│ 写入缓存   │
│ (HttpUrlConn │        │ 后显示     │
│  / OkHttp)   │        └────────────┘
└──────────────┘
```

| 缓存层级 | 存储介质 | 速度 | 特点 |
|---------|---------|------|------|
| **ActiveResources** | 内存（弱引用 HashMap） | 最快 | 正在使用的资源，GC 可回收 |
| **MemoryCache** | 内存（LruCache） | 极快 | 最近使用但当前未活跃的资源 |
| **DiskCache** | 磁盘（DiskLruCache） | 较快 | 持久化存储，App 重启后仍可用 |
| **网络** | 远程服务器 | 最慢 | 首次加载必经路径 |

Glide 的内存缓存实际上分为两层：**ActiveResources**（活跃资源，弱引用）和 **MemoryCache**（LruCache）。当图片正在被 View 使用时存放在 ActiveResources 中；当 View 销毁后资源移入 MemoryCache。这种设计避免了 LruCache 在容量不足时将正在使用的 Bitmap 回收。

### 1.2 Glide 生命周期绑定

Glide 最精妙的设计之一是 **自动感知 Activity / Fragment 生命周期**，在页面不可见时暂停请求，页面销毁时自动取消请求并释放资源。

核心机制：

```java
// 用户调用
Glide.with(activity).load(url).into(imageView);

// Glide.with() 内部：
// 1. 向当前 Activity 添加一个不可见的 SupportRequestManagerFragment
// 2. 该 Fragment 持有 ActivityFragmentLifecycle
// 3. RequestManager 注册为 LifecycleListener
// 4. 当 Activity onStop → Fragment onStop → RequestManager.onStop() → 暂停所有请求
// 5. 当 Activity onDestroy → Fragment onDestroy → RequestManager.onDestroy() → 清理资源
```

关键源码路径：

```java
// RequestManagerRetriever.java
private RequestManager supportFragmentGet(Context context,
        FragmentManager fm, Fragment parentHint, boolean isParentVisible) {
    // 查找或创建 SupportRequestManagerFragment
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        Glide glide = Glide.get(context);
        requestManager = factory.build(glide, current.getGlideLifecycle(),
                current.getRequestManagerTreeNode(), context);
        current.setRequestManager(requestManager);
    }
    return requestManager;
}
```

> ⚠️ 注意：如果传入 `ApplicationContext`，Glide 无法绑定生命周期，请求会跟随 App 进程存活，容易造成 [[内存优化|内存泄漏]]。

### 1.3 Bitmap 复用池（BitmapPool）

频繁创建和回收 Bitmap 会导致大量 GC，Glide 通过 **BitmapPool** 实现 Bitmap 对象复用：

```java
// Glide 内部解码时使用 BitmapPool
BitmapFactory.Options options = new BitmapFactory.Options();
options.inMutable = true;
// 从池中获取可复用的 Bitmap
options.inBitmap = bitmapPool.get(width, height, config);
Bitmap result = BitmapFactory.decodeStream(inputStream, null, options);
```

BitmapPool 的默认实现是 `LruBitmapPool`，按照 Bitmap 的 **宽 × 高 × Config** 进行分组存储。当需要新 Bitmap 时，优先从池中取出尺寸匹配的对象，避免重新分配内存。

在 Android 4.4+ 上，`inBitmap` 只要求复用 Bitmap 的字节数 ≥ 目标 Bitmap 即可，灵活性大幅提升。

### 1.4 Coil 的协程架构

**Coil**（Coroutine Image Loader）是 Kotlin-first 的图片加载库，核心特点：

```kotlin
// 最简用法
imageView.load("https://example.com/image.jpg") {
    crossfade(true)
    placeholder(R.drawable.placeholder)
    error(R.drawable.error)
}
```

Coil 的架构优势：

| 特性 | 说明 |
|------|------|
| **协程驱动** | 所有 I/O 操作在协程中执行，天然支持取消与结构化并发 |
| **Lifecycle 感知** | 通过 `ViewTargetRequestDelegate` 绑定 `ViewTreeLifecycleOwner` |
| **OkHttp 集成** | 默认使用 [[OkHttp与Retrofit|OkHttp]] 作为网络层 |
| **零反射** | 不使用注解处理器或反射，启动速度快 |
| **轻量** | 约 1500 个方法数，远小于 Glide 的 ~8000 |

Coil 内部请求流程：

```kotlin
// ImageLoader.enqueue() 简化流程
suspend fun execute(request: ImageRequest): ImageResult {
    // 1. 检查内存缓存
    val memoryCacheKey = request.memoryCacheKey
    val cached = memoryCache?.get(memoryCacheKey)
    if (cached != null) return SuccessResult(cached, request, DataSource.MEMORY_CACHE)

    // 2. 通过 Interceptor 链处理（类似 OkHttp）
    val result = withContext(request.interceptorDispatcher) {
        interceptorChain.proceed(request)
    }

    // 3. 写入内存缓存
    if (result is SuccessResult) {
        memoryCache?.set(memoryCacheKey, result.drawable)
    }
    return result
}
```

Coil 2.x 采用 **Interceptor 链** 模式（借鉴 OkHttp），每个拦截器负责一个职责（缓存、映射、解码等），扩展性极强。

---


## 二、原理与源码

### 2.1 Glide 的 Engine.load 流程

`Engine` 是 Glide 的核心调度器，`load()` 方法串联了整个缓存查找与加载逻辑：

```java
// Engine.java（简化）
public <R> LoadStatus load(
        GlideContext glideContext, Object model, Key signature,
        int width, int height, Class<?> resourceClass,
        Class<R> transcodeClass, Priority priority,
        DiskCacheStrategy diskCacheStrategy, Map<Class<?>, Transformation<?>> transformations,
        boolean isMemoryCacheable, boolean useUnlimitedSourceGeneratorPool,
        boolean useAnimationPool, boolean onlyRetrieveFromCache,
        ResourceCallback cb, Executor callbackExecutor) {

    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    // ① 生成缓存 Key
    EngineKey key = keyFactory.buildKey(model, signature, width, height,
            transformations, resourceClass, transcodeClass, options);

    // ② 检查 ActiveResources（弱引用缓存）
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
        cb.onResourceReady(active, DataSource.MEMORY_CACHE);
        return null;
    }

    // ③ 检查 MemoryCache（LruCache）
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
        cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
        return null;
    }

    // ④ 检查是否有正在执行的相同任务（避免重复请求）
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
        current.addCallback(cb, callbackExecutor);
        return new LoadStatus(cb, current);
    }

    // ⑤ 创建 EngineJob + DecodeJob，提交到线程池
    EngineJob<R> engineJob = engineJobFactory.build(key, isMemoryCacheable,
            useUnlimitedSourceGeneratorPool, useAnimationPool, onlyRetrieveFromCache);
    DecodeJob<R> decodeJob = decodeJobFactory.build(glideContext, model, key,
            signature, width, height, resourceClass, transcodeClass, priority,
            diskCacheStrategy, transformations, isTransformationRequired,
            isScaleOnlyOrNoTransform, onlyRetrieveFromCache, options, engineJob);

    jobs.put(key, engineJob);
    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);  // 提交到线程池执行

    return new LoadStatus(cb, engineJob);
}
```

完整流程图：

```
Glide.with().load().into()
    │
    ▼
RequestManager.track(request)
    │
    ▼
SingleRequest.begin()
    │
    ▼
Engine.load()
    ├─→ ActiveResources (弱引用) ──→ 命中 → 回调显示
    ├─→ MemoryCache (LruCache)  ──→ 命中 → 回调显示
    ├─→ 已有相同 EngineJob       ──→ 复用 → 添加回调
    └─→ 新建 DecodeJob
            ├─→ ResourceCacheGenerator  → 磁盘缓存(转换后)
            ├─→ DataCacheGenerator      → 磁盘缓存(原始数据)
            └─→ SourceGenerator         → 网络/本地源
                    │
                    ▼
              解码 → 转换 → 写入缓存 → 回调显示
```

### 2.2 缓存 Key 生成

Glide 的缓存 Key 由多个维度组合，确保不同请求参数对应不同缓存：

```java
// EngineKey.java
class EngineKey implements Key {
    private final Object model;           // URL / Uri / File 等
    private final Key signature;          // 自定义签名（版本控制）
    private final int width;              // 目标宽度
    private final int height;             // 目标高度
    private final Map<Class<?>, Transformation<?>> transformations;
    private final Class<?> resourceClass;
    private final Class<?> transcodeClass;
    private final Options options;

    @Override
    public boolean equals(Object o) {
        if (o instanceof EngineKey) {
            EngineKey other = (EngineKey) o;
            return model.equals(other.model)
                    && signature.equals(other.signature)
                    && height == other.height
                    && width == other.width
                    && transformations.equals(other.transformations)
                    && resourceClass.equals(other.resourceClass)
                    && transcodeClass.equals(other.transcodeClass)
                    && options.equals(other.options);
        }
        return false;
    }

    @Override
    public int hashCode() {
        // 基于所有字段计算 hashCode
        int result = model.hashCode();
        result = 31 * result + signature.hashCode();
        result = 31 * result + width;
        result = 31 * result + height;
        result = 31 * result + transformations.hashCode();
        result = 31 * result + resourceClass.hashCode();
        result = 31 * result + transcodeClass.hashCode();
        result = 31 * result + options.hashCode();
        return result;
    }
}
```

> 💡 这意味着：**同一张图片，不同尺寸的 ImageView 加载会产生不同的缓存条目**。这是 Glide 能精确匹配目标尺寸、减少内存占用的关键。

### 2.3 ActiveResources + MemoryCache + DiskCache

**ActiveResources**：

```java
// ActiveResources.java
final class ActiveResources {
    // 核心数据结构：HashMap<Key, WeakReference<EngineResource>>
    final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
    private final ReferenceQueue<EngineResource<?>> resourceReferenceQueue = new ReferenceQueue<>();

    synchronized EngineResource<?> get(Key key) {
        ResourceWeakReference activeRef = activeEngineResources.get(key);
        if (activeRef == null) return null;

        EngineResource<?> active = activeRef.get();
        if (active == null) {
            // 弱引用已被 GC 回收，清理条目
            cleanupActiveReference(activeRef);
        }
        return active;
    }

    // 资源引用计数归零时，从 ActiveResources 移入 MemoryCache
    synchronized void deactivate(Key key) {
        ResourceWeakReference removed = activeEngineResources.remove(key);
        if (removed != null) {
            removed.reset();
        }
    }
}
```

**MemoryCache**（默认 `LruResourceCache`）：

```java
// LruResourceCache.java — 继承自 LruCache
public class LruResourceCache extends LruCache<Key, Resource<?>> implements MemoryCache {

    @Override
    protected int getSize(Resource<?> item) {
        return item.getSize(); // Bitmap 字节数
    }

    @Override
    protected void onItemEvicted(Key key, Resource<?> item) {
        // 被淘汰的资源回收到 BitmapPool
        if (listener != null) {
            listener.onResourceRemoved(item);
        }
    }
}
```

**DiskCache**（默认 `DiskLruCacheWrapper`）：

Glide 的磁盘缓存支持两种策略：
- `DATA`：缓存原始数据（未解码）
- `RESOURCE`：缓存解码 + 转换后的数据

```java
// DiskCacheStrategy 常用选项
DiskCacheStrategy.ALL       // 原始数据 + 转换后数据都缓存
DiskCacheStrategy.DATA      // 只缓存原始数据
DiskCacheStrategy.RESOURCE  // 只缓存转换后数据
DiskCacheStrategy.NONE      // 不使用磁盘缓存
DiskCacheStrategy.AUTOMATIC // 默认，智能选择
```

### 2.4 RequestManager 与 Lifecycle 绑定源码

```java
// RequestManager.java
public class RequestManager implements ComponentCallbacks2,
        LifecycleListener, ModelTypes<RequestBuilder<Drawable>> {

    private final Lifecycle lifecycle;
    private final RequestTracker requestTracker;
    private final TargetTracker targetTracker;

    RequestManager(Glide glide, Lifecycle lifecycle,
            RequestManagerTreeNode treeNode, Context context) {
        this.glide = glide;
        this.lifecycle = lifecycle;
        this.requestTracker = new RequestTracker();
        this.targetTracker = new TargetTracker();

        // 注册生命周期监听
        lifecycle.addListener(this);
    }

    // Activity/Fragment 可见
    @Override
    public synchronized void onStart() {
        resumeRequests();
        targetTracker.onStart();
    }

    // Activity/Fragment 不可见 → 暂停请求
    @Override
    public synchronized void onStop() {
        pauseRequests();
        targetTracker.onStop();
    }

    // Activity/Fragment 销毁 → 清理所有请求
    @Override
    public synchronized void onDestroy() {
        targetTracker.onDestroy();
        for (Target<?> target : targetTracker.getAll()) {
            clear(target);
        }
        targetTracker.clear();
        requestTracker.clearRequests();
        lifecycle.removeListener(this);
        glide.unregisterRequestManager(this);
    }
}
```

`RequestTracker` 内部维护了一个 `Set<Request>`，`pauseRequests()` 遍历所有 Request 调用 `pause()`，`resumeRequests()` 调用 `begin()` 恢复。

### 2.5 Coil 与 Glide 对比表格

| 维度 | Glide 4.x | Coil 2.x |
|------|-----------|----------|
| **语言** | Java（支持 Kotlin 扩展） | Kotlin-first |
| **异步方案** | 自定义线程池 + Handler | Kotlin 协程 |
| **生命周期感知** | 无头 Fragment | `ViewTreeLifecycleOwner` |
| **网络层** | 默认 HttpUrlConnection，可替换 OkHttp | 默认 [[OkHttp与Retrofit\|OkHttp]] |
| **内存缓存** | ActiveResources + LruCache | 强引用 LruCache |
| **磁盘缓存** | DiskLruCache | OkHttp 的 Cache / 自定义 DiskCache |
| **Bitmap 复用** | BitmapPool（LruBitmapPool） | 依赖系统 `inBitmap`，2.x 移除了 BitmapPool |
| **图片格式** | JPEG/PNG/GIF/WebP/AVIF | JPEG/PNG/GIF/WebP/SVG/AVIF（通过扩展） |
| **GIF 支持** | 内置 | 需要 `coil-gif` 扩展 |
| **方法数** | ~8000 | ~1500 |
| **包体积** | ~500KB | ~250KB |
| **扩展机制** | GlideModule + 注解处理器 | Interceptor 链 + ComponentRegistry |
| **Jetpack Compose** | 需要 accompanist 库 | 原生支持 `coil-compose` |
| **最低 API** | 14 | 21 |

---


## 三、高级用法

### 3.1 自定义 GlideModule

Glide 4.x 使用 `@GlideModule` 注解 + `AppGlideModule` 来全局配置：

```java
@GlideModule
public class MyGlideModule extends AppGlideModule {

    @Override
    public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
        // 自定义内存缓存大小（默认为屏幕大小 * 2 的 MemoryCache）
        int memoryCacheSizeBytes = 1024 * 1024 * 50; // 50MB
        builder.setMemoryCache(new LruResourceCache(memoryCacheSizeBytes));

        // 自定义 BitmapPool 大小
        builder.setBitmapPool(new LruBitmapPool(memoryCacheSizeBytes));

        // 自定义磁盘缓存
        builder.setDiskCache(new InternalCacheDiskCacheFactory(context,
                "glide_cache", 1024 * 1024 * 250)); // 250MB

        // 设置默认请求选项
        builder.setDefaultRequestOptions(
                new RequestOptions()
                        .format(DecodeFormat.PREFER_ARGB_8888)
                        .disallowHardwareConfig()
        );

        // 日志级别
        builder.setLogLevel(Log.DEBUG);
    }

    @Override
    public void registerComponents(@NonNull Context context,
            @NonNull Glide glide, @NonNull Registry registry) {
        // 替换网络层为 OkHttp
        OkHttpClient client = new OkHttpClient.Builder()
                .connectTimeout(15, TimeUnit.SECONDS)
                .readTimeout(20, TimeUnit.SECONDS)
                .addInterceptor(new ProgressInterceptor()) // 自定义进度拦截器
                .build();
        registry.replace(GlideUrl.class, InputStream.class,
                new OkHttpUrlLoader.Factory(client));
    }

    // 禁用清单解析，提升初始化速度
    @Override
    public boolean isManifestParsingEnabled() {
        return false;
    }
}
```

编译后会自动生成 `GlideApp` 类，使用方式：

```java
GlideApp.with(this)
        .load(url)
        .placeholder(R.drawable.placeholder)
        .error(R.drawable.error)
        .into(imageView);
```

**Coil 的全局配置**（对比）：

```kotlin
// Application.onCreate()
val imageLoader = ImageLoader.Builder(this)
    .memoryCache {
        MemoryCache.Builder(this)
            .maxSizePercent(0.25) // 使用 25% 可用内存
            .build()
    }
    .diskCache {
        DiskCache.Builder()
            .directory(cacheDir.resolve("image_cache"))
            .maxSizeBytes(250L * 1024 * 1024) // 250MB
            .build()
    }
    .okHttpClient {
        OkHttpClient.Builder()
            .cache(CoilUtils.createDefaultCache(this))
            .build()
    }
    .crossfade(true)
    .build()

Coil.setImageLoader(imageLoader)
```

### 3.2 自定义 Transformation

Transformation 用于对解码后的 Bitmap 进行变换（圆角、模糊、裁剪等）。

**Glide 自定义圆角 Transformation**：

```java
public class RoundedCornersTransformation extends BitmapTransformation {

    private final float radius;

    public RoundedCornersTransformation(float radiusDp, Context context) {
        this.radius = TypedValue.applyDimension(
                TypedValue.COMPLEX_UNIT_DIP, radiusDp,
                context.getResources().getDisplayMetrics());
    }

    @Override
    protected Bitmap transform(@NonNull BitmapPool pool,
            @NonNull Bitmap toTransform, int outWidth, int outHeight) {
        Bitmap result = pool.get(outWidth, outHeight, Bitmap.Config.ARGB_8888);
        Canvas canvas = new Canvas(result);
        Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setShader(new BitmapShader(toTransform,
                Shader.TileMode.CLAMP, Shader.TileMode.CLAMP));
        RectF rect = new RectF(0, 0, outWidth, outHeight);
        canvas.drawRoundRect(rect, radius, radius, paint);
        return result;
    }

    @Override
    public void updateDiskCacheKey(@NonNull MessageDigest messageDigest) {
        String key = "RoundedCorners(radius=" + radius + ")";
        messageDigest.update(key.getBytes(Key.CHARSET));
    }
}

// 使用
Glide.with(context)
        .load(url)
        .transform(new RoundedCornersTransformation(16f, context))
        .into(imageView);
```

**多重变换组合**：

```java
Glide.with(context)
        .load(url)
        .transform(new MultiTransformation<>(
                new CenterCrop(),
                new RoundedCornersTransformation(16f, context)
        ))
        .into(imageView);
```

**Coil 自定义 Transformation**：

```kotlin
class GrayscaleTransformation : Transformation {
    override val cacheKey: String = "GrayscaleTransformation"

    override suspend fun transform(input: Bitmap, size: Size): Bitmap {
        val output = Bitmap.createBitmap(input.width, input.height, input.config)
        val canvas = Canvas(output)
        val paint = Paint().apply {
            colorFilter = ColorMatrixColorFilter(ColorMatrix().apply { setSaturation(0f) })
        }
        canvas.drawBitmap(input, 0f, 0f, paint)
        return output
    }
}

// 使用
imageView.load(url) {
    transformations(CircleCropTransformation(), GrayscaleTransformation())
}
```

### 3.3 预加载

预加载可以在用户即将看到图片之前提前将其加载到缓存中，提升用户体验。

**Glide 预加载**：

```java
// 预加载到内存缓存（指定尺寸）
Glide.with(context)
        .load(url)
        .preload(500, 500);

// 预加载到磁盘缓存（原始尺寸）
Glide.with(context)
        .downloadOnly()
        .load(url)
        .preload();

// RecyclerView 预加载集成
RecyclerView recyclerView = findViewById(R.id.recycler_view);
RecyclerViewPreloader<String> preloader = new RecyclerViewPreloader<>(
        Glide.with(this), preloadModelProvider, preloadSizeProvider, 10 /* maxPreload */);
recyclerView.addOnScrollListener(preloader);
```

**Coil 预加载**：

```kotlin
// 预加载到内存缓存
val request = ImageRequest.Builder(context)
    .data(url)
    .size(500, 500)
    .memoryCachePolicy(CachePolicy.ENABLED)
    .build()
imageLoader.enqueue(request)

// 预加载到磁盘缓存
val request = ImageRequest.Builder(context)
    .data(url)
    .memoryCachePolicy(CachePolicy.DISABLED)
    .diskCachePolicy(CachePolicy.ENABLED)
    .build()
imageLoader.enqueue(request)
```

**批量预加载最佳实践**：

```java
// 在列表页提前预加载下一页图片
public void preloadNextPage(List<String> urls) {
    for (String url : urls) {
        Glide.with(applicationContext)
                .load(url)
                .diskCacheStrategy(DiskCacheStrategy.DATA)
                .preload();
    }
}
```

> ⚠️ 预加载应使用 `applicationContext`，避免绑定到即将销毁的 Activity。同时注意控制并发数量，避免抢占正在显示图片的带宽。

---


## 四、面试题

### Q1: Glide 的三级缓存机制是怎样的？为什么内存缓存要分两层？

**答：** Glide 的缓存分为三级：ActiveResources（活跃资源缓存）、MemoryCache（内存 LruCache）、DiskCache（磁盘缓存）。加上网络请求，实际是"三级缓存 + 网络"的四层结构。

内存缓存分两层的原因：ActiveResources 使用弱引用存储当前正在被 View 使用的资源，MemoryCache 使用 LruCache 存储最近使用但当前未活跃的资源。这样设计的好处是：
1. **避免误回收**：LruCache 在容量不足时会淘汰最久未使用的条目，如果正在显示的 Bitmap 也在 LruCache 中，可能被淘汰导致显示异常。ActiveResources 通过引用计数保护正在使用的资源。
2. **提升命中率**：活跃资源直接从 HashMap 查找，O(1) 复杂度，比 LruCache 更快。
3. **内存友好**：弱引用在内存紧张时可被 GC 回收，资源回收后会自动从 ActiveResources 移除。

### Q2: Glide 是如何绑定生命周期的？为什么不能在子线程调用 Glide.with()？

**答：** Glide 通过向当前 Activity/Fragment 添加一个不可见的 `SupportRequestManagerFragment` 来感知生命周期。这个 Fragment 内部持有 `ActivityFragmentLifecycle`，`RequestManager` 注册为其 `LifecycleListener`。当 Activity 执行 `onStop()` 时，Fragment 同步回调，RequestManager 暂停所有请求；`onDestroy()` 时清理所有资源。

不能在子线程调用 `Glide.with(activity)` 的原因：添加 Fragment 需要操作 `FragmentManager`，这是一个 UI 操作，必须在主线程执行。如果在子线程调用，Glide 会降级使用 `ApplicationContext`，此时无法绑定生命周期，可能导致 [[内存优化|内存泄漏]]。Glide 4.x 在子线程调用时会打印警告日志。

### Q3: Glide 的缓存 Key 包含哪些信息？同一张图片在不同尺寸 ImageView 中会缓存几份？

**答：** Glide 的 `EngineKey` 由以下维度组合生成：`model`（URL 等）、`signature`（自定义签名）、`width`（目标宽度）、`height`（目标高度）、`transformations`（变换列表）、`resourceClass`、`transcodeClass`、`options`。

因此，**同一张图片在不同尺寸的 ImageView 中会生成不同的缓存 Key，对应不同的缓存条目**。例如一张图片在 100×100 和 200×200 的 ImageView 中加载，会在内存中缓存两份不同尺寸的 Bitmap。这是 Glide 的设计哲学——以空间换时间，避免每次显示时重新缩放。

如果希望共享缓存，可以使用 `.override(Target.SIZE_ORIGINAL)` 统一加载原始尺寸，但这会增加内存占用。

### Q4: Coil 相比 Glide 有哪些优势？什么场景下选择 Coil？

**答：** Coil 的核心优势：
1. **Kotlin-first**：API 设计完全面向 Kotlin，扩展函数 + DSL 语法更简洁
2. **协程驱动**：天然支持结构化并发和取消，与 ViewModel/Lifecycle 无缝集成
3. **轻量**：方法数约 1500（Glide 约 8000），包体积约 250KB（Glide 约 500KB）
4. **Compose 原生支持**：`coil-compose` 提供 `AsyncImage` 组件
5. **Interceptor 架构**：借鉴 [[OkHttp与Retrofit|OkHttp]] 的拦截器链，扩展性强
6. **零反射**：不使用注解处理器，编译速度更快

选择建议：
- **新项目且全 Kotlin**：优先选 Coil
- **需要 Compose 支持**：Coil 更自然
- **老项目 Java 为主**：继续使用 Glide
- **需要 GIF 深度支持**：Glide 内置支持更成熟
- **对包体积敏感**：Coil 更轻量

### Q5: BitmapPool 的作用是什么？Coil 2.x 为什么移除了 BitmapPool？

**答：** BitmapPool 的作用是复用已分配的 Bitmap 对象，避免频繁创建和回收 Bitmap 导致的内存抖动和 GC 压力。Glide 在解码图片时通过 `BitmapFactory.Options.inBitmap` 指定一个从池中获取的可复用 Bitmap，系统会直接在该内存区域解码，省去了内存分配的开销。

Coil 2.x 移除 BitmapPool 的原因：
1. **Android 8.0+ 的 Bitmap 内存分配在 native 层**，GC 压力已大幅降低
2. BitmapPool 本身占用大量内存（默认为屏幕大小 × 4），在内存紧张时反而是负担
3. 维护 BitmapPool 的匹配逻辑增加了复杂度
4. 实测表明在现代设备上，移除 BitmapPool 对性能影响很小，但能简化架构并减少内存占用

### Q6: 如何监控和优化 Glide 的内存使用？

**答：** 监控和优化手段：

1. **实现 `ComponentCallbacks2`**：Glide 默认注册了内存回调，在 `onTrimMemory()` 时自动清理缓存
```java
// Glide 内部已实现，也可手动触发
Glide.get(context).clearMemory(); // 主线程调用
Glide.get(context).trimMemory(TRIM_MEMORY_MODERATE);
```

2. **合理设置缓存大小**：通过自定义 GlideModule 调整 MemoryCache 和 BitmapPool 大小

3. **使用 `override()` 控制加载尺寸**：避免加载超出 ImageView 尺寸的大图
```java
Glide.with(context).load(url).override(300, 300).into(imageView);
```

4. **选择合适的 `DecodeFormat`**：`RGB_565` 比 `ARGB_8888` 节省一半内存
```java
Glide.with(context).load(url)
        .format(DecodeFormat.PREFER_RGB_565)
        .into(imageView);
```

5. **及时清理**：在 `onDestroy()` 中确保调用 `Glide.with(this).clear(target)`

6. **使用 `skipMemoryCache(true)`**：对一次性大图跳过内存缓存

更多内存优化策略参见 [[内存优化]]。

---


## 五、实战与踩坑

### 5.1 踩坑：RecyclerView 图片错位

**问题**：RecyclerView 快速滑动时，由于 ViewHolder 复用，旧的图片加载请求完成后会显示在错误的 Item 上。

**解决方案**：

```java
// ✅ 正确做法：Glide 自动处理（into 会取消之前的请求）
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    Glide.with(holder.itemView.getContext())
            .load(dataList.get(position).getImageUrl())
            .placeholder(R.drawable.placeholder)
            .into(holder.imageView);
}

// ❌ 错误做法：手动设置 Bitmap，不经过 Glide
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    // 异步加载后直接 setBitmap，无法取消旧请求
    loadBitmapAsync(url, bitmap -> holder.imageView.setImageBitmap(bitmap));
}
```

Glide 的 `into()` 方法内部会调用 `RequestManager.clear(target)` 取消该 Target 上之前的请求，然后绑定新请求。这是 Glide 解决图片错位的核心机制。

如果使用自定义 Target，需要在 `onViewRecycled` 中手动清理：

```java
@Override
public void onViewRecycled(@NonNull ViewHolder holder) {
    Glide.with(holder.itemView.getContext()).clear(holder.imageView);
    super.onViewRecycled(holder);
}
```

### 5.2 踩坑：Glide 加载圆角图片 + CenterCrop 失效

**问题**：同时使用 `centerCrop()` 和 `transform(new RoundedCorners(radius))` 时，只有最后一个 Transformation 生效。

```java
// ❌ 圆角不生效
Glide.with(context)
        .load(url)
        .centerCrop()
        .transform(new RoundedCorners(30))
        .into(imageView);
```

**原因**：`centerCrop()` 和 `transform()` 都会覆盖之前的 Transformation 设置。

**解决方案**：使用 `MultiTransformation` 或 `transforms()` 方法：

```java
// ✅ 正确做法
Glide.with(context)
        .load(url)
        .transform(new CenterCrop(), new RoundedCorners(30))
        .into(imageView);

// 或者使用 GlideApp 的链式调用
GlideApp.with(context)
        .load(url)
        .transforms(new CenterCrop(), new RoundedCorners(30))
        .into(imageView);
```

### 5.3 踩坑：内存泄漏 — 错误的 Context 使用

```java
// ❌ 在 Adapter 中持有 Activity Context
public class MyAdapter extends RecyclerView.Adapter<ViewHolder> {
    private final Context context; // 如果是 Activity，可能泄漏

    @Override
    public void onBindViewHolder(ViewHolder holder, int position) {
        // 使用 holder.itemView.getContext() 更安全
        Glide.with(context).load(url).into(holder.imageView);
    }
}

// ✅ 推荐做法
@Override
public void onBindViewHolder(ViewHolder holder, int position) {
    Glide.with(holder.itemView) // 自动获取最近的 Activity/Fragment
            .load(dataList.get(position).getUrl())
            .into(holder.imageView);
}
```

### 5.4 踩坑：Coil 在 Java 项目中的兼容问题

Coil 是 Kotlin-first 的库，在 Java 中使用需要额外适配：

```java
// Java 中使用 Coil（不如 Kotlin 简洁）
ImageRequest request = new ImageRequest.Builder(context)
        .data("https://example.com/image.jpg")
        .target(imageView)
        .crossfade(true)
        .build();
Coil.imageLoader(context).enqueue(request);
```

> 💡 如果项目以 Java 为主，建议继续使用 Glide；Coil 更适合纯 Kotlin 或 Kotlin + Compose 项目。

### 5.5 实战：大图加载与长图浏览

加载超大图片（如长截图、高清地图）时，直接加载到内存会导致 OOM：

```java
// ❌ 直接加载 10000×5000 的大图
Glide.with(context).load(hugeImageUrl).into(imageView);

// ✅ 方案一：限制加载尺寸
Glide.with(context)
        .load(hugeImageUrl)
        .override(screenWidth, Target.SIZE_ORIGINAL) // 宽度适配屏幕
        .downsample(DownsampleStrategy.AT_MOST)
        .into(imageView);

// ✅ 方案二：使用 SubsamplingScaleImageView 分块加载
// 先用 Glide 下载到磁盘
Glide.with(context)
        .downloadOnly()
        .load(hugeImageUrl)
        .into(new CustomTarget<File>() {
            @Override
            public void onResourceReady(@NonNull File resource,
                    @Nullable Transition<? super File> transition) {
                // 使用 SubsamplingScaleImageView 分块解码
                subsamplingView.setImage(ImageSource.uri(Uri.fromFile(resource)));
            }

            @Override
            public void onLoadCleared(@Nullable Drawable placeholder) {}
        });
```

### 5.6 实战：加载进度监听

Glide 默认不支持加载进度回调，需要通过 OkHttp 拦截器实现：

```java
// 1. 自定义进度拦截器
public class ProgressInterceptor implements Interceptor {
    static final Map<String, ProgressListener> LISTENERS = new ConcurrentHashMap<>();

    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = chain.proceed(request);
        String url = request.url().toString();
        ProgressListener listener = LISTENERS.get(url);
        if (listener == null) return response;

        return response.newBuilder()
                .body(new ProgressResponseBody(response.body(), listener))
                .build();
    }
}

// 2. 使用
ProgressInterceptor.LISTENERS.put(url, new ProgressListener() {
    @Override
    public void onProgress(int progress) {
        progressBar.setProgress(progress);
    }
});

Glide.with(context).load(url).into(new DrawableImageViewTarget(imageView) {
    @Override
    public void onResourceReady(@NonNull Drawable resource,
            @Nullable Transition<? super Drawable> transition) {
        super.onResourceReady(resource, transition);
        ProgressInterceptor.LISTENERS.remove(url);
    }
});
```

### 5.7 性能优化清单

| 优化项 | Glide 实现 | Coil 实现 |
|-------|-----------|----------|
| 列表预加载 | `RecyclerViewPreloader` | 自定义 `OnScrollListener` + `enqueue` |
| 降低内存 | `format(RGB_565)` | `bitmapConfig(RGB_565)` |
| 跳过内存缓存 | `skipMemoryCache(true)` | `memoryCachePolicy(DISABLED)` |
| 仅缓存原始数据 | `diskCacheStrategy(DATA)` | `diskCachePolicy(ENABLED)` |
| 缩略图 | `.thumbnail(0.3f)` | `.size(100, 100)` 先加载小图 |
| 暂停/恢复 | `pauseRequests()` / `resumeRequests()` | 取消协程 Job / 重新 enqueue |

---

> 📌 **总结**：Glide 是 Android 图片加载的事实标准，功能全面、社区成熟；Coil 是 Kotlin 时代的新选择，架构更现代、体积更轻。理解它们的缓存机制、生命周期绑定和 [[内存优化]] 策略，是 Android 高级开发的必备技能。
> 
> 延伸阅读：[[Bitmap与图片压缩]] · [[RecyclerView性能优化]] · [[OkHttp与Retrofit]] · [[Jetpack Compose]]

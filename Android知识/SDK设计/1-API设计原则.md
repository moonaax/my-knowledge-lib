# API 设计原则（易用性、向后兼容）

> SDK 的公开 API 是与开发者签订的"契约"。一旦发布，每一个公开方法、每一个参数类型都成为承诺。好的 API 让正确的用法显而易见，错误的用法难以编译通过；差的 API 则让调用者在文档和源码之间反复横跳。本文聚焦 **易用性** 与 **向后兼容** 两大主题，结合 Kotlin/Java SDK 实践展开。

相关主题：[[6-版本管理与发布]] · [[4-混淆与ProGuard规则]] · [[模块化设计]]

---

## 一、核心概念

### 1.1 好 API 的三个标准

| 标准 | 含义 | 反面案例 |
|------|------|----------|
| **简洁（Minimal）** | 公开面尽可能小，一个功能只有一种显而易见的调用方式 | 同一功能暴露 `loadData()` / `fetchData()` / `getData()` 三个方法 |
| **一致（Consistent）** | 命名、参数顺序、返回值风格在整个 SDK 内保持统一 | 有的方法返回 `Result<T>`，有的直接抛异常，有的返回 `null` |
| **可发现（Discoverable）** | 开发者通过 IDE 自动补全就能猜到 API 的存在和用法 | 核心功能藏在 `Utils.misc.internal.Helper` 里 |

**简洁性的量化指标**：公开类数量、公开方法数量、必填参数个数。每次 PR 都应关注这些数字的增量。

````java
// ✅ 简洁：一个入口，链式配置
HcClient client = HcClient.builder()
    .appId("demo")
    .timeout(Duration.ofSeconds(30))
    .build();

// ❌ 冗余：多个入口，职责不清
HcConfig config = new HcConfig();
config.setAppId("demo");
HcClientFactory factory = new HcClientFactory(config);
HcClient client = factory.create("default");
````
### 1.2 向后兼容的三个层次

向后兼容（Backward Compatibility）不是一个笼统的概念，它至少包含三个层次：

**① 源码兼容（Source Compatibility）**
升级 SDK 版本后，调用方代码 **无需修改即可编译通过**。这是最基本的要求。

````java
// v1.0: Data load(String url)
// v1.1: Data load(String url, int timeout)  ← 源码兼容 ✅（提供重载）
// 旧代码 load("https://...") 依然编译通过
````
**② 二进制兼容（Binary Compatibility）**
升级 SDK 版本后，调用方 **无需重新编译**，已有的 class/dex 文件可以直接链接新版本运行。这对 Android AAR 分发尤为关键。

````java
// v1.0 编译产物中调用签名: load(Ljava/lang/String;)LData;
// v1.1 如果方法签名变为: load(Ljava/lang/String;I)LData;
// → 二进制不兼容 ❌，旧 class 找不到原签名方法
// 解决：手动保留旧签名重载
````
**③ 行为兼容（Behavioral Compatibility）**
升级后 API 的 **可观测行为** 保持一致。最隐蔽也最容易被忽视。

````java
// v1.0: fetchUsers() 返回按 id 升序排列的列表
// v1.1: fetchUsers() 内部改用 HashMap，返回顺序不确定
// → 行为不兼容 ❌，依赖排序的调用方会出 bug
````
> 三个层次的严格程度：行为兼容 > 二进制兼容 > 源码兼容。SDK 应至少保证 **同一主版本内的二进制兼容**。

### 1.3 语义化版本 SemVer

遵循 [SemVer 2.0](https://semver.org/) 规范，版本号格式 `MAJOR.MINOR.PATCH`：

| 变更类型 | 版本位 | 示例 | 说明 |
|----------|--------|------|------|
| 破坏性变更 | MAJOR | 1.x → 2.0.0 | 删除/重命名公开 API |
| 新增功能（兼容） | MINOR | 1.1 → 1.2.0 | 新增方法、新增可选参数 |
| Bug 修复 | PATCH | 1.2.0 → 1.2.1 | 内部实现修复 |

与 [[6-版本管理与发布]] 配合，在 CI 中自动校验：MINOR 版本升级不允许出现二进制不兼容变更。

````kotlin
// build.gradle.kts 中集成 japicmp
tasks.register<me.champeau.gradle.japicmp.JapicmpTask>("checkBinaryCompat") {
    oldClasspath.from(files("libs/sdk-1.2.0.jar"))
    newClasspath.from(tasks.jar)
    onlyBinaryIncompatibleModified = true
    failOnModification = true // MINOR 升级时开启
}
````
---

## 二、原理与实践

### 2.1 最小化公开 API（internal / 封装）

**原则：没有明确理由公开的，一律不公开。** 公开容易，收回极难——每个 `public` 都是未来的负债。

Kotlin 提供了 `internal` 可见性，限制在同一模块内可见：

````kotlin
// ✅ 只暴露必要接口
public interface HcClient {
    suspend fun fetchUser(id: String): User
    fun close()
}

// 实现类对外不可见
internal class HcClientImpl(
    private val httpEngine: HttpEngine,
    private val config: HcConfig
) : HcClient {
    override suspend fun fetchUser(id: String): User {
        return httpEngine.get("/users/$id").decode()
    }
    override fun close() = httpEngine.shutdown()
}

// 工厂方法作为唯一入口
public fun HcClient(block: HcClientBuilder.() -> Unit): HcClient {
    return HcClientBuilder().apply(block).build()
}
````
**注意 `internal` 的局限性**：Kotlin 的 `internal` 编译为 Java 的 `public`（方法名被 mangle），Java 调用者仍可通过反射或直接调用 mangle 后的方法名访问。配合 [[4-混淆与ProGuard规则]] 可进一步隐藏。

````
# proguard-rules.pro
-keep public class com.hc.sdk.HcClient { public *; }
# internal 类不 keep，混淆后外部无法稳定引用
````
### 2.2 Builder 模式 vs DSL 设计

两种模式都用于构建复杂配置对象，选择取决于目标语言：

**Builder 模式（Java 友好）**：

````java
public class HcClientBuilder {
    private String appId = "";
    private Duration timeout = Duration.ofSeconds(30);
    private RetryPolicy retryPolicy = RetryPolicy.Default;

    public HcClientBuilder appId(String value) { this.appId = value; return this; }
    public HcClientBuilder timeout(Duration value) { this.timeout = value; return this; }
    public HcClientBuilder retryPolicy(RetryPolicy value) { this.retryPolicy = value; return this; }

    public HcClient build() {
        if (appId == null || appId.isBlank()) {
            throw new IllegalArgumentException("appId must not be blank");
        }
        return new HcClientImpl(appId, timeout, retryPolicy);
    }
}

// Java 调用
// HcClient client = new HcClientBuilder()
//     .appId("demo")
//     .timeout(Duration.ofSeconds(30))
//     .build();
````
**DSL 模式（Kotlin 优先）**：

````kotlin
public fun HcClient(block: HcClientConfig.() -> Unit): HcClient {
    val config = HcClientConfig().apply(block)
    require(config.appId.isNotBlank()) { "appId must not be blank" }
    return HcClientImpl(config)
}

@HcDsl
public class HcClientConfig {
    var appId: String = ""
    var timeout: Duration = 30.seconds
    var retryPolicy: RetryPolicy = RetryPolicy.Default

    fun retry(block: RetryPolicyBuilder.() -> Unit) {
        retryPolicy = RetryPolicyBuilder().apply(block).build()
    }
}

// Kotlin 调用
val client = HcClient {
    appId = "demo"
    timeout = 10.seconds
    retry {
        maxAttempts = 3
        backoff = exponential(base = 1.seconds)
    }
}
````
**如何兼顾两种语言？** 同时提供 Builder 和 DSL 入口：

````kotlin
// Kotlin 用户用 DSL
public fun HcClient(block: HcClientConfig.() -> Unit): HcClient

// Java 用户用 Builder
public class HcClient {
    companion object {
        @JvmStatic fun builder(): HcClientBuilder = HcClientBuilder()
    }
}
````
### 2.3 @Deprecated 迁移策略

API 演进不可避免，`@Deprecated` 是平滑过渡的关键工具。Kotlin 的 `@Deprecated` 比 Java 更强大，支持 `ReplaceWith` 自动迁移：

````kotlin
@Deprecated(
    message = "Use fetchUser(id) instead. Will be removed in 3.0.",
    replaceWith = ReplaceWith("fetchUser(id)"),
    level = DeprecationLevel.WARNING  // WARNING → ERROR → HIDDEN
)
public fun getUser(id: String): User = fetchUser(id)

public fun fetchUser(id: String): User { /* ... */ }
````
**三阶段迁移策略**：

````
v2.1: 新增 fetchUser()，标记 getUser() 为 WARNING
      → 调用方看到警告，IDE 提供一键替换
v2.3: 将 getUser() 升级为 ERROR
      → 调用方必须迁移，否则编译失败
v3.0: 将 getUser() 升级为 HIDDEN 或直接删除
      → 彻底移除，MAJOR 版本允许破坏性变更
````
**保持二进制兼容的技巧**：即使标记为 `HIDDEN`，方法仍存在于字节码中，旧的二进制产物仍可调用。只有物理删除才会破坏二进制兼容。

### 2.4 @JvmOverloads 与 Java 互操作

Kotlin 的默认参数是源码级特性，Java 看不到。`@JvmOverloads` 让编译器生成多个重载方法：

````kotlin
@JvmOverloads
fun search(
    query: String,
    page: Int = 1,
    pageSize: Int = 20,
    sortBy: String = "relevance"
): SearchResult { /* ... */ }

// 编译器生成的 Java 重载：
// search(String)
// search(String, int)
// search(String, int, int)
// search(String, int, int, String)
````
**⚠️ @JvmOverloads 的二进制兼容陷阱**：

````kotlin
// v1.0
@JvmOverloads
fun search(query: String, page: Int = 1): SearchResult

// 生成: search(String), search(String, int)

// v1.1 —— 在中间插入参数
@JvmOverloads
fun search(query: String, filter: Filter = Filter.NONE, page: Int = 1): SearchResult

// 生成: search(String), search(String, Filter), search(String, Filter, int)
// ❌ 旧的 search(String, int) 签名消失了！二进制不兼容！
````
**最佳实践**：新参数只追加在末尾，或手动编写重载而非依赖 `@JvmOverloads`。

### 2.5 默认参数与扩展函数

默认参数和扩展函数是 Kotlin API 简洁性的两大利器，但在 SDK 设计中需要注意边界。

**默认参数——减少重载，提升可读性**：

````kotlin
public suspend fun HcClient.uploadFile(
    path: String,
    contentType: String = "application/octet-stream",
    onProgress: ((Float) -> Unit)? = null
): UploadResult {
    // ...
}

// 调用方只需关心必填参数
client.uploadFile("/data/report.pdf")
// 或按需指定
client.uploadFile("/data/img.png", contentType = "image/png") {
    progress -> updateUI(progress)
}
````
**扩展函数——在不修改核心接口的前提下增加便捷方法**：

````kotlin
// 核心接口保持精简
public interface HcClient {
    suspend fun execute(request: Request): Response
}

// 便捷方法通过扩展提供，不增加接口的实现负担
public suspend inline fun <reified T> HcClient.get(
    url: String,
    params: Map<String, String> = emptyMap()
): T {
    val request = Request.get(url, params)
    return execute(request).decode<T>()
}

public suspend inline fun <reified T> HcClient.post(
    url: String,
    body: Any
): T {
    val request = Request.post(url, body)
    return execute(request).decode<T>()
}
````
**扩展函数的兼容性注意点**：
- 扩展函数编译为静态方法，移动到不同文件会改变类名 → **二进制不兼容**
- 如果后续接口新增同名成员方法，成员方法优先级高于扩展 → **行为不兼容**

````kotlin
// v1.0: 扩展函数
fun HcClient.close() { /* cleanup */ }

// v1.1: 接口新增同名方法
interface HcClient {
    fun close()  // 成员方法优先，扩展函数被"遮蔽"
}
// ⚠️ 如果两者行为不同，调用方会遇到意外行为变更
````
---

## 三、面试题

### Q1: 什么是二进制兼容？为什么对 SDK 很重要？

**答：** 二进制兼容指升级依赖库后，已编译的调用方 class 文件无需重新编译即可正常链接和运行。对 SDK 而言，使用者可能通过 Maven/Gradle 间接依赖你的库，无法控制重新编译的时机。如果 MINOR 版本升级破坏了二进制兼容，会导致 `NoSuchMethodError`、`IncompatibleClassChangeError` 等运行时崩溃。典型的二进制不兼容变更包括：删除/重命名公开方法、改变方法签名、将类从 `class` 改为 `interface`。可以使用 japicmp 或 metalava 等工具在 CI 中自动检测。

### Q2: Kotlin 的 `internal` 修饰符能否真正阻止外部访问？

**答：** 不能完全阻止。`internal` 在 Kotlin 编译后变为 Java 的 `public`，只是方法名会被 mangle（如追加模块名哈希）。Java 代码可以直接调用 mangle 后的方法名，反射也不受限制。因此 `internal` 是"君子协定"而非强制隔离。要进一步保护，需配合 ProGuard/R8 混淆（参见 [[4-混淆与ProGuard规则]]），将 internal 类的名称混淆掉，使外部无法稳定引用。此外，也可以通过将实现放在独立的 `impl` 模块中，只发布 API 模块的产物来实现物理隔离。

### Q3: 设计一个 SDK 初始化 API，如何同时兼顾 Kotlin 和 Java 调用者的体验？

**答：** 推荐同时提供 DSL 风格和 Builder 风格两个入口。Kotlin 侧提供顶层函数 + lambda 配置（如 `HcClient { appId = "x" }`），Java 侧提供经典 Builder（如 `HcClient.builder().appId("x").build()`）。内部共享同一个配置类和构建逻辑，避免重复代码。关键点：Builder 的 setter 返回 `this` 以支持链式调用；DSL 配置类用 `@DslMarker` 注解防止作用域泄漏；必填参数在 `build()` 中通过 `require` 校验而非构造函数强制，以保持两种风格的一致性。

### Q4: 为什么不建议在 `@JvmOverloads` 方法的中间位置插入新的默认参数？

**答：** `@JvmOverloads` 按参数从右到左的顺序依次省略来生成重载。如果在中间插入新参数，生成的重载签名集合会发生变化，原有的某些签名会消失。例如原来有 `f(String, Int)` 的重载，插入参数后变成 `f(String, Filter, Int)`，旧的 `f(String, Int)` 签名不再生成，导致二进制不兼容。安全做法是只在参数列表末尾追加新参数，或者放弃 `@JvmOverloads`，手动编写重载方法以完全控制签名。

### Q5: 如何制定一个 API 废弃（Deprecation）计划？从标记到删除需要几个版本？

**答：** 推荐三阶段策略。第一阶段（MINOR 版本）：新增替代 API，将旧 API 标记为 `@Deprecated(level = WARNING)`，并通过 `ReplaceWith` 提供自动迁移。第二阶段（下一个 MINOR 版本）：将级别升级为 `ERROR`，强制调用方迁移。第三阶段（下一个 MAJOR 版本）：将级别设为 `HIDDEN` 或物理删除。至少跨越两个 MINOR 版本给调用方迁移窗口。在 HIDDEN 阶段，方法仍存在于字节码中，保持二进制兼容；只有在 MAJOR 版本才允许物理删除。整个过程应在 CHANGELOG 中明确记录时间线。

### Q6: 扩展函数适合用来定义 SDK 的核心 API 吗？有什么风险？

**答：** 扩展函数适合定义便捷方法和语法糖，但不适合作为核心 API 的唯一定义方式。风险有三：（1）扩展函数编译为静态方法，所在文件的类名成为 ABI 的一部分，移动文件会破坏二进制兼容；（2）如果接口后续新增同名成员方法，成员方法优先级更高，会"遮蔽"扩展函数，导致行为静默变更；（3）扩展函数无法被 `override`，不支持多态。推荐做法是核心契约用接口成员方法定义，扩展函数只用于提供基于核心 API 的组合便捷操作（如 `client.get<User>(url)` 基于 `client.execute(request)` 实现）。

---

## 四、实战与踩坑

### 4.1 API 破坏性变更的真实案例

**案例一：OkHttp 3.x → 4.x 的包名保持策略**

OkHttp 4.0 用 Kotlin 重写，但刻意保持了 `okhttp3` 包名而非改为 `okhttp4`。原因是改包名会导致所有调用方必须修改 import，这是最大规模的源码不兼容。OkHttp 团队选择牺牲包名的"语义正确性"来换取平滑迁移。

教训：**包名是 API 表面积最大的部分，改包名的代价远超改方法名。**

**案例二：Kotlin 协程 `ExperimentalCoroutinesApi` 的长期实验状态**

Kotlin 协程中 `Flow.flatMapMerge` 等 API 长期标记为 `@ExperimentalCoroutinesApi`，调用方需要 `@OptIn` 才能使用。这种策略让团队可以在不承诺稳定性的前提下收集反馈，但也导致社区抱怨"永远实验"。

教训：**实验性标记是好工具，但要有明确的毕业时间线，否则会损害开发者信任。**

**案例三：Android `AsyncTask` 的废弃**

`AsyncTask` 在 API 30 被标记为 `@Deprecated`，但由于 Android 的向后兼容承诺，它至今仍存在于 SDK 中。Google 选择不删除而是标记废弃，引导开发者迁移到协程或 `java.util.concurrent`。

教训：**对于平台级 SDK，废弃但不删除是常态。二进制兼容的权重远高于代码整洁。**

### 4.2 二进制兼容检查工具

#### japicmp（Java API Compatibility Checker）

适用于 Java/Kotlin JVM 库，对比两个 JAR 文件的公开 API 差异：

````kotlin
// build.gradle.kts
plugins {
    id("me.champeau.gradle.japicmp") version "0.4.2"
}

tasks.register<me.champeau.gradle.japicmp.JapicmpTask>("checkApiCompat") {
    oldClasspath.from(files("baseline/sdk-1.2.0.jar"))
    newClasspath.from(tasks.named("jar"))
    onlyBinaryIncompatibleModified = true
    failOnModification = true
    txtOutputFile = file("$buildDir/reports/japicmp.txt")
}
````
典型输出：

````
MODIFIED CLASS: com.hc.sdk.HcClient
  MODIFIED METHOD: search(java.lang.String, int)
    REMOVED → BINARY INCOMPATIBLE
  ADDED METHOD: search(java.lang.String, com.hc.sdk.Filter, int)
````
#### metalava（Android 官方工具）

Google 内部用于 Android SDK 的 API 兼容性检查，也适用于 Android Library：

````bash
# 生成 API 签名文件
metalava --source-path src/main/java \
         --api api/current.txt \
         --hide HiddenSuperclass

# 对比兼容性
metalava --check-compatibility:api:released api/released.txt \
         --source-path src/main/java
````
`api/current.txt` 示例：

````
// Signature format: 4.0
package com.hc.sdk {
  public interface HcClient {
    method public suspend Object? fetchUser(String id, kotlin.coroutines.Continuation<? super com.hc.sdk.User>);
    method public void close();
  }
}
````
**CI 集成建议**：在 PR 流程中，MINOR 版本分支强制运行兼容性检查，不通过则阻止合并。API 签名文件（`api/current.txt`）纳入版本控制，变更需要显式审批。详见 [[6-版本管理与发布]]。

### 4.3 Kotlin 与 Java 混合 API 的坑

#### 坑 1：`internal` 泄漏到 Java

````kotlin
// Kotlin 模块
internal class SecretEngine {
    fun process(data: ByteArray): ByteArray = /* ... */
}
````
编译后 Java 可见签名：`public final class SecretEngine$sdk_release { ... }`

Java 调用方虽然看到奇怪的类名，但技术上可以调用。**解决方案**：在 ProGuard 中不 keep internal 类，混淆后类名变为 `a.b.c`，无法稳定引用。

#### 坑 2：Kotlin 属性的 getter/setter 命名

````kotlin
// Kotlin
class Config {
    var isEnabled: Boolean = true   // getter: isEnabled(), setter: setEnabled(boolean)
    var hasHeader: Boolean = false  // getter: getHasHeader() ← 注意不是 hasHeader()!
}
````
`is` 前缀的布尔属性，getter 保持 `isXxx()`；但 `has` 前缀不会被特殊处理，getter 变成 `getHasXxx()`。Java 调用者会感到困惑。

**解决方案**：使用 `@get:JvmName` 显式控制：

````kotlin
class Config {
    @get:JvmName("hasHeader")
    var hasHeader: Boolean = false
}
````
#### 坑 3：协程函数对 Java 的不友好

````kotlin
// Kotlin suspend 函数
interface HcClient {
    suspend fun fetchUser(id: String): User
}

// Java 看到的签名：
// Object fetchUser(String id, Continuation<? super User> cont)
// → Java 调用者完全无法正常使用
````
**解决方案**：为 Java 调用者提供 `CompletableFuture` 桥接：

````kotlin
// 扩展函数，放在单独的 -jvm 模块中
@JvmName("fetchUserAsync")
fun HcClient.fetchUserFuture(id: String): CompletableFuture<User> {
    val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())
    return scope.future { fetchUser(id) }
}
````
#### 坑 4：Kotlin 默认参数对 Java 不可见

````kotlin
fun createRequest(
    url: String,
    method: String = "GET",
    headers: Map<String, String> = emptyMap()
): Request { /* ... */ }

// Java 只能看到: createRequest(String, String, Map)
// 无法省略 method 和 headers
````
**解决方案**：添加 `@JvmOverloads`，或在需要精确控制签名时手动编写重载。

#### 坑 5：`Nothing` 类型与 Java 互操作

````kotlin
fun fail(message: String): Nothing = throw IllegalStateException(message)

// Java 看到返回类型为 Void，但实际永远抛异常
// Java 编译器不知道这个方法永远不会正常返回，无法做流程分析优化
````
这类工具方法建议仅在 Kotlin 内部使用，不作为公开 API 暴露给 Java 调用者。

---

## 总结

| 设计决策 | 推荐做法 |
|----------|----------|
| 可见性 | 默认 `internal`/`private`，显式选择 `public` |
| 配置 API | Kotlin DSL + Java Builder 双入口 |
| 新增参数 | 追加在末尾，配合默认值 |
| 废弃 API | WARNING → ERROR → HIDDEN，跨 2+ MINOR 版本 |
| 兼容性检查 | CI 集成 japicmp/metalava，签名文件入库 |
| Java 互操作 | `@JvmOverloads` / `@JvmStatic` / `@JvmName` 显式标注 |
| 协程 API | Kotlin 用 suspend，Java 用 `CompletableFuture` 桥接 |

> API 设计的终极目标：让调用者写出的代码看起来像是语言内置的功能，而非在"使用某个库"。

相关主题：[[6-版本管理与发布]] · [[4-混淆与ProGuard规则]] · [[模块化设计]] · [[Kotlin 协程在 SDK 中的应用]]

# 混淆与 ProGuard 规则

> 本文系统梳理 Android 混淆体系，涵盖 ProGuard / R8 核心概念、keep 规则语法、SDK 发布混淆最佳实践及常见踩坑，帮助开发者在 [[5-AAR与APK打包]] 流程中正确配置混淆，同时兼顾 [[3-最小化依赖与体积控制]]。

---

## 一、核心概念

### 1.1 ProGuard vs R8

| 维度 | ProGuard | R8 |
|------|----------|----|
| 维护方 | GuardSquare（第三方） | Google（官方） |
| 集成方式 | AGP 3.3 以前默认 | AGP 3.4+ 默认，完全替代 ProGuard |
| 输入 | `.class` → `.class` | `.class` / `.dex` → `.dex`（直接出 dex） |
| 性能 | 多步串行，编译较慢 | 单 pass 流水线，编译更快 |
| 兼容性 | 使用 ProGuard 规则文件 | **完全兼容** ProGuard 规则语法 |
| 特有能力 | — | 支持 Kotlin metadata 处理、更激进的优化 |

> R8 在 AGP 7.0+ 已无法关闭（`android.enableR8=false` 被移除），因此本文以 R8 为主线讲解，但规则语法与 ProGuard 通用。

### 1.2 四大步骤

混淆工具的处理流水线包含四个阶段：

````
┌──────────┐    ┌──────────┐    ┌───────────┐    ┌────────────┐
│  Shrink  │───▶│ Optimize │───▶│ Obfuscate │───▶│ Preverify  │
│  压缩    │    │  优化    │    │  混淆     │    │  预校验    │
└──────────┘    └──────────┘    └───────────┘    └────────────┘
````
1. **Shrink（压缩/树摇）**：移除未被引用的类、字段、方法。这是体积优化的核心步骤，与 [[3-最小化依赖与体积控制]] 直接相关。
2. **Optimize（优化）**：内联短方法、移除无用分支、常量折叠等字节码级优化。
3. **Obfuscate（混淆）**：将类名、方法名、字段名重命名为 `a`、`b`、`c` 等短名称，增加逆向难度。
4. **Preverify（预校验）**：为 Java 6+ 的 class 文件添加预校验信息。R8 直接输出 dex，此步骤已内化。

在 `build.gradle.kts` 中的开关：

````kotlin
android {
    buildTypes {
        release {
            // 开启压缩（shrink）+ 混淆（obfuscate）
            isMinifyEnabled = true
            // 开启资源压缩（需配合 isMinifyEnabled）
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
````
> `proguard-android-optimize.txt` 包含优化步骤；若使用 `proguard-android.txt` 则默认关闭 optimize。

### 1.3 keep 规则语法

keep 规则是混淆配置的核心，语法结构如下：

````
-keep[,modifier,...] class_specification
````
#### 六大 keep 指令对比

| 指令 | 防止被移除（Shrink） | 防止被重命名（Obfuscate） |
|------|:---:|:---:|
| `-keep` | ✅ | ✅ |
| `-keepnames` | ❌ | ✅ |
| `-keepclassmembers` | ✅（仅成员） | ✅（仅成员） |
| `-keepclassmembernames` | ❌ | ✅（仅成员） |
| `-keepclasseswithmembers` | ✅（类+匹配成员） | ✅（类+匹配成员） |
| `-keepclasseswithmembernames` | ❌ | ✅（类+匹配成员） |

#### 通配符说明

````proguard
# * 匹配类名中除包分隔符外的任意部分
-keep class com.example.sdk.api.* { *; }

# ** 匹配任意类名（含子包）
-keep class com.example.sdk.api.** { *; }

# <init> 匹配所有构造方法
-keepclassmembers class * {
    <init>(...);
}

# <fields> 匹配所有字段，<methods> 匹配所有方法
-keepclassmembers class * implements java.io.Serializable {
    <fields>;
}

# *** 匹配任意类型（含基本类型和数组）
-keepclassmembers class * {
    public *** get*();
    public void set*(***);
}
````
#### 常用修饰符

````proguard
# allowshrinking：允许被压缩移除，但如果保留则不混淆
-keep,allowshrinking class com.example.sdk.Optional { *; }

# allowobfuscation：允许被混淆，但不允许被移除
-keep,allowobfuscation class com.example.sdk.Internal { *; }

# includedescriptorclasses：同时保留方法签名中引用的类
-keep,includedescriptorclasses class com.example.sdk.Api {
    public *;
}
````
---

## 二、原理与实践

### 2.1 R8 的工作流程

````
输入(.class/.jar/.aar)
  → 合并 ProGuard 规则(app + library consumer + default)
  → 构建调用图，确定 Entry Points
  → Tree Shaking（移除不可达代码）
  → Optimization（内联、常量传播、分支消除）
  → Obfuscation（重命名标识符）
  → Desugaring + Dex 转换 → 输出 .dex
````
关键点：R8 将 desugaring（脱糖，如 Java 8 lambda 转换）与混淆合并到同一 pass 中，避免了 D8 + ProGuard 的两次遍历，显著提升编译速度。

### 2.2 consumerProguardFiles 的作用

在 SDK（Library Module）开发中，`consumerProguardFiles` 是最重要的混淆配置入口：

````kotlin
// SDK 模块的 build.gradle.kts
android {
    defaultConfig {
        // 这些规则会打包进 AAR，被 App 消费时自动合并
        consumerProguardFiles("consumer-rules.pro")
    }
}
````
**工作机制：**

````
┌─────────────────┐     ┌──────────────────────┐
│  SDK Module      │     │  App Module           │
│                  │     │                       │
│ consumer-rules   │────▶│  自动合并到 App 的     │
│ .pro             │     │  混淆规则中            │
│ (打包进 AAR)     │     │                       │
└─────────────────┘     └──────────────────────┘
````
- `proguardFiles`：仅在**当前模块**编译时生效（Library 模块自身不做混淆，所以这个对 Library 意义不大）
- `consumerProguardFiles`：规则文件会被打包进 [[5-AAR与APK打包]] 产物的 `proguard.txt` 中，App 集成时 R8 自动读取

**SDK 的 consumer-rules.pro 示例：**

````proguard
# ===== SDK 公开 API 保护 =====
# 保留所有 public/protected 的类和成员
-keep public class com.example.sdk.api.** {
    public protected *;
}

# 保留 SDK 初始化入口
-keep class com.example.sdk.SdkInitializer {
    public static *** getInstance();
    public void init(android.content.Context);
}

# 保留回调接口
-keep interface com.example.sdk.callback.** { *; }

# 保留数据模型（用于 JSON 序列化）
-keep class com.example.sdk.model.** {
    <fields>;
    <init>(...);
}
````
### 2.3 常见 keep 规则模板

#### 模板一：序列化类（Gson / Moshi / Kotlinx.serialization）

````proguard
# Gson 通用规则
-keepattributes Signature
-keepattributes *Annotation*

# 保留使用 @SerializedName 注解的字段
-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}

# 保留泛型签名（TypeToken 需要）
-keep class com.google.gson.reflect.TypeToken { *; }
-keep class * extends com.google.gson.reflect.TypeToken

# SDK 数据模型 —— 推荐直接 keep 整个包
-keep class com.example.sdk.model.** {
    <fields>;
    <init>();
}
````
#### 模板二：反射调用

````proguard
# 通过反射访问的类必须保留类名和被反射的成员
-keep class com.example.sdk.internal.ReflectTarget {
    # 保留被反射调用的特定方法
    public void targetMethod(java.lang.String);
    # 保留被反射读取的字段
    public java.lang.String targetField;
}

# 如果使用注解标记反射目标，可以用通配规则
-keep @com.example.sdk.annotation.Reflectable class * { *; }
````
#### 模板三：JNI / Native 方法

````proguard
# native 方法名必须与 C/C++ 侧一致，不能被混淆
-keepclasseswithmembernames class * {
    native <methods>;
}

# 如果 JNI 回调 Java 方法，也需要保留
-keep class com.example.sdk.jni.NativeBridge {
    # C++ 侧通过 JNI 调用的方法
    void onNativeCallback(int, java.lang.String);
}
````
#### 模板四：枚举

````proguard
# 枚举的 values() 和 valueOf() 通过反射调用，必须保留
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
````
#### 模板五：Parcelable

````proguard
-keepclassmembers class * implements android.os.Parcelable {
    public static final ** CREATOR;
    <fields>;
}
````
### 2.4 -printmapping 与 mapping 文件

````proguard
# 在 proguard-rules.pro 中指定输出路径
-printmapping build/outputs/mapping/release/mapping.txt
````
mapping 文件记录了混淆前后的名称映射关系：

````
# mapping.txt 示例
com.example.sdk.api.UserService -> a.b.c:
    java.lang.String getUserName() -> a
    void setUserName(java.lang.String) -> b
    int userId -> a
    java.lang.String userName -> b
com.example.sdk.model.UserInfo -> a.b.d:
    1:1:void <init>():15:15 -> <init>
    2:5:java.lang.String getName():20:23 -> a
````
**mapping 文件的用途：**

1. **崩溃日志还原**：使用 `retrace` 工具将混淆后的堆栈还原为原始类名/方法名

````bash
# R8 retrace（AGP 自带）
java -jar retrace.jar mapping.txt stacktrace.txt

# 或使用 Android Studio: Build → Analyze APK → 加载 mapping
````
2. **增量混淆**：通过 `-applymapping` 复用上一版本的 mapping，保持混淆名称稳定

````proguard
# 增量发布时复用旧 mapping（谨慎使用）
-applymapping previous-mapping.txt
````
3. **SDK 发布必须保留**：每次发布 SDK 版本时，mapping 文件应归档保存，便于排查线上问题。

---

## 三、面试题

### Q1：ProGuard 和 R8 有什么区别？为什么 Google 要用 R8 替代 ProGuard？

**A：** 核心区别有三点：

1. **流水线合并**：ProGuard 输出 `.class`，还需要 D8 再转 `.dex`，是两步操作；R8 将 shrink/optimize/obfuscate 与 dex 转换合并为单一 pass，编译速度更快。
2. **更好的 Kotlin 支持**：R8 能理解 Kotlin metadata，避免因混淆 metadata 导致 Kotlin 反射失败。
3. **更激进的优化**：R8 支持 class merging（合并小类）、enum unboxing（枚举拆箱为 int）等 ProGuard 不具备的优化。

Google 替代 ProGuard 的根本原因是**掌控 Android 工具链的完整性**，减少对第三方的依赖，同时提升编译性能。

### Q2：`-keep` 和 `-keepclassmembers` 有什么区别？

**A：**

````
-keep class A { void foo(); }
````
→ 类 A **不会被移除**，也**不会被重命名**；方法 `foo()` 同理。

````
-keepclassmembers class A { void foo(); }
````
→ **不保护类 A 本身**。如果 A 因为其他原因被保留（比如被引用），则 `foo()` 不会被移除和重命名。但如果 A 整个类都不可达，A 和 `foo()` 都会被移除。

简单记忆：`-keep` 保护类 + 成员，`-keepclassmembers` 只保护成员（前提是类本身存活）。

### Q3：SDK 开发中，混淆规则应该放在 `proguardFiles` 还是 `consumerProguardFiles`？

**A：** SDK（Library Module）应该使用 `consumerProguardFiles`。原因：

- Library Module 自身编译时**不执行混淆**（混淆只在 App 最终打包时统一执行）
- `consumerProguardFiles` 指定的规则文件会被打包进 AAR 的 `proguard.txt`
- 当 App 集成该 AAR 时，R8 会自动合并这些规则
- 这样 SDK 使用者**无需手动添加混淆规则**，降低接入成本

````kotlin
// ✅ 正确：SDK 模块
android {
    defaultConfig {
        consumerProguardFiles("consumer-rules.pro")
    }
}

// ❌ 错误：这只在 Library 自身编译时生效，对 App 无效
android {
    buildTypes {
        release {
            proguardFiles("proguard-rules.pro")
        }
    }
}
````
### Q4：为什么枚举类需要特殊的 keep 规则？

**A：** Java 枚举在编译后会生成 `values()` 和 `valueOf(String)` 两个静态方法。JVM 在以下场景通过**反射**调用它们：

- `switch` 语句编译后使用 `values()` 构建跳转表
- 序列化/反序列化时通过 `valueOf()` 还原枚举实例
- `EnumSet`、`EnumMap` 内部依赖 `values()`

如果这些方法被混淆或移除，运行时会抛出 `NoSuchMethodException`。标准规则：

````proguard
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
````
> R8 默认规则文件 `proguard-android-optimize.txt` 已包含此规则，但 SDK 的 consumer-rules 中建议显式声明，避免依赖 App 侧配置。

### Q5：什么是 `-printusage` 和 `-whyareyoukeeping`？如何用它们调试混淆问题？

**A：**

- **`-printusage unused.txt`**：输出所有被 Tree Shaking 移除的类和成员，用于检查是否有不该被移除的代码。
- **`-whyareyoukeeping class com.example.Foo`**：打印 R8 保留某个类的原因链（哪条 keep 规则或哪个引用路径导致它被保留），用于排查"为什么包体积没减小"。

````proguard
# 调试时临时添加
-printusage build/outputs/usage.txt
-whyareyoukeeping class com.example.sdk.internal.HeavyModule
````
实际调试流程：

````java
// 1. 发现某个内部类没被移除，包体积偏大
// 2. 添加 -whyareyoukeeping 规则
// 3. 编译后查看输出，发现是某条过于宽泛的 keep 规则导致
// 4. 收窄 keep 规则范围，重新编译验证
````
### Q6：R8 Full Mode 是什么？和默认模式有什么区别？

**A：** R8 Full Mode（AGP 8.0+ 默认开启）比兼容模式更激进：

| 行为 | 兼容模式（Compat） | Full Mode |
|------|:---:|:---:|
| 保留 `<init>()` 无参构造 | 默认保留 | 可能移除 |
| 保留未使用的 `-keep` 目标 | 保留 | 可能移除 |
| 枚举 `valueOf` 返回类型 | 保持原类型 | 可能优化为基类 |
| `-keepattributes` 默认行为 | 宽松 | 严格 |

开启/关闭方式（`gradle.properties`）：

````properties
# AGP 8.0+ 默认 true
android.enableR8.fullMode=true
````
Full Mode 下需要更精确的 keep 规则，否则容易出现运行时崩溃。SDK 开发者应在 Full Mode 下充分测试。

---

## 四、实战与踩坑

### 4.1 SDK 发布时的混淆最佳实践

#### 原则：SDK 自己保护自己，不给使用者添麻烦

````kotlin
// SDK 模块 build.gradle.kts 完整配置
android {
    defaultConfig {
        // ✅ 核心：SDK 自带混淆规则，打包进 AAR
        consumerProguardFiles("consumer-rules.pro")
    }

    buildTypes {
        release {
            // SDK Library 自身不开启混淆
            // 混淆由 App 最终打包时统一执行
            isMinifyEnabled = false
        }
    }
}
````
**consumer-rules.pro 编写清单：**

````proguard
# ========================================
# SDK Consumer ProGuard Rules
# ========================================

# 1. 公开 API 类 —— 必须保留类名和公开成员
-keep public class com.example.sdk.** {
    public *;
    protected *;
}

# 2. 公开接口（回调、监听器）
-keep public interface com.example.sdk.callback.** { *; }

# 3. 数据模型（JSON 序列化）
-keep class com.example.sdk.model.** {
    <fields>;
    <init>();
    <init>(...);
}

# 4. 枚举
-keepclassmembers enum com.example.sdk.** {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# 5. Native 方法
-keepclasseswithmembernames class com.example.sdk.** {
    native <methods>;
}

# 6. 保留注解和泛型签名
-keepattributes Signature,*Annotation*,InnerClasses,EnclosingMethod

# 7. Parcelable
-keepclassmembers class com.example.sdk.** implements android.os.Parcelable {
    public static final ** CREATOR;
}
````
#### SDK 混淆验证流程

1. 在 SDK 项目中创建 sample app 模块，开启 `isMinifyEnabled = true`
2. 用 release 构建 sample app，运行所有 SDK 功能
3. 确认无 `ClassNotFoundException` / `NoSuchMethodException`
4. 使用 APK Analyzer 检查：内部类已混淆，公开 API 类名保留

### 4.2 Gson/Moshi 序列化混淆坑

#### 问题现象

````kotlin
// SDK 数据模型
data class UserInfo(
    val userId: String,
    val userName: String,
    val avatarUrl: String
)

// 使用 Gson 反序列化
val user = Gson().fromJson(json, UserInfo::class.java)
// 混淆后：user.userId == null, user.userName == null ❌
````
#### 根因分析

Gson 默认通过**字段名**匹配 JSON key。混淆后字段名变成 `a`、`b`、`c`，与 JSON 中的 `"userId"` 不匹配，导致反序列化失败（字段为 null 或默认值）。

````
混淆前：userId   → JSON key "userId"    ✅ 匹配
混淆后：a        → JSON key "userId"    ❌ 不匹配
````
#### 解决方案

**方案一：keep 整个 model 包（推荐 SDK 使用）**

````proguard
-keep class com.example.sdk.model.** {
    <fields>;
    <init>(...);
}
````
**方案二：使用 @SerializedName 注解（字段级精确控制）**

````kotlin
data class UserInfo(
    @SerializedName("userId")
    val userId: String,
    @SerializedName("userName")
    val userName: String,
    @SerializedName("avatarUrl")
    val avatarUrl: String
)
````
配合规则：

````proguard
-keepclassmembers,allowobfuscation class * {
    @com.google.gson.annotations.SerializedName <fields>;
}
````
> 方案二允许类名被混淆，只保留带注解的字段名，体积更优。但 SDK 场景下方案一更安全。

**方案三：Moshi + @JsonClass（推荐新项目）**

````kotlin
@JsonClass(generateAdapter = true)
data class UserInfo(
    @Json(name = "userId")
    val userId: String,
    @Json(name = "userName")
    val userName: String
)
````
Moshi 的 codegen 模式在编译期生成 Adapter，**不依赖反射**，天然抗混淆。只需保留 `@JsonClass` 注解：

````proguard
-keep @com.squareup.moshi.JsonClass class * { *; }
````
### 4.3 Kotlin data class 混淆

Kotlin data class 编译后会生成以下方法：

````
- componentN() 方法（解构声明）
- copy() 方法
- toString() / hashCode() / equals()
- 无参构造（如果所有参数有默认值）
````
#### 坑点一：copy() 方法参数名丢失

````kotlin
// 混淆前
val updated = user.copy(userName = "newName")

// 混淆后，如果通过反射调用 copy()，参数名变成 a, b, c
// 命名参数语法在编译期解析，直接调用不受影响
// 但如果通过 KClass.memberFunctions 反射调用 copy()，会出问题
````
#### 坑点二：Kotlin 反射 + data class

````kotlin
// 使用 Kotlin 反射获取属性
val props = UserInfo::class.memberProperties
props.forEach { prop ->
    // 混淆后 prop.name 变成 "a", "b"
    println("${prop.name} = ${prop.get(user)}")
}
````
#### 解决方案

````proguard
# 如果 SDK 内部使用 Kotlin 反射操作 data class
# 必须保留 Kotlin Metadata 注解
-keep class kotlin.Metadata { *; }

# 保留需要反射的 data class
-keep class com.example.sdk.model.** {
    <fields>;
    <init>(...);
    # 保留 componentN 和 copy
    *** component*();
    *** copy(...);
}
````
> 如果 SDK 不使用 Kotlin 反射，只是普通的 data class 作为数据载体 + Gson 序列化，只需保留字段和构造方法即可。

### 4.4 反射调用被混淆的排查

#### 典型报错

````
java.lang.NoSuchMethodException: com.example.sdk.a.b.c()
    at java.lang.Class.getMethod(Class.java:2072)
    at com.example.sdk.a.d.invoke(Unknown Source:12)
````
#### 排查步骤

**Step 1：用 mapping 文件还原堆栈**

````bash
# 使用 R8 retrace
cd $ANDROID_HOME/tools/proguard/bin
./retrace.sh mapping.txt obfuscated_stacktrace.txt
````
还原结果：

````
java.lang.NoSuchMethodException: com.example.sdk.internal.PluginLoader.loadPlugin()
    at java.lang.Class.getMethod(Class.java:2072)
    at com.example.sdk.internal.PluginManager.invoke(PluginManager.kt:45)
````
**Step 2：定位反射调用点**

````java
// PluginManager.java:45
public class PluginManager {
    public void invoke(String className, String methodName) throws Exception {
        // 🔴 问题：className 和 methodName 是硬编码的原始名称
        // 但目标类已被混淆，名称不匹配
        Class<?> clazz = Class.forName(className);
        Method method = clazz.getMethod(methodName);
        method.invoke(null);
    }
}
````
**Step 3：添加 keep 规则**

````proguard
# 方案 A：保留被反射调用的目标类
-keep class com.example.sdk.internal.PluginLoader {
    public void loadPlugin();
}

# 方案 B：使用自定义注解标记 + 通配规则（更可维护）
-keep @com.example.sdk.annotation.KeepForReflection class * { *; }
````
**Step 4：自定义注解方案（推荐）**

````java
// 定义注解
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD})
public @interface KeepForReflection {}

// 使用注解标记
@KeepForReflection
public class PluginLoader {
    @KeepForReflection
    public void loadPlugin() {
        // ...
    }
}
````
对应 ProGuard 规则：

````proguard
-keep @com.example.sdk.annotation.KeepForReflection class * {
    @com.example.sdk.annotation.KeepForReflection *;
}
````
#### 预防措施

- 自定义 `@KeepForReflection` 注解 — 代码即文档
- CI 中开启混淆构建 + 自动化测试
- 使用 `-printusage` 检查移除列表
- mapping 文件每版本归档
- 避免字符串硬编码类名，用 `Foo::class.java.name` 替代

---

## 参考与延伸

- [[3-最小化依赖与体积控制]] — 混淆的 shrink 步骤是体积优化的核心手段
- [[5-AAR与APK打包]] — consumerProguardFiles 如何随 AAR 分发
- [R8 官方文档](https://developer.android.com/build/shrink-code)
- [ProGuard 规则语法参考](https://www.guardsquare.com/manual/configuration/usage)

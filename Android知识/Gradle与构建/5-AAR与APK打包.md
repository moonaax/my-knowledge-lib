# AAR 与 APK 打包

## 一、什么是 APK？

APK（Android Package）就是 Android 应用的安装包，类似于 Windows 上的 `.exe` 文件。你写的所有代码、图片、布局文件，最终都会被打包成一个 `.apk` 文件，用户安装的就是它。

APK 本质上就是一个 **zip 压缩包**，你把任何 `.apk` 文件后缀改成 `.zip`，就能用解压软件打开看里面的内容。

---

## 二、APK 里面有什么？

把一个 APK 解压后，你会看到这些东西：

````
my-app.apk（解压后）
├── classes.dex          ← 你写的所有代码，编译后的产物
├── classes2.dex         ← 如果方法数超过 65536，会有多个 dex
├── resources.arsc       ← 资源映射表（记录资源 ID 和实际文件的对应关系）
├── res/                 ← 资源文件（图片、布局 XML 等）
│   ├── layout/
│   ├── drawable/
│   └── ...
├── assets/              ← 原始资源（不会被编译，原样打包）
├── lib/                 ← so 动态库（C/C++ 编译的）
│   ├── armeabi-v7a/
│   ├── arm64-v8a/
│   └── x86_64/
├── AndroidManifest.xml  ← 应用清单（已编译为二进制格式）
├── META-INF/            ← 签名信息
│   ├── MANIFEST.MF
│   ├── CERT.SF
│   └── CERT.RSA
└── kotlin/              ← Kotlin metadata（如果用了 Kotlin）
````
### 逐个解释

**classes.dex**
- 你写的 Java/Kotlin 代码最终都变成了这个文件
- Android 不能直接运行 `.class` 文件（那是 JVM 的格式），需要转成 `.dex`（Dalvik Executable）
- `.dex` 是专门为 Android 虚拟机（Dalvik / ART）设计的格式，更紧凑、更省内存

**resources.arsc**
- 这是一张"查找表"
- 你在代码里写 `R.drawable.icon`，系统就是通过这张表找到对应的图片文件
- 它记录了所有资源 ID → 实际文件路径的映射

**res/ 目录**
- 存放编译后的资源文件
- 注意：XML 文件（布局、动画等）已经被编译成二进制格式，不是原始文本了
- 图片（png、jpg、webp）保持原样

**assets/ 目录**
- 和 `res/` 的区别：assets 里的文件**完全不会被编译**，原样打包
- 通过 `AssetManager` 访问，不会生成资源 ID
- 适合放字体文件、预置数据库、HTML 等

**lib/ 目录**
- 存放 `.so` 文件（C/C++ 编译的动态链接库）
- 按 CPU 架构分目录：`armeabi-v7a`（32位ARM）、`arm64-v8a`（64位ARM）、`x86_64`（模拟器）
- 系统会根据设备架构自动选择对应目录的 so

**AndroidManifest.xml**
- 应用的"身份证"，声明了包名、权限、四大组件等
- APK 里的是二进制格式，不能直接阅读（需要用 `aapt2 dump` 查看）

**META-INF/**
- 签名相关文件
- 系统安装 APK 时会校验签名，确保没被篡改

---

## 三、APK 打包完整流程

从你点击 Android Studio 的 "Build" 按钮开始，到生成最终 APK，经历了以下步骤：

````
┌─────────────────────────────────────────────────────┐
│  第1步：AAPT2 编译资源                                │
│  res/ 下的 XML、图片等 → 编译为二进制 .flat 文件        │
│  同时生成 R.java（资源 ID 常量类）                      │
├─────────────────────────────────────────────────────┤
│  第2步：AAPT2 链接                                    │
│  合并所有 .flat 文件 → 生成 resources.arsc             │
│  合并所有 AndroidManifest.xml（主模块 + 库模块）        │
├─────────────────────────────────────────────────────┤
│  第3步：编译源码                                       │
│  .java → javac → .class                              │
│  .kt → kotlinc → .class                              │
├─────────────────────────────────────────────────────┤
│  第4步：字节码处理（可选）                               │
│  Transform API / ASM 字节码插桩                        │
│  比如：埋点、日志注入、路由表生成                         │
├─────────────────────────────────────────────────────┤
│  第5步：D8 / R8 编译                                  │
│  .class → .dex                                       │
│  D8：只做转换                                         │
│  R8：转换 + 混淆 + 优化 + 树摇（去掉没用的代码）         │
├─────────────────────────────────────────────────────┤
│  第6步：打包                                          │
│  把 dex + resources.arsc + res/ + assets/ + lib/     │
│  → 合并为一个未签名的 APK                              │
├─────────────────────────────────────────────────────┤
│  第7步：签名                                          │
│  用 keystore 对 APK 进行数字签名                       │
├─────────────────────────────────────────────────────┤
│  第8步：zipalign 对齐                                 │
│  将未压缩数据按 4 字节对齐                              │
│  → 最终可安装的 APK                                   │
└─────────────────────────────────────────────────────┘
````
### 3.1 AAPT2 是什么？

AAPT2（Android Asset Packaging Tool 2）是 Android 的资源编译工具。

**它做了两件事：**

**编译阶段（Compile）：**
- 把每个资源文件单独编译成 `.flat` 二进制格式
- 比如 `activity_main.xml` → `activity_main.xml.flat`
- 图片不需要编译，但也会被处理（比如 PNG 优化）

**链接阶段（Link）：**
- 把所有 `.flat` 文件合并
- 生成 `resources.arsc`（资源查找表）
- 生成 `R.java`（每个资源对应一个 int 常量）
- 合并来自不同模块的 `AndroidManifest.xml`

````java
// R.java 长这样（自动生成的）
public final class R {
    public static final class drawable {
        public static final int icon = 0x7f020001;
        public static final int background = 0x7f020002;
    }
    public static final class layout {
        public static final int activity_main = 0x7f030001;
    }
}
````
你在代码里写 `R.layout.activity_main`，实际上就是在用这个 int 值去 `resources.arsc` 里查找对应的文件。

### 3.2 D8 和 R8 的区别

这是面试高频题，一定要搞清楚。

**D8（默认的 dex 编译器）：**
- 职责单一：把 `.class` 文件转成 `.dex` 文件
- 同时做"脱糖"（desugaring）：把 Java 8+ 的语法（Lambda、Stream 等）转成低版本兼容的代码
- 不做任何优化和混淆

**R8（D8 的升级版，现在是默认）：**
- 包含 D8 的所有功能
- 额外做了 ProGuard 的工作：
  - **混淆（Obfuscation）**：把类名、方法名改成 a、b、c，防止反编译
  - **优化（Optimization）**：内联方法、删除无用分支等
  - **树摇（Tree Shaking）**：删除没有被引用的代码
  - **资源缩减**：配合 `shrinkResources` 删除未使用的资源

````groovy
// build.gradle 中开启 R8
android {
    buildTypes {
        release {
            minifyEnabled true      // 开启代码混淆和优化
            shrinkResources true    // 开启资源缩减
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'),
                    'proguard-rules.pro'
        }
    }
}
````
### 3.3 签名机制详解

Android 要求所有 APK 必须签名才能安装。签名的作用：
- **验证身份**：证明这个 APK 确实是你发布的
- **防篡改**：如果 APK 被修改，签名校验会失败
- **升级校验**：新版本必须和旧版本使用相同的签名才能覆盖安装

| 签名方案 | 最低版本 | 原理 | 特点 |
|---------|---------|------|------|
| v1 | 所有版本 | 对每个文件单独计算摘要，存在 META-INF/ | 可以在 META-INF/ 中添加文件而不破坏签名（安全漏洞） |
| v2 | Android 7.0 | 对整个 APK 文件计算签名 | 任何修改都会导致签名失败，安装更快（不需要解压校验） |
| v3 | Android 9.0 | 在 v2 基础上支持密钥轮换 | 可以更换签名密钥而不影响升级 |
| v4 | Android 11 | 基于 Merkle 哈希树 | 支持 ADB 增量安装，开发调试更快 |

````bash
# 生成 keystore（只需要做一次）
keytool -genkey -v -keystore my-release.keystore \
    -alias my-alias -keyalg RSA -keysize 2048 -validity 10000

# 签名 APK
apksigner sign \
    --ks my-release.keystore \
    --ks-key-alias my-alias \
    --out app-signed.apk \
    app-unsigned.apk

# 验证签名
apksigner verify -v --print-certs app-signed.apk
````
### 3.4 zipalign 是什么？

zipalign 是一个对齐工具，它把 APK 中**未压缩的数据**按 4 字节边界对齐。

**为什么要对齐？**
- 对齐后，系统可以用 `mmap`（内存映射）直接读取文件，不需要把整个文件拷贝到内存
- 减少运行时内存占用
- 提高读取速度

````bash
# 对齐命令（签名之前执行）
zipalign -v 4 input.apk output.apk

# 检查是否已对齐
zipalign -c -v 4 app.apk
````
> 注意：如果用 v2+ 签名，zipalign 必须在签名**之前**执行，因为 v2 签名覆盖整个文件，之后的任何修改都会破坏签名。

### 3.5 MultiDex（多 dex）

**问题背景：**
- 单个 `.dex` 文件最多只能包含 **65536 个方法**（因为方法索引用的是 16 位，2^16 = 65536）
- 一个稍大的项目加上第三方库，很容易超过这个限制

**解决方案：**
- 把代码拆分成多个 dex 文件：`classes.dex`、`classes2.dex`、`classes3.dex`...

**不同 Android 版本的处理：**

| 版本 | 虚拟机 | 支持情况 |
|------|--------|---------|
| 5.0 以下 | Dalvik | 默认只加载 classes.dex，需要 MultiDex 库手动加载其他 dex |
| 5.0 及以上 | ART | 安装时把所有 dex 合并编译为 OAT 文件，原生支持 |

````groovy
// 5.0 以下需要的配置
android {
    defaultConfig {
        multiDexEnabled true
    }
}

dependencies {
    implementation 'androidx.multidex:multidex:2.0.1'
}
````
````java
// Application 中初始化（5.0 以下需要）
public class MyApp extends MultiDexApplication {
    // 或者手动调用：
    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
    }
}
````
---

## 四、什么是 AAR？

AAR（Android Archive）是 Android Library 的打包格式。

**通俗理解：**
- APK 是给用户安装的完整应用
- AAR 是给开发者用的"半成品"，是一个可复用的 Android 库
- 你做组件开发，最终产出的就是 AAR

AAR 本质也是一个 **zip 压缩包**，改后缀为 `.zip` 就能解压。

---

## 五、AAR 里面有什么？

````
my-library.aar（解压后）
├── classes.jar          ← 编译后的 Java/Kotlin 字节码（注意是 .jar 不是 .dex）
├── res/                 ← 资源文件
├── R.txt                ← 资源符号表（资源 ID 列表）
├── public.txt           ← 公开资源列表（可选）
├── AndroidManifest.xml  ← 库的清单文件（文本格式，不是二进制）
├── proguard.txt         ← 消费者混淆规则
├── libs/                ← 依赖的第三方 jar
├── jni/                 ← so 动态库
├── assets/              ← 原始资源
└── lint.jar             ← 自定义 Lint 规则（可选）
````
### AAR vs JAR vs APK 对比

| 对比项 | APK | AAR | JAR |
|--------|-----|-----|-----|
| 用途 | 安装到设备 | Android 库分发 | Java 库分发 |
| 包含代码 | .dex | .jar（.class） | .class |
| 包含资源 | ✅ | ✅ | ❌ |
| 包含 so | ✅ | ✅ | ❌ |
| AndroidManifest | ✅（二进制） | ✅（文本） | ❌ |
| 混淆规则 | 已混淆 | 提供给消费者 | ❌ |
| 需要签名 | ✅ | ❌ | ❌ |
| 可直接运行 | ✅ | ❌ | ❌ |

### 为什么 AAR 里是 .jar 而不是 .dex？

因为 AAR 是"中间产物"，不是最终产物：
- AAR 会被其他项目依赖，最终和主项目的代码一起编译成 `.dex`
- 如果 AAR 里就是 `.dex`，就没法和主项目的代码一起做混淆、优化了

---

## 六、AAR 的打包流程

````
Library 模块的源码 + 资源
        ↓
AAPT2 编译资源 → R.txt
        ↓
javac/kotlinc → .class → classes.jar
        ↓
收集 res/、assets/、jni/、AndroidManifest.xml
        ↓
打包为 .aar（zip 格式）
````
和 APK 的区别：
- **不做 dex 转换**（保留 .class 格式）
- **不签名**
- **不做 zipalign**
- **资源不合并**（留给消费者合并）

---

## 七、AAR 的依赖与发布

### 7.1 本地依赖 AAR

**方式一：放在 libs 目录**
````groovy
// settings.gradle 或 build.gradle
dependencies {
    implementation files('libs/my-library.aar')
}
````
**方式二：flatDir（推荐本地调试用）**
````groovy
// 项目根 build.gradle
allprojects {
    repositories {
        flatDir { dirs 'libs' }
    }
}

// 模块 build.gradle
dependencies {
    implementation(name: 'my-library', ext: 'aar')
}
````
### 7.2 发布到 Maven 仓库

这是正式的分发方式，其他项目通过 GAV 坐标（groupId:artifactId:version）引用。

````groovy
// library 模块的 build.gradle
plugins {
    id 'com.android.library'
    id 'maven-publish'
}

android {
    // ...
    publishing {
        singleVariant("release") {
            withSourcesJar()    // 附带源码
            withJavadocJar()    // 附带文档
        }
    }
}

afterEvaluate {
    publishing {
        publications {
            release(MavenPublication) {
                from components.release
                groupId = 'com.example'
                artifactId = 'my-component'
                version = '1.0.0'
            }
        }
        repositories {
            maven {
                url = "https://your-maven-repo.com/releases"
                credentials {
                    username = project.findProperty("mavenUser") ?: ""
                    password = project.findProperty("mavenPassword") ?: ""
                }
            }
        }
    }
}
````
发布后，其他项目这样引用：
````groovy
dependencies {
    implementation 'com.example:my-component:1.0.0'
}
````
### 7.3 传递依赖问题（重要！）

这是做组件开发必须理解的概念。

假设你的 AAR 库依赖了 OkHttp：

````groovy
// 你的库的 build.gradle
dependencies {
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'  // implementation
}
````
**问题：** 使用你的库的 App 项目，能不能直接用 OkHttp 的类？

**答案：不能！**

因为 `implementation` 是"内部依赖"，不会暴露给消费者。

**解决方案：**

| 声明方式 | 是否传递 | 使用场景 |
|---------|---------|---------|
| `implementation` | ❌ 不传递 | 库内部使用的依赖，不暴露给消费者 |
| `api` | ✅ 传递 | 库的公开 API 中用到的依赖，消费者也需要 |
| `compileOnly` | ❌ 不传递 | 编译时需要，运行时由宿主提供 |
| `runtimeOnly` | ✅ 传递 | 编译时不需要，运行时需要 |

````groovy
// 如果你的库的公开接口返回了 OkHttp 的类型，就要用 api
dependencies {
    api 'com.squareup.okhttp3:okhttp:4.12.0'       // 消费者也能用
    implementation 'com.google.code.gson:gson:2.10'  // 消费者用不到
}
````
**发布到 Maven 后的依赖传递：**
- 发布时会生成 `.pom` 文件，记录了依赖关系
- `api` 依赖 → pom 中标记为 `compile` scope → 消费者自动获得
- `implementation` 依赖 → pom 中标记为 `runtime` scope → 消费者运行时有，但编译时不可见

---

## 八、AAR 中的混淆策略

### 两种混淆规则文件

| 文件 | 作用 | 生效时机 |
|------|------|---------|
| `proguardFiles` | 库自身编译时的混淆规则 | 库模块打 release 包时 |
| `consumerProguardFiles` | 给消费者的混淆规则 | 消费者打 release 包时自动应用 |

````groovy
android {
    buildTypes {
        release {
            minifyEnabled false  // 库模块通常不自己混淆！
            consumerProguardFiles 'consumer-rules.pro'  // 这个会打进 AAR
        }
    }
}
````
### 为什么库模块通常不自己混淆？

- 如果库自己混淆了，消费者就没法对库的代码做进一步优化
- 混淆应该在最终打 APK 时统一做，这样 R8 能看到全部代码，优化效果最好
- 库只需要通过 `consumerProguardFiles` 告诉消费者"哪些类不能混淆"

### consumer-rules.pro 示例

````proguard
# 保留公开 API（消费者需要调用的类和方法）
-keep public class com.example.sdk.MySDK { public *; }
-keep public interface com.example.sdk.Callback { *; }

# 保留注解（反射需要）
-keepattributes *Annotation*

# 保留泛型签名（Gson、Retrofit 等序列化框架需要）
-keepattributes Signature

# 保留枚举（混淆后枚举的 valueOf 会出问题）
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
````
---

## 九、资源冲突与合并

当 App 依赖多个 AAR 时，资源可能冲突（比如两个库都有 `R.string.app_name`）。

### 合并规则
- **同名资源**：App 模块的优先级最高，会覆盖库的资源
- **多个库冲突**：按依赖顺序，后声明的覆盖先声明的
- **Manifest 合并**：自动合并，冲突时需要用 `tools:replace` 等标记解决

### 避免冲突的最佳实践

````groovy
android {
    // 给库的资源加前缀，避免和其他库冲突
    resourcePrefix 'mylib_'
}
````
加了这个配置后，如果你的资源名不以 `mylib_` 开头，IDE 会警告你。

````xml
<!-- 好的命名 -->
<string name="mylib_title">标题</string>
<color name="mylib_primary">#FF0000</color>

<!-- 不好的命名（容易和其他库冲突） -->
<string name="title">标题</string>
````
---

## 十、常见面试题

### Q1：描述一下 APK 的打包流程？
> 资源编译（AAPT2）→ 源码编译（javac/kotlinc）→ 字节码处理（Transform/ASM）→ dex 编译（D8/R8）→ 打包 → 签名 → zipalign。重点讲 AAPT2 的编译和链接两个阶段，以及 R8 做了混淆+优化+树摇。

### Q2：D8 和 R8 有什么区别？
> D8 只做 class → dex 的转换和脱糖。R8 = D8 + ProGuard，额外做混淆、代码优化、树摇（删除未使用代码）。现在 R8 是默认的，已经替代了 ProGuard。

### Q3：APK 签名 v1 和 v2 的区别？
> v1 对每个文件单独签名，存在 META-INF 目录，可以在该目录添加文件而不破坏签名（安全隐患）。v2 对整个 APK 文件签名，任何修改都会导致校验失败，更安全，安装也更快。

### Q4：AAR 和 JAR 的区别？
> JAR 只包含 .class 字节码，适合纯 Java 库。AAR 除了代码还包含 Android 资源（res、assets）、AndroidManifest、so 库、混淆规则等，是 Android Library 的标准分发格式。

### Q5：为什么 AAR 里是 .jar 而不是 .dex？
> 因为 AAR 是中间产物，最终要和 App 的代码一起编译成 dex。保留 .class 格式可以让 R8 在最终打包时统一做混淆和优化，效果更好。

### Q6：implementation 和 api 的区别？对 AAR 发布有什么影响？
> implementation 的依赖不会暴露给消费者（编译隔离），api 的依赖会传递。发布到 Maven 时，implementation 对应 runtime scope，api 对应 compile scope。库的公开接口用到的类型应该用 api 声明。

### Q7：为什么库模块不建议自己做混淆？
> 库自己混淆后，消费者的 R8 就无法对库代码做进一步优化（比如内联、树摇）。应该通过 consumerProguardFiles 提供 keep 规则，让消费者在最终打包时统一混淆。

### Q8：MultiDex 是怎么回事？
> 单个 dex 方法数上限 65536（16位索引）。超过后需要拆分为多个 dex。Android 5.0+ 的 ART 原生支持多 dex（安装时合并编译）。5.0 以下的 Dalvik 需要 MultiDex 库在 Application.attachBaseContext 中手动加载。

### Q9：zipalign 的作用是什么？为什么要在签名之前执行？
> zipalign 将未压缩数据按 4 字节对齐，使系统可以用 mmap 直接读取，减少内存拷贝。必须在 v2 签名之前执行，因为 v2 签名覆盖整个文件，之后的任何修改都会破坏签名。

### Q10：如何避免多个 AAR 之间的资源冲突？
> 使用 `resourcePrefix` 给资源加前缀。合并时 App 模块优先级最高。Manifest 冲突用 `tools:replace` 解决。

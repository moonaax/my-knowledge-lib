# 自定义 Gradle Plugin

> 系统梳理 Gradle Plugin 的核心概念、实现方式、发布流程与实战踩坑。
> 关联：[[1-Gradle生命周期与Task]] · [[4-Transform-API与ASM字节码插桩]] · [[5-AAR与APK打包]]

---

## 一、核心概念

### 1.1 Plugin 的作用

Gradle Plugin 本质是**可复用的构建逻辑封装**，核心职责：

- **封装 Task**：将一组相关 Task 打包，apply 即可使用
- **扩展 DSL**：通过 Extension 向 `build.gradle` 注入自定义配置块
- **统一规范**：多模块项目中强制统一编译版本、依赖、代码检查等
- **字节码处理**：结合 [[4-Transform-API与ASM字节码插桩]] 实现编译期插桩

Plugin 在 Configuration 阶段被 apply，注册的 Task 在 Execution 阶段执行，详见 [[1-Gradle生命周期与Task]]。

### 1.2 三种实现方式

| 方式 | 位置 | 适用场景 | 可复用 |
|------|------|----------|--------|
| build.gradle 内联 | 当前模块 | 快速验证 | ❌ 仅当前模块 |
| buildSrc | 项目根目录 `buildSrc/` | 多模块共享 | ⚠️ 仅当前项目 |
| 独立项目 | 独立 module/仓库 | 跨项目复用、发布 Maven | ✅ 完全复用 |

#### 方式一：build.gradle 内联

```groovy
class GreetingPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.task('hello') {
            doLast { println "Hello from GreetingPlugin!" }
        }
    }
}
apply plugin: GreetingPlugin
```

优点：零配置。缺点：无法跨模块复用。

#### 方式二：buildSrc

Gradle 自动编译 `buildSrc/` 并加入构建 classpath：

```
buildSrc/
├── build.gradle          // plugins { id 'groovy' }
└── src/main/groovy/com/example/VersionPlugin.groovy
```

```groovy
package com.example

class VersionPlugin implements Plugin<Project> {
    void apply(Project project) {
        project.ext.set("minSdkVersion", 21)
        project.ext.set("compileSdkVersion", 34)
    }
}
```

优点：IDE 支持好。缺点：`buildSrc` 改动导致**全项目**重新配置。

#### 方式三：独立项目（推荐）

将 Plugin 作为独立 module，发布到 Maven 后通过 `plugins {}` 引入，后续章节详细展开。

### 1.3 Extension 配置

Extension 是 Plugin 向外暴露的 DSL 配置入口：

```groovy
// 使用者视角
myPlugin {
    appName = "HelloApp"
    enableLog = true
}
```

Plugin 中通过 `project.extensions.create()` 注册：

```groovy
class MyExtension {
    String appName = "default"
    boolean enableLog = false
}

class MyPlugin implements Plugin<Project> {
    void apply(Project project) {
        def ext = project.extensions.create("myPlugin", MyExtension)
        project.afterEvaluate {
            println "appName=${ext.appName}, enableLog=${ext.enableLog}"
        }
    }
}
```

> ⚠️ Extension 值在 Configuration 阶段结束后才确定，Task 中读取需用 `afterEvaluate` 或 `Property<T>` 惰性 API。

---

## 二、原理与实践

### 2.1 Plugin 接口实现

所有 Plugin 实现 `org.gradle.api.Plugin<T>`，泛型通常为 `Project`。

**Groovy 实现：**

```groovy
class AuditPlugin implements Plugin<Project> {
    void apply(Project project) {
        def ext = project.extensions.create("audit", AuditExtension)
        project.tasks.register("auditReport", AuditTask) {
            group = "audit"
            description = "Generate audit report"
            moduleName.set(ext.moduleName)
            verbose.set(ext.verbose)
            outputDir.set(project.layout.buildDirectory.dir("audit"))
        }
    }
}
```

**Java 实现：**

```java
public class AuditPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        AuditExtension ext = project.getExtensions()
                .create("audit", AuditExtension.class);
        project.getTasks().register("auditReport", AuditTask.class, task -> {
            task.setGroup("audit");
            task.getModuleName().set(ext.getModuleName());
            task.getVerbose().set(ext.getVerbose());
            task.getOutputDir().set(project.getLayout().getBuildDirectory().dir("audit"));
        });
    }
}
```

> 💡 `tasks.register()` 是惰性注册，Task 只在需要执行时才实例化，优于 `tasks.create()`。

### 2.2 Extension：Property API（推荐）

```groovy
abstract class AuditExtension {
    abstract Property<String> getModuleName()
    abstract Property<Boolean> getVerbose()

    AuditExtension() {
        moduleName.convention("")
        verbose.convention(false)
    }
}
```

`abstract` + `Property<T>` 的好处：Gradle 自动实例化、惰性求值、可直接与 Task `@Input` 关联。

嵌套 Extension 示例：

```groovy
abstract class MyPluginExtension {
    abstract Property<String> getAppName()
    abstract ServerExtension getServer()
}

// 使用者配置
myPlugin {
    appName = "demo"
    server { host = "localhost"; port = 8080 }
}
```

### 2.3 自定义 Task

```groovy
abstract class AuditTask extends DefaultTask {
    @Input abstract Property<String> getModuleName()
    @Input abstract Property<Boolean> getVerbose()
    @OutputDirectory abstract DirectoryProperty getOutputDir()

    @TaskAction
    void execute() {
        def dir = outputDir.get().asFile
        dir.mkdirs()
        new File(dir, "report.txt").text =
            "Module: ${moduleName.get()}\nTime: ${new Date()}"
        if (verbose.get()) logger.lifecycle("[Audit] Report → ${dir}")
    }
}
```

`@Input` / `@OutputDirectory` 注解让 Gradle 实现**增量构建**：输入不变时跳过执行。

### 2.4 发布到 Maven

**Plugin ID 映射（推荐 java-gradle-plugin）：**

```groovy
plugins {
    id 'groovy'
    id 'java-gradle-plugin'
    id 'maven-publish'
}

group = 'com.example'
version = '1.0.0'

gradlePlugin {
    plugins {
        audit {
            id = 'com.example.audit'
            implementationClass = 'com.example.AuditPlugin'
        }
    }
}

publishing {
    repositories {
        maven { url = uri("${rootProject.projectDir}/local-repo") }
    }
}
```

执行 `./gradlew publish` 后，消费端使用：

```groovy
// settings.gradle
pluginManagement {
    repositories {
        maven { url uri("../my-plugin/local-repo") }
        gradlePluginPortal()
    }
}

// app/build.gradle
plugins {
    id 'com.example.audit' version '1.0.0'
}
audit { moduleName = "app"; verbose = true }
```

> ⚠️ 发布到远程 Maven 时，凭证应放在 `~/.gradle/gradle.properties` 或环境变量中，不要硬编码。

### 2.5 Convention Plugin（多模块公共配置）

Convention Plugin 是官方推荐的多模块配置统一方案，替代 `subprojects {}` 写法。

**传统方式的问题：**

```groovy
// ❌ 反模式：所有子模块被强制统一，不够灵活
subprojects {
    apply plugin: 'java'
    sourceCompatibility = JavaVersion.VERSION_17
}
```

**Convention Plugin 实现（Composite Build）：**

```
build-logic/
├── settings.gradle
└── conventions/
    ├── build.gradle
    └── src/main/groovy/
        ├── android-lib-conventions.gradle
        └── java-conventions.gradle
```

`build-logic/conventions/build.gradle`：

```groovy
plugins { id 'groovy-gradle-plugin' }
dependencies { implementation 'com.android.tools.build:gradle:8.2.0' }
```

`java-conventions.gradle`（文件名即 Plugin ID）：

```groovy
plugins { id 'java' }
java {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}
tasks.withType(JavaCompile).configureEach { options.encoding = 'UTF-8' }
```

根 `settings.gradle`：

```groovy
pluginManagement { includeBuild("build-logic") }
include ':app', ':lib-network', ':lib-utils'
```

子模块按需引用：

```groovy
// lib-utils/build.gradle
plugins { id 'java-conventions' }
```

优势：模块**主动选择**约定，而非被动继承；配置可测试、可版本化。

---

## 三、面试题

### Q1：三种 Plugin 实现方式如何选择？

**A：** 快速验证用内联；项目内多模块共享用 buildSrc（或 Composite Build）；跨项目复用或发布用独立项目。buildSrc 改动会触发全量重配置，大型项目建议迁移到 Composite Build。

### Q2：为什么要用 `afterEvaluate` 读取 Extension？有更好的方式吗？

**A：** Extension 值在 Configuration 阶段由用户脚本赋值，`apply()` 中直接读取可能拿到默认值。`afterEvaluate` 在脚本执行完毕后回调。更好的方式是使用 `Property<T>` 惰性 API：

```groovy
// Property 是 Provider，执行时才求值，无需 afterEvaluate
project.tasks.register("greet") {
    def name = ext.appName  // Provider
    doLast { println "Hello ${name.get()}" }
}
```

### Q3：`tasks.register()` 和 `tasks.create()` 的区别？

**A：** `create()` 立即创建实例，即使不执行也有开销；`register()` 惰性注册，按需实例化。大型项目中 `register()` 可显著减少 Configuration 阶段耗时，属于 Configuration Avoidance API。

### Q4：如何在 Plugin 中获取 Android 的 applicationVariants？

**A：** 需在 `afterEvaluate` 中访问（Variant 在 Configuration 末尾生成），或使用 AGP 7.0+ 的新 Variant API：

```groovy
def androidComponents = project.extensions.getByType(
    com.android.build.api.variant.AndroidComponentsExtension)
androidComponents.onVariants(androidComponents.selector().all()) { variant ->
    println "Variant: ${variant.name}"
}
```

### Q5：Convention Plugin 和 `subprojects {}` 的区别？

**A：** `subprojects` 是隐式耦合——子模块不知道被施加了什么配置；Convention Plugin 让每个模块在 `plugins {}` 中显式声明约定，更灵活、可维护、可测试。Gradle 官方推荐 Convention Plugin。

### Q6：如何调试自定义 Plugin？

**A：** 三种方式：
1. **日志**：`project.logger.lifecycle()` 输出关键信息
2. **Remote Debug**：`./gradlew task -Dorg.gradle.debug=true --no-daemon`，IDE Attach 到 5005 端口
3. **TestKit**：Gradle 官方测试框架，模拟完整构建：

```groovy
def result = GradleRunner.create()
    .withProjectDir(testProjectDir)
    .withArguments('auditReport')
    .withPluginClasspath()
    .build()
assert result.output.contains("[Audit]")
```

---

## 四、实战与踩坑

### 4.1 buildSrc 改动导致全量重编译

**现象：** 修改 `buildSrc` 任意文件，所有模块重新配置。

**解决：** 迁移到 Composite Build（`includeBuild`），只有被修改的模块重新编译：

```groovy
// settings.gradle
pluginManagement { includeBuild("build-logic") }
```

### 4.2 Extension 读取时机错误

**现象：** 读取 Extension 值始终为 null。

**修复：** 使用 `Property<T>` API，Task 中通过 Provider 惰性读取，无需 `afterEvaluate`。

### 4.3 Plugin ID 找不到

**排查清单：**
1. `META-INF/gradle-plugins/<plugin-id>.properties` 文件名与 ID 完全一致
2. properties 内容：`implementation-class=com.example.MyPlugin`（全限定名）
3. 使用 `java-gradle-plugin` 可自动生成，避免手动出错
4. 确认 jar 已发布且消费端 `pluginManagement` 可达

### 4.4 Plugin 与 AGP 版本不兼容

**现象：** 升级 AGP 后报 `NoSuchMethodError`。

**最佳实践：**
- 避免使用 `com.android.build.gradle.internal.*` 内部 API
- 优先使用 Variant API（`AndroidComponentsExtension`）
- AGP 依赖声明为 `compileOnly`，避免版本冲突：

```groovy
dependencies {
    compileOnly 'com.android.tools.build:gradle-api:8.2.0'
}
```

### 4.5 完整项目模板

```
my-audit-plugin/
├── build.gradle
├── src/main/groovy/com/example/audit/
│   ├── AuditPlugin.groovy
│   ├── AuditExtension.groovy
│   └── AuditTask.groovy
└── src/main/resources/META-INF/gradle-plugins/
    └── com.example.audit.properties
```

### 4.6 Plugin 开发 Checklist

| 检查项 | 说明 |
|--------|------|
| ✅ `tasks.register()` | 惰性注册，避免不必要实例化 |
| ✅ `Property<T>` Extension | 惰性求值，避免配置顺序问题 |
| ✅ AGP `compileOnly` | 避免与消费端版本冲突 |
| ✅ 不用 AGP internal API | 防止升级崩溃 |
| ✅ `java-gradle-plugin` | 自动生成 Plugin Marker |
| ✅ TestKit 测试 | 保证 Plugin 行为正确 |
| ✅ Composite Build | 替代 buildSrc，避免全量重配置 |
| ✅ 凭证外置 | `gradle.properties` 或环境变量 |

---

> 📌 延伸阅读：[[1-Gradle生命周期与Task]] · [[4-Transform-API与ASM字节码插桩]] · [[5-AAR与APK打包]]

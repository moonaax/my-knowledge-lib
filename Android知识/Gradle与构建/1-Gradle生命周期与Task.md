# Gradle 生命周期与 Task

> Gradle 构建的核心在于理解其三阶段生命周期和 Task 的执行模型。本文从概念、原理、面试、实战四个维度全面梳理。
> 相关笔记：[[3-自定义Plugin]] · [[5-AAR与APK打包]] · [[Gradle依赖管理]]

---

## 一、核心概念

### 1.1 Gradle 三大阶段

Gradle 的每一次构建都严格经历三个阶段，顺序不可跳过：

| 阶段 | 英文 | 核心动作 | 关键产物 |
|------|------|---------|---------|
| 初始化 | Initialization | 解析 `settings.gradle`，确定哪些项目参与构建 | `Project` 实例树 |
| 配置 | Configuration | 执行所有参与项目的 `build.gradle`，构建 Task DAG | Task 有向无环图 |
| 执行 | Execution | 按 DAG 拓扑序执行被选中的 Task 及其依赖 | 构建产物（APK/AAR/JAR 等） |

````
settings.gradle          build.gradle (所有项目)        选中的 Task 链
┌──────────────┐      ┌─────────────────────┐      ┌──────────────────┐
│ Initialization │ ──▶ │   Configuration     │ ──▶ │    Execution     │
└──────────────┘      └─────────────────────┘      └──────────────────┘
   确定项目集合            构建 Task DAG              执行 Task Action
````
**常见误区**：`build.gradle` 中直接写的代码（非 `doFirst`/`doLast` 块）在**配置阶段**就会执行，而不是执行阶段。这是新手最容易踩的坑。

````groovy
// build.gradle
task myTask {
    // ⚠️ 这里是配置阶段执行的代码
    println "配置阶段: 我每次 sync 都会打印"

    doLast {
        // ✅ 这里才是执行阶段的代码
        println "执行阶段: 只有运行 myTask 才会打印"
    }
}
````
### 1.2 Project vs Task

**Project** 是 Gradle 构建的组织单元，对应一个 `build.gradle` 文件：

````
rootProject (settings.gradle)
├── app          (build.gradle)  → Project ':app'
├── lib-network  (build.gradle)  → Project ':lib-network'
└── lib-common   (build.gradle)  → Project ':lib-common'
````
**Task** 是 Gradle 构建的最小执行单元，每个 Project 包含若干 Task：

````groovy
// 查看某个项目的所有 Task
./gradlew :app:tasks --all
````
两者关系：
- 一个 `Settings` 对象 → 多个 `Project`
- 一个 `Project` → 多个 `Task`
- 一个 `Task` → 多个 `Action`（doFirst / doLast 添加的闭包）

````groovy
// settings.gradle
include ':app', ':lib-network', ':lib-common'

// app/build.gradle
project.tasks.whenTaskAdded { task ->
    println "Task added: ${task.name}"
}
````
### 1.3 Task 依赖与 DAG

Gradle 在配置阶段结束后会生成一张 **DAG（有向无环图）**，节点是 Task，边是依赖关系。执行阶段按拓扑排序依次执行。

**声明依赖的方式**：

````groovy
// 方式一：dependsOn
task compile {
    doLast { println 'compiling...' }
}
task assemble {
    dependsOn compile
    doLast { println 'assembling...' }
}

// 方式二：mustRunAfter / shouldRunAfter（仅控制顺序，不触发依赖）
task unitTest {
    mustRunAfter compile
    doLast { println 'testing...' }
}

// 方式三：finalizedBy（无论成功失败都执行）
task deploy {
    finalizedBy 'cleanup'
    doLast { println 'deploying...' }
}
task cleanup {
    doLast { println 'cleaning up...' }
}
````
**dependsOn vs mustRunAfter 区别**：

| 特性 | dependsOn | mustRunAfter |
|------|-----------|-------------|
| 是否触发依赖执行 | ✅ 是 | ❌ 否 |
| 是否保证顺序 | ✅ 是 | ✅ 是 |
| 典型场景 | assemble 依赖 compile | test 在 compile 之后（但可单独运行） |

**可视化 DAG**：

````groovy
// build.gradle — 监听 DAG 构建完成
gradle.taskGraph.whenReady { graph ->
    println "=== Task DAG ==="
    graph.allTasks.each { println "  - ${it.path}" }
}
````
### 1.4 增量构建（Incremental Build）

Gradle 的增量构建机制通过 **inputs/outputs** 判断 Task 是否需要重新执行：

- 如果 Task 的 inputs 和 outputs 都没有变化 → **UP-TO-DATE**，跳过执行
- 如果任一发生变化 → 重新执行

````
第一次执行:  :compileJava  → 编译（执行）
第二次执行:  :compileJava  → UP-TO-DATE（跳过）
修改源码后:  :compileJava  → 编译（重新执行）
````
增量构建是 Gradle 性能优势的核心来源之一，配合 [[Build Cache]] 可以实现跨机器的缓存复用。

---


## 二、原理与实践

### 2.1 各阶段详细执行内容

#### 初始化阶段（Initialization）

1. Gradle 查找并执行 `init.gradle`（全局初始化脚本，位于 `~/.gradle/init.d/`）
2. 查找 `settings.gradle`（从当前目录向上搜索）
3. 执行 `settings.gradle`，创建 `Settings` 对象
4. 根据 `include` 声明创建 `Project` 实例树

````groovy
// settings.gradle
rootProject.name = 'MyApp'
include ':app', ':lib-common'

// 动态 include
file('modules').eachDir { dir ->
    if (new File(dir, 'build.gradle').exists()) {
        include ":modules:${dir.name}"
    }
}
````
#### 配置阶段（Configuration）

1. 按项目依赖顺序执行每个 `build.gradle`
2. 应用插件（`apply plugin`）、解析依赖、注册 Task
3. 构建完整的 Task DAG

````groovy
// build.gradle
apply plugin: 'com.android.application'

android {
    compileSdkVersion 34
    // DSL 块在配置阶段执行
}

task printConfig {
    description = 'Print build config'  // 配置阶段
    group = 'custom'
    doLast {
        println "Build type: ${android.buildTypes.names}"  // 执行阶段
    }
}
````
**性能提示**：每次执行任何 Task（甚至 `./gradlew help`）都经历完整配置阶段，因此配置代码应尽量轻量。

#### 执行阶段（Execution）

1. 根据命令行指定的 Task，从 DAG 中提取子图
2. 按拓扑排序执行 Task 链
3. 每个 Task 先检查 UP-TO-DATE，再决定是否执行 Action

````groovy
// 生命周期回调
gradle.settingsEvaluated { println "✅ settings.gradle 执行完毕" }
gradle.projectsLoaded    { println "✅ 所有 Project 实例已创建" }
gradle.projectsEvaluated { println "✅ 所有 build.gradle 执行完毕" }
gradle.buildFinished     { result -> println "✅ 构建${result.failure ? '失败' : '成功'}" }
````
### 2.2 自定义 Task 详解

#### doFirst / doLast

`doFirst` 和 `doLast` 向 Task 的 Action 列表头部/尾部添加闭包：

````groovy
task greet {
    doFirst { println '1. doFirst-A' }
    doFirst { println '2. doFirst-B' }  // 后添加的 doFirst 排在前面
    doLast  { println '3. doLast-A' }
    doLast  { println '4. doLast-B' }
}
// 输出顺序: 2 → 1 → 3 → 4
````
**注意**：`doFirst` 是栈结构（LIFO），`doLast` 是队列结构（FIFO）。

#### 带类型的 Task

````groovy
// 继承已有 Task 类型
task copyDocs(type: Copy) {
    from 'src/docs'
    into 'build/docs'
    include '**/*.md'
}

task cleanBuild(type: Delete) {
    delete 'build/outputs'
}

task zipRelease(type: Zip) {
    from 'build/outputs/apk/release'
    archiveFileName = "release-${version}.zip"
    destinationDirectory = file('build/distributions')
}
````
#### 自定义 Task 类（inputs/outputs）

这是实现增量构建的关键 — 通过声明 `@Input`、`@InputFiles`、`@OutputFile` 等注解：

````groovy
// buildSrc/src/main/groovy/GenerateConfigTask.groovy
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.*

abstract class GenerateConfigTask extends DefaultTask {

    @Input
    String appName

    @Input
    String versionName

    @InputFile
    File templateFile

    @OutputFile
    File outputFile

    @TaskAction
    void generate() {
        def content = templateFile.text
            .replace('{{APP_NAME}}', appName)
            .replace('{{VERSION}}', versionName)
        outputFile.text = content
        println "Generated config: ${outputFile.path}"
    }
}
````
注册并使用：

````groovy
// build.gradle
task generateConfig(type: GenerateConfigTask) {
    appName = 'MyApp'
    versionName = '2.0.0'
    templateFile = file('config/template.properties')
    outputFile = file("$buildDir/generated/config.properties")
}
````
常用注解一览：

| 注解 | 作用 |
|------|------|
| `@Input` | 简单值输入（String, int, boolean） |
| `@InputFile` / `@InputFiles` | 文件/文件集合输入 |
| `@InputDirectory` | 目录输入 |
| `@OutputFile` / `@OutputDirectory` | 文件/目录输出 |
| `@Optional` | 标记为可选 |
| `@Internal` | 不参与增量判断 |



### 2.3 Task 增量构建原理

#### 工作流程

````
Task 执行前检查:
1. 是否声明了 inputs/outputs？ → 否：每次都执行
2. outputs 是否存在？ → 否：执行（首次构建）
3. inputs 指纹与上次一致？ → 是：UP-TO-DATE 跳过 / 否：重新执行
````
#### 指纹（Fingerprint）机制

Gradle 对 inputs 的指纹基于**文件内容 hash**（非修改时间），简单值用 `equals()` 比较，文件集合则组合所有文件路径 + 内容 hash。

````groovy
// 验证增量构建
task processData {
    inputs.file 'data/input.json'
    outputs.file "$buildDir/processed/output.json"

    doLast {
        println "Processing data..."
        def input = file('data/input.json')
        def output = file("$buildDir/processed/output.json")
        output.parentFile.mkdirs()
        output.text = input.text.toUpperCase()
    }
}
````
````bash
# 第一次执行
$ ./gradlew processData
> Task :processData
Processing data...

# 第二次执行（无修改）
$ ./gradlew processData
> Task :processData UP-TO-DATE

# 修改 input.json 后
$ ./gradlew processData
> Task :processData
Processing data...
````
#### 增量 Task（Incremental Task）

对于大量文件的处理场景，可以使用 `@Incremental` + `InputChanges` 只处理变化的文件：

````groovy
abstract class IncrementalProcessTask extends DefaultTask {
    @Incremental @InputDirectory File inputDir
    @OutputDirectory File outputDir

    @TaskAction
    void execute(InputChanges inputChanges) {
        inputChanges.getFileChanges(inputDir).each { change ->
            def target = new File(outputDir, change.file.name)
            if (change.changeType == ChangeType.REMOVED) {
                target.delete()
            } else {
                target.text = change.file.text.toUpperCase()
            }
        }
    }
}
````
### 2.4 Configuration Cache

Configuration Cache 是 Gradle 7.0+ 引入的重要优化，**缓存配置阶段的结果**，后续构建直接跳过配置阶段。

#### 启用方式

````properties
# gradle.properties
org.gradle.configuration-cache=true
org.gradle.configuration-cache.max-problems=5
````
#### 工作原理

````
首次构建:
  Initialization → Configuration → [序列化 Task DAG 到缓存] → Execution

后续构建（配置未变）:
  [从缓存反序列化 Task DAG] → Execution
  ⚡ 跳过了 Initialization + Configuration！
````
#### 兼容性约束

Configuration Cache 要求 Task 不能在执行阶段引用某些对象：

````groovy
// ❌ 不兼容 Configuration Cache — 执行阶段引用 Project
task badTask {
    doLast {
        println project.name  // ❌ 执行阶段不能引用 project
    }
}

// ✅ 兼容 — 在配置阶段捕获值
task goodTask {
    def name = project.name  // ✅ 配置阶段获取
    doLast {
        println name  // ✅ 使用局部变量
    }
}
````
不能在执行阶段引用的对象：`Project`、`Gradle`、`Settings`、`TaskContainer`。

这与 [[3-自定义Plugin]] 的开发密切相关，编写 Plugin 时需要注意 Configuration Cache 兼容性。

### 2.5 Task 避免配置时间浪费

使用 `register` 替代 `create` 实现 Task 的**惰性注册**（Lazy Configuration）：

````groovy
// ❌ 旧方式：立即创建并配置（即使不执行也会消耗配置时间）
tasks.create('oldWay') {
    doLast { println 'old' }
}

// ✅ 新方式：惰性注册（只有真正需要执行时才配置）
tasks.register('newWay') {
    doLast { println 'new' }
}
````
配合 `Provider` API 实现惰性属性：

````groovy
abstract class MyTask extends DefaultTask {
    @Input
    abstract Property<String> getMessage()

    @TaskAction
    void run() {
        println message.get()
    }
}

tasks.register('lazy', MyTask) {
    // Provider — 值在需要时才计算
    message.set(provider { 
        "Computed at: ${new Date()}" 
    })
}
````
---


## 三、面试题

### Q1：Gradle 构建的三个阶段分别做了什么？配置阶段的代码和执行阶段的代码有什么区别？

**A**：

1. **初始化阶段**：解析 `settings.gradle`，确定参与构建的项目集合，创建 `Project` 实例树
2. **配置阶段**：执行所有 `build.gradle`，应用插件、声明依赖、注册 Task，构建 Task DAG
3. **执行阶段**：根据命令行目标 Task，从 DAG 提取子图，按拓扑排序执行

核心区别：配置阶段代码（Task `{ }` 块中直接写的代码）**每次构建都执行**；执行阶段代码（`doFirst`/`doLast`/`@TaskAction`）**只有 Task 被选中时才执行**。

````groovy
task example {
    println "我在配置阶段执行"     // 每次 sync/build 都打印
    doLast {
        println "我在执行阶段执行"  // 只有 ./gradlew example 才打印
    }
}
````
---

### Q2：dependsOn、mustRunAfter、finalizedBy 有什么区别？

**A**：

| 关键字 | 语义 | 是否触发依赖 | 典型场景 |
|--------|------|-------------|---------|
| `dependsOn` | A 依赖 B → 执行 A 前必须先执行 B | ✅ 是 | `assemble.dependsOn compile` |
| `mustRunAfter` | 如果 A 和 B 都要执行，B 必须在 A 之后 | ❌ 否 | `test.mustRunAfter compile` |
| `shouldRunAfter` | 同上，但允许 Gradle 在有循环时忽略 | ❌ 否 | 弱排序约束 |
| `finalizedBy` | A 执行后（无论成败）必须执行 B | ✅ 是 | `deploy.finalizedBy cleanup` |

关键区别：`dependsOn` 会**拉取**依赖 Task 到执行图中；`mustRunAfter` 只在两个 Task 都已经在执行图中时才生效。

---

### Q3：什么是增量构建？如何让自定义 Task 支持增量构建？

**A**：

增量构建是 Gradle 通过比较 Task 的 inputs/outputs 指纹来判断是否需要重新执行的机制。如果 inputs 和 outputs 都没有变化，Task 标记为 `UP-TO-DATE` 并跳过。

让自定义 Task 支持增量构建的方法：
1. 继承 `DefaultTask`
2. 用 `@Input`、`@InputFile`、`@OutputFile` 等注解声明输入输出
3. 用 `@TaskAction` 标注执行方法

````groovy
abstract class MyTask extends DefaultTask {
    @InputFile File source
    @OutputFile File target

    @TaskAction
    void process() {
        target.text = source.text.toUpperCase()
    }
}
````
也可以用运行时 API：`inputs.file()`、`outputs.file()`、`inputs.property()` 等。

---

### Q4：Configuration Cache 是什么？和 Build Cache 有什么区别？

**A**：

| 特性 | Configuration Cache | Build Cache |
|------|-------------------|-------------|
| 缓存内容 | 配置阶段的结果（Task DAG） | 执行阶段的 Task 输出 |
| 跳过阶段 | 初始化 + 配置 | 单个 Task 的执行 |
| 生效条件 | 构建脚本/插件未变化 | Task inputs 未变化 |
| 引入版本 | Gradle 7.0 | Gradle 3.5 |
| 共享范围 | 本地 | 本地 + 远程（CI 共享） |

Configuration Cache 的约束：Task 执行阶段不能引用 `Project`/`Gradle`/`Settings` 对象，需在配置阶段将值捕获到局部变量或 `Property` 中。

---

### Q5：为什么推荐用 `tasks.register` 而不是 `tasks.create`？

**A**：

`tasks.create` 会**立即**创建并配置 Task 实例，即使该 Task 在本次构建中不会被执行。在大型项目中，数百个不需要执行的 Task 都被创建和配置，浪费大量配置时间。

`tasks.register` 采用**惰性注册**（Lazy Configuration），只有当 Task 真正需要执行或被其他 Task 依赖时，才会创建和配置实例。

````groovy
// 100 个 Task 全部立即创建 — 配置阶段慢
100.times { i ->
    tasks.create("task$i") { doLast { println i } }
}

// 100 个 Task 惰性注册 — 配置阶段快
100.times { i ->
    tasks.register("task$i") { doLast { println i } }
}
````
在 [[3-自定义Plugin]] 中应始终使用 `register` 方式注册 Task。

---

### Q6：Gradle Task 的 DAG 是如何构建的？能否有环？

**A**：

DAG 构建过程：配置阶段收集所有 Task 及其依赖关系 → 配置结束后根据命令行目标 Task 递归解析依赖构建子图 → 拓扑排序确定执行顺序。

**不能有环**。循环依赖会在配置阶段抛出 `CircularDependencyException`。

````groovy
// ❌ 循环依赖 — 构建失败
task a { dependsOn 'b' }
task b { dependsOn 'c' }
task c { dependsOn 'a' }
// Circular dependency between the following tasks:
// :a -> :b -> :c -> :a
````
排查方式：`./gradlew <task> --dry-run` 可以打印 Task 执行顺序而不实际执行。

---


## 四、实战与踩坑

### 4.1 踩坑：配置阶段执行了耗时操作

**现象**：每次 Android Studio Sync 都特别慢，即使什么都没改。

**原因**：在 `build.gradle` 配置块中执行了网络请求或文件 I/O。

````groovy
// ❌ 错误示范 — 配置阶段执行网络请求
task fetchVersion {
    // 这段代码在配置阶段执行！每次 sync 都会请求网络
    def url = new URL('https://api.example.com/version')
    def version = url.text.trim()
    
    doLast {
        println "Version: $version"
    }
}

// ✅ 正确做法 — 将耗时操作移到执行阶段
task fetchVersion {
    outputs.file "$buildDir/version.txt"
    
    doLast {
        def url = new URL('https://api.example.com/version')
        file("$buildDir/version.txt").text = url.text.trim()
    }
}
````
### 4.2 踩坑：Task 输出目录冲突

**现象**：两个 Task 输出到同一目录，增量构建失效，每次都重新执行。

````groovy
// ❌ 两个 Task 输出到同一目录
task generateA {
    outputs.dir "$buildDir/generated"
    doLast { /* ... */ }
}
task generateB {
    outputs.dir "$buildDir/generated"  // 冲突！
    doLast { /* ... */ }
}

// ✅ 每个 Task 使用独立输出目录
task generateA {
    outputs.dir "$buildDir/generated/a"
    doLast { /* ... */ }
}
task generateB {
    outputs.dir "$buildDir/generated/b"
    doLast { /* ... */ }
}
````
### 4.3 实战：构建耗时分析

````bash
# 生成构建扫描报告
./gradlew assembleDebug --scan

# 生成 profile 报告（本地 HTML）
./gradlew assembleDebug --profile

# 查看 Task 执行耗时
./gradlew assembleDebug --info | grep "Task :"
````
推荐使用 `--scan` 上传到 Gradle 官方平台查看详细的可视化报告。

### 4.4 实战：自定义 Task 完整案例 — APK 产物拷贝

在 Android 项目中，构建完成后自动将 APK 拷贝到指定目录并重命名：

````groovy
// app/build.gradle
android.applicationVariants.all { variant ->
    variant.outputs.all { output ->
        tasks.register("copy${variant.name.capitalize()}Apk") {
            group = 'distribution'
            def apkFile = output.outputFile
            def targetFile = file("${rootDir}/release/${rootProject.name}-${variant.name}-${variant.versionName}.apk")
            inputs.file apkFile
            outputs.file targetFile

            doLast {
                targetFile.parentFile.mkdirs()
                apkFile.withInputStream { is -> targetFile.withOutputStream { os -> os << is } }
                println "✅ APK copied to: ${targetFile.path}"
            }
        }
    }
}
````
这个案例与 [[5-AAR与APK打包]] 中的打包流程紧密关联。

### 4.5 踩坑：Configuration Cache 不兼容

**现象**：启用 Configuration Cache 后构建报错 `Invocation of Task.project at execution time is unsupported`。

````groovy
// ❌ 执行阶段引用 project
task report {
    doLast {
        def name = project.name           // ❌
        def buildDir = project.buildDir   // ❌
        println "$name: $buildDir"
    }
}

// ✅ 配置阶段捕获值
task report {
    def name = project.name               // ✅ 配置阶段
    def dir = project.buildDir            // ✅ 配置阶段
    doLast {
        println "$name: $dir"             // ✅ 使用局部变量
    }
}
````
### 4.6 实战：利用 TaskGraph 动态控制行为

````groovy
// 根据是否执行 release 构建来决定是否签名
gradle.taskGraph.whenReady { graph ->
    if (graph.hasTask(':app:assembleRelease')) {
        println "🔐 Release build detected, loading signing config..."
        android.signingConfigs.release.storePassword = 
            System.getenv('STORE_PASSWORD') ?: 'default'
    }
}
````
### 4.7 踩坑：Task 依赖声明时机

**现象**：`dependsOn` 引用的 Task 还未注册，报 `Task not found`。

````groovy
// ❌ 引用尚未注册的 Task（取决于脚本执行顺序）
task a {
    dependsOn 'b'  // 如果 b 在后面定义，用字符串是安全的
}
task b { doLast { println 'b' } }

// ✅ 使用字符串引用（惰性解析）或 provider
tasks.register('x') {
    dependsOn tasks.named('y')  // ✅ TaskProvider 惰性引用
}
tasks.register('y') {
    doLast { println 'y' }
}
````
### 4.8 调试技巧速查

````bash
# 查看 Task 依赖树
./gradlew :app:assembleDebug --dry-run

# 强制重新执行（忽略 UP-TO-DATE）
./gradlew :app:assembleDebug --rerun-tasks

# 查看某个 Task 的 inputs/outputs
./gradlew :app:compileDebugJavaWithJavac --info

# 查看配置缓存状态
./gradlew :app:assembleDebug --configuration-cache

# 并行执行
./gradlew assembleDebug --parallel

# 持续构建（文件变化自动重新执行）
./gradlew processData --continuous
````
---

## 总结

| 知识点 | 关键要点 |
|--------|---------|
| 三大阶段 | 初始化 → 配置 → 执行，配置阶段代码每次都执行 |
| Task DAG | 配置阶段构建，执行阶段按拓扑序执行，不允许环 |
| 增量构建 | inputs/outputs 指纹比较，UP-TO-DATE 跳过 |
| Configuration Cache | 缓存配置阶段结果，执行阶段不能引用 Project |
| 惰性注册 | `tasks.register` 替代 `tasks.create` |
| 性能优化 | 配置阶段避免 I/O、使用 `--profile`/`--scan` 分析 |

> 延伸阅读：[[3-自定义Plugin]] · [[5-AAR与APK打包]] · [[Gradle依赖管理]] · [[Build Cache]]

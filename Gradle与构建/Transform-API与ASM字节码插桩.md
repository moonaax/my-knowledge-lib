# Transform API 与 ASM 字节码插桩

> Android 构建过程中，字节码插桩是实现无侵入式功能注入的核心手段。本文梳理 Transform API（已废弃）与 ASM 框架的原理、实践及迁移方案。
> 相关主题：[[自定义Plugin]] · [[Gradle生命周期与Task]] · [[注解处理器APT与KSP]]

---

## 一、核心概念

### 1.1 Transform API 的作用（已废弃）

Transform API 是 AGP 提供的字节码处理管线接口，允许在 `.class → .dex` 阶段拦截和修改字节码。

核心流程：

```
.java/.kt → javac/kotlinc → .class → [Transform 链] → .dex → APK
```

关键类与方法：

```java
public class MyTransform extends Transform {
    @Override public String getName() { return "myTransform"; }

    @Override public Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }

    @Override public Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }

    @Override public boolean isIncremental() { return true; }

    @Override
    public void transform(TransformInvocation invocation) {
        for (TransformInput input : invocation.getInputs()) {
            for (JarInput jarInput : input.getJarInputs()) { /* 处理 jar */ }
            for (DirectoryInput dirInput : input.getDirectoryInputs()) { /* 处理 class */ }
        }
    }
}
```

在 [[自定义Plugin]] 中注册：

```java
public class MyPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        AppExtension app = project.getExtensions().getByType(AppExtension.class);
        app.registerTransform(new MyTransform());
    }
}
```

**废弃原因（AGP 7.0）：**

| 问题 | 说明 |
|------|------|
| 性能差 | 每个 Transform 都需要完整复制 I/O，多个 Transform 串联导致大量磁盘读写 |
| 增量支持复杂 | 开发者需自行处理增量逻辑，容易出错 |
| 缺乏隔离 | Transform 之间共享输入输出，调试困难 |
| API 不稳定 | 内部实现频繁变动，兼容性成本高 |

### 1.2 ASM 字节码操作框架

ASM 是轻量级、高性能的 Java 字节码操作框架，直接操作 `.class` 二进制结构。体积小（约 100KB），基于事件驱动的访问者模式，Android 官方推荐。

**两种 API 模型：**

| 模型 | 特点 | 适用场景 |
|------|------|----------|
| Core API（事件模型） | 基于 Visitor 回调，流式处理，内存友好 | 简单插桩、修改 |
| Tree API（对象模型） | 整个 class 加载为 ClassNode 树 | 复杂分析、多次遍历 |

### 1.3 访问者模式：ClassVisitor / MethodVisitor

ASM 核心设计基于访问者模式，通过 Visitor 回调遍历和修改字节码：

```
ClassReader → ClassVisitor → ClassWriter
                  ↓
            MethodVisitor / FieldVisitor / AnnotationVisitor
```

**ClassVisitor 回调顺序：** `visit → visitSource? → visitOuterClass? → (visitAnnotation | visitAttribute)* → (visitInnerClass | visitField | visitMethod)* → visitEnd`

**MethodVisitor 回调顺序：** `visitCode → (visitInsn | visitVarInsn | visitMethodInsn | ...)* → visitMaxs → visitEnd`

每个 `visitXxx` 对应字节码中的一个结构元素，重写即可实现插桩。

### 1.4 AGP 7.0+ 替代方案：AsmClassVisitorFactory

AGP 7.0 引入 `AsmClassVisitorFactory` 作为 Transform API 的官方替代。AGP 自动管理 I/O 和增量编译，多个 Factory 可高效串联，通过 Instrumentation API 注册，与 [[Gradle生命周期与Task]] 深度集成。

**基本结构：**

```java
// 1. 定义参数接口（可选）
public interface TimeCostParams extends InstrumentationParameters {
    @Input
    Property<Boolean> getEnabled();
}

// 2. 实现 AsmClassVisitorFactory
public abstract class TimeCostClassVisitorFactory
        implements AsmClassVisitorFactory<TimeCostParams> {

    @Override
    public ClassVisitor createClassVisitor(
            ClassContext classContext,
            ClassVisitor nextClassVisitor) {
        return new TimeCostClassVisitor(
                Opcodes.ASM9, nextClassVisitor);
    }

    @Override
    public boolean isInstrumentable(ClassData classData) {
        // 过滤需要处理的类
        return classData.getClassName()
                .startsWith("com.example.app");
    }
}
```

在 [[自定义Plugin]] 中注册：

```java
public class MyPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        AndroidComponentsExtension components =
            project.getExtensions().getByType(AndroidComponentsExtension.class);
        components.onVariants(components.selector().all(), variant -> {
            variant.getInstrumentation().transformClassesWith(
                TimeCostClassVisitorFactory.class, InstrumentationScope.ALL,
                params -> params.getEnabled().set(true));
        });
    }
}
```

---


## 二、原理与实践

### 2.1 ASM 的 ClassReader / ClassWriter / ClassVisitor 流程

ASM 字节码处理遵循「读取 → 访问 → 写入」三步流水线：

| 类 | 职责 |
|---|------|
| `ClassReader` | 解析 `.class` 字节数组，驱动 Visitor 回调 |
| `ClassVisitor` | 接收回调事件，在此修改字节码（插桩核心） |
| `ClassWriter` | 将 Visitor 事件序列化为新的 `.class` 字节数组 |

**基本模板：**

```java
public static byte[] transform(byte[] classBytes) {
    ClassReader cr = new ClassReader(classBytes);
    ClassWriter cw = new ClassWriter(
            cr, ClassWriter.COMPUTE_FRAMES);
    ClassVisitor cv = new MyClassVisitor(Opcodes.ASM9, cw);
    cr.accept(cv, ClassReader.EXPAND_FRAMES);
    return cw.toByteArray();
}
```

**参数：** `COMPUTE_FRAMES` 自动计算栈帧映射，`EXPAND_FRAMES` 展开栈帧信息，两者配合使用。

### 2.2 方法插桩示例：方法耗时埋点

目标：在每个方法的入口和出口插入计时代码，等价于：

```java
// 插桩前
public void doWork() {
    // 业务逻辑
}

// 插桩后
public void doWork() {
    long _startTime = System.currentTimeMillis();
    // 业务逻辑
    long _cost = System.currentTimeMillis() - _startTime;
    Log.d("TimeCost", "doWork cost: " + _cost + "ms");
}
```

**Step 1：自定义 ClassVisitor**

```java
public class TimeCostClassVisitor extends ClassVisitor {
    private String className;

    public TimeCostClassVisitor(int api, ClassVisitor cv) { super(api, cv); }

    @Override
    public void visit(int version, int access, String name,
            String signature, String superName, String[] interfaces) {
        super.visit(version, access, name, signature, superName, interfaces);
        this.className = name;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name,
            String descriptor, String signature, String[] exceptions) {
        MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
        if ("<init>".equals(name) || "<clinit>".equals(name)) return mv;
        return new TimeCostMethodVisitor(api, mv, access, name, descriptor, className);
    }
}
```

**Step 2：自定义 MethodVisitor（使用 AdviceAdapter）**

```java
public class TimeCostMethodVisitor extends AdviceAdapter {
    private final String methodName;
    private final String className;
    private int startTimeVarIndex;

    protected TimeCostMethodVisitor(int api, MethodVisitor mv,
            int access, String name, String desc, String className) {
        super(api, mv, access, name, desc);
        this.methodName = name;
        this.className = className;
    }

    @Override
    protected void onMethodEnter() {
        // long _startTime = System.currentTimeMillis();
        startTimeVarIndex = newLocal(Type.LONG_TYPE);
        mv.visitMethodInsn(INVOKESTATIC,
                "java/lang/System", "currentTimeMillis",
                "()J", false);
        mv.visitVarInsn(LSTORE, startTimeVarIndex);
    }

    @Override
    protected void onMethodExit(int opcode) {
        // long _cost = System.currentTimeMillis() - _startTime;
        mv.visitMethodInsn(INVOKESTATIC,
                "java/lang/System", "currentTimeMillis", "()J", false);
        mv.visitVarInsn(LLOAD, startTimeVarIndex);
        mv.visitInsn(LSUB);
        int costVarIndex = newLocal(Type.LONG_TYPE);
        mv.visitVarInsn(LSTORE, costVarIndex);

        // Log.d("TimeCost", className + "." + methodName + " cost: " + cost + "ms");
        mv.visitLdcInsn("TimeCost");
        // 使用 StringBuilder 拼接日志字符串
        mv.visitTypeInsn(NEW, "java/lang/StringBuilder");
        mv.visitInsn(DUP);
        mv.visitMethodInsn(INVOKESPECIAL, "java/lang/StringBuilder", "<init>", "()V", false);
        mv.visitLdcInsn(className.replace('/', '.') + "." + methodName + " cost: ");
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append",
                "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
        mv.visitVarInsn(LLOAD, costVarIndex);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append",
                "(J)Ljava/lang/StringBuilder;", false);
        mv.visitLdcInsn("ms");
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "append",
                "(Ljava/lang/String;)Ljava/lang/StringBuilder;", false);
        mv.visitMethodInsn(INVOKEVIRTUAL, "java/lang/StringBuilder", "toString",
                "()Ljava/lang/String;", false);
        mv.visitMethodInsn(INVOKESTATIC, "android/util/Log", "d",
                "(Ljava/lang/String;Ljava/lang/String;)I", false);
        mv.visitInsn(POP);
    }
}
```

> **为什么用 AdviceAdapter？** 它封装了 `onMethodEnter()` / `onMethodExit()` 回调，自动处理构造方法中 `super()` 调用前不能插入代码的问题。

### 2.3 AsmClassVisitorFactory 完整示例

基于 AGP 7.0+ 的方法耗时插桩方案，build.gradle（插件模块）：

```groovy
plugins { id 'java-gradle-plugin' }
dependencies {
    implementation 'com.android.tools.build:gradle:7.4.2'
    implementation 'org.ow2.asm:asm:9.5'
    implementation 'org.ow2.asm:asm-commons:9.5'
}
gradlePlugin {
    plugins {
        timeCostPlugin {
            id = 'com.example.timecost'
            implementationClass = 'com.example.TimeCostPlugin'
        }
    }
}
```

**TimeCostPlugin.java：**

```java
public class TimeCostPlugin implements Plugin<Project> {
    @Override
    public void apply(Project project) {
        AndroidComponentsExtension components =
            project.getExtensions().getByType(AndroidComponentsExtension.class);
        components.onVariants(components.selector().all(), variant -> {
            variant.getInstrumentation().transformClassesWith(
                TimeCostFactory.class, InstrumentationScope.ALL,
                params -> params.getEnabled().set(true));
            variant.getInstrumentation().setAsmFramesComputationMode(
                FramesComputationMode.COMPUTE_FRAMES_FOR_INSTRUMENTED_METHODS);
        });
    }
}
```

**TimeCostFactory.java：**

```java
public abstract class TimeCostFactory
        implements AsmClassVisitorFactory<TimeCostParams> {
    @Override
    public ClassVisitor createClassVisitor(ClassContext ctx, ClassVisitor next) {
        return new TimeCostClassVisitor(Opcodes.ASM9, next);
    }
    @Override
    public boolean isInstrumentable(ClassData classData) {
        return classData.getClassName().startsWith("com.example.");
    }
}

public interface TimeCostParams extends InstrumentationParameters {
    @Input Property<Boolean> getEnabled();
}
```

`TimeCostClassVisitor` 和 `TimeCostMethodVisitor` 复用 2.2 节代码。应用插件只需 `id 'com.example.timecost'`，与旧版 Transform API 对比代码量减少约 60%。

---


## 三、应用场景

### 3.1 方法耗时统计

最经典的插桩场景，已在第二节详细展示。核心思路：

- 在 `onMethodEnter()` 记录起始时间
- 在 `onMethodExit()` 计算差值并上报
- 可配合注解过滤（如 `@TraceMethod`），避免全量插桩带来的性能开销

**注解过滤示例：** 通过检测 `@TraceMethod` 注解决定是否插桩：

```java
@Override
public MethodVisitor visitMethod(int access, String name,
        String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    return new MethodVisitor(api, mv) {
        boolean needInstrument = false;
        @Override
        public AnnotationVisitor visitAnnotation(String desc, boolean visible) {
            if ("Lcom/example/TraceMethod;".equals(desc)) needInstrument = true;
            return super.visitAnnotation(desc, visible);
        }
        @Override
        public void visitCode() {
            super.visitCode();
            if (needInstrument) { /* 插入 onMethodEnter 逻辑 */ }
        }
    };
}
```

结合 [[注解处理器APT与KSP]]，可以在编译期生成注解标记，再由 ASM 插桩读取。

### 3.2 无侵入埋点

业务代码无需任何修改，通过字节码插桩自动在关键位置注入埋点代码。

**典型场景：**
- `Activity.onCreate()` / `onResume()` 页面曝光
- `View.OnClickListener.onClick()` 点击事件
- `Fragment` 生命周期

**onClick 插桩思路：** 找到所有实现了 `View.OnClickListener` 接口的类，在其 `onClick` 方法入口插入埋点：

```java
@Override
public void visit(int version, int access, String name,
        String signature, String superName, String[] interfaces) {
    super.visit(version, access, name, signature, superName, interfaces);
    if (interfaces != null) {
        for (String iface : interfaces) {
            if ("android/view/View$OnClickListener".equals(iface)) {
                this.isClickListener = true;
            }
        }
    }
}
```

### 3.3 权限检查

在调用敏感 API 前自动插入权限检查，避免遗漏导致崩溃：

```java
@Override
public void visitMethodInsn(int opcode, String owner,
        String name, String descriptor, boolean isInterface) {
    if ("android/hardware/Camera".equals(owner) && "open".equals(name)) {
        mv.visitVarInsn(ALOAD, 0);
        mv.visitLdcInsn("android.permission.CAMERA");
        mv.visitMethodInsn(INVOKESTATIC, "com/example/PermissionHelper",
                "check", "(Ljava/lang/Object;Ljava/lang/String;)Z", false);
        Label continueLabel = new Label();
        mv.visitJumpInsn(IFNE, continueLabel);
        mv.visitInsn(RETURN);
        mv.visitLabel(continueLabel);
    }
    super.visitMethodInsn(opcode, owner, name, descriptor, isInterface);
}
```

### 3.4 线程切换检测

检测主线程中执行耗时操作（如数据库、网络），Debug 模式下抛出警告：

```java
@Override
protected void onMethodEnter() {
    mv.visitMethodInsn(INVOKESTATIC, "android/os/Looper",
            "myLooper", "()Landroid/os/Looper;", false);
    mv.visitMethodInsn(INVOKESTATIC, "android/os/Looper",
            "getMainLooper", "()Landroid/os/Looper;", false);
    Label notMainThread = new Label();
    mv.visitJumpInsn(IF_ACMPNE, notMainThread);
    mv.visitLdcInsn("ThreadCheck");
    mv.visitLdcInsn(className + "." + methodName + " called on main thread!");
    mv.visitMethodInsn(INVOKESTATIC, "android/util/Log", "w",
            "(Ljava/lang/String;Ljava/lang/String;)I", false);
    mv.visitInsn(POP);
    mv.visitLabel(notMainThread);
}
```

可通过 `isInstrumentable()` 或注解过滤，仅对数据库操作类、网络请求类等进行检测。

---


## 四、面试题

### Q1：Transform API 为什么被废弃？AGP 7.0+ 用什么替代？

**A：** 三大问题：①性能差，每个 Transform 完整读写所有 class，串联时 I/O 成倍增长；②增量编译需自行实现，极易出错；③API 不稳定，适配成本高。AGP 7.0+ 使用 `AsmClassVisitorFactory` 替代，AGP 自动管理 I/O、增量编译和多 Factory 串联。

### Q2：ASM 的 Core API 和 Tree API 有什么区别？

**A：**

| 维度 | Core API | Tree API |
|------|----------|----------|
| 工作方式 | Visitor 回调，流式处理 | 加载为 ClassNode 树 |
| 内存占用 | 低 | 高 |
| 适用场景 | 简单插桩 | 复杂分析、多次遍历 |

90% 的插桩需求用 Core API + AdviceAdapter 即可。

### Q3：AdviceAdapter 相比直接使用 MethodVisitor 有什么优势？

**A：** AdviceAdapter 是 ASM 提供的 MethodVisitor 子类：
1. 提供语义清晰的 `onMethodEnter()` 和 `onMethodExit(int opcode)` 回调
2. 自动处理构造方法中 `super()` 调用前不能插入代码的问题
3. `onMethodExit()` 覆盖所有返回路径（正常 return 和异常 throw）

直接使用 MethodVisitor 需自行在每个 `visitInsn(RETURN)` / `visitInsn(ARETURN)` 处插入出口代码，容易遗漏。

### Q4：ClassWriter 的 COMPUTE_FRAMES 和 COMPUTE_MAXS 有什么区别？

**A：**
- `COMPUTE_MAXS`：自动计算 max_stack 和 max_locals，不计算栈帧映射
- `COMPUTE_FRAMES`：自动计算栈帧映射 + max_stack + max_locals

推荐 `COMPUTE_FRAMES`，Java 6+ 要求 StackMapFrame，手动计算极易出错。配合 `ClassReader.EXPAND_FRAMES` 使用。

### Q5：如何在 ASM 插桩中实现增量编译？

**A：**
- **旧版 Transform API：** 需在 `transform()` 中检查 `isIncremental()`，遍历 `getChangedFiles()` 按 Status（ADDED/CHANGED/REMOVED）分别处理
- **AsmClassVisitorFactory：** AGP 自动处理增量逻辑，只对变更的 class 调用 `createClassVisitor()`，开发者无需关心

### Q6：字节码插桩和 APT/KSP 有什么区别？

**A：**

| 维度 | ASM 插桩 | APT / KSP |
|------|---------|-----------|
| 时机 | `.class` 编译后、打包前 | 编译期（源码级） |
| 能否修改已有代码 | ✅ 可修改任意 class | ❌ 只能生成新代码 |
| 典型场景 | 方法插桩、无侵入埋点 | 代码生成（Router 等） |

两者可配合：[[注解处理器APT与KSP]] 生成标记，ASM 读取标记进行插桩。

---

## 五、实战与踩坑

### 5.1 常见错误与解决方案

**❌ 错误 1：`VerifyError` / `ClassFormatError`**

未使用 `COMPUTE_FRAMES` 或插入指令后栈不平衡：

```java
// ✅ 正确
ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_FRAMES);
cr.accept(cv, ClassReader.EXPAND_FRAMES);
// ❌ 错误
ClassWriter cw = new ClassWriter(0);
```

**❌ 错误 2：构造方法插桩 `VerifyError`**

在 `super()` 前插入使用 `this` 的代码。解决：使用 AdviceAdapter 的 `onMethodEnter()`，它自动延迟到 `super()` 之后。

**❌ 错误 3：Lambda 导致意外插桩**

Lambda 编译为合成方法（synthetic），需过滤：

```java
@Override
public MethodVisitor visitMethod(int access, String name,
        String descriptor, String signature, String[] exceptions) {
    MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);
    if ((access & Opcodes.ACC_SYNTHETIC) != 0 || (access & Opcodes.ACC_BRIDGE) != 0) return mv;
    return new TimeCostMethodVisitor(api, mv, access, name, descriptor, className);
}
```

### 5.2 调试技巧

**1. ASM Bytecode Viewer 插件：** Android Studio 安装此插件，可直接查看任意类对应的 ASM 代码，极大降低手写 visitXxx 的难度。

**2. 输出插桩前后的 class 对比：** 保存中间产物后用 `javap -c -p` 反编译对比。

**3. 使用 CheckClassAdapter 验证：**

```java
ClassWriter cw = new ClassWriter(cr, ClassWriter.COMPUTE_FRAMES);
CheckClassAdapter checker = new CheckClassAdapter(cw);
ClassVisitor cv = new MyClassVisitor(Opcodes.ASM9, checker);
cr.accept(cv, ClassReader.EXPAND_FRAMES);
```

插桩代码有问题时会在编译期直接抛出异常，而不是等到运行时才 VerifyError。

### 5.3 性能优化建议

| 优化点 | 说明 |
|--------|------|
| 缩小插桩范围 | 通过 `isInstrumentable()` 或包名过滤 |
| 避免 Tree API | 除非需要复杂分析，Core API 内存更低 |
| 减少字符串拼接 | StringBuilder 会增加方法体积 |
| 注意 Kotlin inline | inline 函数复制到调用处，可能重复插桩 |

### 5.4 AGP 版本迁移清单

从 Transform API 迁移到 AsmClassVisitorFactory：

1. 移除 Transform 子类和 `registerTransform()` 调用
2. 创建 `AsmClassVisitorFactory` 实现类
3. 删除文件遍历/I/O 逻辑（AGP 自动处理）
4. 保留 ClassVisitor / MethodVisitor 核心逻辑
5. 在 Plugin 中通过 `AndroidComponentsExtension` 注册
6. 设置 `FramesComputationMode`

```groovy
// 新版依赖
implementation 'com.android.tools.build:gradle:7.0+'
implementation 'org.ow2.asm:asm-commons:9.5' // AdviceAdapter 需要
```

---

> **总结：** 字节码插桩是 Android 工程化的高级技能。理解 ASM Visitor 模式和 AGP 构建管线（参考 [[Gradle生命周期与Task]]），是实现高质量插桩的基础。新项目应直接使用 AsmClassVisitorFactory。

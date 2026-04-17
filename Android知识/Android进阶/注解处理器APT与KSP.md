# 注解处理器 APT 与 KSP

## 一、注解基础

### 1.1 什么是注解？

注解（Annotation）是代码的"标签"，本身不影响程序运行，但可以被编译器或运行时工具读取并做相应处理。

```java
// 你每天都在用注解
@Override           // 编译时检查是否正确重写
@Nullable           // 标记可空
@Deprecated         // 标记已废弃
@Entity             // Room 的表定义
@Inject             // Hilt 的依赖注入
@GET("/users")      // Retrofit 的网络请求
```

### 1.2 注解的保留策略

| 策略 | 含义 | 使用场景 |
|------|------|---------|
| `SOURCE` | 只在源码中存在，编译后丢弃 | @Override、Lint 检查 |
| `CLASS` | 保留到 class 文件，运行时不可见 | APT 处理（默认值） |
| `RUNTIME` | 保留到运行时，可以通过反射读取 | Retrofit、Gson、依赖注入 |

```java
// 自定义注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)  // 只在编译时使用
public @interface AutoRegister {
    String name();
}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)  // 运行时可反射读取
public @interface BindView {
    int id();
}
```

### 1.3 编译时注解 vs 运行时注解

| 对比项 | 编译时注解（APT/KSP） | 运行时注解（反射） |
|--------|---------------------|------------------|
| 处理时机 | 编译期 | 运行期 |
| 性能 | 无运行时开销 | 反射有性能损耗 |
| 产物 | 生成新的源码文件 | 无 |
| 代表框架 | Room、Dagger/Hilt、ARouter | Retrofit、Gson、ButterKnife（早期） |

---

## 二、APT（Annotation Processing Tool）

### 2.1 是什么？

APT 是 Java 的编译时注解处理工具。它在编译阶段扫描源码中的注解，根据注解信息**生成新的 Java 源文件**，这些生成的文件会和你的代码一起编译。

### 2.2 工作流程

```
你的源码（带注解）
    ↓
javac 编译开始
    ↓
APT 扫描注解 → 调用你写的 Processor
    ↓
Processor 生成新的 .java 文件
    ↓
javac 编译生成的文件（可能触发新一轮 APT）
    ↓
所有文件编译完成 → .class
```

### 2.3 实战：自动生成路由表

假设我们要实现一个简单的路由框架，通过注解自动收集所有页面。

**第一步：定义注解（annotation 模块，纯 Java/Kotlin 库）**

```java
// Route.java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
public @interface Route {
    String path();
}
```

**第二步：编写注解处理器（compiler 模块，Java 库）**

```java
// RouteProcessor.java
@AutoService(Processor.class)  // 自动注册处理器
public class RouteProcessor extends AbstractProcessor {

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return Collections.singleton(Route.class.getCanonicalName());
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        Map<String, String> routeMap = new HashMap<>();

        // 收集所有被 @Route 标注的类
        for (Element element : roundEnv.getElementsAnnotatedWith(Route.class)) {
            Route annotation = element.getAnnotation(Route.class);
            String path = annotation.path();
            String className = ((TypeElement) element).getQualifiedName().toString();
            routeMap.put(path, className);
        }

        if (!routeMap.isEmpty()) {
            generateRouteTable(routeMap);
        }

        return true;
    }

    private void generateRouteTable(Map<String, String> routeMap) {
        // 用 JavaPoet 生成代码
        ParameterizedTypeName mapType = ParameterizedTypeName.get(
            ClassName.get(Map.class),
            ClassName.get(String.class),
            ClassName.get(Class.class)
        );

        MethodSpec.Builder methodBuilder = MethodSpec.methodBuilder("getRoutes")
            .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
            .returns(mapType)
            .addStatement("Map<String, Class<?>> map = new HashMap<>()");

        for (Map.Entry<String, String> entry : routeMap.entrySet()) {
            methodBuilder.addStatement("map.put($S, $T.class)",
                entry.getKey(), ClassName.bestGuess(entry.getValue()));
        }

        methodBuilder.addStatement("return map");

        TypeSpec typeSpec = TypeSpec.classBuilder("RouteTable_Generated")
            .addModifiers(Modifier.PUBLIC)
            .addMethod(methodBuilder.build())
            .build();

        try {
            JavaFile.builder("com.example.router", typeSpec)
                .build()
                .writeTo(processingEnv.getFiler());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**第三步：使用注解**

```java
@Route(path = "/home")
public class HomeActivity extends AppCompatActivity { }

@Route(path = "/detail")
public class DetailActivity extends AppCompatActivity { }
```

**编译后自动生成：**

```java
// build/generated/source/apt/...
public class RouteTable_Generated {
    public static Map<String, Class<?>> getRoutes() {
        Map<String, Class<?>> map = new HashMap<>();
        map.put("/home", HomeActivity.class);
        map.put("/detail", DetailActivity.class);
        return map;
    }
}
```

**第四步：Gradle 配置**

```groovy
// annotation 模块
plugins { id 'java-library' }

// compiler 模块
plugins { id 'java-library' }
dependencies {
    implementation project(':annotation')
    implementation 'com.google.auto.service:auto-service:1.0'
    annotationProcessor 'com.google.auto.service:auto-service:1.0'
    implementation 'com.squareup:javapoet:1.13.0'
}

// app 模块
dependencies {
    implementation project(':annotation')
    annotationProcessor project(':compiler')  // Java
    // 或 kapt project(':compiler')           // Kotlin
}
```

---

## 三、KSP（Kotlin Symbol Processing）

### 3.1 是什么？

KSP 是 Google 推出的 Kotlin 专用的注解处理工具，用来替代 kapt。

### 3.2 为什么需要 KSP？

**kapt 的问题：**
- kapt 是 APT 的 Kotlin 适配层，工作原理是先把 Kotlin 代码生成 Java Stub，再交给 APT 处理
- 生成 Java Stub 的过程很慢，占了编译时间的 1/3
- 不能直接理解 Kotlin 特有的语法（扩展函数、密封类、协程等）

**KSP 的优势：**
- 直接分析 Kotlin 代码，不需要生成 Java Stub
- 编译速度比 kapt 快 2 倍以上
- 原生支持 Kotlin 特性
- API 更简洁

### 3.3 KSP vs kapt vs APT

| 对比项 | APT | kapt | KSP |
|--------|-----|------|-----|
| 语言 | Java | Kotlin（通过 Java Stub） | Kotlin（原生） |
| 速度 | 快 | 慢（需要生成 Stub） | 快（比 kapt 快 2x） |
| Kotlin 支持 | 不理解 Kotlin 语法 | 通过 Stub 间接支持 | 原生支持 |
| 增量编译 | 支持 | 部分支持 | 完善支持 |
| 生态 | 成熟 | 成熟 | 快速增长 |

### 3.4 KSP 实战

```kotlin
// KSP 处理器
class RouteProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return RouteProcessor(environment.codeGenerator, environment.logger)
    }
}

class RouteProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {

    override fun process(resolver: Resolver): List<KSAnnotated> {
        // 找到所有被 @Route 标注的类
        val symbols = resolver.getSymbolsWithAnnotation(Route::class.qualifiedName!!)
        val unprocessed = mutableListOf<KSAnnotated>()

        symbols.forEach { symbol ->
            if (symbol is KSClassDeclaration) {
                val annotation = symbol.annotations.first {
                    it.shortName.asString() == "Route"
                }
                val path = annotation.arguments.first { it.name?.asString() == "path" }
                    .value as String
                val className = symbol.qualifiedName?.asString() ?: return@forEach

                logger.info("Found route: $path -> $className")
                // 生成代码...
            } else {
                unprocessed.add(symbol)
            }
        }

        return unprocessed
    }
}
```

```groovy
// Gradle 配置
plugins {
    id 'com.google.devtools.ksp' version '1.9.0-1.0.13'
}

dependencies {
    implementation project(':annotation')
    ksp project(':compiler')  // 用 ksp 替代 kapt
}
```

### 3.5 主流框架的 KSP 支持

| 框架 | kapt | KSP |
|------|------|-----|
| Room | ✅ | ✅（推荐） |
| Hilt/Dagger | ✅ | ✅ |
| Moshi | ✅ | ✅ |
| Glide | ✅ | ✅ |
| ARouter | ✅ | ❌（社区版有） |

---

## 四、面试题库

### Q1：编译时注解和运行时注解的区别？各有什么优缺点？

> 编译时注解（APT/KSP）在编译期处理，生成新的源码文件，运行时零开销，但只能生成代码不能修改已有代码。运行时注解通过反射在运行期读取，灵活性高可以动态处理，但反射有性能损耗（方法调用慢 5-10 倍），而且反射会绕过编译检查，类型安全性差。实际项目中能用编译时注解就不用运行时注解。Room、Hilt、ARouter 都是编译时注解，Retrofit 的 @GET/@POST 是运行时注解（因为需要在运行时动态构建请求）。

### Q2：APT 的工作原理？

> APT 在 javac 编译过程中被触发。javac 编译源码时会检查是否有注册的 Processor，如果有就调用 Processor 的 process 方法，传入被注解标记的元素信息。Processor 可以读取注解的值和被注解元素的类型信息，然后通过 Filer 生成新的 Java 源文件。生成的文件会参与下一轮编译，如果新文件中也有注解，会触发新一轮 APT 处理，直到没有新文件生成为止。Processor 通过 @AutoService 注解或 META-INF/services 文件注册到编译器。

### Q3：KSP 和 kapt 的区别？为什么 KSP 更快？

> kapt 的工作原理是先把 Kotlin 代码生成 Java Stub（只有方法签名没有实现的 Java 文件），然后交给 Java 的 APT 处理。生成 Stub 的过程非常耗时，大约占编译时间的三分之一。KSP 直接分析 Kotlin 的符号信息，不需要生成 Java Stub，所以快了大约 2 倍。而且 KSP 原生理解 Kotlin 语法（扩展函数、密封类、默认参数等），kapt 通过 Java Stub 会丢失这些信息。目前主流框架都已支持 KSP，新项目应该优先使用 KSP。

### Q4：JavaPoet / KotlinPoet 是什么？

> JavaPoet 和 KotlinPoet 是 Square 开源的代码生成库，分别用于生成 Java 和 Kotlin 源文件。它们提供了类型安全的 API 来构建类、方法、字段等代码结构，避免了手动拼接字符串容易出错的问题。比如 TypeSpec 构建类、MethodSpec 构建方法、FieldSpec 构建字段，最终通过 JavaFile/FileSpec 写入文件。几乎所有使用 APT/KSP 的框架都用它们来生成代码。

### Q5：ARouter 的路由表是怎么生成的？

> ARouter 在编译时通过 APT 扫描所有被 @Route 标注的类，收集路径和类名的映射关系，为每个模块生成一个路由表类（如 ARouter$$Group$$xxx）。运行时 ARouter 通过反射加载这些生成的类，把路由信息注册到内存中的 HashMap。当调用 ARouter.getInstance().build("/path").navigation() 时，从 HashMap 中查找目标类并通过 Intent 启动。这样就实现了模块间不直接依赖也能跳转的效果。

### Q6：注解处理器能修改已有的类吗？

> 标准的 APT/KSP 不能修改已有的类，只能生成新的类。如果需要修改已有类的字节码，需要用 Transform API + ASM 在编译后的 class 文件上做字节码操作。比如 Hilt 的 @AndroidEntryPoint 看起来像是修改了 Activity 类，实际上是生成了一个 Hilt_XxxActivity 基类，然后通过 Gradle Transform 把你的 Activity 的父类从 AppCompatActivity 改为 Hilt_XxxActivity。这是 APT 和字节码操作配合使用的典型案例。

### Q7：如何调试注解处理器？

> 几种方式：一是在 Processor 中用 processingEnv.getMessager().printMessage() 输出日志，编译时会在 Gradle 控制台显示。二是在 Gradle 命令中加 --no-daemon -Dorg.gradle.debug=true，然后在 IDE 中 Remote Debug 连接到 Gradle 进程，可以在 Processor 代码中打断点。三是写单元测试，用 google 的 compile-testing 库模拟编译过程验证生成的代码是否正确。调试注解处理器确实比较麻烦，建议多写单元测试。

### Q8：增量编译对注解处理器有什么影响？

> 增量编译只重新编译修改过的文件，但注解处理器可能需要扫描所有文件才能正确生成代码。如果处理器不支持增量编译，Gradle 会退化为全量编译，导致编译变慢。APT 通过 @IncrementalAnnotationProcessor 声明增量支持类型（ISOLATING 或 AGGREGATING）。KSP 原生支持增量编译，只会把变化的符号传给处理器。这也是 KSP 编译更快的原因之一。

### Q9：实际项目中注解处理器有哪些应用场景？

> 非常多：Room 用 APT/KSP 根据 @Entity 和 @Dao 生成数据库操作代码；Hilt/Dagger 根据 @Inject 和 @Module 生成依赖注入代码；ARouter 根据 @Route 生成路由表；Glide 根据 @GlideModule 生成 API 扩展；Moshi/Gson 根据数据类生成 JSON 序列化适配器；ButterKnife 根据 @BindView 生成 View 绑定代码。在组件开发中，我们也可以用注解处理器自动生成组件注册表、SPI 实现类发现等。

### Q10：如果让你实现一个简单的 ButterKnife，思路是什么？

> 定义 @BindView(R.id.xxx) 注解标记 View 字段。编写 APT Processor，扫描所有被 @BindView 标注的字段，收集字段名、类型和资源 ID。用 JavaPoet 为每个 Activity 生成一个 XxxActivity_ViewBinding 类，在其构造方法中调用 activity.findViewById(id) 并赋值给对应字段。提供一个 ButterKnife.bind(activity) 方法，内部通过反射找到生成的 ViewBinding 类并实例化。这样运行时只有一次反射（找到生成的类），后续的 findViewById 都是直接调用，性能接近手写代码。

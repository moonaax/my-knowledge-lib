# Clean Architecture

> 洁净架构（Clean Architecture）由 Robert C. Martin（Uncle Bob）于 2012 年提出，通过**分层**和**依赖规则**将业务逻辑与框架、UI、数据库等外部细节彻底解耦。

相关主题：[[1-MVC-MVP-MVVM-MVI]] · [[3-依赖注入]]

---

## 一、核心概念

### 1.1 同心圆分层模型

```
┌─────────────────────────────────────────────┐
│            Frameworks & Drivers              │  ← Android SDK、Retrofit、Room
│  ┌───────────────────────────────────────┐   │
│  │        Interface Adapters             │   │  ← ViewModel、Presenter、Mapper
│  │  ┌───────────────────────────────┐    │   │
│  │  │        Use Cases              │    │   │  ← 每个用例一个类
│  │  │  ┌───────────────────────┐    │    │   │
│  │  │  │      Entities         │    │    │   │  ← 核心业务实体
│  │  │  └───────────────────────┘    │    │   │
│  │  └───────────────────────────────┘    │   │
│  └───────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

- **Entities**：核心业务对象与规则，纯 Java 类，不依赖任何框架
- **Use Cases**：封装具体业务操作，编排 Entity 交互
- **Interface Adapters**：数据格式转换层，包含 ViewModel、Mapper、Repository 实现
- **Frameworks & Drivers**：最外层技术实现，Android SDK、Retrofit、Room 等

```java
// Entity —— 纯业务实体
public class User {
    private String id;
    private String name;
    private String email;

    public User(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public boolean isEmailValid() {
        return email != null && email.contains("@");
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}
```

### 1.2 依赖规则（The Dependency Rule）

> **源码依赖只能指向内层，内层不能知道外层的任何信息。**

```
Frameworks → Interface Adapters → Use Cases → Entities
                    依赖方向：始终向内 →
```

内层定义接口（抽象），外层提供实现（具体）——这是**依赖反转原则（DIP）**的直接应用。

### 1.3 与传统三层架构的区别

| 对比维度 | 传统三层架构 | Clean Architecture |
|---------|------------|-------------------|
| 依赖方向 | 上层依赖下层具体实现 | 内层定义接口，外层实现，依赖反转 |
| 业务逻辑 | 常散落在 Controller/Activity 中 | 严格封装在 UseCase 和 Entity 中 |
| 可测试性 | Business 层依赖具体实现，难以单测 | UseCase 只依赖接口，轻松 Mock |
| 框架耦合 | 业务逻辑常与框架绑定 | 核心业务完全不依赖框架 |
| 可替换性 | 换数据库需改动业务层 | 只需替换外层实现 |

---

## 二、原理与实践

### 2.1 Android 中的分层落地

```
app/
├── data/           ← Frameworks + Interface Adapters 的数据部分
│   ├── remote/     ← Retrofit 网络数据源
│   ├── local/      ← Room / SP 本地数据源
│   ├── mapper/     ← DTO ↔ Domain Model 映射
│   └── repository/ ← Repository 实现
├── domain/         ← Entities + Use Cases（纯 Java，不依赖 Android SDK）
│   ├── model/
│   ├── repository/ ← Repository 接口
│   └── usecase/
└── presentation/   ← Interface Adapters 的 UI 部分 + Frameworks
    ├── ui/
    └── viewmodel/
```

### 2.2 Repository 模式

Repository 作为**数据访问的统一抽象**，屏蔽数据来源细节：

```java
// domain 层：定义接口
public interface ArticleRepository {
    List<Article> getArticles(int page, int pageSize);
    Article getArticleById(String id);
}

// data 层：实现接口
public class ArticleRepositoryImpl implements ArticleRepository {
    private final ArticleApi api;
    private final ArticleDao dao;
    private final ArticleMapper mapper;

    public ArticleRepositoryImpl(ArticleApi api, ArticleDao dao, ArticleMapper mapper) {
        this.api = api;
        this.dao = dao;
        this.mapper = mapper;
    }

    @Override
    public List<Article> getArticles(int page, int pageSize) {
        try {
            List<ArticleDto> dtos = api.getArticles(page, pageSize).execute().body();
            if (dtos != null) {
                dao.insertAll(mapper.toEntityList(dtos));
                return mapper.toDomainList(dtos);
            }
        } catch (Exception e) { /* 降级到本地 */ }
        return mapper.entityToDomainList(dao.getArticles(page, pageSize));
    }

    @Override
    public Article getArticleById(String id) {
        ArticleEntity entity = dao.getArticleById(id);
        return entity != null ? mapper.entityToDomain(entity) : null;
    }
}
```

### 2.3 UseCase 设计

**一个 UseCase 只做一件事**（单一职责）：

```java
public class GetArticleListUseCase {
    private final ArticleRepository repository;

    public GetArticleListUseCase(ArticleRepository repository) {
        this.repository = repository;
    }

    public List<Article> execute(int page, int pageSize) {
        return repository.getArticles(page, pageSize);
    }
}
```

异步场景使用 Kotlin 协程（Kotlin 特有能力）：

```kotlin
abstract class SuspendUseCase<in P, out R> {
    abstract suspend operator fun invoke(param: P): Result<R>
}

class GetArticleListUseCase(
    private val repository: ArticleRepository
) : SuspendUseCase<Pair<Int, Int>, List<Article>>() {
    override suspend fun invoke(param: Pair<Int, Int>): Result<List<Article>> = runCatching {
        repository.getArticles(param.first, param.second)
    }
}
```

### 2.4 依赖反转原则

依赖反转是 Clean Architecture 运转的**基石**，参见 [[3-依赖注入]]。

- UseCase（内层）依赖 Repository **接口**（定义在 domain 层）
- RepositoryImpl（外层）实现该接口
- 通过 DI 框架在运行时绑定

```java
@Module
@InstallIn(SingletonComponent.class)
public abstract class RepositoryModule {
    @Binds @Singleton
    public abstract UserRepository bindUserRepository(UserRepositoryImpl impl);

    @Binds @Singleton
    public abstract ArticleRepository bindArticleRepository(ArticleRepositoryImpl impl);
}
```

### 2.5 完整项目结构示例

```
news-app/
├── app/di/                           ← Hilt DI 模块
├── data/
│   ├── remote/api/NewsApi.java
│   ├── remote/dto/NewsDto.java
│   ├── local/db/NewsDao.java
│   ├── local/entity/NewsEntity.java
│   ├── mapper/NewsMapper.java
│   └── repository/NewsRepositoryImpl.java
├── domain/
│   ├── model/News.java
│   ├── repository/NewsRepository.java
│   └── usecase/GetNewsListUseCase.java
└── presentation/
    ├── newslist/NewsListViewModel.java
    └── newslist/NewsListActivity.java
```

数据流向：`UI → ViewModel → UseCase → Repository接口 → RepositoryImpl → DataSource → 逐层回传`

---

## 三、面试题

### Q1: 什么是 Clean Architecture？核心原则是什么？

**答：** Clean Architecture 是 Uncle Bob 提出的架构思想，核心是通过同心圆分层（Entities → Use Cases → Interface Adapters → Frameworks）和**依赖规则**（源码依赖只能从外层指向内层）实现业务逻辑与技术细节的解耦。核心原则包括：依赖规则（向内依赖）、关注点分离、依赖反转（内层定义接口，外层实现）、可测试性（业务逻辑脱离框架可单测）。它与 [[1-MVC-MVP-MVVM-MVI]] 是互补关系——Clean Architecture 关注整体系统架构，MVVM/MVI 关注表现层组织。

### Q2: 依赖规则如何通过依赖反转实现？

**答：** UseCase（内层）需要调用数据库/网络（外层）获取数据，看似违反依赖规则。解决方案是 DIP：在 domain 层定义 Repository 接口，data 层提供实现，通过 [[3-依赖注入]] 在运行时绑定。这样 domain 层的编译依赖不会指向 data 层。

```java
// domain 层定义接口
public interface OrderRepository {
    Order getOrderById(String orderId);
}
// data 层实现
public class OrderRepositoryImpl implements OrderRepository {
    @Override
    public Order getOrderById(String orderId) { /* 访问网络/数据库 */ }
}
// UseCase 只依赖接口
public class GetOrderUseCase {
    private final OrderRepository repository;
    public GetOrderUseCase(OrderRepository repository) { this.repository = repository; }
    public Order execute(String orderId) { return repository.getOrderById(orderId); }
}
```

### Q3: 每层为什么要用不同的数据模型？

**答：** 为了解耦。`NewsDto` 对应 API 的 JSON 结构，`NewsEntity` 对应数据库表（含 Room 注解），`News` 是纯业务模型，`NewsUiModel` 是 UI 展示格式。共用模型会导致 API 变化影响 UI、数据库注解污染业务模型。但对于简单项目，可以适当共用——务实比教条更重要。

### Q4: UseCase 只有一行代码还有必要存在吗？

**答：** 有争议但有价值：①统一业务入口，ViewModel 调用方式一致；②未来扩展点（加缓存、校验、埋点）；③独立测试单元；④命名本身就是业务文档。但在小型项目中，如果确定业务不会复杂化，可以省略 UseCase，让 ViewModel 直接调用 Repository。

### Q5: Clean Architecture 如何提升可测试性？

**答：** 分层和依赖反转使每层可独立测试。UseCase 只依赖 Repository 接口，用 Mock 即可测试：

```java
@Test
public void execute_withValidId_returnsUser() {
    UserRepository mock = mock(UserRepository.class);
    when(mock.getUserById("1")).thenReturn(new User("1", "张三", "test@example.com"));

    GetUserProfileUseCase useCase = new GetUserProfileUseCase(mock);
    User result = useCase.execute("1");

    assertEquals("张三", result.getName());
}
```

对比传统架构：业务逻辑在 Activity 中需要 Instrumented Test，慢且不稳定。Clean Architecture 让 80% 以上逻辑可用纯 JUnit 测试。

### Q6: 多模块项目中各层如何划分模块？

**答：** 推荐将各层拆为独立 Gradle 模块：`:core:domain`（纯 Java 模块，无 Android 依赖）、`:core:data`（Android Library）、`:feature:xxx`（功能模块含 presentation）、`:app`（壳模块组装）。好处是编译隔离、强制依赖规则（编译期检查）、团队可独立开发。

---

## 四、实战与踩坑

### 4.1 过度设计的坑

```java
// ❌ 坑1：每个 CRUD 操作都建 UseCase，一个 User 产生 8 个类
// ✅ 简单查询可合并为一个 UseCase

// ❌ 坑2：简单配置项也要四层模型 + Mapper
// ✅ 简单模型可跨层共用

// ❌ 坑3：为 DataSource 也在 domain 层定义接口
// ✅ DataSource 是 data 层内部细节，domain 只需 Repository 接口
```

### 4.2 小项目是否需要 Clean Architecture？

| 项目规模 | 推荐架构 | 理由 |
|---------|---------|------|
| Demo / 原型 | 简单 MVVM | 快速验证 |
| 小型（< 20 页面） | MVVM + Repository | 不需要 UseCase |
| 中型（20-50 页面） | 简化版 Clean Arch | 引入 UseCase，模型适当共用 |
| 大型（> 50 页面） | 完整 Clean Arch + 多模块 | 完整分层保障可维护性 |

**渐进式演进**：先 MVVM + Repository，需要时再加 UseCase，比一开始就上全套更合理。

### 4.3 与 MVVM / MVI 的结合

Clean Architecture 定义**整体系统架构**，[[1-MVC-MVP-MVVM-MVI]] 中的 MVVM/MVI 定义**表现层架构**，天然互补。

**Clean Arch + MVVM**（最主流组合）：

```java
public class NewsListViewModel extends ViewModel {
    private final GetNewsListUseCase useCase;
    private final MutableLiveData<List<NewsUiModel>> newsLiveData = new MutableLiveData<>();

    public NewsListViewModel(GetNewsListUseCase useCase) { this.useCase = useCase; }

    public void loadNews(int page) {
        List<News> list = useCase.execute(page, 20);
        List<NewsUiModel> uiModels = new ArrayList<>();
        for (News n : list) {
            uiModels.add(new NewsUiModel(n.getTitle(), formatDate(n.getPublishTime())));
        }
        newsLiveData.setValue(uiModels);
    }
}
```

**Clean Arch + MVI**（StateFlow/sealed class 为 Kotlin 特有）：

```kotlin
data class NewsUiState(
    val newsList: List<NewsUiModel> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class NewsListViewModel(private val useCase: GetNewsListUseCase) : ViewModel() {
    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    fun loadNews(page: Int) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true) }
            useCase(Pair(page, 20))
                .onSuccess { list -> _uiState.update { it.copy(newsList = list.map { n -> n.toUiModel() }, isLoading = false) } }
                .onFailure { e -> _uiState.update { it.copy(error = e.message, isLoading = false) } }
        }
    }
}
```

### 4.4 实战建议

1. **从 Repository 模式开始**——即使不用完整 Clean Arch 也值得采用
2. **domain 层保持纯净**——不引入 Android 依赖
3. **UseCase 命名体现业务**——`GetUserProfileUseCase` 而非 `UserUseCase`
4. **Mapper 按需创建**——简单模型不必强制分离
5. **测试驱动验证架构**——某层难以单测说明架构有问题
6. **团队共识优先**——架构选型要团队一起决定

---

> Clean Architecture 不是银弹，它是一种**思维方式**——让你始终思考：哪些是核心业务？哪些是可替换的细节？依赖方向是否正确？掌握这种思维，比死记分层结构更重要。

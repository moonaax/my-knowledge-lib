# Jetpack - Room 与 Navigation

## 一、Room

### 1.1 是什么？

Room 是 Google 官方的 SQLite 抽象层，通过注解在编译时生成数据库操作代码，类型安全、编译时校验 SQL。

### 1.2 三大核心组件

````kotlin
// 1. Entity（实体）→ 对应数据库中的一张表
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    @ColumnInfo(name = "user_name") val name: String,
    val age: Int,
    @Ignore val tempData: String = ""  // 不存入数据库
)

// 2. Dao（数据访问对象）→ 定义数据库操作
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAll(): Flow<List<User>>  // 返回 Flow，数据变化自动通知

    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getById(userId: Int): User?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)

    @Update
    suspend fun update(user: User)

    @Delete
    suspend fun delete(user: User)

    @Query("DELETE FROM users")
    suspend fun deleteAll()
}

// 3. Database（数据库）→ 数据库持有者
@Database(entities = [User::class], version = 1, exportSchema = true)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                ).build().also { INSTANCE = it }
            }
        }
    }
}
````
### 1.3 数据库迁移

````java
// 版本 1 → 版本 2：新增 email 字段
static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(@NonNull SupportSQLiteDatabase db) {
        db.execSQL("ALTER TABLE users ADD COLUMN email TEXT NOT NULL DEFAULT ''");
    }
};

// 构建时添加迁移
Room.databaseBuilder(context, AppDatabase.class, "app_database")
    .addMigrations(MIGRATION_1_2)
    .build();

// 自动迁移（Room 2.4+，简单场景）
@Database(
    entities = {User.class},
    version = 2,
    autoMigrations = {@AutoMigration(from = 1, to = 2)}
)
public abstract class AppDatabase extends RoomDatabase { }
````
### 1.4 TypeConverter

Room 只支持基本类型，复杂类型需要转换器。

````java
public class Converters {
    @TypeConverter
    public Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }
}

@Database(entities = {User.class}, version = 1)
@TypeConverters(Converters.class)
public abstract class AppDatabase extends RoomDatabase { }
````
### 1.5 关联查询

````kotlin
// 一对多：User 有多个 Post
data class UserWithPosts(
    @Embedded val user: User,
    @Relation(parentColumn = "id", entityColumn = "userId")
    val posts: List<Post>
)

@Dao
interface UserDao {
    @Transaction
    @Query("SELECT * FROM users")
    fun getUsersWithPosts(): Flow<List<UserWithPosts>>
}
````
---

## 二、Navigation

### 2.1 是什么？

Navigation 是 Jetpack 的导航组件，用于管理 Fragment 之间的跳转，支持 Deep Link、动画、参数传递、返回栈管理。

### 2.2 核心概念

| 概念 | 说明 |
|------|------|
| NavGraph | 导航图，定义所有页面和跳转关系（XML 或 Kotlin DSL） |
| NavHost | 导航容器，显示当前页面（NavHostFragment） |
| NavController | 导航控制器，执行跳转操作 |
| Destination | 目的地（Fragment、Activity、Dialog） |
| Action | 跳转动作，定义从哪到哪 |

### 2.3 基本使用

````xml
<!-- nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment">
        <argument
            android:name="itemId"
            app:argType="integer" />
    </fragment>
</navigation>
````
````java
// 跳转
NavHostFragment.findNavController(this)
    .navigate(R.id.action_home_to_detail, new Bundle() {{ putInt("itemId", 123); }});

// Safe Args（推荐，类型安全）
HomeFragmentDirections.ActionHomeToDetail action =
    HomeFragmentDirections.actionHomeToDetail(123);
NavHostFragment.findNavController(this).navigate(action);

// 接收参数
DetailFragmentArgs args = DetailFragmentArgs.fromBundle(getArguments());
int itemId = args.getItemId();
````
### 2.4 Deep Link

````xml
<!-- nav_graph.xml 中配置 -->
<fragment android:id="@+id/detailFragment">
    <deepLink app:uri="myapp://detail/{itemId}" />
</fragment>

<!-- AndroidManifest.xml 中添加 -->
<activity android:name=".MainActivity">
    <nav-graph android:value="@navigation/nav_graph" />
</activity>
````
---

## 三、面试题库

### Q1：Room 的原理是什么？和直接用 SQLite 有什么区别？

> Room 是 SQLite 的抽象层，通过注解处理器（APT）在编译时生成 Dao 接口的实现代码。和直接用 SQLiteOpenHelper 相比，Room 的优势是编译时校验 SQL 语句（写错了编译就报错）、类型安全（不需要手动操作 Cursor）、原生支持 Flow/LiveData 实现数据变化监听、支持数据库迁移管理。底层还是 SQLite，Room 只是在上面做了一层更安全更方便的封装。生成的代码可以在 build/generated 目录下看到，面试时能说出这一点会加分。

### Q2：Room 的数据库迁移怎么做？如果忘了写迁移会怎样？

> Room 通过 Migration 对象定义版本间的迁移逻辑，在 databaseBuilder 中通过 addMigrations 添加。如果版本号变了但没有提供对应的 Migration，App 启动时会直接崩溃（IllegalStateException）。如果不想写迁移可以用 fallbackToDestructiveMigration()，但这会删除所有数据重建数据库，只适合开发阶段。Room 2.4+ 支持 AutoMigration，对于简单的变更（加字段、加表）可以自动生成迁移代码，复杂变更（改字段类型、重命名）还是需要手动写。

### Q3：Room 怎么实现数据变化自动通知 UI？

> Room 的 Dao 方法返回 Flow 或 LiveData 时，底层会注册一个 InvalidationTracker。当数据库表发生写操作（insert/update/delete）时，InvalidationTracker 检测到表被修改，自动重新执行查询并发射新数据。这是通过 SQLite 的触发器机制实现的。所以你在一个地方插入数据，另一个地方观察的 Flow 会自动收到更新，不需要手动刷新。

### Q4：Navigation 组件的优势是什么？有什么局限？

> 优势：可视化的导航图、Safe Args 类型安全传参、统一的返回栈管理、内置 Deep Link 支持、转场动画配置简单。局限：学习成本较高、大型项目导航图复杂难以维护、不适合动态路由（组件化场景下通常用 ARouter）、Fragment 的创建和销毁由 Navigation 控制，灵活性不如手动管理。实际项目中如果是单模块应用推荐用 Navigation，多模块组件化项目通常用 ARouter 或自定义路由方案。

### Q5：Room 中 @Embedded 和 @Relation 的区别？

> @Embedded 是把一个对象的字段展开到当前表中，相当于字段内联，不会创建新表。@Relation 是定义表之间的关联关系（一对一、一对多），需要配合 @Transaction 使用确保查询的原子性。@Embedded 适合把一个大实体拆分成多个小对象方便管理，@Relation 适合跨表查询。

### Q6：Room 为什么不允许在主线程执行数据库操作？

> 数据库操作是 IO 操作，可能耗时较长，在主线程执行会阻塞 UI 导致卡顿甚至 ANR。Room 默认会检查当前线程，如果在主线程调用同步查询会抛 IllegalStateException。解决方案是用 suspend 函数配合协程在 IO 调度器上执行，或者返回 Flow/LiveData 让 Room 自动在后台线程查询。虽然可以用 allowMainThreadQueries() 关闭检查，但这只应该在测试或 demo 中使用。

### Q7：Navigation 的 Safe Args 是什么？

> Safe Args 是 Navigation 的 Gradle 插件，根据导航图中定义的 argument 自动生成类型安全的参数类和 Direction 类。比如定义了 itemId: Int 参数，会生成 DetailFragmentArgs 类（包含 itemId 属性）和 HomeFragmentDirections 类（包含 actionHomeToDetail(itemId) 方法）。好处是编译时检查参数类型和是否缺少必要参数，避免运行时因为 key 拼写错误或类型不匹配导致的崩溃。

### Q8：Room 支持哪些返回类型？

> Room 的 Dao 方法支持多种返回类型：普通类型（同步查询，需要在后台线程）、suspend 函数（配合协程）、Flow（数据变化自动通知）、LiveData（同 Flow 但是 Android 专属）、RxJava 的 Observable/Single/Maybe/Completable、Paging 的 PagingSource（分页加载）。推荐用 suspend 函数做增删改，用 Flow 做查询，这样既能异步执行又能自动监听数据变化。

### Q9：Navigation 怎么处理返回栈？popUpTo 和 popUpToInclusive 是什么？

> Navigation 维护了一个 Fragment 返回栈。popUpTo 指定跳转时弹出返回栈到某个目的地，popUpToInclusive 决定是否包含该目的地本身。典型场景：登录成功后跳转到首页，需要清除登录页的返回栈，否则用户按返回键会回到登录页。配置 popUpTo="@id/loginFragment" + popUpToInclusive="true" 就能实现。

### Q10：Room 的 TypeConverter 有什么注意事项？

> TypeConverter 用于将 Room 不支持的类型（Date、List、自定义对象）转换为支持的基本类型存储。注意事项：一是转换器是全局生效的，注册在 Database 上会影响所有 Entity；二是 List 类型通常转成 JSON 字符串存储，但这样就失去了 SQL 查询能力，如果需要查询列表中的元素应该用关联表；三是转换器的性能要注意，频繁的序列化反序列化可能影响查询速度。

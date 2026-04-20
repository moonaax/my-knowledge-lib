# Fragment 全解析

## 一、Fragment 是什么？

Fragment 是一个"可复用的 UI 片段"，必须依附于 Activity 存在。你可以把它理解为 Activity 里的一个"子页面"。

**为什么需要 Fragment？**
- 一个 Activity 里展示多个页面（如 ViewPager 的每一页）
- 平板适配（左侧列表 + 右侧详情）
- UI 复用（同一个 Fragment 可以放在不同 Activity 中）
- 更灵活的导航（Navigation 组件基于 Fragment）

---

## 二、生命周期

### 2.1 Fragment 完整生命周期

````
onAttach() → onCreate() → onCreateView() → onViewCreated()
→ onStart() → onResume()
→ onPause() → onStop() → onDestroyView() → onDestroy() → onDetach()
````
| 回调 | 含义 | 典型操作 |
|------|------|---------|
| `onAttach()` | 与 Activity 关联 | 获取 Activity 引用 |
| `onCreate()` | Fragment 被创建 | 初始化非 UI 数据（ViewModel、参数） |
| `onCreateView()` | 创建视图 | inflate 布局，返回 View |
| `onViewCreated()` | 视图已创建 | 初始化 UI 控件、设置监听器 |
| `onStart()` | 可见 | — |
| `onResume()` | 可交互 | — |
| `onPause()` | 失去焦点 | — |
| `onStop()` | 不可见 | — |
| `onDestroyView()` | 视图被销毁 | 清理 View 引用（避免内存泄漏） |
| `onDestroy()` | Fragment 被销毁 | 释放资源 |
| `onDetach()` | 与 Activity 解除关联 | — |

### 2.2 与 Activity 生命周期对比

````
Activity.onCreate()
    ↓
Fragment.onAttach()
Fragment.onCreate()
Fragment.onCreateView()
Fragment.onViewCreated()
    ↓
Activity.onStart()
    ↓
Fragment.onStart()
    ↓
Activity.onResume()
    ↓
Fragment.onResume()

--- 销毁时反过来 ---

Fragment.onPause()
    ↓
Activity.onPause()
    ↓
Fragment.onStop()
    ↓
Activity.onStop()
    ↓
Fragment.onDestroyView()
Fragment.onDestroy()
Fragment.onDetach()
    ↓
Activity.onDestroy()
````
### 2.3 View 生命周期 vs Fragment 生命周期

这是一个容易忽略的重点：**Fragment 的 View 和 Fragment 本身的生命周期是分开的。**

````
Fragment 实例存活期间：onCreate ─────────────────────────── onDestroy
View 存活期间：          onCreateView ── onDestroyView
                                        （View 可能被销毁重建多次）
````
典型场景：ViewPager 中的 Fragment，滑走后 View 被销毁（onDestroyView），但 Fragment 实例还在（没有 onDestroy）。滑回来时重新走 onCreateView。

**所以：**
- 在 `onCreateView/onViewCreated` 中初始化 UI
- 在 `onDestroyView` 中清理 View 引用
- 用 `viewLifecycleOwner` 而不是 `this` 来观察 LiveData

````java
// ✅ 正确：用 viewLifecycleOwner
viewModel.getData().observe(getViewLifecycleOwner(), data -> {
    textView.setText(data);
});

// ❌ 错误：用 this（Fragment 实例），View 销毁后还在观察，可能 NPE
viewModel.getData().observe(this, data -> {
    textView.setText(data);  // View 可能已经销毁了！
});
````
---

## 三、FragmentManager 与事务

### 3.1 FragmentManager

| 类型 | 获取方式 | 管理范围 |
|------|---------|---------|
| Activity 的 | `supportFragmentManager` | Activity 中的 Fragment |
| Fragment 的 | `childFragmentManager` | Fragment 中嵌套的子 Fragment |
| 父级的 | `parentFragmentManager` | 当前 Fragment 所在的 FragmentManager |

### 3.2 Fragment 事务

````java
// 添加
getSupportFragmentManager().beginTransaction()
    .add(R.id.container, new MyFragment())
    .addToBackStack("tag")  // 加入返回栈
    .commit();

// 替换
getSupportFragmentManager().beginTransaction()
    .replace(R.id.container, new NewFragment())
    .addToBackStack(null)
    .commit();

// 移除
getSupportFragmentManager().beginTransaction()
    .remove(fragment)
    .commit();
````
**add vs replace：**

| 操作 | 行为 | 原 Fragment |
|------|------|------------|
| `add` | 在容器中添加新 Fragment | 保留，被覆盖（还在） |
| `replace` | 移除容器中所有 Fragment，添加新的 | 被移除（走 onDestroyView） |

**commit 的几种方式：**

| 方法 | 特点 |
|------|------|
| `commit()` | 异步提交，在下一帧执行。不能在 onSaveInstanceState 之后调用 |
| `commitAllowingStateLoss()` | 异步提交，允许状态丢失（不会抛异常） |
| `commitNow()` | 同步立即执行，不能和 addToBackStack 一起用 |
| `commitNowAllowingStateLoss()` | 同步 + 允许状态丢失 |

### 3.3 返回栈（Back Stack）

````java
// 添加到返回栈
.addToBackStack("name");

// 按返回键时，系统会弹出栈顶事务（恢复之前的 Fragment）
// 如果没有加入返回栈，按返回键直接关闭 Activity

// 手动弹出
getSupportFragmentManager().popBackStack();

// 弹出到指定位置
getSupportFragmentManager().popBackStack("name", FragmentManager.POP_BACK_STACK_INCLUSIVE);
````
---

## 四、Fragment 通信

### 4.1 共享 ViewModel（推荐）

````java
// 共享 ViewModel（作用域为 Activity）
public class SharedViewModel extends ViewModel {
    private final MutableLiveData<String> selectedItem = new MutableLiveData<>();
    public MutableLiveData<String> getSelectedItem() { return selectedItem; }
}

// Fragment A（发送方）
public class FragmentA extends Fragment {
    private SharedViewModel sharedVM;

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        sharedVM = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
    }

    public void onItemClick(String item) {
        sharedVM.getSelectedItem().setValue(item);
    }
}

// Fragment B（接收方）
public class FragmentB extends Fragment {
    private SharedViewModel sharedVM;

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        sharedVM = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        sharedVM.getSelectedItem().observe(getViewLifecycleOwner(), item -> {
            // 收到数据
        });
    }
}
````
### 4.2 Fragment Result API（Jetpack 推荐）

````java
// Fragment A（接收结果）
getParentFragmentManager().setFragmentResultListener("requestKey", this, (key, bundle) -> {
    String result = bundle.getString("data");
});

// Fragment B（发送结果）
Bundle resultBundle = new Bundle();
resultBundle.putString("data", "hello");
getParentFragmentManager().setFragmentResult("requestKey", resultBundle);
````
### 4.3 接口回调（传统方式）

````java
// 定义接口
public interface OnItemSelectedListener {
    void onItemSelected(String item);
}

// Fragment 中
public class MyFragment extends Fragment {
    private OnItemSelectedListener listener;

    @Override
    public void onAttach(@NonNull Context context) {
        super.onAttach(context);
        if (context instanceof OnItemSelectedListener) {
            listener = (OnItemSelectedListener) context;
        }
    }

    public void onItemClick(String item) {
        if (listener != null) {
            listener.onItemSelected(item);
        }
    }
}

// Activity 实现接口
public class MyActivity extends AppCompatActivity implements OnItemSelectedListener {
    @Override
    public void onItemSelected(String item) {
        // 处理
    }
}
````
### 4.4 通信方式对比

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| 共享 ViewModel | 解耦、生命周期安全 | 需要共同的作用域 | Fragment 之间通信 |
| Result API | 简单、解耦 | 只能传 Bundle | 返回结果 |
| 接口回调 | 直观 | 耦合度高 | 简单场景 |
| EventBus | 完全解耦 | 难以追踪、调试困难 | 不推荐 |

---

## 五、ViewPager2 + Fragment 懒加载

### 5.1 旧方案（ViewPager + setUserVisibleHint）

````java
// 已废弃，但面试可能问
@Override
public void setUserVisibleHint(boolean isVisibleToUser) {
    super.setUserVisibleHint(isVisibleToUser);
    if (isVisibleToUser) {
        // 对用户可见，加载数据
    }
}
````
### 5.2 新方案（ViewPager2 + Lifecycle）

ViewPager2 默认支持基于 Lifecycle 的懒加载，不可见的 Fragment 最多走到 `onStart()`，不会走 `onResume()`。

````java
// ViewPager2 + FragmentStateAdapter
public class MyPagerAdapter extends FragmentStateAdapter {
    public MyPagerAdapter(@NonNull FragmentActivity activity) {
        super(activity);
    }

    @Override
    public int getItemCount() { return 3; }

    @NonNull
    @Override
    public Fragment createFragment(int position) {
        switch (position) {
            case 0: return new HomeFragment();
            case 1: return new ListFragment();
            default: return new ProfileFragment();
        }
    }
}

// Fragment 中，在 onResume 加载数据就是懒加载
public class ListFragment extends Fragment {
    private boolean isLoaded = false;

    @Override
    public void onResume() {
        super.onResume();
        if (!isLoaded) {
            loadData();
            isLoaded = true;
        }
    }
}
````
**offscreenPageLimit：**
````java
viewPager2.setOffscreenPageLimit(1);  // 预加载左右各1页
// 默认是 OFFSCREEN_PAGE_LIMIT_DEFAULT（-1），使用 RecyclerView 的缓存机制
````
---

## 六、Fragment 的坑

### 6.1 getActivity() 返回 null

Fragment 在 `onDetach()` 之后，`getActivity()` 返回 null。如果异步回调中使用，可能 NPE。

````java
// ❌ 危险
someAsyncCall(result -> {
    requireActivity().runOnUiThread(() -> { ... }); // 可能崩溃
});

// ✅ 安全
someAsyncCall(result -> {
    Activity activity = getActivity();
    if (activity != null) {
        activity.runOnUiThread(() -> { ... });
    }
});
````
### 6.2 Fragment 重叠

Activity 被系统回收后重建，会自动恢复之前的 Fragment。如果在 onCreate 中无条件 add Fragment，就会重叠。

````java
// ❌ 错误
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    getSupportFragmentManager().beginTransaction()
        .add(R.id.container, new MyFragment())
        .commit();
}

// ✅ 正确：判断是否是首次创建
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    if (savedInstanceState == null) {
        getSupportFragmentManager().beginTransaction()
            .add(R.id.container, new MyFragment())
            .commit();
    }
}
````
### 6.3 Fragment 必须有无参构造函数

系统恢复 Fragment 时通过反射调用无参构造函数。传参必须用 `arguments`。

````java
// ❌ 错误：有参构造函数，系统恢复时会崩溃
public class MyFragment extends Fragment {
    private int id;
    public MyFragment(int id) { this.id = id; }
}

// ✅ 正确：无参构造 + arguments 传参
public class MyFragment extends Fragment {
    public static MyFragment newInstance(int id) {
        MyFragment fragment = new MyFragment();
        Bundle args = new Bundle();
        args.putInt("id", id);
        fragment.setArguments(args);
        return fragment;
    }

    private int getId() {
        return requireArguments().getInt("id");
    }
}
````
---

## 七、面试题库

### 基础题

**Q1：Fragment 的生命周期？和 Activity 的对应关系？**
> onAttach → onCreate → onCreateView → onViewCreated → onStart → onResume → onPause → onStop → onDestroyView → onDestroy → onDetach。创建时 Fragment 先于 Activity（onStart 之前），销毁时 Fragment 先于 Activity。

**Q2：add 和 replace 的区别？**
> add 是在容器中叠加 Fragment，原来的还在（不走 onDestroyView）。replace 是先移除容器中所有 Fragment 再添加新的（原来的走 onDestroyView）。配合 addToBackStack 使用时，replace 的 Fragment 按返回键可以恢复。

**Q3：commit 和 commitNow 的区别？**
> commit 是异步的，加入消息队列等待执行。commitNow 是同步立即执行，但不能和 addToBackStack 一起用。commit 不能在 onSaveInstanceState 之后调用（会抛异常），commitAllowingStateLoss 可以。

**Q4：为什么 Fragment 必须有无参构造函数？**
> 系统在配置变更（如旋转屏幕）后恢复 Fragment 时，通过反射调用无参构造函数创建实例。传参应该用 arguments（Bundle），系统会自动保存和恢复。

### 通信题

**Q5：Fragment 之间如何通信？**
> 推荐：共享 ViewModel（activityViewModels）或 Fragment Result API。传统方式：接口回调、EventBus（不推荐）。核心原则是 Fragment 之间不应该直接引用对方。

**Q6：什么是 viewLifecycleOwner？为什么要用它？**
> viewLifecycleOwner 是 Fragment 中 View 的生命周期所有者，范围是 onCreateView → onDestroyView。用它观察 LiveData 可以避免 View 销毁后还收到回调导致 NPE。用 this（Fragment 实例）的话，Fragment 实例还在但 View 已销毁时会出问题。

### 进阶题

**Q7：ViewPager2 中 Fragment 的懒加载怎么实现？**
> ViewPager2 基于 Lifecycle，不可见的 Fragment 最多到 STARTED 状态（onStart），不会走 onResume。所以在 onResume 中加载数据就是懒加载。旧的 ViewPager 需要用 setUserVisibleHint（已废弃）。

**Q8：Fragment 重叠问题怎么解决？**
> Activity 被系统回收后重建会自动恢复 Fragment。如果在 onCreate 中无条件 add，就会重叠。解决：判断 savedInstanceState == null 时才 add Fragment。

**Q9：FragmentManager、childFragmentManager、parentFragmentManager 的区别？**
> FragmentManager（supportFragmentManager）管理 Activity 中的 Fragment。childFragmentManager 管理 Fragment 中嵌套的子 Fragment。parentFragmentManager 是当前 Fragment 所在的 FragmentManager（可能是 Activity 的或父 Fragment 的）。

**Q10：Fragment 和 Activity 相比有什么优势？**
> 更轻量、可复用、支持嵌套、适配多屏幕、配合 Navigation 实现单 Activity 架构。缺点是生命周期更复杂、有一些坑（重叠、getActivity 为 null 等）。

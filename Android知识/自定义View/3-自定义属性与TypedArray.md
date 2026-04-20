# 自定义属性与 TypedArray

> 本篇聚焦 Android 自定义 View 中**属性声明、解析与回收**的完整链路，涵盖 `declare-styleable`、`TypedArray` 源码、属性优先级以及常见踩坑。
> 关联笔记：[[自定义View绘制流程]] · [[View的构造函数]] · [[Android资源系统]]

---

## 一、核心概念

### 1.1 declare-styleable 与 attr

在 `res/values/attrs.xml` 中通过 `<declare-styleable>` 为自定义 View 声明属性集合，编译后会在 `R.styleable` 中生成 `int[]` 数组。

````xml
<!-- res/values/attrs.xml -->
<resources>
    <declare-styleable name="CircleImageView">
        <!-- string 类型 -->
        <attr name="civ_title" format="string" />
        <!-- dimension 类型 -->
        <attr name="civ_borderWidth" format="dimension" />
        <!-- reference 类型（引用资源） -->
        <attr name="civ_src" format="reference" />
        <!-- color 类型 -->
        <attr name="civ_borderColor" format="color" />
        <!-- enum 类型：互斥，只能选一个 -->
        <attr name="civ_shape" format="enum">
            <enum name="circle" value="0" />
            <enum name="round_rect" value="1" />
        </attr>
        <!-- flag 类型：可按位或组合 -->
        <attr name="civ_gravity" format="flag">
            <flag name="top" value="0x01" />
            <flag name="bottom" value="0x02" />
            <flag name="center" value="0x10" />
        </attr>
        <!-- 多类型混合 -->
        <attr name="civ_background" format="color|reference" />
    </declare-styleable>
</resources>
````
#### attr format 速查表

| format      | 说明               | 取值方法                        |
| ----------- | ------------------ | ------------------------------- |
| `string`    | 字符串             | `getString()`                   |
| `dimension` | 尺寸 dp/sp/px      | `getDimension()`                |
| `reference` | 资源引用 `@drawable`| `getResourceId()`               |
| `color`     | 颜色值             | `getColor()`                    |
| `integer`   | 整数               | `getInt()` / `getInteger()`     |
| `float`     | 浮点数             | `getFloat()`                    |
| `boolean`   | 布尔               | `getBoolean()`                  |
| `enum`      | 枚举（互斥）       | `getInt()`                      |
| `flag`      | 标志位（可组合）   | `getInt()` 后按位判断           |
| `fraction`  | 百分比             | `getFraction()`                 |

### 1.2 enum vs flag 的本质区别

````java
// enum — 互斥，XML 中只能写一个值
// android:layout_width="match_parent"

// flag — 可用 | 组合
// android:gravity="top|center_horizontal"
````
- `enum`：编译期检查互斥，值不可组合
- `flag`：值按位设计，运行时通过 `and` 判断

````java
int gravity = ta.getInt(R.styleable.CircleImageView_civ_gravity, 0);
boolean isTop = (gravity & 0x01) != 0;
boolean isCenter = (gravity & 0x10) != 0;
````
### 1.3 defStyleAttr vs defStyleRes

[[View的构造函数]] 中四参数构造函数签名：

````java
public class MyView extends View {
    public MyView(Context context) { this(context, null); }
    public MyView(Context context, AttributeSet attrs) { this(context, attrs, 0); }
    public MyView(Context context, AttributeSet attrs, int defStyleAttr) { this(context, attrs, defStyleAttr, 0); }
    public MyView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
}
````
两者的区别：

| 参数           | 含义                                         | 生效条件                          |
| -------------- | -------------------------------------------- | --------------------------------- |
| `defStyleAttr` | 指向当前 Theme 中的一个 attr，间接引用 style | 优先级高于 `defStyleRes`          |
| `defStyleRes`  | 直接指定一个 style 资源 `R.style.xxx`        | 仅当 `defStyleAttr == 0` 或未在 Theme 中定义时生效 |

典型用法：

````xml
<!-- 1. 在 attrs.xml 声明一个 attr 指向 style -->
<attr name="myViewStyle" format="reference" />

<!-- 2. 在 Theme 中赋值 -->
<style name="AppTheme" parent="...">
    <item name="myViewStyle">@style/Widget.MyView</item>
</style>

<!-- 3. 定义默认 style -->
<style name="Widget.MyView">
    <item name="civ_borderWidth">2dp</item>
    <item name="civ_borderColor">#FF0000</item>
</style>
````
````java
public class MyView extends View {
    public MyView(Context context) { this(context, null); }
    public MyView(Context context, AttributeSet attrs) { this(context, attrs, R.attr.myViewStyle); }
    public MyView(Context context, AttributeSet attrs, int defStyleAttr) { this(context, attrs, defStyleAttr, R.style.Widget_MyView); }
    public MyView(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
}
````
---

## 二、原理与源码

### 2.1 obtainStyledAttributes 四参数版本

`Context.obtainStyledAttributes()` 是属性解析的入口，最终调用 `Resources.Theme.obtainStyledAttributes()`：

````java
// android.content.res.Resources.Theme
public TypedArray obtainStyledAttributes(
    AttributeSet set,    // XML 中写的属性
    int[] attrs,         // R.styleable.XxxView 数组
    int defStyleAttr,    // Theme 中的 attr 引用
    int defStyleRes      // 兜底 style 资源
)
````
**源码关键流程**（基于 AOSP API 33）：

````java
// ResourcesImpl.java -> ThemeImpl
public TypedArray resolveAttributes(AttributeSet set, int[] attrs,
        int defStyleAttr, int defStyleRes) {
    // 1. 从 XML AttributeSet 中提取属性值
    // 2. 如果 XML 没有，查找 style="@style/xxx" 中的值
    // 3. 如果 style 没有，查找 defStyleAttr 指向的 Theme attr
    // 4. 如果 defStyleAttr 无效，查找 defStyleRes
    // 5. 最后查找 Theme 中的默认值
    // 底层调用 native: AssetManager.applyStyle()
}
````
在 Kotlin 自定义 View 中的标准用法：

````java
// In constructor
TypedArray ta = context.obtainStyledAttributes(
    attrs,                              // XML 属性集
    R.styleable.CircleImageView,        // 声明的属性数组
    defStyleAttr,                       // Theme attr
    defStyleRes                         // 兜底 style
);

float borderWidth = ta.getDimension(
    R.styleable.CircleImageView_civ_borderWidth, 0f
);
int borderColor = ta.getColor(
    R.styleable.CircleImageView_civ_borderColor, Color.BLACK
);
int shape = ta.getInt(
    R.styleable.CircleImageView_civ_shape, 0
);

ta.recycle(); // 必须回收！
````
### 2.2 属性优先级（从高到低）

````
XML 直接设置  >  style="@style/xxx"  >  defStyleAttr(Theme attr)  >  defStyleRes  >  Theme 默认值
````
用一张表说明：

| 优先级 | 来源                     | 示例                                      |
| ------ | ------------------------ | ----------------------------------------- |
| 1（最高）| XML 属性直接赋值       | `app:civ_borderWidth="4dp"`               |
| 2      | XML 中 `style` 属性     | `style="@style/MyBorder"`                 |
| 3      | `defStyleAttr` 指向的 style | Theme 中 `<item name="myViewStyle">@style/X</item>` |
| 4      | `defStyleRes` 兜底 style | 构造函数中传入 `R.style.Widget_MyView`    |
| 5（最低）| Theme 中直接定义的属性 | `<item name="civ_borderWidth">1dp</item>` |

验证优先级的代码：

````java
// 在 XML 中同时设置直接属性和 style，观察哪个生效
// layout.xml
// <com.example.MyView
//     style="@style/StyleA"           ← borderWidth=2dp
//     app:civ_borderWidth="8dp" />    ← 直接设置 8dp
//
// 结果：borderWidth = 8dp（XML 直接设置优先）
````
### 2.3 TypedArray 的获取与回收机制

#### 对象池设计

`TypedArray` 采用**对象池**（Pool）复用，避免频繁创建对象：

````java
// TypedArray.java (AOSP 简化)
public class TypedArray {
    // 同步池，默认容量 5
    private static final SynchronizedPool<TypedArray> sPool =
        new SynchronizedPool<>(5);

    static TypedArray obtain(Resources res, int len) {
        TypedArray ta = sPool.acquire();
        if (ta == null) {
            ta = new TypedArray(res);
        }
        // 初始化内部数组 ...
        return ta;
    }

    public void recycle() {
        // 清理内部数据
        mData = null;
        mIndices = null;
        mResources = null;
        mRecycled = true;
        sPool.release(this);  // 归还到池中
    }
}
````
#### 回收流程图

````
obtainStyledAttributes()
    ↓
sPool.acquire()  →  池中有对象？ → 复用
                     ↓ 没有
                  new TypedArray()
    ↓
填充属性数据（native applyStyle）
    ↓
返回给调用方使用
    ↓
ta.recycle()
    ↓
清理数据 → sPool.release() → 归还池中
````
### 2.4 use 扩展函数（推荐写法）

AndroidX 提供了 `TypedArray.use {}` 扩展，自动回收：

````java
// In constructor
TypedArray ta = context.obtainStyledAttributes(
    attrs, R.styleable.CircleImageView, defStyleAttr, defStyleRes
);
try {
    borderWidth = ta.getDimension(
        R.styleable.CircleImageView_civ_borderWidth, 0f
    );
    borderColor = ta.getColor(
        R.styleable.CircleImageView_civ_borderColor, Color.BLACK
    );
} finally {
    ta.recycle();
}
// 离开 try-finally 块自动 recycle，无需手动调用
````
`use` 的实现原理：

````kotlin
// androidx.core.content
inline fun <R> TypedArray.use(block: (TypedArray) -> R): R {
    return try {
        block(this)
    } finally {
        recycle()
    }
}
````
---

## 三、面试题

### Q1：TypedArray 为什么要调用 recycle()？不调用会怎样？

**A：** `TypedArray` 内部使用**对象池**（`SynchronizedPool`）复用实例。调用 `recycle()` 将对象归还池中，供下次 `obtainStyledAttributes()` 复用，避免频繁 GC。

不调用的后果：
1. 对象池耗尽后每次都要 `new TypedArray()`，增加内存分配和 GC 压力
2. 在 `StrictMode` 下会抛出 `RuntimeException`
3. 内部持有 `Resources` 引用，可能导致间接内存泄漏

````java
// StrictMode 检测代码
StrictMode.setVmPolicy(
    new StrictMode.VmPolicy.Builder()
        .detectLeakedClosableObjects()
        .penaltyLog()
        .build()
);
````
### Q2：obtainStyledAttributes 的四个参数分别是什么？属性优先级如何？

**A：**

````java
context.obtainStyledAttributes(
    set,          // 1. XML AttributeSet
    attrs,        // 2. R.styleable.XxxView 属性数组
    defStyleAttr, // 3. Theme 中的 attr 引用（间接指向 style）
    defStyleRes   // 4. 兜底 style 资源 ID
);
````
优先级从高到低：**XML 直接属性 > style 属性 > defStyleAttr > defStyleRes > Theme 默认值**。

`defStyleRes` 仅在 `defStyleAttr == 0` 或 Theme 中未定义该 attr 时才生效。

### Q3：enum 和 flag 有什么区别？分别在什么场景使用？

**A：**

| 对比项   | enum             | flag                    |
| -------- | ---------------- | ----------------------- |
| 互斥性   | 互斥，只能选一个 | 可按位或组合多个        |
| 值设计   | 任意整数         | 必须是 2 的幂次方       |
| 典型场景 | shape 类型选择   | gravity、文字样式组合   |
| XML 写法 | `app:shape="circle"` | `app:gravity="top\|center"` |

````java
// flag 判断
int flags = ta.getInt(R.styleable.Xxx_myFlag, 0);
boolean hasTop = (flags & FLAG_TOP) != 0;
boolean hasCenter = (flags & FLAG_CENTER) != 0;
````
### Q4：defStyleAttr 和 defStyleRes 的区别？什么时候用哪个？

**A：**

- `defStyleAttr`：指向 Theme 中的一个 `attr`，该 attr 的值是一个 style 引用。好处是**可以通过切换 Theme 改变默认样式**，适合需要主题适配的控件（如 MaterialButton 的 `materialButtonStyle`）。
- `defStyleRes`：直接指定一个 `R.style.xxx`，不受 Theme 影响。适合**不需要主题切换**的固定默认样式。

优先级：`defStyleAttr` > `defStyleRes`。当 `defStyleAttr` 在 Theme 中有定义时，`defStyleRes` 被忽略。

````java
// MaterialButton 的做法：
public class MaterialButton extends AppCompatButton {
    public MaterialButton(Context context) { this(context, null); }
    public MaterialButton(Context context, AttributeSet attrs) {
        this(context, attrs, com.google.android.material.R.attr.materialButtonStyle);
    }
    public MaterialButton(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }
}
// 通过 Theme 中的 materialButtonStyle attr 控制默认样式
````
### Q5：自定义属性在 Library 模块中需要注意什么？

**A：**

1. **命名前缀**：必须加前缀（如 `civ_`），避免与宿主 App 或其他 Library 的属性冲突
2. **R 类引用**：Library 中 `R.styleable` 的值不是 `final`（AGP 非 final R），不能用在 `switch-case` 中，需用 `if-else`
3. **属性复用**：如果要复用系统属性（如 `android:text`），在 `declare-styleable` 中引用时不要加 `format`：

````xml
<declare-styleable name="MyView">
    <!-- 复用系统属性，不加 format -->
    <attr name="android:text" />
    <!-- 自定义属性，需要 format -->
    <attr name="mv_radius" format="dimension" />
</declare-styleable>
````
4. **namespace**：Library 发布为 AAR 后，使用方用 `app:` 命名空间即可，无需关心原始包名

### Q6：TypedArray 的 getDimension()、getDimensionPixelSize()、getDimensionPixelOffset() 有什么区别？

**A：**

| 方法                       | 返回类型 | 舍入方式         | 典型用途         |
| -------------------------- | -------- | ---------------- | ---------------- |
| `getDimension()`           | `Float`  | 不舍入           | 需要精确浮点值   |
| `getDimensionPixelSize()`  | `Int`    | 四舍五入，至少 1 | 宽高、padding    |
| `getDimensionPixelOffset()`| `Int`    | 直接截断取整     | 偏移量           |

````java
// 10.5dp 在 2x 屏幕上：
// getDimension()           → 21.0f
// getDimensionPixelSize()  → 21  (四舍五入，且保证 ≥ 1)
// getDimensionPixelOffset()→ 21  (截断取整)

// 0.4dp 在 1x 屏幕上：
// getDimension()           → 0.4f
// getDimensionPixelSize()  → 1   (保证至少为 1)
// getDimensionPixelOffset()→ 0   (截断为 0)
````
---

## 四、实战与踩坑

### 4.1 TypedArray 忘记 recycle 的后果

#### 现象

- Debug 模式下 `StrictMode` 报错：`A resource was acquired but never released`
- Release 模式下不会崩溃，但对象池无法回收，导致**内存抖动**
- 高频创建 View（如 RecyclerView Item）时尤为明显

#### 根因分析

````java
// TypedArray.recycle() 源码
public void recycle() {
    if (mRecycled) {
        throw new RuntimeException("Already recycled!");
    }
    mRecycled = true;
    // 清空内部数组引用
    mXml = null;
    mTheme = null;
    mAssets = null;
    sPool.release(this);  // 关键：归还池
}
````
不调用 `recycle()` → 对象不归还池 → 池中可用对象减少 → 后续每次都 `new` → GC 压力增大。

#### 解决方案

始终使用 `use` 扩展函数，从根本上杜绝遗漏：

````java
TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.MyView);
try {
    // 在这里读取属性
    myColor = ta.getColor(R.styleable.MyView_mv_color, Color.GRAY);
} finally {
    ta.recycle();
}
// 自动 recycle，即使发生异常也能保证回收
````
#### Lint 检查

Android Lint 内置了 `Recycle` 检查规则，会在未调用 `recycle()` 时发出警告：

````
Warning: This TypedArray should be recycled after use with #recycle()
````
### 4.2 自定义属性命名冲突

#### 场景一：两个 Library 定义了同名 attr

````xml
<!-- library-a 的 attrs.xml -->
<attr name="title" format="string" />

<!-- library-b 的 attrs.xml -->
<attr name="title" format="reference" />
````
编译报错：`Attribute "title" has already been defined`。

#### 解决方案

1. **加前缀**（推荐）：所有自定义属性加模块前缀

````xml
<declare-styleable name="LibAView">
    <attr name="liba_title" format="string" />
</declare-styleable>

<declare-styleable name="LibBView">
    <attr name="libb_title" format="reference" />
</declare-styleable>
````
2. **抽取公共 attr**：如果两个 Library 确实需要同一个属性，抽到公共模块

````xml
<!-- common/attrs.xml -->
<attr name="common_title" format="string" />

<!-- library-a 引用，不重复声明 format -->
<declare-styleable name="LibAView">
    <attr name="common_title" />
</declare-styleable>
````
#### 场景二：与系统属性冲突

````xml
<!-- 错误：重新定义了 android:text 的 format -->
<attr name="android:text" format="string" />

<!-- 正确：引用系统属性，不加 format -->
<declare-styleable name="MyView">
    <attr name="android:text" />
</declare-styleable>
````
### 4.3 在 Library 中使用自定义属性的注意事项

#### 4.3.1 非 final R 的影响

AGP 中 Library 模块的 `R` 类字段不是 `final`，因此：

````java
// ❌ 编译错误：R.styleable.MyView_xxx 不是常量，不能用于 switch
switch (index) {
    case R.styleable.MyView_mv_color: /* ... */ break;
}

// ✅ 使用 if-else 替代
if (index == R.styleable.MyView_mv_color) { /* ... */ }
````
> 从 AGP 8.0 起可通过 `android.nonFinalResIds=false` 恢复 final R，但官方不推荐。

#### 4.3.2 属性命名规范

````
模块缩写_属性名
````
示例：

| 模块              | 前缀   | 属性示例              |
| ----------------- | ------ | --------------------- |
| CircleImageView   | `civ_` | `civ_borderWidth`     |
| RatingBarView     | `rbv_` | `rbv_starCount`       |
| LoadingButton     | `lb_`  | `lb_loadingColor`     |

#### 4.3.3 完整实战示例

一个完整的自定义 View 属性声明与解析流程：

````xml
<!-- res/values/attrs.xml -->
<resources>
    <attr name="progressBarStyle" format="reference" />

    <declare-styleable name="GradientProgressBar">
        <attr name="gpb_progress" format="integer" />
        <attr name="gpb_max" format="integer" />
        <attr name="gpb_startColor" format="color" />
        <attr name="gpb_endColor" format="color" />
        <attr name="gpb_cornerRadius" format="dimension" />
        <attr name="gpb_animDuration" format="integer" />
    </declare-styleable>
</resources>
````
````xml
<!-- res/values/styles.xml -->
<style name="Widget.GradientProgressBar">
    <item name="gpb_progress">0</item>
    <item name="gpb_max">100</item>
    <item name="gpb_startColor">#FF6B6B</item>
    <item name="gpb_endColor">#FFA500</item>
    <item name="gpb_cornerRadius">4dp</item>
    <item name="gpb_animDuration">300</item>
</style>

<!-- Theme 中关联 -->
<style name="AppTheme" parent="Theme.Material3.Light">
    <item name="progressBarStyle">@style/Widget.GradientProgressBar</item>
</style>
````
````java
public class GradientProgressBar extends View {

    private int progress = 0;
    private int max = 100;
    private int startColor = Color.RED;
    private int endColor = Color.YELLOW;
    private float cornerRadius = 0f;
    private int animDuration = 300;

    private final Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final RectF rect = new RectF();

    public GradientProgressBar(Context context) { this(context, null); }
    public GradientProgressBar(Context context, AttributeSet attrs) { this(context, attrs, R.attr.progressBarStyle); }
    public GradientProgressBar(Context context, AttributeSet attrs, int defStyleAttr) { this(context, attrs, defStyleAttr, R.style.Widget_GradientProgressBar); }
    public GradientProgressBar(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        TypedArray ta = context.obtainStyledAttributes(
            attrs, R.styleable.GradientProgressBar, defStyleAttr, defStyleRes
        );
        try {
            progress = ta.getInt(R.styleable.GradientProgressBar_gpb_progress, 0);
            max = ta.getInt(R.styleable.GradientProgressBar_gpb_max, 100);
            startColor = ta.getColor(
                R.styleable.GradientProgressBar_gpb_startColor, Color.RED
            );
            endColor = ta.getColor(
                R.styleable.GradientProgressBar_gpb_endColor, Color.YELLOW
            );
            cornerRadius = ta.getDimension(
                R.styleable.GradientProgressBar_gpb_cornerRadius, 0f
            );
            animDuration = ta.getInt(
                R.styleable.GradientProgressBar_gpb_animDuration, 300
            );
        } finally {
            ta.recycle();
        }
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        float ratio = (float) progress / max;
        rect.set(0f, 0f, getWidth() * ratio, (float) getHeight());
        paint.setShader(new LinearGradient(
            0f, 0f, rect.right, 0f,
            startColor, endColor, Shader.TileMode.CLAMP
        ));
        canvas.drawRoundRect(rect, cornerRadius, cornerRadius, paint);
    }

    public void setProgress(int value, boolean animate) {
        int clamped = Math.max(0, Math.min(value, max));
        if (animate) {
            ValueAnimator animator = ValueAnimator.ofInt(progress, clamped);
            animator.setDuration(animDuration);
            animator.addUpdateListener(it -> {
                progress = (int) it.getAnimatedValue();
                invalidate();
            });
            animator.start();
        } else {
            progress = clamped;
            invalidate();
        }
    }

    public void setProgress(int value) {
        setProgress(value, true);
    }
}
````
XML 中使用：

````xml
<com.example.widget.GradientProgressBar
    android:layout_width="match_parent"
    android:layout_height="8dp"
    app:gpb_progress="60"
    app:gpb_startColor="#4CAF50"
    app:gpb_endColor="#8BC34A"
    app:gpb_cornerRadius="4dp" />
````
### 4.4 其他常见踩坑

#### 踩坑 1：在 RecyclerView.ViewHolder 中重复解析属性

````java
// ❌ 每次 onBindViewHolder 都解析属性 — 浪费性能
@Override
public void onBindViewHolder(VH holder, int position) {
    // 不要在这里 obtainStyledAttributes
}

// ✅ 属性解析应在 View 的构造函数中一次性完成
````
#### 踩坑 2：recycle 后继续使用 TypedArray

````java
TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.MyView);
ta.recycle();
// ❌ recycle 后再读取会崩溃或返回错误值
int color = ta.getColor(R.styleable.MyView_mv_color, 0);
````
#### 踩坑 3：多类型 format 取值方式

````xml
<attr name="mv_bg" format="color|reference" />
````
````java
// 需要先尝试 getResourceId，再 fallback 到 getColor
TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.MyView);
try {
    int resId = ta.getResourceId(R.styleable.MyView_mv_bg, 0);
    Drawable bg;
    if (resId != 0) {
        bg = ContextCompat.getDrawable(context, resId);
    } else {
        bg = new ColorDrawable(ta.getColor(R.styleable.MyView_mv_bg, Color.WHITE));
    }
} finally {
    ta.recycle();
}
````
---

## 参考

- [[自定义View绘制流程]]
- [[View的构造函数]]
- [[Android资源系统]]
- [AOSP TypedArray 源码](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/core/java/android/content/res/TypedArray.java)
- [Android Developers - Create custom view components](https://developer.android.com/develop/ui/views/layout/custom-views/create-view)

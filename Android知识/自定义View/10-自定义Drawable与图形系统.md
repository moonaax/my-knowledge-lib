# 自定义 Drawable 与图形系统

> Drawable 是 Android 图形系统的基石，理解 Drawable 体系对自定义 UI 和性能优化至关重要。
> 关联：[[2-Canvas-bindPaint-bindPath实战]] · [[9-Paint进阶与特效]] · [[4-动画-属性动画-插值器-估值器]]

---

## 一、Drawable 体系概览

### 1.1 什么是 Drawable

Drawable 是 Android 中**可绘制对象**的抽象基类，它不一定是图片——它可以是颜色、形状、动画、状态列表等任何可以画到 Canvas 上的东西。

```
Drawable（抽象基类）
├── BitmapDrawable        → 图片
├── ShapeDrawable         → 形状（矩形、圆、线）
├── GradientDrawable      → 渐变形状（XML shape 标签）
├── ColorDrawable         → 纯色
├── StateListDrawable     → 状态列表（selector）
├── LayerDrawable         → 图层叠加
├── LevelListDrawable     → 等级列表
├── TransitionDrawable    → 渐变过渡
├── InsetDrawable         → 内边距
├── ClipDrawable          → 裁剪
├── ScaleDrawable         → 缩放
├── RotateDrawable        → 旋转
├── AnimationDrawable     → 帧动画
├── VectorDrawable        → 矢量图
├── AnimatedVectorDrawable → 矢量动画
└── 自定义 Drawable        → 继承 Drawable 实现
```

### 1.2 Drawable vs View

| 维度 | Drawable | View |
|------|---------|------|
| 能否交互 | 不能（纯绘制） | 能（事件处理） |
| 内存占用 | 轻量 | 重量（有 measure/layout） |
| 使用场景 | 背景、图标、分割线 | 可交互的 UI 元素 |
| 绘制方式 | `draw(Canvas)` | `onDraw(Canvas)` |

**什么时候用 Drawable**：只需要显示，不需要交互——背景、图标、分割线、角标等。

---

## 二、常用 Drawable 详解

### 2.1 GradientDrawable（XML shape）

最常用的 Drawable，对应 XML 中的 `<shape>` 标签。

```xml
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">
    <solid android:color="#FF6200EE" />
    <corners android:radius="8dp" />
    <stroke android:width="1dp" android:color="#333333" />
    <padding android:left="16dp" android:top="8dp"
             android:right="16dp" android:bottom="8dp" />
    <gradient
        android:startColor="#FF6200EE"
        android:endColor="#FF03DAC5"
        android:angle="45" />
</shape>
```

**Java 代码动态创建**：

```java
GradientDrawable drawable = new GradientDrawable();
drawable.setShape(GradientDrawable.RECTANGLE);
drawable.setColor(0xFF6200EE);
drawable.setCornerRadius(dp2px(8));
drawable.setStroke(dp2px(1), 0xFF333333);
view.setBackground(drawable);
```

### 2.2 StateListDrawable（selector）

根据 View 的状态显示不同的 Drawable。

```xml
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:state_pressed="true" android:drawable="@drawable/btn_pressed" />
    <item android:state_focused="true" android:drawable="@drawable/btn_focused" />
    <item android:state_enabled="false" android:drawable="@drawable/btn_disabled" />
    <item android:drawable="@drawable/btn_normal" /> <!-- 默认 -->
</selector>
```

**代码动态创建**：

```java
StateListDrawable sld = new StateListDrawable();
sld.addState(new int[]{android.R.attr.state_pressed}, pressedDrawable);
sld.addState(new int[]{android.R.attr.state_enabled}, normalDrawable);
sld.addState(new int[]{}, defaultDrawable);
view.setBackground(sld);
```

### 2.3 LayerDrawable（图层叠加）

多个 Drawable 从下到上叠加。

```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/bg_shadow" />  <!-- 底层阴影 -->
    <item android:drawable="@drawable/bg_main"       <!-- 主体 -->
          android:left="2dp" android:top="2dp" />
    <item android:drawable="@drawable/ic_icon"       <!-- 图标 -->
          android:gravity="center" />
</layer-list>
```

### 2.4 ClipDrawable（裁剪）

按水平/垂直方向裁剪 Drawable，常用作进度条。

```xml
<clip xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/progress_fill"
    android:clipOrientation="horizontal"
    android:gravity="left">
</clip>
```

```java
// level 范围 0-10000
clipDrawable.setLevel(5000); // 显示 50%
```

---

## 三、自定义 Drawable

### 3.1 基本模板

```java
public class CircleDrawable extends Drawable {

    private Paint paint;
    private int color;

    public CircleDrawable(int color) {
        this.color = color;
        paint = new Paint(Paint.ANTI_ALIAS_FLAG);
        paint.setColor(color);
    }

    @Override
    public void draw(@NonNull Canvas canvas) {
        Rect bounds = getBounds();
        float cx = bounds.exactCenterX();
        float cy = bounds.exactCenterY();
        float radius = Math.min(bounds.width(), bounds.height()) / 2f;
        canvas.drawCircle(cx, cy, radius, paint);
    }

    @Override
    public void setAlpha(int alpha) {
        paint.setAlpha(alpha);
        invalidateSelf(); // 通知重绘
    }

    @Override
    public void setColorFilter(@Nullable ColorFilter colorFilter) {
        paint.setColorFilter(colorFilter);
        invalidateSelf();
    }

    @Override
    public int getOpacity() {
        return PixelFormat.TRANSLUCENT;
    }
}
```

### 3.2 实战：渐变进度条 Drawable

```java
public class GradientProgressDrawable extends Drawable {

    private Paint paint;
    private float progress = 0f; // 0-1
    private int[] colors = {0xFF00BFFF, 0xFFFF6B6B};

    public GradientProgressDrawable() {
        paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    }

    @Override
    public void draw(@NonNull Canvas canvas) {
        Rect bounds = getBounds();
        float width = bounds.width() * progress;

        // 渐变着色器
        LinearGradient gradient = new LinearGradient(
            0, 0, bounds.width(), 0,
            colors, null, Shader.TileMode.CLAMP);
        paint.setShader(gradient);

        // 画圆角矩形
        float radius = bounds.height() / 2f;
        canvas.drawRoundRect(
            bounds.left, bounds.top,
            bounds.left + width, bounds.bottom,
            radius, radius, paint);
    }

    public void setProgress(float progress) {
        this.progress = Math.max(0, Math.min(1, progress));
        invalidateSelf();
    }

    @Override
    public void setAlpha(int alpha) { paint.setAlpha(alpha); }

    @Override
    public void setColorFilter(ColorFilter cf) { paint.setColorFilter(cf); }

    @Override
    public int getOpacity() { return PixelFormat.TRANSLUCENT; }
}
```

### 3.3 实战：带头像的圆形 Drawable

```java
public class AvatarDrawable extends Drawable {

    private Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private Paint borderPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private Paint textPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private Bitmap bitmap;
    private String name;
    private int borderColor = Color.WHITE;
    private float borderWidth = dp2px(2);

    public AvatarDrawable(Bitmap bitmap, String name) {
        this.bitmap = bitmap;
        this.name = name;
        borderPaint.setStyle(Paint.Style.STROKE);
        borderPaint.setStrokeWidth(borderWidth);
        borderPaint.setColor(borderColor);
        textPaint.setColor(Color.WHITE);
        textPaint.setTextSize(dp2px(24));
        textPaint.setTextAlign(Paint.Align.CENTER);
    }

    @Override
    public void draw(@NonNull Canvas canvas) {
        Rect bounds = getBounds();
        float cx = bounds.exactCenterX();
        float cy = bounds.exactCenterY();
        float radius = Math.min(bounds.width(), bounds.height()) / 2f - borderWidth;

        if (bitmap != null) {
            // 有头像：画圆形头像
            BitmapShader shader = new BitmapShader(bitmap,
                Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
            Matrix matrix = new Matrix();
            float scale = (radius * 2) / Math.min(bitmap.getWidth(), bitmap.getHeight());
            matrix.setScale(scale, scale);
            shader.setLocalMatrix(matrix);
            paint.setShader(shader);
            canvas.drawCircle(cx, cy, radius, paint);
        } else {
            // 无头像：画名字首字
            paint.setColor(getColorFromName(name));
            canvas.drawCircle(cx, cy, radius, paint);

            String firstChar = name != null && !name.isEmpty() ?
                String.valueOf(name.charAt(0)) : "?";
            Paint.FontMetrics fm = textPaint.getFontMetrics();
            float textY = cy - (fm.top + fm.bottom) / 2f;
            canvas.drawText(firstChar, cx, textY, textPaint);
        }

        // 白色边框
        canvas.drawCircle(cx, cy, radius, borderPaint);
    }

    private int getColorFromName(String name) {
        int[] colors = {0xFF4CAF50, 0xFF2196F3, 0xFFFF9800,
                        0xFFE91E63, 0xFF9C27B0, 0xFF00BCD4};
        int hash = name != null ? Math.abs(name.hashCode()) : 0;
        return colors[hash % colors.length];
    }

    private int dp2px(int dp) {
        return (int) (dp * 3 + 0.5f); // 简化，实际用 context
    }

    @Override public void setAlpha(int alpha) {}
    @Override public void setColorFilter(ColorFilter cf) {}
    @Override public int getOpacity() { return PixelFormat.TRANSLUCENT; }
}
```

---

## 四、Drawable 的缓存与复用

### 4.1 getConstantState() 机制

Drawable 的 `getConstantState()` 返回一个 `ConstantState` 对象，可以用来高效复制 Drawable。

```java
// 高效复制（共享 ConstantState，省内存）
Drawable copy = original.getConstantState().newDrawable().mutate();
copy.setTint(Color.RED); // mutate() 确保修改不影响原对象
```

**mutate() 的作用**：默认情况下，从同一个资源加载的 Drawable 共享状态（改一个全变）。mutate() 创建独立副本，修改不会影响其他实例。

### 4.2 Bitmap 复用（BitmapPool）

Glide/Coil 等图片库使用 BitmapPool 复用 Bitmap，减少内存分配。

```java
// 原理：BitmapDrawable 内部的 Bitmap 可以被回收复用
BitmapPool pool = new LruBitmapPool(maxSize);

// 获取可复用的 Bitmap
Bitmap reusable = pool.get(width, height, config);
if (reusable != null) {
    // 复用已有 Bitmap（避免 new 分配）
    reusable.eraseColor(Color.TRANSPARENT);
} else {
    reusable = Bitmap.createBitmap(width, height, config);
}
```

---

## 五、Drawable 与 Tint（着色）

### 5.1 基本用法

```java
// 给 Drawable 着色（改变整体色调）
drawable.setTint(0xFF6200EE);

// XML
android:tint="#6200EE"
```

### 5.2 tintMode

```java
// 着色模式（类似 Xfermode）
drawable.setTintMode(PorterDuff.Mode.SRC_IN);
// SRC_IN  → 只保留着色后的效果（最常用）
// SRC_ATOP → 叠加着色
// MULTIPLY → 颜色相乘
```

### 5.3 实战：图标动态着色

```java
// 根据主题切换图标颜色
Drawable icon = ContextCompat.getDrawable(context, R.drawable.ic_settings);
if (isDarkMode) {
    icon.setTint(Color.WHITE);
} else {
    icon.setTint(Color.BLACK);
}
imageView.setImageDrawable(icon);
```

---

## 六、VectorDrawable 与 AnimatedVectorDrawable

### 6.1 VectorDrawable

矢量图，放大不失真，体积小。

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="#FF000000"
        android:pathData="M12,2C6.48,2 2,6.48 2,12s4.48,10 10,10
                          10,-4.48 10,-10S17.52,2 12,2z
                          M12,17l-5,-5h3V9h4v3h3z"/>
</vector>
```

### 6.2 AnimatedVectorDrawable

矢量图动画——路径描边、填充变化、旋转变换。

```xml
<!-- res/drawable/avd_check.xml -->
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_check_path">
    <target
        android:name="checkmark"
        android:animation="@anim/anim_check_draw" />
</animated-vector>

<!-- res/anim/anim_check_draw.xml -->
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <objectAnimator
        android:propertyName="trimPathEnd"
        android:valueFrom="0"
        android:valueTo="1"
        android:duration="300"
        android:interpolator="@android:interpolator/fast_out_slow_in" />
</set>
```

```java
// 播放
AnimatedVectorDrawable avd = (AnimatedVectorDrawable)
    ContextCompat.getDrawable(this, R.drawable.avd_check);
imageView.setImageDrawable(avd);
avd.start();
```

---

## 七、面试要点

**Q: Drawable 和 Bitmap 的区别？**

Drawable 是抽象类，表示任何可绘制的对象（形状、颜色、图片、动画等），通过 draw(Canvas) 绘制。Bitmap 是具体的像素矩阵。BitmapDrawable 是 Bitmap 的 Drawable 包装。Drawable 不一定是 Bitmap，但 Bitmap 可以包装成 BitmapDrawable。

**Q: mutate() 是做什么的？**

从同一个资源加载的多个 Drawable 实例共享同一个 ConstantState，修改一个会影响所有。mutate() 创建独立的 ConstantState 副本，使修改只影响当前实例。典型场景：列表中不同 item 需要给同一个图标不同颜色。

**Q: 什么时候应该自定义 Drawable 而不是自定义 View？**

当只需要绘制不需要交互时用 Drawable——比如背景、图标、分割线。Drawable 比 View 轻量得多（没有 measure/layout 流程）。需要点击事件、动画联动、复杂布局时用 View。

**Q: VectorDrawable 相比 BitmapDrawable 的优势？**

矢量图放大不失真，体积通常比同尺寸 PNG 小很多，支持 tint 着色（一个图标适配多种颜色），支持 AnimatedVectorDrawable 做路径动画。缺点是复杂图形渲染性能不如 Bitmap。

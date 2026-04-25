# Paint 进阶与特效

> 深入 Paint 的高级能力：Xfermode 混合模式、ColorFilter、MaskFilter、文字特效。这些是实现复杂 UI 效果的关键。
> 关联：[[2-Canvas-bindPaint-bindPath实战]] · [[1-View绘制流程-measure-layout-draw]]

---

## 一、Xfermode（图层混合模式）

Xfermode 控制两个图层重叠时的像素混合方式，是实现圆角头像、擦除效果、倒影等效果的核心。

### 1.1 PorterDuff.Mode 原理

```
源图（Src）：新绘制的内容
目标图（Dst）：Canvas 上已有的内容

16 种混合模式：
┌─────────┬───────┬───────┬───────┬───────┐
│         │Clear  │Src    │Dst    │SrcOver│
├─────────┼───────┼───────┼───────┼───────┤
│ 结果    │透明   │只留源 │只留目 │源覆盖│
└─────────┴───────┴───────┴───────┴───────┘
```

### 1.2 最常用的 5 种模式

| 模式 | 效果 | 典型场景 |
|------|------|---------|
| `SRC_IN` | 只保留源和目标重叠的源部分 | 圆角头像 |
| `SRC_OUT` | 只保留源中不在目标的部分 | 擦除效果 |
| `DST_IN` | 只保留目标和源重叠的目标部分 | 遮罩效果 |
| `SRC_OVER` | 源覆盖在目标上面（默认） | 正常绘制 |
| `CLEAR` | 清除所有像素 | 清空画布 |

### 1.3 实战：圆角头像

```java
public class RoundImageView extends View {

    private Paint paint;
    private Bitmap bitmap;
    private BitmapShader shader;

    public RoundImageView(Context context, AttributeSet attrs) {
        super(context, attrs);
        paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (bitmap == null) return;

        // 方法一：BitmapShader（推荐，性能好）
        shader = new BitmapShader(bitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP);
        // 缩放 bitmap 到 View 大小
        float scaleX = (float) getWidth() / bitmap.getWidth();
        float scaleY = (float) getHeight() / bitmap.getHeight();
        Matrix matrix = new Matrix();
        matrix.setScale(scaleX, scaleY);
        shader.setLocalMatrix(matrix);
        paint.setShader(shader);

        // 画圆角矩形
        float radius = dp2px(16);
        canvas.drawRoundRect(0, 0, getWidth(), getHeight(), radius, radius, paint);
    }

    public void setImageBitmap(Bitmap bmp) {
        this.bitmap = bmp;
        invalidate();
    }
}
```

### 1.4 实战：Xfermode 实现圆形头像

```java
@Override
protected void onDraw(Canvas canvas) {
    // 1. 创建离屏缓冲
    int layerId = canvas.saveLayer(0, 0, getWidth(), getHeight(), null);

    // 2. 画目标（圆形遮罩）
    paint.setColor(Color.WHITE);
    float radius = Math.min(getWidth(), getHeight()) / 2f;
    canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, radius, paint);

    // 3. 设置混合模式
    paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));

    // 4. 画源（头像图片）
    canvas.drawBitmap(bitmap, null, new RectF(0, 0, getWidth(), getHeight()), paint);

    // 5. 清除 Xfermode
    paint.setXfermode(null);

    // 6. 恢复图层
    canvas.restoreToCount(layerId);
}
```

> **离屏缓冲是必须的**：Xfermode 作用于整个 Canvas 的已有像素，如果不创建离屏缓冲，会和背景混合，产生意外效果。

---

## 二、ColorFilter（颜色滤镜）

ColorFilter 在绘制时对每个像素的颜色进行变换。

### 2.1 三种 ColorFilter

```java
// 1. LightingColorFilter：颜色调整（模拟光照）
// 公式：result = src * mul + add
paint.setColorFilter(new LightingColorFilter(0xFFFFFF, 0x00FF00));
// mul=白色(不变), add=绿色 → 整体偏绿

// 2. PorterDuffColorFilter：颜色混合
paint.setColorFilter(new PorterDuffColorFilter(
    0x88FF0000, PorterDuff.Mode.SRC_IN));
// 红色半透明覆盖

// 3. ColorMatrixColorFilter：矩阵变换（最强大）
float[] matrix = {
    0.33f, 0.59f, 0.11f, 0, 0,  // R = 灰度
    0.33f, 0.59f, 0.11f, 0, 0,  // G = 灰度
    0.33f, 0.59f, 0.11f, 0, 0,  // B = 灰度
    0,    0,    0,    1, 0       // A = 不变
};
paint.setColorFilter(new ColorMatrixColorFilter(matrix));
// → 灰度效果
```

### 2.2 常见效果的 ColorMatrix

```java
// 灰度
float[] gray = {
    0.33f, 0.59f, 0.11f, 0, 0,
    0.33f, 0.59f, 0.11f, 0, 0,
    0.33f, 0.59f, 0.11f, 0, 0,
    0, 0, 0, 1, 0
};

// 反色
float[] invert = {
    -1, 0, 0, 0, 255,
    0, -1, 0, 0, 255,
    0, 0, -1, 0, 255,
    0, 0, 0, 1, 0
};

// 高饱和度
float[] saturate = {
    1.5f, -0.3f, -0.2f, 0, 0,
    -0.3f, 1.5f, -0.2f, 0, 0,
    -0.2f, -0.3f, 1.5f, 0, 0,
    0, 0, 0, 1, 0
};

// 怀旧
float[] vintage = {
    0.393f, 0.769f, 0.189f, 0, 0,
    0.349f, 0.686f, 0.168f, 0, 0,
    0.272f, 0.534f, 0.131f, 0, 0,
    0, 0, 0, 1, 0
};
```

### 2.3 实战：图片灰色/彩色切换动画

```java
// 属性动画驱动 ColorMatrix
ObjectAnimator animator = ObjectAnimator.ofFloat(this, "saturation", 0f, 1f);
animator.setDuration(500);
animator.start();

public void setSaturation(float value) {
    ColorMatrix cm = new ColorMatrix();
    cm.setSaturation(value); // 0=灰度, 1=原色
    paint.setColorFilter(new ColorMatrixColorFilter(cm));
    invalidate();
}
```

---

## 三、MaskFilter（遮罩滤镜）

### 3.1 BlurMaskFilter（模糊）

```java
// 模糊半径和样式
paint.setMaskFilter(new BlurMaskFilter(20, BlurMaskFilter.Blur.NORMAL));

// 四种样式：
// NORMAL  - 标准模糊
// SOLID   - 边缘实心，内部模糊
// OUTER   - 只模糊边缘外部
// INNER   - 只模糊边缘内部
```

**实战：发光文字效果**

```java
@Override
protected void onDraw(Canvas canvas) {
    String text = "Hello";

    // 1. 底层发光
    textPaint.setMaskFilter(new BlurMaskFilter(30, BlurMaskFilter.Blur.NORMAL));
    textPaint.setColor(0xFF00BFFF);
    canvas.drawText(text, x, y, textPaint);

    // 2. 清除模糊，画清晰文字
    textPaint.setMaskFilter(null);
    textPaint.setColor(Color.WHITE);
    canvas.drawText(text, x, y, textPaint);
}
```

### 3.2 EmbossMaskFilter（浮雕）

```java
// 方向向量、环境光、反射光、模糊半径
float[] direction = {1, 1, 1}; // 光照方向
paint.setMaskFilter(new EmbossMaskFilter(
    direction,   // 光照方向 [x, y, z]
    0.5f,        // 环境光强度 (0-1)
    0.8f,        // 反射光强度
    3f           // 模糊半径
));
```

---

## 四、文字绘制进阶

### 4.1 文字居中

```java
@Override
protected void onDraw(Canvas canvas) {
    String text = "居中文字";

    // 水平居中
    float x = getWidth() / 2f;

    // 垂直居中（关键！）
    Paint.FontMetrics fm = textPaint.getFontMetrics();
    // fm.top    = 文字最高点到 baseline 的距离（负数）
    // fm.bottom = 文字最低点到 baseline 的距离（正数）
    // fm.ascent = 字符最高点到 baseline（比 top 小一点）
    // fm.descent= 字符最低点到 baseline（比 bottom 小一点）
    float y = getHeight() / 2f - (fm.top + fm.bottom) / 2f;

    textPaint.setTextAlign(Paint.Align.CENTER);
    canvas.drawText(text, x, y, textPaint);
}
```

### 4.2 文字测量

```java
// 测量文字宽度
float width = textPaint.measureText("Hello");

// 获取文字边界
Rect bounds = new Rect();
textPaint.getTextBounds("Hello", 0, 5, bounds);
int textWidth = bounds.width();
int textHeight = bounds.height();

// measureText vs getTextBounds：
// measureText 返回的是 advance width（文字前进宽度，含字间距）
// getTextBounds 返回的是实际像素边界（更紧凑）
// 用 measureText 做布局，用 getTextBounds 做精确绘制
```

### 4.3 文字沿路径绘制

```java
Path path = new Path();
path.addCircle(cx, cy, radius, Path.Direction.CW);

canvas.drawTextOnPath("文字沿圆弧排列", path, 0, 0, textPaint);
// offset 参数：沿路径的起始偏移量
// hOffset：沿路径的横向偏移
// vOffset：垂直于路径的偏移（正值向下）
```

### 4.4 StaticLayout（多行文字）

`canvas.drawText` 不支持自动换行，多行文字需要用 `StaticLayout`。

```java
@Override
protected void onDraw(Canvas canvas) {
    String text = "这是一段很长的文字，需要自动换行显示...";

    StaticLayout layout = new StaticLayout(
        text,
        textPaint,
        getWidth() - getPaddingLeft() - getPaddingRight(), // 最大宽度
        Layout.Alignment.ALIGN_NORMAL,  // 对齐方式
        1.0f,   // 行间距倍数
        0f,     // 行间距额外值
        false   // include padding
    );

    canvas.save();
    canvas.translate(getPaddingLeft(), getPaddingTop());
    layout.draw(canvas);
    canvas.restore();
}
```

---

## 五、Shader（着色器）进阶

### 5.1 五种 Shader

```java
// 1. LinearGradient：线性渐变
new LinearGradient(x0, y0, x1, y1, colorStart, colorEnd, TileMode.CLAMP);

// 2. RadialGradient：径向渐变
new RadialGradient(cx, cy, radius, colorCenter, colorEdge, TileMode.CLAMP);

// 3. SweepGradient：扫描渐变（雷达效果）
new SweepGradient(cx, cy, colorStart, colorEnd);

// 4. BitmapShader：位图填充
new BitmapShader(bitmap, TileMode.REPEAT, TileMode.REPEAT);

// 5. ComposeShader：组合着色器
new ComposeShader(shader1, shader2, PorterDuff.Mode.MULTIPLY);
```

### 5.2 TileMode（平铺模式）

| 模式 | 效果 |
|------|------|
| `CLAMP` | 边缘颜色延伸 |
| `REPEAT` | 重复平铺 |
| `MIRROR` | 镜像平铺 |

### 5.3 实战：文字渐变色

```java
@Override
protected void onDraw(Canvas canvas) {
    String text = "渐变色文字";

    // 文字宽度的线性渐变
    float textWidth = textPaint.measureText(text);
    LinearGradient gradient = new LinearGradient(
        0, 0, textWidth, 0,
        new int[]{0xFF00BFFF, 0xFFFF6B6B, 0xFF51CF66},
        null,
        Shader.TileMode.CLAMP
    );
    textPaint.setShader(gradient);
    canvas.drawText(text, x, y, textPaint);
}
```

---

## 六、Paint 的硬件加速限制

部分 Paint API 在硬件加速下不生效：

| API | 软件渲染 | 硬件加速 |
|-----|---------|---------|
| `BlurMaskFilter` | 支持 | 部分支持 |
| `EmbossMaskFilter` | 支持 | 不支持 |
| `setMaskFilter` | 支持 | 部分支持 |
| `setXfermode` | 支持 | 部分支持 |
| `drawBitmapMesh` | 支持 | 不支持 |
| `drawPicture` | 支持 | 不支持 |

**解决方案**：

```java
// 关闭单个 View 的硬件加速
setLayerType(View.LAYER_TYPE_SOFTWARE, null);

// 或在 XML 中
android:layerType="software"
```

---

## 七、面试要点

**Q: Xfermode 的 SRC_IN 和 SRC_OUT 分别什么效果？**

SRC_IN 只保留源图和目标图重叠部分的源图像素——典型场景是圆形头像（圆形遮罩 + 头像图片）。SRC_OUT 只保留源图中不在目标图中的部分——典型场景是擦除效果（橡皮擦路径 + 原图）。

**Q: ColorMatrix 怎么实现灰度效果？**

灰度公式 Gray = 0.33R + 0.59G + 0.11B（人眼对绿色最敏感）。用 ColorMatrix 把 RGB 三行都设为 [0.33, 0.59, 0.11, 0, 0]，这样每个通道都输出灰度值。也可以直接用 ColorMatrix.setSaturation(0) 设置饱和度为 0 实现灰度。

**Q: drawText 的垂直居中怎么算？**

用 Paint.FontMetrics 获取 baseline 的位置。fm.top 是文字最高点到 baseline 的距离（负数），fm.bottom 是最低点到 baseline 的距离（正数）。文字中心到 baseline 的偏移 = -(fm.top + fm.bottom) / 2，所以 y = centerY - (fm.top + fm.bottom) / 2。

**Q: hardware acceleration 对 Paint 有什么影响？**

硬件加速用 GPU 渲染，部分 Paint API 不支持：BlurMaskFilter、EmbossMaskFilter、部分 Xfermode。解决方法是 setLayerType(LAYER_TYPE_SOFTWARE, null) 关闭单个 View 的硬件加速，或用 setXfermode 前创建离屏缓冲 canvas.saveLayer。

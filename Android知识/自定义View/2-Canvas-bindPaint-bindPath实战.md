# Canvas / Paint / Path 实战

> 本文系统梳理 Android 自定义 View 三大绘图核心：[[Canvas]]、[[Paint]]、[[Path]]，涵盖坐标变换、混合模式、路径填充、实战案例与面试高频题。

---

## 一、核心概念

### 1.1 Canvas 坐标系与变换

Canvas 默认坐标系：**左上角为原点**，x 轴向右，y 轴向下。所有绘制操作都基于当前坐标系。

四种基本变换：

| 变换 | 方法 | 说明 |
|------|------|------|
| 平移 | `translate(dx, dy)` | 将坐标原点移动 (dx, dy) |
| 旋转 | `rotate(degrees, px, py)` | 以 (px, py) 为中心旋转 |
| 缩放 | `scale(sx, sy, px, py)` | 以 (px, py) 为中心缩放 |
| 错切 | `skew(sx, sy)` | sx 为 x 轴倾斜角正切值 |

````java
// 变换示例：在 View 中心绘制旋转矩形
@Override
protected void onDraw(Canvas canvas) {
    float cx = getWidth() / 2f;
    float cy = getHeight() / 2f;
    canvas.save();
    canvas.translate(cx, cy);
    canvas.rotate(45f);
    canvas.drawRect(-50f, -50f, 50f, 50f, paint);
    canvas.restore();
}
````
> 关键：变换操作的顺序会影响最终结果，**后调用的变换先作用**（矩阵右乘）。

### 1.2 Paint 核心属性

[[Paint]] 控制「怎么画」，核心属性：

````java
Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
// style：填充模式
paint.setStyle(Paint.Style.FILL);          // FILL / STROKE / FILL_AND_STROKE
paint.setStrokeWidth(dpToPx(4f));          // 描边宽度（仅 STROKE 生效）
paint.setStrokeCap(Paint.Cap.ROUND);       // 线帽：BUTT / ROUND / SQUARE
paint.setStrokeJoin(Paint.Join.ROUND);     // 拐角：MITER / ROUND / BEVEL

// Shader：着色器（渐变/位图填充）
paint.setShader(new LinearGradient(
    0f, 0f, (float) width, 0f,
    new int[]{Color.RED, Color.BLUE},
    null, Shader.TileMode.CLAMP
));

// Xfermode：混合模式（图层合成核心）
paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));
````
常用 [[Shader]] 子类：
- `LinearGradient` — 线性渐变
- `RadialGradient` — 径向渐变
- `SweepGradient` — 扫描渐变
- `BitmapShader` — 位图着色（圆形头像核心）
- `ComposeShader` — 组合着色器

### 1.3 Path 核心 API

[[Path]] 控制「画什么形状」：

````java
Path path = new Path();
path.moveTo(0f, 0f);                          // 移动画笔（不画线）
path.lineTo(100f, 100f);                      // 直线
path.quadTo(150f, 0f, 200f, 100f);            // 二阶贝塞尔
path.cubicTo(250f, 0f, 300f, 200f, 350f, 100f); // 三阶贝塞尔
path.arcTo(new RectF(0f, 0f, 100f, 100f), 0f, 90f); // 圆弧
path.close();                                  // 闭合路径
````
| 方法 | 控制点 | 适用场景 |
|------|--------|----------|
| `lineTo` | 0 | 直线、多边形 |
| `quadTo` | 1 | 简单曲线、波浪 |
| `cubicTo` | 2 | 复杂曲线、贝塞尔动画 |
| `arcTo` | 椭圆+角度 | 圆弧、扇形、进度条 |

---

## 二、原理与源码

### 2.1 Canvas 的 save/restore 状态栈

Canvas 内部维护一个**状态栈**，每次 `save()` 压入当前状态（矩阵 + 裁剪区域），`restore()` 弹出恢复。

````
save()    → 栈: [State0, State1]
操作...
save()    → 栈: [State0, State1, State2]
操作...
restore() → 栈: [State0, State1]  ← 恢复到 State1
restore() → 栈: [State0]          ← 恢复到 State0
````
源码关键路径（`android.graphics.Canvas`）：

````java
// Canvas.java（简化）
public int save() {
    return nSave(mNativeCanvasWrapper, MATRIX_SAVE_FLAG | CLIP_SAVE_FLAG);
}

public void restore() {
    if (!nRestore(mNativeCanvasWrapper)) {
        throw new IllegalStateException("Underflow in restore");
    }
}
````
> `saveLayer()` 会额外创建**离屏缓冲区**（offscreen bitmap），是 [[Xfermode]] 生效的前提，但开销较大。

````java
// saveLayer 典型用法
int layerId = canvas.saveLayer(0f, 0f, (float) getWidth(), (float) getHeight(), null);
// 在离屏缓冲区绘制...
canvas.restoreToCount(layerId);
````
### 2.2 Paint 的 Xfermode — PorterDuff 16 种模式

[[PorterDuff]] 模式定义了 **Src（源图）** 与 **Dst（目标图）** 的像素合成规则：

| 模式 | 公式 | 典型用途 |
|------|------|----------|
| `CLEAR` | [0, 0] | 清除 |
| `SRC` | [Sa, Sc] | 只显示源 |
| `DST` | [Da, Dc] | 只显示目标 |
| `SRC_OVER` | [Sa+(1-Sa)*Da, ...] | 默认模式，源覆盖目标 |
| `DST_OVER` | [Da+(1-Da)*Sa, ...] | 目标覆盖源 |
| `SRC_IN` | [Sa*Da, Sc*Da] | **圆形头像**核心 |
| `DST_IN` | [Da*Sa, Dc*Sa] | 遮罩效果 |
| `SRC_OUT` | [Sa*(1-Da), ...] | **刮刮卡**核心 |
| `DST_OUT` | [Da*(1-Sa), ...] | 橡皮擦效果 |
| `SRC_ATOP` | [...] | 源绘制在目标上方（仅交集） |
| `DST_ATOP` | [...] | 目标绘制在源上方（仅交集） |
| `XOR` | [...] | 异或，去除交集 |
| `DARKEN` | [...] | 取暗 |
| `LIGHTEN` | [...] | 取亮 |
| `MULTIPLY` | [Sa*Da, Sc*Dc] | 正片叠底 |
| `SCREEN` | [Sa+Da-Sa*Da, ...] | 滤色 |

> 使用 Xfermode 时**必须配合 `saveLayer`** 创建离屏缓冲区，否则 Src/Dst 关系不正确。

### 2.3 Path 的 FillType

Path 有 4 种填充规则，决定路径围成区域的「内外」判定：

````java
path.setFillType(Path.FillType.EVEN_ODD); // 默认 WINDING
````
| FillType | 规则 | 说明 |
|----------|------|------|
| `WINDING` | 非零环绕数 | 默认，所有围成区域都填充 |
| `EVEN_ODD` | 奇偶规则 | 交叉区域交替填充（镂空效果） |
| `INVERSE_WINDING` | WINDING 取反 | 填充路径外部 |
| `INVERSE_EVEN_ODD` | EVEN_ODD 取反 | 填充路径外部（奇偶） |

````java
// EVEN_ODD 实现环形效果
Path path = new Path();
path.setFillType(Path.FillType.EVEN_ODD);
path.addCircle(cx, cy, outerR, Path.Direction.CW);
path.addCircle(cx, cy, innerR, Path.Direction.CW);
canvas.drawPath(path, paint); // 只填充两圆之间的环形区域
````
---

## 三、实战案例

### 3.1 圆形头像（BitmapShader + Xfermode 两种方案）

#### 方案一：BitmapShader（推荐，性能更优）

````java
public class CircleAvatarView extends View {

    private final Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private float radius = 0f;
    private Bitmap bitmap;

    public CircleAvatarView(Context context) { this(context, null); }
    public CircleAvatarView(Context context, AttributeSet attrs) { super(context, attrs); }

    public void setImageBitmap(Bitmap bmp) {
        bitmap = bmp;
        invalidate();
    }

    @Override
    protected void onSizeChanged(int w, int h, int oldw, int oldh) {
        radius = Math.min(w, h) / 2f;
        if (bitmap != null) updateShader(bitmap);
    }

    private void updateShader(Bitmap bmp) {
        int size = (int) (radius * 2);
        Bitmap scaled = Bitmap.createScaledBitmap(bmp, size, size, true);
        paint.setShader(new BitmapShader(scaled, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP));
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (bitmap == null) return;
        canvas.drawCircle(radius, radius, radius, paint);
    }
}
````
#### 方案二：Xfermode（SRC_IN）

````java
public class XfermodeAvatarView extends View {

    private final Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final Xfermode xfermode = new PorterDuffXfermode(PorterDuff.Mode.SRC_IN);
    private Bitmap bitmap;

    public XfermodeAvatarView(Context context) { this(context, null); }
    public XfermodeAvatarView(Context context, AttributeSet attrs) { super(context, attrs); }

    public void setImageBitmap(Bitmap bmp) {
        bitmap = bmp;
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (bitmap == null) return;
        float r = Math.min(getWidth(), getHeight()) / 2f;

        // 必须 saveLayer 创建离屏缓冲
        int layerId = canvas.saveLayer(0f, 0f, (float) getWidth(), (float) getHeight(), null);

        // 先画 Dst：圆形
        canvas.drawCircle(r, r, r, paint);

        // 设置 Xfermode，再画 Src：位图
        paint.setXfermode(xfermode);
        canvas.drawBitmap(bitmap, null, new RectF(0f, 0f, r * 2, r * 2), paint);

        paint.setXfermode(null);
        canvas.restoreToCount(layerId);
    }
}
````
### 3.2 波浪进度条（Path + 贝塞尔 + 动画）

````java
public class WaveProgressView extends View {

    private float progress = 0.5f;  // 0f..1f

    public void setProgress(float value) {
        progress = Math.max(0f, Math.min(value, 1f));
        invalidate();
    }

    private final Paint wavePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final Paint circlePaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final Paint textPaint = new Paint(Paint.ANTI_ALIAS_FLAG);

    private final Path wavePath = new Path();
    private final PorterDuffXfermode xfermode = new PorterDuffXfermode(PorterDuff.Mode.SRC_IN);
    private float waveOffset = 0f;
    private final float waveHeight = 20f;
    private float getWaveLength() { return (float) getWidth(); }

    private final ValueAnimator animator;

    public WaveProgressView(Context context) { this(context, null); }
    public WaveProgressView(Context context, AttributeSet attrs) {
        super(context, attrs);
        wavePaint.setColor(Color.parseColor("#4FC3F7"));
        wavePaint.setStyle(Paint.Style.FILL);
        circlePaint.setColor(Color.parseColor("#E0E0E0"));
        circlePaint.setStyle(Paint.Style.STROKE);
        circlePaint.setStrokeWidth(4f);
        textPaint.setColor(Color.WHITE);
        textPaint.setTextSize(48f);
        textPaint.setTextAlign(Paint.Align.CENTER);

        animator = ValueAnimator.ofFloat(0f, 1f);
        animator.setDuration(1500L);
        animator.setRepeatCount(ValueAnimator.INFINITE);
        animator.setInterpolator(new LinearInterpolator());
        animator.addUpdateListener(it -> {
            waveOffset = (float) it.getAnimatedValue() * getWaveLength();
            invalidate();
        });
    }

    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        animator.start();
    }

    @Override
    protected void onDetachedFromWindow() {
        animator.cancel();
        super.onDetachedFromWindow();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        float cx = getWidth() / 2f;
        float cy = getHeight() / 2f;
        float r = Math.min(cx, cy) - circlePaint.getStrokeWidth();

        // 离屏缓冲
        int layerId = canvas.saveLayer(0f, 0f, (float) getWidth(), (float) getHeight(), null);

        // Dst：圆形
        canvas.drawCircle(cx, cy, r, wavePaint);

        // Src：波浪 + SRC_IN 裁剪到圆内
        wavePaint.setXfermode(xfermode);
        buildWavePath(r, cx, cy);
        canvas.drawPath(wavePath, wavePaint);
        wavePaint.setXfermode(null);

        canvas.restoreToCount(layerId);

        // 外圈描边
        canvas.drawCircle(cx, cy, r, circlePaint);

        // 文字
        String text = (int) (progress * 100) + "%";
        Paint.FontMetrics fontMetrics = textPaint.getFontMetrics();
        float baseline = cy - (fontMetrics.top + fontMetrics.bottom) / 2;
        canvas.drawText(text, cx, baseline, textPaint);
    }

    private void buildWavePath(float r, float cx, float cy) {
        float waveLength = getWaveLength();
        float waterLevel = cy + r - progress * r * 2;
        wavePath.reset();
        wavePath.moveTo(-waveLength + waveOffset, waterLevel);

        float x = -waveLength + waveOffset;
        while (x <= getWidth() + waveLength) {
            wavePath.quadTo(
                x + waveLength / 4, waterLevel - waveHeight,
                x + waveLength / 2, waterLevel
            );
            wavePath.quadTo(
                x + waveLength * 3 / 4, waterLevel + waveHeight,
                x + waveLength, waterLevel
            );
            x += waveLength;
        }
        wavePath.lineTo((float) getWidth(), (float) getHeight());
        wavePath.lineTo(0f, (float) getHeight());
        wavePath.close();
    }
}
````
### 3.3 刮刮卡效果（SRC_OUT + 手势）

````java
public class ScratchCardView extends View {

    private Bitmap foregroundBmp;  // 覆盖层
    private Bitmap resultBmp;      // 底部结果图
    private Canvas mCanvas;

    private final Paint scratchPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final Paint bmpPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final Path scratchPath = new Path();
    private float lastX = 0f;
    private float lastY = 0f;

    public ScratchCardView(Context context) { this(context, null); }
    public ScratchCardView(Context context, AttributeSet attrs) {
        super(context, attrs);
        scratchPaint.setStyle(Paint.Style.STROKE);
        scratchPaint.setStrokeWidth(60f);
        scratchPaint.setStrokeCap(Paint.Cap.ROUND);
        scratchPaint.setStrokeJoin(Paint.Join.ROUND);
        // 核心：DST_OUT 让手指划过的区域变透明
        scratchPaint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.DST_OUT));
    }

    public void setup(Bitmap result, int foreColor) {
        resultBmp = result;
        foregroundBmp = Bitmap.createBitmap(result.getWidth(), result.getHeight(), Bitmap.Config.ARGB_8888);
        mCanvas = new Canvas(foregroundBmp);
        mCanvas.drawColor(foreColor);
        invalidate();
    }

    public void setup(Bitmap result) {
        setup(result, Color.GRAY);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                scratchPath.moveTo(event.getX(), event.getY());
                lastX = event.getX(); lastY = event.getY();
                return true;
            case MotionEvent.ACTION_MOVE:
                float midX = (lastX + event.getX()) / 2;
                float midY = (lastY + event.getY()) / 2;
                scratchPath.quadTo(lastX, lastY, midX, midY);
                lastX = event.getX(); lastY = event.getY();

                // 在前景 Bitmap 上用 DST_OUT 擦除
                if (mCanvas != null) mCanvas.drawPath(scratchPath, scratchPaint);
                scratchPath.reset();
                scratchPath.moveTo(lastX, lastY);
                invalidate();
                break;
        }
        return super.onTouchEvent(event);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (resultBmp != null) canvas.drawBitmap(resultBmp, 0f, 0f, bmpPaint);
        if (foregroundBmp != null) canvas.drawBitmap(foregroundBmp, 0f, 0f, bmpPaint);
    }
}
````
使用方式：

````java
ScratchCardView scratchView = findViewById(R.id.scratch);
Bitmap resultBmp = BitmapFactory.decodeResource(getResources(), R.drawable.prize);
scratchView.setup(resultBmp);
````
---

## 四、面试题

### Q1：Canvas.save() 和 saveLayer() 有什么区别？

`save()` 仅保存**矩阵和裁剪区域**到状态栈，开销极小。`saveLayer()` 额外创建一个**离屏 Bitmap 缓冲区**，后续绘制都在该缓冲区上进行，`restore()` 时再合并回主 Canvas。`saveLayer` 是 [[Xfermode]] 正确生效的前提，但会增加内存开销和 GPU 负担。

### Q2：PorterDuff.Mode.SRC_IN 和 DST_IN 的区别？

- `SRC_IN`：取 Src 和 Dst 的**交集区域**，显示 **Src 的像素**。典型用途：圆形头像（Dst 画圆，Src 画位图）。
- `DST_IN`：取交集区域，显示 **Dst 的像素**。典型用途：遮罩效果（Src 作为 alpha 蒙版）。

关键：谁先画谁是 Dst，后画的是 Src。

### Q3：Path.FillType.WINDING 和 EVEN_ODD 的区别？

- `WINDING`（默认）：从某点向外引射线，路径从左到右穿过 +1，从右到左 -1，结果非零则在路径内。所有闭合区域都会被填充。
- `EVEN_ODD`：射线与路径交叉次数为奇数则在内部。嵌套路径会产生**镂空效果**。

实际应用：用 `EVEN_ODD` + 两个 `addCircle` 可以画环形，无需额外计算。

### Q4：为什么 Xfermode 不生效？常见原因有哪些？

1. **没有使用 `saveLayer`**：直接在 View 的 Canvas 上操作，Dst 包含了 View 背景甚至 Window 背景，导致混合结果异常。
2. **硬件加速不支持**：部分 Xfermode 在硬件加速下行为异常，需要 `setLayerType(LAYER_TYPE_SOFTWARE, null)`。
3. **Src/Dst 顺序搞反**：先画的是 Dst，后画的是 Src，顺序错误会导致效果相反。
4. **Bitmap 尺寸不匹配**：Src 和 Dst 的 Bitmap 大小不一致，透明区域参与混合。

### Q5：自定义 View 中如何避免 onDraw 中频繁创建对象？

- Paint、Path、RectF 等对象在**构造函数或 `onSizeChanged`** 中创建，`onDraw` 中复用。
- Path 使用 `reset()` 或 `rewind()` 清空后复用（`rewind` 保留内部数据结构，性能更优）。
- 避免在 `onDraw` 中调用 `String.format()`、创建 `Bitmap` 等分配内存的操作。
- 使用 Android Studio 的 **Allocation Tracker** 检测 `onDraw` 中的内存分配。

### Q6：Canvas.clipPath() 和 Xfermode 实现圆形裁剪，哪个更好？

| 维度 | clipPath | Xfermode/BitmapShader |
|------|----------|----------------------|
| 抗锯齿 | ❌ 无法抗锯齿（API 18 以下） | ✅ Paint 自带抗锯齿 |
| 性能 | ✅ 无需离屏缓冲 | ⚠️ Xfermode 需要 saveLayer |
| 推荐度 | 简单场景可用 | **推荐 BitmapShader 方案** |

结论：优先使用 `BitmapShader`，兼顾抗锯齿和性能。

---

## 五、实战与踩坑

### 5.1 硬件加速不支持的 API

Android 硬件加速（默认开启）不支持部分 Canvas/Paint API：

| 不支持的 API | 影响 | 解决方案 |
|-------------|------|----------|
| `Canvas.drawBitmapMesh()` | 网格变形失效 | 关闭硬件加速 |
| `Paint.setMaskFilter(BlurMaskFilter)` | 模糊阴影不生效 | View 级别关闭 |
| `Canvas.drawPicture()` | 录制回放失效 | 关闭硬件加速 |
| 部分 `PorterDuff.Mode` | 混合结果异常 | `saveLayer` 或关闭硬件加速 |
| `Paint.setShadowLayer()` | 文字以外无效 | 使用 `BlurMaskFilter` 替代 |

````java
// View 级别关闭硬件加速（推荐，影响范围最小）
setLayerType(LAYER_TYPE_SOFTWARE, null);

// 也可在 AndroidManifest.xml 中 Activity 级别关闭
// android:hardwareAccelerated="false"
````
> 最佳实践：**仅在必要时关闭**，优先使用 View 级别而非全局关闭。可通过 `canvas.isHardwareAccelerated` 运行时判断。

### 5.2 Bitmap 内存管理

自定义 View 中 [[Bitmap]] 是内存大户，关键注意点：

````java
// 1. 按需采样，避免加载原图
public static Bitmap decodeSampledBitmap(Resources res, int resId, int reqW, int reqH) {
    BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);
    options.inSampleSize = calculateInSampleSize(options, reqW, reqH);
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}

// 2. 使用 inBitmap 复用内存（API 19+）
options.inMutable = true;
options.inBitmap = reusableBitmap;  // 尺寸 >= 目标的可复用 Bitmap

// 3. 及时回收
@Override
protected void onDetachedFromWindow() {
    super.onDetachedFromWindow();
    if (bitmap != null) bitmap.recycle();
    bitmap = null;
}
````
内存计算公式：`宽 × 高 × 每像素字节数`

| Config | 每像素 | 适用场景 |
|--------|--------|----------|
| `ARGB_8888` | 4 字节 | 默认，高质量 |
| `RGB_565` | 2 字节 | 不需要透明度，省一半内存 |
| `ALPHA_8` | 1 字节 | 仅 alpha 通道（遮罩） |
| `HARDWARE` | GPU 管理 | API 26+，只读位图 |

### 5.3 自定义 Drawable

将绘制逻辑封装为 [[Drawable]]，可复用于 ImageView、背景等场景：

````java
public class RoundRectDrawable extends Drawable {

    private final Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final float cornerRadius;
    private final RectF rectF = new RectF();

    public RoundRectDrawable(int color, float cornerRadius) {
        this.cornerRadius = cornerRadius;
        paint.setColor(color);
        paint.setStyle(Paint.Style.FILL);
    }

    @Override
    public void draw(@NonNull Canvas canvas) {
        rectF.set(getBounds());
        canvas.drawRoundRect(rectF, cornerRadius, cornerRadius, paint);
    }

    @Override public void setAlpha(int alpha) { paint.setAlpha(alpha); }
    @Override public void setColorFilter(ColorFilter colorFilter) { paint.setColorFilter(colorFilter); }
    @Override public int getOpacity() { return PixelFormat.TRANSLUCENT; }
}

// 使用
imageView.setBackground(new RoundRectDrawable(Color.BLUE, dpToPx(16f)));
````
自定义 Drawable 踩坑：
- 必须重写 `getIntrinsicWidth/Height`，否则 `wrap_content` 时尺寸为 -1
- `setBounds` 在 `draw` 之前由系统调用，绘制时用 `bounds` 获取区域
- 支持状态变化（按压/选中）需重写 `isStateful()` 返回 `true` 并处理 `onStateChange`

### 5.4 性能优化清单

| 问题 | 解决方案 |
|------|----------|
| `onDraw` 中 new 对象 | 提前创建，`onDraw` 中复用 |
| 频繁 `invalidate()` | 使用 `invalidate(Rect)` 局部刷新 |
| `saveLayer` 过多 | 减少离屏缓冲层数，合并绘制 |
| 大 Bitmap 未缩放 | `inSampleSize` 按需采样 |
| 复杂 Path 每帧重建 | `Path.reset()` 复用 + 缓存不变部分 |
| 硬件加速全局关闭 | 仅 View 级别按需关闭 |

---

## 参考

- [[自定义View]] 绘制流程：`onMeasure` → `onLayout` → `onDraw`
- [[硬件加速]] 官方文档：不支持的操作列表
- [[PorterDuff]] 原始论文：Porter & Duff, 1984, *Compositing Digital Images*
- [[Bitmap]] 内存优化与 inBitmap 复用机制
- [[BitmapShader]] 与 [[Xfermode]] 的性能对比

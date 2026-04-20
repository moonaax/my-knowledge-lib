# View 绘制流程：measure / layout / draw

> 自定义 View 的三大核心阶段，从 MeasureSpec 到像素上屏的完整链路。
> 关联：[[8-事件分发机制]] · [[3-自定义属性与TypedArray]]

---

## 一、核心概念

### 1.1 MeasureSpec 三种模式

`MeasureSpec` 是一个 32 位 int 值，高 2 位表示 **模式（SpecMode）**，低 30 位表示 **尺寸（SpecSize）**。

| 模式 | 常量值 | 含义 | 对应 LayoutParams |
|------|--------|------|-------------------|
| `EXACTLY` | `1 << 30` | 父容器已确定子 View 的精确大小 | `match_parent` 或具体 dp 值 |
| `AT_MOST` | `2 << 30` | 子 View 不能超过 SpecSize | `wrap_content` |
| `UNSPECIFIED` | `0` | 父容器不限制，子 View 想多大就多大 | ScrollView 内部、系统内部测量 |

```java
// MeasureSpec 的打包与解包
int mode = MeasureSpec.getMode(measureSpec);   // 高 2 位
int size = MeasureSpec.getSize(measureSpec);   // 低 30 位
int spec = MeasureSpec.makeMeasureSpec(size, mode); // 合成
```

### 1.2 measure / layout / draw 三阶段职责

```
ViewRootImpl.performTraversals()
    ├── performMeasure()   → measure 阶段：确定 View 的宽高（measuredWidth / measuredHeight）
    ├── performLayout()    → layout 阶段：确定 View 在父容器中的位置（left, top, right, bottom）
    └── performDraw()      → draw 阶段：将 View 绘制到屏幕上
```

| 阶段 | 回调方法 | 产出 | 可多次调用？ |
|------|---------|------|-------------|
| measure | `onMeasure()` | `measuredWidth` / `measuredHeight` | 是，可能被多次测量 |
| layout | `onLayout()` | `mLeft / mTop / mRight / mBottom` | 是，但通常一次 |
| draw | `onDraw()` | Canvas 像素 | 是，invalidate 触发 |

> **关键区别**：`getMeasuredWidth()` 在 measure 后就有值；`getWidth()` = `mRight - mLeft`，在 layout 后才有值。两者大多数情况相等，但在 layout 中手动修改坐标时会不同。

---

## 二、原理与源码

### 2.1 ViewRootImpl.performTraversals 调用链

`ViewRootImpl` 是连接 `WindowManager` 和 `DecorView` 的桥梁。当界面需要刷新时，`Choreographer` 在下一个 VSYNC 信号到来时回调 `doTraversal()`：

```
Choreographer.doFrame()
  → ViewRootImpl.doTraversal()
    → ViewRootImpl.performTraversals()
      ├── measureHierarchy() / performMeasure()
      │     → DecorView.measure()
      │       → DecorView.onMeasure()
      │         → 递归子 View.measure()
      ├── performLayout()
      │     → DecorView.layout()
      │       → DecorView.onLayout()
      │         → 递归子 View.layout()
      └── performDraw()
            → draw() → drawSoftware() 或 ThreadedRenderer.draw()
              → DecorView.draw()
                → DecorView.onDraw() + dispatchDraw()
```

核心流程：先判断 `mLayoutRequested` 决定是否 measure，再 `performLayout()`，最后 `performDraw()`。三个阶段按需执行，并非每帧都走完整流程。

### 2.2 onMeasure 中 MeasureSpec 的合成规则

父容器在测量子 View 时，通过 `ViewGroup.getChildMeasureSpec()` 将 **父的 MeasureSpec** 与 **子的 LayoutParams** 合成为子 View 的 MeasureSpec：

核心逻辑：根据 `specMode` 和 `childDimension` 的组合，决定子 View 的测量模式和尺寸上限。`wrap_content` 在 `EXACTLY` 父容器下会得到 `AT_MOST` + 父剩余空间作为上限。

**合成规则速查表**：

| 父 SpecMode \ 子 LayoutParams | 具体值 (dp) | `match_parent` | `wrap_content` |
|-------------------------------|------------|----------------|----------------|
| **EXACTLY** | EXACTLY, childSize | EXACTLY, parentSize | AT_MOST, parentSize |
| **AT_MOST** | EXACTLY, childSize | AT_MOST, parentSize | AT_MOST, parentSize |
| **UNSPECIFIED** | EXACTLY, childSize | UNSPECIFIED, 0 | UNSPECIFIED, 0 |

### 2.3 onLayout 的坐标系

`layout()` 方法确定 View 的四个顶点位置，坐标系以 **父容器左上角为原点**：

```
父容器 (0,0)
  ┌──────────────────────┐
  │  (left, top)         │
  │    ┌────────┐        │
  │    │ 子View │ height │
  │    └────────┘        │
  │      width           │
  │         (right, bottom)
  └──────────────────────┘
```

```java
// View.layout() 简化流程
public void layout(int l, int t, int r, int b) {
    boolean changed = setFrame(l, t, r, b);  // 设置 mLeft, mTop, mRight, mBottom
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) != 0) {
        onLayout(changed, l, t, r, b);   // ViewGroup 必须重写，摆放子 View
    }
}
```

对于自定义 ViewGroup，`onLayout()` 是必须重写的方法：

```java
public class SimpleVerticalLayout extends ViewGroup {

    public SimpleVerticalLayout(Context context) { this(context, null); }
    public SimpleVerticalLayout(Context context, AttributeSet attrs) { super(context, attrs); }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int currentTop = getPaddingTop();
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (child.getVisibility() == GONE) continue;
            MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
            int childLeft = getPaddingLeft() + lp.leftMargin;
            int childRight = childLeft + child.getMeasuredWidth();
            int childBottom = currentTop + lp.topMargin + child.getMeasuredHeight();
            child.layout(childLeft, currentTop + lp.topMargin, childRight, childBottom);
            currentTop = childBottom + lp.bottomMargin;
        }
    }
}
```

### 2.4 onDraw 与硬件加速 DisplayList

#### 软件绘制 vs 硬件加速

| 特性 | 软件绘制 | 硬件加速（默认开启） |
|------|---------|-------------------|
| 渲染线程 | 主线程 | RenderThread |
| 绘制载体 | Bitmap-backed Canvas | DisplayList (RenderNode) |
| invalidate 范围 | 需要重绘整个 View 树 | 只重录当前 View 的 DisplayList |
| 支持的 API | 全部 Canvas API | 大部分，少数不支持（如 `setXfermode` 部分模式） |

硬件加速下，`onDraw()` 中的 Canvas 操作会被录制为 **DisplayList**（一组 GPU 指令），由 RenderThread 异步提交给 GPU 执行：

```
View.draw()
  → updateDisplayListIfDirty()
    → RenderNode.beginRecording()   // 获取 RecordingCanvas
    → onDraw(canvas)                // 录制绘制指令
    → RenderNode.endRecording()     // 生成 DisplayList
```

```java
// 自定义 View 的 onDraw 示例
public class CircleView extends View {

    private final Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);

    public CircleView(Context context) { this(context, null); }
    public CircleView(Context context, AttributeSet attrs) {
        super(context, attrs);
        paint.setColor(Color.RED);
        paint.setStyle(Paint.Style.FILL);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        float cx = getWidth() / 2f;
        float cy = getHeight() / 2f;
        float radius = Math.min(cx, cy) - getPaddingStart();
        canvas.drawCircle(cx, cy, radius, paint);
    }
}
```

### 2.5 requestLayout vs invalidate 区别

这是面试高频考点，也是日常开发中最容易混淆的两个方法：

```
requestLayout()                          invalidate()
  │                                        │
  ├→ 标记 PFLAG_FORCE_LAYOUT              ├→ 标记 PFLAG_DIRTY
  ├→ 向上传递到 ViewRootImpl              ├→ 向上传递脏区域
  ├→ 触发 measure + layout + draw         ├→ 只触发 draw
  └→ 用于：尺寸/位置变化                  └→ 用于：外观变化（颜色、透明度等）
```

| 方法 | 触发阶段 | 使用场景 |
|------|---------|---------|
| `requestLayout()` | measure → layout → draw | 尺寸或位置发生变化 |
| `invalidate()` | draw | 仅外观变化（颜色、文字等） |
| `postInvalidateOnAnimation()` | draw（下一帧） | 动画中的连续重绘 |
| `forceLayout()` | 标记 flag，不主动触发 | 配合父容器的 requestLayout 使用 |

```java
// 典型用法
public void updateColor(int newColor) {
    paint.setColor(newColor);
    invalidate();          // 只需重绘，不需要重新测量
}

public void updateTextSize(float newSize) {
    paint.setTextSize(newSize);
    requestLayout();       // 文字大小变了 → 尺寸可能变 → 需要重新 measure + layout
}
```


---

## 三、面试题

### Q1：View 的 measure、layout、draw 分别做了什么？可以跳过某个阶段吗？

**A**：
- **measure**：递归测量 View 树，确定每个 View 的 `measuredWidth` 和 `measuredHeight`。核心回调是 `onMeasure()`，必须调用 `setMeasuredDimension()` 保存结果。
- **layout**：递归摆放 View 树，确定每个 View 在父容器坐标系中的 `left/top/right/bottom`。ViewGroup 必须重写 `onLayout()` 来摆放子 View。
- **draw**：递归绘制 View 树到 Canvas 上。`onDraw()` 绘制自身内容，`dispatchDraw()` 绘制子 View。

可以"跳过"：调用 `invalidate()` 只触发 draw，不会重新 measure 和 layout。但 `requestLayout()` 会触发完整的三阶段流程。

### Q2：getMeasuredWidth() 和 getWidth() 有什么区别？什么时候不一样？

**A**：
- `getMeasuredWidth()` = `mMeasuredWidth & MEASURED_SIZE_MASK`，在 `setMeasuredDimension()` 后赋值，即 **measure 阶段结束后**可用。
- `getWidth()` = `mRight - mLeft`，在 `layout()` 中 `setFrame()` 后赋值，即 **layout 阶段结束后**可用。

大多数情况两者相等。不一样的场景：
```java
// 在 onLayout 中人为修改了坐标
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    // measuredWidth = 200，但 layout 时给了 300 的宽度
    child.layout(0, 0, 300, getMeasuredHeight());
    // 此时 child.getMeasuredWidth() = 200, child.getWidth() = 300
}
```

### Q3：为什么自定义 View 直接继承 View 时，wrap_content 不生效？

**A**：`View.onMeasure()` 的默认实现调用 `getDefaultSize()`，其中 `AT_MOST` 和 `EXACTLY` 都返回 `specSize`（父容器给的最大值），导致 `wrap_content` 效果等同于 `match_parent`。解决方案见第四节。

### Q4：requestLayout 和 invalidate 的区别？什么时候用哪个？

**A**：

| 对比项 | `requestLayout()` | `invalidate()` |
|--------|-------------------|----------------|
| 触发阶段 | measure → layout → draw | 仅 draw |
| 标记 flag | `PFLAG_FORCE_LAYOUT` | `PFLAG_DIRTY` |
| 传递方向 | 向上到 ViewRootImpl | 向上传递脏区域 |
| 使用场景 | 尺寸/位置变化 | 仅外观变化 |

经验法则：
- 改了 `paint.textSize`、`padding`、内容数量 → `requestLayout()`
- 改了 `paint.color`、`alpha`、背景色 → `invalidate()`
- 不确定时用 `requestLayout()`（它最终也会触发 draw）

### Q5：View.post(Runnable) 为什么能获取到宽高？

**A**：`View.post()` 会将 Runnable 投递到主线程 Handler 的消息队列中。如果 View 已经 attach 到 Window，Runnable 会在当前消息（包含 `performTraversals`）执行完毕后才执行，此时 measure 和 layout 已经完成。

```java
// 常见用法：在 onCreate 中获取 View 宽高
view.post(() -> {
    int w = view.getWidth();    // 此时已经 layout 完成
    int h = view.getHeight();
});
```

如果 View 尚未 attach，`post()` 会将 Runnable 存入 `RunQueue`，在 `dispatchAttachedToWindow()` 时重新投递，同样保证在 layout 之后执行。

### Q6：硬件加速下 invalidate 的流程和软件绘制有什么不同？

**A**：

**软件绘制**：`invalidate()` → 向上传递脏区域 → `drawSoftware()` → 从 DecorView 递归 `draw()`，所有与脏区域相交的 View 都会重绘。

**硬件加速**：`invalidate()` → 标记当前 View 的 `RenderNode` dirty → `ThreadedRenderer.draw()` → 只重录该 View 的 DisplayList → RenderThread 提交 GPU。

核心优势：硬件加速下 `invalidate()` 不会导致父 View 和兄弟 View 重绘，只更新当前 View 的 DisplayList。这也是属性动画（`setAlpha()`、`setTranslationX()`）在硬件加速下不需要 `invalidate()` 的原因——它们直接修改 RenderNode 属性，由 RenderThread 在下一帧应用。


---

## 四、实战与踩坑

### 4.1 踩坑：自定义 View 中 wrap_content 不生效

如 Q3 所述，直接继承 `View` 时如果不重写 `onMeasure()`，`wrap_content` 会表现得和 `match_parent` 一样。

**正确做法**：在 `onMeasure()` 中针对 `AT_MOST` 模式提供默认尺寸。

```java
public class TagView extends View {

    private String text = "";
    private final Paint paint = new Paint(Paint.ANTI_ALIAS_FLAG);

    public TagView(Context context) { this(context, null); }
    public TagView(Context context, AttributeSet attrs) {
        super(context, attrs);
        paint.setTextSize(spToPx(14f));
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 计算内容所需的宽高
        int contentWidth = (int) paint.measureText(text) + getPaddingStart() + getPaddingEnd();
        int contentHeight = (int) (paint.getFontMetrics().bottom - paint.getFontMetrics().top) +
                getPaddingTop() + getPaddingBottom();

        // 使用 resolveSize 简化 MeasureSpec 处理
        int w = resolveSize(contentWidth, widthMeasureSpec);
        int h = resolveSize(contentHeight, heightMeasureSpec);
        setMeasuredDimension(w, h);
    }
}
```

`resolveSize()` 是 `View` 提供的工具方法，内部逻辑：

```java
// View.resolveSize 等价逻辑
public static int resolveSize(int desiredSize, int measureSpec) {
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
        case MeasureSpec.EXACTLY: return specSize;                          // 父说了算
        case MeasureSpec.AT_MOST: return Math.min(desiredSize, specSize);   // 不超过父的限制
        default:                  return desiredSize;                        // UNSPECIFIED，想要多少给多少
    }
}
```

**更完整的模板**——同时处理 ViewGroup 的 `wrap_content`：

```java
public class FlowLayout extends ViewGroup {

    public FlowLayout(Context context) { this(context, null); }
    public FlowLayout(Context context, AttributeSet attrs) { super(context, attrs); }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int maxWidth = MeasureSpec.getSize(widthMeasureSpec);
        int usedWidth = getPaddingStart();
        int totalHeight = getPaddingTop();
        int lineHeight = 0;

        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (child.getVisibility() == GONE) continue;
            // 让子 View 在父的约束下自行测量
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, totalHeight);
            MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
            int childW = child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin;
            int childH = child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin;

            if (usedWidth + childW + getPaddingEnd() > maxWidth) {
                // 换行
                usedWidth = getPaddingStart();
                totalHeight += lineHeight;
                lineHeight = 0;
            }
            usedWidth += childW;
            lineHeight = Math.max(lineHeight, childH);
        }
        totalHeight += lineHeight + getPaddingBottom();

        setMeasuredDimension(
            resolveSize(usedWidth + getPaddingEnd(), widthMeasureSpec),
            resolveSize(totalHeight, heightMeasureSpec)
        );
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int x = getPaddingStart(); int y = getPaddingTop(); int lineH = 0;
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            if (child.getVisibility() == GONE) continue;
            MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
            if (x + lp.leftMargin + child.getMeasuredWidth() + lp.rightMargin + getPaddingEnd() > getWidth()) {
                x = getPaddingStart(); y += lineH; lineH = 0;
            }
            int cl = x + lp.leftMargin; int ct = y + lp.topMargin;
            child.layout(cl, ct, cl + child.getMeasuredWidth(), ct + child.getMeasuredHeight());
            x += lp.leftMargin + child.getMeasuredWidth() + lp.rightMargin;
            lineH = Math.max(lineH, child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
        }
    }

    // 支持 MarginLayoutParams
    @Override public LayoutParams generateLayoutParams(AttributeSet attrs) { return new MarginLayoutParams(getContext(), attrs); }
    @Override protected LayoutParams generateDefaultLayoutParams() { return new MarginLayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT); }
}
```

### 4.2 踩坑：多次 measure 的性能问题

#### 问题根源

某些 ViewGroup 会对子 View 进行 **两次甚至多次测量**：

| ViewGroup | 多次测量场景 |
|-----------|-------------|
| `LinearLayout` | 使用 `layout_weight` 时，先按 0 权重测量一次，再按权重分配剩余空间测量第二次 |
| `RelativeLayout` | 水平和垂直方向各测量一次（至少 2 次） |
| `ConstraintLayout` | 内部优化，通常只需一次 |

嵌套层级越深，多次测量的指数效应越明显：

```
层级深度 N，每层 2 次测量 → 叶子节点被测量 2^N 次
3 层嵌套 RelativeLayout → 叶子 View 被测量 8 次！
```

#### 检测方法

```java
// 在自定义 View 中统计 onMeasure 调用次数
public class DebugView extends View {

    private int measureCount = 0;

    public DebugView(Context context) { this(context, null); }
    public DebugView(Context context, AttributeSet attrs) { super(context, attrs); }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        measureCount++;
        Log.d("DebugView", "onMeasure #" + measureCount + ", " +
            "width=" + MeasureSpec.toString(widthMeasureSpec) + ", " +
            "height=" + MeasureSpec.toString(heightMeasureSpec));
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }
}
```

也可以通过 **Layout Inspector** 或 **Systrace** 中的 `measure` 标签观察耗时。

#### 优化策略

1. **减少嵌套层级**：用 `ConstraintLayout` 替代多层 `LinearLayout` + `RelativeLayout` 嵌套
2. **避免无意义的 requestLayout**：频繁更新文本时，如果尺寸不变，用 `invalidate()` 代替
3. **自定义 View 中缓存测量结果**：当 MeasureSpec 未变化时，直接复用上次的 `measuredWidth/Height`，跳过昂贵的计算逻辑。

4. **使用 `isLayoutRequested` 避免重复请求**：在频繁更新场景中，先检查再调用。

### 4.3 实战：完整自定义 View 模板

结合以上知识点，一个规范的自定义 View 模板：

```java
public class ProgressRingView extends View {

    // --- 属性 ---
    private float progress = 0f;
    public void setProgress(float value) {
        progress = Math.max(0f, Math.min(value, 1f));
        invalidate();  // 仅外观变化，不需要 requestLayout
    }

    private float ringWidth = dpToPx(12f);
    public void setRingWidth(float value) {
        ringWidth = value;
        bgPaint.setStrokeWidth(value);
        fgPaint.setStrokeWidth(value);
        requestLayout();  // 可能影响尺寸
    }

    private final Paint bgPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final Paint fgPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
    private final RectF rectF = new RectF();

    public ProgressRingView(Context context) { this(context, null); }
    public ProgressRingView(Context context, AttributeSet attrs) { this(context, attrs, 0); }
    public ProgressRingView(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        bgPaint.setStyle(Paint.Style.STROKE);
        bgPaint.setColor(Color.LTGRAY);
        bgPaint.setStrokeWidth(ringWidth);
        fgPaint.setStyle(Paint.Style.STROKE);
        fgPaint.setColor(Color.BLUE);
        fgPaint.setStrokeWidth(ringWidth);
        fgPaint.setStrokeCap(Paint.Cap.ROUND);
    }

    // --- Measure ---
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int defaultSize = (int) (dpToPx(100f) + ringWidth);
        int w = resolveSize(defaultSize + getPaddingStart() + getPaddingEnd(), widthMeasureSpec);
        int h = resolveSize(defaultSize + getPaddingTop() + getPaddingBottom(), heightMeasureSpec);
        setMeasuredDimension(w, h);
    }

    // --- Draw ---
    @Override
    protected void onDraw(Canvas canvas) {
        float half = ringWidth / 2f;
        rectF.set(
            getPaddingStart() + half,
            getPaddingTop() + half,
            getWidth() - getPaddingEnd() - half,
            getHeight() - getPaddingBottom() - half
        );
        // 背景圆环
        canvas.drawArc(rectF, 0f, 360f, false, bgPaint);
        // 前景进度
        canvas.drawArc(rectF, -90f, 360f * progress, false, fgPaint);
    }
}

// dp 转换工具方法
private static float dpToPx(float dp) {
    return dp * Resources.getSystem().getDisplayMetrics().density;
}

private static float spToPx(float sp) {
    return sp * Resources.getSystem().getDisplayMetrics().scaledDensity;
}
```

### 4.4 小结：绘制流程 Checklist

| 检查项 | 说明 |
|--------|------|
| ✅ `onMeasure` 处理了 `wrap_content` | 避免 `AT_MOST` 退化为 `EXACTLY` |
| ✅ 调用了 `setMeasuredDimension()` | 否则抛 `IllegalStateException` |
| ✅ `onLayout` 考虑了 padding 和 margin | 使用 `paddingStart` 而非硬编码 |
| ✅ `onDraw` 中不创建对象 | Paint、Path、RectF 等提前创建为成员变量 |
| ✅ 区分 `invalidate` 和 `requestLayout` | 外观变化用前者，尺寸变化用后者 |
| ✅ 减少嵌套层级 | 避免多次 measure 的指数膨胀 |
| ✅ 硬件加速兼容性 | 检查 `canvas.isHardwareAccelerated`，必要时降级 |

---

> 相关笔记：[[8-事件分发机制]] · [[3-自定义属性与TypedArray]] · [[硬件加速与RenderThread]] · [[ConstraintLayout原理]]

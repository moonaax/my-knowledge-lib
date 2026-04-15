# View 绘制流程

## 一、从 setContentView 到屏幕显示

你在 Activity 里调用 `setContentView(R.layout.activity_main)` 之后，到用户真正看到界面，中间经历了很多步骤。先看全貌：

```
Activity.setContentView()
    ↓
PhoneWindow.setContentView()
    ↓
LayoutInflater.inflate() → 解析 XML，创建 View 树
    ↓
DecorView 添加到 WindowManager
    ↓
ViewRootImpl.setView()
    ↓
ViewRootImpl.requestLayout()
    ↓
ViewRootImpl.scheduleTraversals()
    ↓
Choreographer 等待 VSYNC 信号
    ↓
ViewRootImpl.performTraversals()
    ├── performMeasure()   → View 树的测量
    ├── performLayout()    → View 树的布局
    └── performDraw()      → View 树的绘制
    ↓
用户看到界面
```

---

## 二、View 树的结构

```
DecorView (FrameLayout)
├── LinearLayout
│   ├── ActionBar
│   └── FrameLayout (id: android.R.id.content)
│       └── 你的布局（setContentView 设置的）
│           ├── LinearLayout
│           │   ├── TextView
│           │   └── Button
│           └── ImageView
```

- `DecorView` 是最顶层的 View，是一个 FrameLayout
- `ViewRootImpl` 不是 View，它是 View 树和 WindowManager 之间的桥梁
- 绘制从 `ViewRootImpl.performTraversals()` 开始，自顶向下遍历整棵 View 树

---

## 三、Measure（测量）

测量的目的是确定每个 View 的**宽高**。

### 3.1 MeasureSpec

MeasureSpec 是一个 32 位的 int 值，高 2 位是模式（mode），低 30 位是大小（size）。

| 模式 | 含义 | 对应 XML |
|------|------|---------|
| `EXACTLY` | 精确值，View 的大小就是 specSize | `match_parent` 或具体数值（如 `100dp`） |
| `AT_MOST` | 最大值，View 的大小不能超过 specSize | `wrap_content` |
| `UNSPECIFIED` | 不限制，View 想多大就多大 | 系统内部使用（如 ScrollView 的子 View） |

```java
// MeasureSpec 的组成
int measureSpec = MeasureSpec.makeMeasureSpec(size, mode);
int mode = MeasureSpec.getMode(measureSpec);
int size = MeasureSpec.getSize(measureSpec);
```

### 3.2 MeasureSpec 的确定规则

子 View 的 MeasureSpec 由**父 View 的 MeasureSpec** + **子 View 的 LayoutParams** 共同决定：

| 父 MeasureSpec \ 子 LayoutParams | 具体数值（如 100dp） | match_parent | wrap_content |
|----------------------------------|---------------------|-------------|-------------|
| EXACTLY (parentSize) | EXACTLY (childSize) | EXACTLY (parentSize) | AT_MOST (parentSize) |
| AT_MOST (parentSize) | EXACTLY (childSize) | AT_MOST (parentSize) | AT_MOST (parentSize) |
| UNSPECIFIED | EXACTLY (childSize) | UNSPECIFIED (0) | UNSPECIFIED (0) |

### 3.3 测量流程

```
ViewRootImpl.performMeasure()
    ↓
DecorView.measure(widthSpec, heightSpec)
    ↓
DecorView.onMeasure()  → 作为 ViewGroup，遍历子 View
    ↓
child.measure() → child.onMeasure()
    ↓
继续递归...直到叶子节点
```

**View 的 measure 流程：**
```java
// View.java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    // measure 是 final 的，不能重写！
    // 内部会调用 onMeasure
    onMeasure(widthMeasureSpec, heightMeasureSpec);
}

// 默认的 onMeasure
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(
        getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec)
    );
}
```

**自定义 View 重写 onMeasure：**
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);

    int width;
    if (widthMode == MeasureSpec.EXACTLY) {
        width = widthSize;                          // match_parent 或具体值
    } else if (widthMode == MeasureSpec.AT_MOST) {
        width = Math.min(desiredWidth, widthSize);  // wrap_content
    } else {
        width = desiredWidth;                        // UNSPECIFIED
    }

    // 必须调用 setMeasuredDimension，否则会抛异常
    setMeasuredDimension(width, height);
}
```

> **重要：** 如果自定义 View 不重写 onMeasure，`wrap_content` 和 `match_parent` 效果一样！因为默认实现中 AT_MOST 模式也返回 specSize（父容器大小）。

### 3.4 ViewGroup 的测量

ViewGroup 需要先测量所有子 View，再根据子 View 的大小确定自己的大小。

```java
// ViewGroup 提供的辅助方法
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec) {
    for (int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        measureChild(child, widthMeasureSpec, heightMeasureSpec);
    }
}

protected void measureChild(View child, int parentWidthSpec, int parentHeightSpec) {
    LayoutParams lp = child.getLayoutParams();
    int childWidthSpec = getChildMeasureSpec(parentWidthSpec, padding, lp.width);
    int childHeightSpec = getChildMeasureSpec(parentHeightSpec, padding, lp.height);
    child.measure(childWidthSpec, childHeightSpec);
}
```

---

## 四、Layout（布局）

布局的目的是确定每个 View 的**位置**（left, top, right, bottom）。

### 4.1 布局流程

```
ViewRootImpl.performLayout()
    ↓
DecorView.layout(left, top, right, bottom)
    ↓
DecorView.onLayout()  → 遍历子 View，调用 child.layout()
    ↓
child.layout() → child.onLayout()
    ↓
继续递归...
```

```java
// View.java
public void layout(int l, int t, int r, int b) {
    // 1. 保存旧位置
    int oldL = mLeft, oldT = mTop, oldR = mRight, oldB = mBottom;

    // 2. 设置新位置
    boolean changed = setFrame(l, t, r, b);

    // 3. 如果位置变了或者需要重新布局
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) != 0) {
        onLayout(changed, l, t, r, b);  // 回调 onLayout
    }
}
```

**View 的 onLayout：** 空实现（叶子节点不需要布局子 View）

**ViewGroup 的 onLayout：** 必须重写，确定每个子 View 的位置

```java
// 自定义 ViewGroup 示例：垂直排列子 View
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    int currentTop = getPaddingTop();
    for (int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        if (child.getVisibility() != View.GONE) {
            child.layout(
                getPaddingLeft(),
                currentTop,
                getPaddingLeft() + child.getMeasuredWidth(),
                currentTop + child.getMeasuredHeight()
            );
            currentTop += child.getMeasuredHeight();
        }
    }
}
```

### 4.2 getMeasuredWidth vs getWidth

| 方法 | 含义 | 可用时机 |
|------|------|---------|
| `getMeasuredWidth()` | 测量宽度（measure 后的值） | onMeasure 之后 |
| `getWidth()` | 实际宽度（right - left） | onLayout 之后 |

通常两者相等，但如果在 layout 时故意改变了位置，它们可能不同：
```java
// 故意让实际宽度和测量宽度不同
child.layout(0, 0, child.getMeasuredWidth() + 100, child.getMeasuredHeight());
// getMeasuredWidth() = 原始值
// getWidth() = 原始值 + 100
```

---

## 五、Draw（绘制）

绘制的目的是把 View 的内容画到屏幕上。

### 5.1 绘制流程（6 步）

```java
// View.draw() 的流程
public void draw(Canvas canvas) {
    // 1. 绘制背景
    drawBackground(canvas);

    // 2. 保存 canvas 图层（为 fading edge 准备，通常跳过）

    // 3. 绘制自身内容
    onDraw(canvas);

    // 4. 绘制子 View
    dispatchDraw(canvas);

    // 5. 绘制前景（滚动条等）
    onDrawForeground(canvas);

    // 6. 绘制装饰（焦点高亮等）
}
```

**自定义 View 主要重写 `onDraw()`：**
```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    // 画圆
    canvas.drawCircle(getWidth() / 2f, getHeight() / 2f, radius, paint);

    // 画文字
    canvas.drawText("Hello", x, y, textPaint);

    // 画路径
    canvas.drawPath(path, paint);
}
```

**ViewGroup 的 `dispatchDraw()`：**
```java
// 遍历子 View，调用 child.draw()
protected void dispatchDraw(Canvas canvas) {
    for (int i = 0; i < childCount; i++) {
        drawChild(canvas, getChildAt(i), drawingTime);
    }
}
```

### 5.2 Canvas 和 Paint

```java
// Paint：画笔，控制颜色、粗细、样式
Paint paint = new Paint();
paint.setColor(Color.RED);
paint.setStrokeWidth(5f);
paint.setStyle(Paint.Style.STROKE);  // FILL（填充）/ STROKE（描边）/ FILL_AND_STROKE
paint.setAntiAlias(true);  // 抗锯齿

// Canvas：画布，提供绑定方法
canvas.drawLine(0f, 0f, 100f, 100f, paint);      // 画线
canvas.drawRect(rect, paint);                      // 画矩形
canvas.drawCircle(cx, cy, radius, paint);          // 画圆
canvas.drawBitmap(bitmap, left, top, paint);       // 画图片
canvas.drawPath(path, paint);                      // 画路径
canvas.drawText("text", x, y, paint);              // 画文字

// Canvas 变换
canvas.save();            // 保存当前状态
canvas.translate(dx, dy); // 平移
canvas.rotate(degrees);   // 旋转
canvas.scale(sx, sy);     // 缩放
canvas.restore();         // 恢复到 save 时的状态
```

---

## 六、requestLayout vs invalidate vs postInvalidate

| 方法 | 触发流程 | 使用场景 |
|------|---------|---------|
| `requestLayout()` | measure → layout → draw | View 的大小或位置变了 |
| `invalidate()` | 只触发 draw | View 的内容变了（颜色、文字等），大小没变 |
| `postInvalidate()` | 同 invalidate，但可以在子线程调用 | 子线程中需要刷新 UI |

```java
// 改变了文字颜色 → 只需要重绘
textView.setTextColor(Color.RED);
textView.invalidate();

// 改变了文字内容（可能影响大小）→ 需要重新测量
textView.setText("longer text");
textView.requestLayout();
```

**requestLayout 的传递：**
```
子 View.requestLayout()
    ↓ 设置 PFLAG_FORCE_LAYOUT 标记
父 View.requestLayout()
    ↓ 继续向上传递
ViewRootImpl.requestLayout()
    ↓
scheduleTraversals() → performTraversals()
```

---

## 七、硬件加速

### 7.1 软件绘制 vs 硬件加速

| 对比项 | 软件绘制 | 硬件加速 |
|--------|---------|---------|
| 渲染方式 | CPU 逐像素绘制到 Bitmap | GPU 通过 OpenGL/Vulkan 渲染 |
| 性能 | 慢 | 快（GPU 擅长图形计算） |
| 内存 | 低 | 高（需要额外的 GPU 缓存） |
| 默认状态 | Android 3.0 之前 | Android 4.0+ 默认开启 |

### 7.2 硬件加速的绘制流程

```
软件绘制：
View.draw() → Canvas（CPU）→ Bitmap → SurfaceFlinger → 屏幕

硬件加速：
View.draw() → DisplayList（记录绘制命令）→ RenderThread → GPU → SurfaceFlinger → 屏幕
```

**DisplayList 的优势：**
- 只记录绘制命令，不立即执行
- View 内容没变时可以复用 DisplayList，不需要重新执行 onDraw
- invalidate 时只需要重新执行 DisplayList 中的命令，不需要重新遍历 View 树

### 7.3 不支持硬件加速的操作

某些 Canvas 操作在硬件加速下不支持：
- `Canvas.drawBitmapMesh()`
- `Paint.setMaskFilter()`（BlurMaskFilter 部分支持）
- `Canvas.drawPicture()`

```java
// 对单个 View 关闭硬件加速
view.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
```

---

## 八、VSYNC 与 Choreographer

### 8.1 VSYNC 信号

VSYNC（Vertical Synchronization）是屏幕的垂直同步信号，通常 60Hz 屏幕每 16.6ms 发送一次。

**没有 VSYNC 的问题：**
- CPU/GPU 的绘制时机和屏幕刷新不同步
- 导致画面撕裂（Tearing）和卡顿（Jank）

### 8.2 Choreographer

Choreographer（编舞者）负责协调 VSYNC 信号和 UI 绘制。

```
VSYNC 信号到来
    ↓
Choreographer.doFrame()
    ├── 处理 Input 事件（CALLBACK_INPUT）
    ├── 处理动画（CALLBACK_ANIMATION）
    ├── 处理 View 绘制（CALLBACK_TRAVERSAL）
    └── 处理提交（CALLBACK_COMMIT）
```

```java
// ViewRootImpl.scheduleTraversals()
void scheduleTraversals() {
    mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
    mChoreographer.postCallback(
        Choreographer.CALLBACK_TRAVERSAL,
        mTraversalRunnable, null);
    // 等待下一个 VSYNC 信号到来时执行 mTraversalRunnable
}
```

**利用 Choreographer 监控帧率：**
```java
Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
    long lastFrameTimeNanos = 0;

    @Override
    public void doFrame(long frameTimeNanos) {
        if (lastFrameTimeNanos != 0) {
            long frameTime = (frameTimeNanos - lastFrameTimeNanos) / 1_000_000;
            if (frameTime > 16) {
                Log.w("FPS", "掉帧！帧耗时: " + frameTime + "ms");
            }
        }
        lastFrameTimeNanos = frameTimeNanos;
        Choreographer.getInstance().postFrameCallback(this);
    }
});
```

---

## 九、面试题库

### Q1：View 的绘制流程是什么？

> View 的绘制从 ViewRootImpl.performTraversals() 开始，依次经历三个阶段：measure（测量）确定 View 的宽高、layout（布局）确定 View 在父容器中的位置、draw（绘制）将 View 的内容画到屏幕上。整个过程是自顶向下递归的，从 DecorView 开始遍历整棵 View 树。触发时机是 ViewRootImpl.scheduleTraversals() 通过 Choreographer 注册 VSYNC 回调，等下一个 VSYNC 信号到来时执行。在此之前会设置同步屏障确保绘制消息优先执行。

### Q2：MeasureSpec 是什么？子 View 的 MeasureSpec 怎么确定的？

> MeasureSpec 是一个 32 位 int 值，高 2 位表示测量模式（EXACTLY、AT_MOST、UNSPECIFIED），低 30 位表示大小。子 View 的 MeasureSpec 由父 View 的 MeasureSpec 和子 View 自身的 LayoutParams 共同决定。比如父是 EXACTLY 且子是 match_parent，子就是 EXACTLY + 父的大小；父是 EXACTLY 且子是 wrap_content，子就是 AT_MOST + 父的大小。这个计算逻辑在 ViewGroup.getChildMeasureSpec() 方法中。理解这个规则对自定义 ViewGroup 非常重要。

### Q3：自定义 View 不重写 onMeasure 会怎样？wrap_content 为什么会失效？

> 默认的 onMeasure 实现调用 getDefaultSize()，在 AT_MOST 模式下返回的是 specSize（父容器给的最大值），和 EXACTLY 模式一样。所以 wrap_content 和 match_parent 效果相同，都是填满父容器。解决方案是重写 onMeasure，在 AT_MOST 模式下根据内容计算实际需要的大小，取内容大小和 specSize 的较小值。这是自定义 View 最基本的要求，面试中如果说不出来基本就凉了。

### Q4：requestLayout 和 invalidate 的区别？

> requestLayout 会触发完整的 measure → layout → draw 流程，适用于 View 的大小或位置发生变化的场景。invalidate 只触发 draw 流程，适用于 View 内容变化但大小不变的场景（比如改变颜色、文字）。requestLayout 会从当前 View 向上逐级传递到 ViewRootImpl，每个父 View 都会被标记为需要重新布局。invalidate 只标记当前 View 的绘制区域为脏区域。性能上 invalidate 开销更小，能用 invalidate 就不要用 requestLayout。另外 invalidate 必须在主线程调用，子线程用 postInvalidate。

### Q5：getMeasuredWidth 和 getWidth 的区别？

> getMeasuredWidth 是 measure 阶段确定的测量宽度，在 onMeasure 之后就有值了。getWidth 是 layout 阶段确定的实际宽度，等于 right - left，在 onLayout 之后才有值。正常情况下两者相等，但如果在 layout 时人为改变了 View 的位置（比如 child.layout 时传入了不同于测量值的参数），两者就会不同。实际开发中在 onLayout 之后用 getWidth 更准确。

### Q6：View.post(Runnable) 为什么能获取到 View 的宽高？

> View.post 的 Runnable 会被加入到 ViewRootImpl 的 Handler 消息队列中。由于 performTraversals（measure + layout + draw）也是通过 Handler 消息触发的，而 post 的消息排在 performTraversals 之后，所以执行 Runnable 时 View 已经完成了测量和布局，自然能获取到正确的宽高。如果 View 还没有 attach 到 Window，post 的 Runnable 会被暂存在 View 的 RunQueue 中，等 attach 后再发送到 Handler。

### Q7：硬件加速和软件绘制的区别？

> 软件绘制是 CPU 通过 Skia 库逐像素绘制到 Bitmap，然后提交给 SurfaceFlinger 合成显示。硬件加速是将绘制命令记录到 DisplayList 中，由 RenderThread 通过 GPU（OpenGL/Vulkan）执行渲染。硬件加速的优势是 GPU 擅长图形计算，性能更好；而且 DisplayList 可以复用，View 内容没变时不需要重新执行 onDraw。劣势是某些 Canvas 操作不支持硬件加速（如 drawBitmapMesh），需要对单个 View 设置 LAYER_TYPE_SOFTWARE 回退到软件绘制。Android 4.0 之后默认开启硬件加速。

### Q8：VSYNC 和 Choreographer 的作用？

> VSYNC 是屏幕的垂直同步信号，60Hz 屏幕每 16.6ms 发一次。Choreographer 负责协调 VSYNC 信号和应用的 UI 工作，它在收到 VSYNC 信号后按顺序处理输入事件、动画、View 绘制。ViewRootImpl.scheduleTraversals() 就是通过 Choreographer 注册回调，等下一个 VSYNC 到来时执行 performTraversals。这个机制保证了 CPU/GPU 的绘制和屏幕刷新同步，避免画面撕裂。我们也可以利用 Choreographer.postFrameCallback 来监控帧率，检测掉帧。

### Q9：一个 View 从创建到显示在屏幕上经历了什么？

> 首先 Activity.setContentView 通过 PhoneWindow 创建 DecorView 并将我们的布局 inflate 添加到 ContentView 区域。然后在 Activity.onResume 之后，WindowManager 将 DecorView 添加到窗口，创建 ViewRootImpl。ViewRootImpl.setView 中调用 requestLayout 触发首次绘制。scheduleTraversals 通过 Choreographer 注册 VSYNC 回调，设置同步屏障。VSYNC 信号到来后执行 performTraversals，依次完成 measure、layout、draw 三个阶段。硬件加速模式下绘制命令被记录到 DisplayList，由 RenderThread 提交给 GPU 渲染，最终通过 SurfaceFlinger 合成显示到屏幕上。

### Q10：如何优化 View 的绘制性能？

> 几个关键方向：一是减少布局层级，用 ConstraintLayout 替代多层嵌套的 LinearLayout/RelativeLayout，减少 measure 和 layout 的递归深度。二是避免在 onDraw 中创建对象（Paint、Rect 等），因为 onDraw 每帧都可能调用，频繁创建对象会触发 GC 导致卡顿。三是使用 clipRect 缩小绘制区域，避免过度绘制。四是合理使用硬件加速的 View Layer，对频繁变化的复杂 View 设置 LAYER_TYPE_HARDWARE 缓存 DisplayList。五是用 ViewStub 延迟加载不常用的布局。六是用 merge 标签减少不必要的层级。可以用 GPU 过度绘制调试工具和 Layout Inspector 来定位问题。

# 自定义 ViewGroup 布局

> 重写 onMeasure + onLayout 实现自定义排列逻辑，流式布局是最经典的面试题。
> 关联：[[1-View绘制流程-measure-layout-draw]] · [[5-事件分发与冲突处理]] · [[7-复合组件与自定义组合控件]]

---

## 一、自定义 ViewGroup 的核心区别

| 继承自 | 核心职责 | 需要重写 |
|--------|---------|---------|
| **View** | 绘制自己 | `onMeasure` + `onDraw` |
| **ViewGroup** | 排列子 View | `onMeasure` + `onLayout`（通常不需要 onDraw） |

ViewGroup 的核心工作是：
1. **测量**：确定每个子 View 的大小和自己的大小
2. **布局**：确定每个子 View 放在哪个位置

---

## 二、onMeasure 模板

### 2.1 测量子 View

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // 1. 测量所有子 View
    measureChildren(widthMeasureSpec, heightMeasureSpec);

    // 2. 计算自己的宽高
    int width = calculateWidth(widthMeasureSpec);
    int height = calculateHeight(heightMeasureSpec);

    // 3. 设置自己的测量宽高
    setMeasuredDimension(width, height);
}
```

### 2.2 处理 wrap_content

ViewGroup 默认不处理 wrap_content，必须手动计算。

```java
private int calculateWidth(int widthMeasureSpec) {
    int mode = MeasureSpec.getMode(widthMeasureSpec);
    int size = MeasureSpec.getSize(widthMeasureSpec);

    if (mode == MeasureSpec.EXACTLY) {
        // match_parent 或固定值 → 直接用
        return size;
    }

    // wrap_content → 计算所有子 View 的宽高
    int maxWidth = 0;
    for (int i = 0; i < getChildCount(); i++) {
        View child = getChildAt(i);
        maxWidth = Math.max(maxWidth, child.getMeasuredWidth());
    }
    maxWidth += getPaddingLeft() + getPaddingRight();

    // AT_MOST 不能超过父容器给的 size
    return Math.min(maxWidth, size);
}
```

---

## 三、实战一：流式布局（FlowLayout）

流式布局是最经典的自定义 ViewGroup 面试题——子 View 一行排满后自动换行。

```
[标签1] [标签2] [标签3] [标签4] [标签5]
[标签6] [标签7]
[标签8] [标签9] [标签10]
```

### 3.1 完整实现

```java
public class FlowLayout extends ViewGroup {

    // 每行的子 View 列表（用于 layout）
    private List<List<View>> allLines = new ArrayList<>();
    // 每行的高度
    private List<Integer> lineHeights = new ArrayList<>();

    public FlowLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        allLines.clear();
        lineHeights.clear();

        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);

        // 当前行已用宽度
        int lineWidth = 0;
        // 当前行最大高度
        int lineHeight = 0;
        // 整体宽高
        int totalWidth = 0;
        int totalHeight = 0;

        List<View> currentLine = new ArrayList<>();

        int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            if (child.getVisibility() == GONE) continue;

            // 测量子 View（考虑父容器的限制）
            measureChild(child, widthMeasureSpec, heightMeasureSpec);

            int childWidth = child.getMeasuredWidth();
            int childHeight = child.getMeasuredHeight();

            // 判断是否需要换行
            if (lineWidth + childWidth > widthSize - getPaddingLeft() - getPaddingRight()) {
                // 换行：保存当前行
                allLines.add(currentLine);
                lineHeights.add(lineHeight);
                totalHeight += lineHeight;
                totalWidth = Math.max(totalWidth, lineWidth);

                // 新的一行
                currentLine = new ArrayList<>();
                lineWidth = 0;
                lineHeight = 0;
            }

            // 加入当前行
            currentLine.add(child);
            lineWidth += childWidth;
            lineHeight = Math.max(lineHeight, childHeight);
        }

        // 处理最后一行
        if (!currentLine.isEmpty()) {
            allLines.add(currentLine);
            lineHeights.add(lineHeight);
            totalHeight += lineHeight;
            totalWidth = Math.max(totalWidth, lineWidth);
        }

        // 加上 padding
        totalWidth += getPaddingLeft() + getPaddingRight();
        totalHeight += getPaddingTop() + getPaddingBottom();

        // 设置最终宽高
        int finalWidth = (widthMode == MeasureSpec.EXACTLY) ? widthSize : totalWidth;
        int finalHeight = (heightMode == MeasureSpec.EXACTLY) ? heightSize : totalHeight;
        setMeasuredDimension(finalWidth, finalHeight);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int curX = getPaddingLeft();
        int curY = getPaddingTop();

        for (int i = 0; i < allLines.size(); i++) {
            List<View> line = allLines.get(i);
            int lineHeight = lineHeights.get(i);

            for (View child : line) {
                int childWidth = child.getMeasuredWidth();
                int childHeight = child.getMeasuredHeight();

                // 垂直居中
                int childTop = curY + (lineHeight - childHeight) / 2;
                child.layout(curX, childTop, curX + childWidth, childTop + childHeight);

                curX += childWidth;
            }

            // 换行
            curX = getPaddingLeft();
            curY += lineHeight;
        }
    }
}
```

### 3.2 带间距的版本

上面的实现没有间距，实际使用需要加 `horizontalSpacing` 和 `verticalSpacing`。

```java
// 在 onMeasure 中
if (lineWidth + horizontalSpacing + childWidth > availableWidth) {
    // 换行时，lineWidth 减去最后一个 horizontalSpacing
    lineWidth -= horizontalSpacing;
}

lineWidth += childWidth + horizontalSpacing;

// 在 onLayout 中
curX += childWidth + horizontalSpacing;
curY += lineHeight + verticalSpacing;
```

### 3.3 XML 中使用

```xml
<com.example.widget.FlowLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="8dp">

    <TextView android:text="Android" style="@style/Tag" />
    <TextView android:text="Kotlin" style="@style/Tag" />
    <TextView android:text="Java" style="@style/Tag" />
    <TextView android:text="Jetpack Compose" style="@style/Tag" />
    <TextView android:text="React Native" style="@style/Tag" />
</com.example.widget.FlowLayout>
```

---

## 四、实战二：圆形布局（CircleLayout）

子 View 围绕中心点均匀分布。

```java
public class CircleLayout extends ViewGroup {

    public CircleLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 测量所有子 View
        measureChildren(widthMeasureSpec, heightMeasureSpec);

        // 取宽高中的较小值作为直径
        int size = Math.min(
            MeasureSpec.getSize(widthMeasureSpec),
            MeasureSpec.getSize(heightMeasureSpec));
        setMeasuredDimension(size, size);
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int childCount = getChildCount();
        if (childCount == 0) return;

        int centerX = getWidth() / 2;
        int centerY = getHeight() / 2;
        int radius = Math.min(centerX, centerY) - dp2px(40); // 留边距

        float angleStep = 360f / childCount;

        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            int childWidth = child.getMeasuredWidth();
            int childHeight = child.getMeasuredHeight();

            // 计算子 View 中心点
            double angle = Math.toRadians(angleStep * i - 90); // 从顶部开始
            int childCenterX = (int) (centerX + radius * Math.cos(angle));
            int childCenterY = (int) (centerY + radius * Math.sin(angle));

            // 计算左上角坐标
            int childLeft = childCenterX - childWidth / 2;
            int childTop = childCenterY - childHeight / 2;
            child.layout(childLeft, childTop,
                childLeft + childWidth, childTop + childHeight);
        }
    }
}
```

---

## 五、实战三：标签选择器（TagLayout）

和 FlowLayout 类似，但子 View 是可点击的标签，支持单选/多选。

```java
public class TagLayout extends FlowLayout {

    private int selectMode = 0; // 0=单选, 1=多选
    private int selectedPosition = -1; // 单选用
    private Set<Integer> selectedSet = new HashSet<>(); // 多选用
    private OnTagSelectedListener listener;

    public TagLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        TypedArray ta = context.obtainStyledAttributes(attrs, R.styleable.TagLayout);
        selectMode = ta.getInt(R.styleable.TagLayout_selectMode, 0);
        ta.recycle();
    }

    @Override
    public void addView(View child) {
        super.addView(child);
        // 每个子 View 自动设置点击事件
        int position = getChildCount() - 1;
        child.setOnClickListener(v -> onTagClick(position));
    }

    private void onTagClick(int position) {
        if (selectMode == 0) {
            // 单选
            if (selectedPosition == position) return;
            // 取消上一个
            if (selectedPosition >= 0) {
                getChildAt(selectedPosition).setSelected(false);
            }
            // 选中当前
            selectedPosition = position;
            getChildAt(position).setSelected(true);
        } else {
            // 多选
            View child = getChildAt(position);
            child.setSelected(!child.isSelected());
            if (child.isSelected()) {
                selectedSet.add(position);
            } else {
                selectedSet.remove(position);
            }
        }
        if (listener != null) listener.onSelected(getSelectedPositions());
    }

    public List<Integer> getSelectedPositions() {
        if (selectMode == 0) {
            return selectedPosition >= 0 ?
                Collections.singletonList(selectedPosition) : Collections.emptyList();
        }
        return new ArrayList<>(selectedSet);
    }

    public void setOnTagSelectedListener(OnTagSelectedListener listener) {
        this.listener = listener;
    }

    public interface OnTagSelectedListener {
        void onSelected(List<Integer> positions);
    }
}
```

---

## 六、onMeasure 与 onLayout 的调用时机

```
触发 measure 的场景：
├── View 加入 ViewGroup 时
├── View 的 visibility 从 GONE 变为 VISIBLE
├── requestLayout() 被调用
└── 父容器重新布局

触发 layout 的场景：
├── measure 完成后（通常一起调用）
├── requestLayout() → 会触发 measure + layout
└── layout() 被显式调用

注意：invalidate() 只触发 draw，不触发 measure/layout
      requestLayout() 触发 measure + layout，可能触发 draw
```

---

## 七、面试要点

**Q: 自定义 ViewGroup 的 onMeasure 和 onLayout 分别做什么？**

onMeasure 负责确定每个子 View 的大小和 ViewGroup 自己的大小——遍历子 View 调用 measureChild，根据子 View 尺寸计算自己的宽高。onLayout 负责确定每个子 View 的位置——遍历子 View 调用 child.layout(l, t, r, b) 设置坐标。

**Q: 流式布局怎么实现？**

核心思路：onMeasure 中遍历子 View，累加宽度判断是否换行，记录每行的子 View 列表和行高；onLayout 中遍历每行，给每个子 View 设置坐标。关键点是处理 wrap_content（自己计算宽高）、padding/margin（累加偏移）。

**Q: requestLayout 和 invalidate 的区别？**

requestLayout 触发 measure + layout + draw 全流程，用于布局发生变化（大小/位置改变）。invalidate 只触发 draw，用于内容变化但布局不变（如背景色改变、文本更新）。自定义 ViewGroup 中，子 View 大小改变用 requestLayout，子 View 内容改变用 invalidate。

**Q: measureChild 和 measureChildWithMargins 的区别？**

measureChild 只考虑 ViewGroup 的 padding；measureChildWithMargins 额外考虑子 View 的 margin。如果子 View 需要 margin 效果，必须用 measureChildWithMargins，并在 LayoutParams 中获取 margin 值。

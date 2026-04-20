# RecyclerView 自定义 LayoutManager

> 深入理解 [[RecyclerView]] 的布局机制，掌握自定义 [[LayoutManager]] 的核心原理与实战技巧。

---

## 一、核心概念

### 1.1 LayoutManager 的职责

[[LayoutManager]] 是 [[RecyclerView]] 架构中负责 **子 View 测量与布局** 的核心组件。RecyclerView 本身不关心子 View 如何排列，所有布局逻辑都委托给 LayoutManager。其核心职责包括：

1. **布局子 View**：决定每个 ItemView 的位置和大小（`onLayoutChildren()`）
2. **处理滚动**：响应用户的滑动手势，计算滚动偏移量（`scrollHorizontallyBy()` / `scrollVerticallyBy()`）
3. **回收与复用**：在滚动过程中，将移出屏幕的 View 回收，将即将进入屏幕的 View 从缓存中取出或重新创建
4. **焦点管理**：处理键盘导航和焦点查找
5. **无障碍支持**：提供 AccessibilityEvent 信息

系统内置了三种 LayoutManager：

| LayoutManager | 布局方式 | 典型场景 |
|---|---|---|
| [[LinearLayoutManager]] | 线性列表（水平/垂直） | 聊天列表、设置页 |
| [[GridLayoutManager]] | 网格布局 | 图片墙、商品列表 |
| [[StaggeredGridLayoutManager]] | 瀑布流 | Pinterest 风格 |

### 1.2 RecyclerView 四级缓存机制

[[RecyclerView]] 的高性能核心在于其精心设计的 **四级缓存机制**，由内部类 `Recycler` 管理：

````
┌─────────────────────────────────────────────────┐
│              Recycler 缓存查找链                  │
│                                                   │
│  第一级  mAttachedScrap / mChangedScrap (Scrap)  │
│    ↓ 未命中                                       │
│  第二级  mCachedViews (Cache)                     │
│    ↓ 未命中                                       │
│  第三级  ViewCacheExtension (自定义扩展)           │
│    ↓ 未命中                                       │
│  第四级  RecycledViewPool (回收池)                 │
│    ↓ 未命中                                       │
│  创建新 ViewHolder (onCreateViewHolder)           │
└─────────────────────────────────────────────────┘
````
#### 第一级：Scrap 缓存

````java
// RecyclerView.Recycler 源码
ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
ArrayList<ViewHolder> mChangedScrap = null;
````
- **mAttachedScrap**：存储当前仍 attached 到 RecyclerView 但被标记为"临时移除"的 ViewHolder。在 `onLayoutChildren()` 重新布局时，先将所有子 View detach 到 scrap，布局完成后未被使用的再移入其他缓存
- **mChangedScrap**：存储数据已变更（`notifyItemChanged`）的 ViewHolder，需要重新 `onBindViewHolder()`
- **特点**：不经过 `onBindViewHolder()`（mAttachedScrap），直接复用，速度最快

#### 第二级：Cache 缓存（mCachedViews）

````java
ArrayList<ViewHolder> mCachedViews = new ArrayList<>();
int mViewCacheMax = DEFAULT_CACHE_SIZE; // 默认为 2
````
- 存储最近被移出屏幕的 ViewHolder，**按 position 精确匹配**
- 默认容量为 **2**，可通过 `setItemViewCacheSize()` 修改
- 命中时 **不需要** 重新 `onBindViewHolder()`，因为数据未变
- 采用 FIFO 策略，满了之后最早的会被移入 RecycledViewPool

#### 第三级：ViewCacheExtension

````java
public abstract class ViewCacheExtension {
    public abstract View getViewForPositionAndType(
        Recycler recycler, int position, int type
    );
}
````
- 开发者自定义的缓存层，很少使用
- 典型场景：固定位置的广告位 View 缓存

#### 第四级：RecycledViewPool

````java
public class RecycledViewPool {
    // 按 viewType 分组存储
    SparseArray<ScrapData> mScrap = new SparseArray<>();

    class ScrapData {
        ArrayList<ViewHolder> mScrapHeap = new ArrayList<>(DEFAULT_MAX_SCRAP); // 默认 5
    }
}
````
- 按 **viewType** 分组存储，每种类型默认最多 **5** 个
- 命中时 **需要** 重新 `onBindViewHolder()`，因为只匹配类型不匹配位置
- 可以在多个 [[RecyclerView]] 之间共享（如 [[ViewPager2]] 的多个页面）

### 1.3 回收与复用流程

**回收流程**（View 滑出屏幕时）：

1. LayoutManager 调用 `removeAndRecycleView()` 或 `detachAndScrapView()`
2. 优先放入 `mCachedViews`（如果未满且 position 有效）
3. `mCachedViews` 已满 → 最早的 ViewHolder 被移入 `RecycledViewPool`
4. `RecycledViewPool` 对应 viewType 也满了 → ViewHolder 被丢弃等待 GC

**复用流程**（View 滑入屏幕时）：

1. 调用 `Recycler.getViewForPosition(position)`
2. 按四级缓存顺序查找（详见第二章源码分析）
3. 找到后根据缓存层级决定是否需要 `onBindViewHolder()`
4. 全部未命中 → `onCreateViewHolder()` + `onBindViewHolder()`



---

## 二、原理与源码

### 2.1 LinearLayoutManager.onLayoutChildren 源码分析

`onLayoutChildren()` 是 LayoutManager 最核心的方法，[[LinearLayoutManager]] 的实现流程如下：

````java
// LinearLayoutManager.java（简化版）
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // 1. 确定锚点（Anchor Point）
    //    锚点 = 布局的起始参考位置，包含 position 和 coordinate
    updateAnchorInfoForLayout(recycler, state, mAnchorInfo);

    // 2. 将当前所有子 View detach 到 Scrap 缓存
    detachAndScrapAttachedViews(recycler);

    // 3. 从锚点向两个方向填充
    if (mAnchorInfo.mLayoutFromEnd) {
        // 先向上填充，再向下填充
        updateLayoutStateToFillStart(mAnchorInfo);
        fill(recycler, mLayoutState, state, false);  // ← 核心填充方法
        updateLayoutStateToFillEnd(mAnchorInfo);
        fill(recycler, mLayoutState, state, false);
    } else {
        // 先向下填充，再向上填充
        updateLayoutStateToFillEnd(mAnchorInfo);
        fill(recycler, mLayoutState, state, false);
        updateLayoutStateToFillStart(mAnchorInfo);
        fill(recycler, mLayoutState, state, false);
    }

    // 4. 回收多余的 Scrap View
    layoutForPredictiveAnimations(recycler, state, startOffset, endOffset);
}
````
**关键步骤解读**：

1. **确定锚点**：找到一个参考位置（通常是第一个可见 item 或焦点 item），作为布局的起点
2. **Detach & Scrap**：将所有子 View 临时放入 Scrap 缓存，而不是直接 remove。这样在后续 fill 时可以快速从 Scrap 中取回
3. **双向填充**：从锚点分别向起始方向和结束方向填充 View，直到填满可见区域
4. **清理**：将 Scrap 中未被复用的 ViewHolder 移入 Cache 或 Pool

### 2.2 fill() 与 layoutChunk() 流程

`fill()` 是实际填充 View 的循环方法：

````java
// LinearLayoutManager.java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
         RecyclerView.State state, boolean stopOnFocusable) {
    final int start = layoutState.mAvailable; // 剩余可用空间
    int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;

    while ((layoutState.mInfinite || remainingSpace > 0)
            && layoutState.hasMore(state)) {
        // 逐个填充子 View
        layoutChunkResult.resetInternal();
        layoutChunk(recycler, state, layoutState, layoutChunkResult);

        // 更新已消耗的空间
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;
        layoutState.mAvailable -= layoutChunkResult.mConsumed;
        remainingSpace -= layoutChunkResult.mConsumed;
    }
    // 返回实际填充的像素数
    return start - layoutState.mAvailable;
}
````
`layoutChunk()` 负责单个 View 的获取、测量和布局：

````java
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
                 LayoutState layoutState, LayoutChunkResult result) {
    // 1. 从 Recycler 获取 View（核心！触发缓存查找链）
    View view = layoutState.next(recycler);

    // 2. 添加到 RecyclerView
    if (layoutState.mScrapList == null) {
        addView(view); // 内部调用 ViewGroup.addView
    }

    // 3. 测量（考虑 ItemDecoration 的 insets）
    measureChildWithMargins(view, 0, 0);

    // 4. 布局（确定 left, top, right, bottom）
    layoutDecoratedWithMargins(view, left, top, right, bottom);

    result.mConsumed = orientationHelper.getDecoratedMeasurement(view);
}
````
**流程图**：

````
onLayoutChildren()
    ├── detachAndScrapAttachedViews()  → 子 View 进入 Scrap
    └── fill()
         └── while (有剩余空间 && 有更多数据)
              └── layoutChunk()
                   ├── recycler.getViewForPosition()  → 从缓存获取 View
                   ├── addView()                       → 添加到 RecyclerView
                   ├── measureChildWithMargins()       → 测量
                   └── layoutDecoratedWithMargins()    → 布局
````
### 2.3 Recycler.getViewForPosition 缓存查找链

这是缓存机制的核心入口，完整查找链如下：

````java
// RecyclerView.Recycler（简化版）
ViewHolder tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs) {
    ViewHolder holder = null;

    // 0. 如果是预布局，先从 mChangedScrap 查找
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
    }

    // 1. 从 mAttachedScrap 或 mCachedViews 按 position 查找
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }

    // 2. 如果 Adapter 有 stableIds，按 id 从 scrap/cache 查找
    if (holder == null && mAdapter.hasStableIds()) {
        holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
    }

    // 3. 从 ViewCacheExtension 查找
    if (holder == null && mViewCacheExtension != null) {
        View view = mViewCacheExtension.getViewForPositionAndType(this, position, type);
        if (view != null) {
            holder = getChildViewHolder(view);
        }
    }

    // 4. 从 RecycledViewPool 按 viewType 查找
    if (holder == null) {
        holder = getRecycledViewPool().getRecycledView(type);
        if (holder != null) {
            holder.resetInternal(); // 重置状态，需要重新 bind
        }
    }

    // 5. 全部未命中，创建新的 ViewHolder
    if (holder == null) {
        holder = mAdapter.createViewHolder(RecyclerView.this, type);
    }

    // 6. 如果需要绑定数据
    if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        mAdapter.bindViewHolder(holder, offsetPosition);
    }

    return holder;
}
````
**各级缓存是否需要 bind 的对比**：

| 缓存级别 | 匹配方式 | 需要 onBind? | 需要 onCreate? |
|---|---|---|---|
| mAttachedScrap | position | ❌ | ❌ |
| mChangedScrap | position | ✅ | ❌ |
| mCachedViews | position | ❌ | ❌ |
| ViewCacheExtension | 自定义 | 视实现而定 | ❌ |
| RecycledViewPool | viewType | ✅ | ❌ |
| 新建 | — | ✅ | ✅ |



---

## 三、自定义 LayoutManager 实战

### 3.1 FlowLayoutManager 设计思路

实现一个 **流式布局**（类似标签云），子 View 从左到右排列，一行放不下时自动换行。

**核心要点**：

1. 重写 `generateDefaultLayoutParams()` — 必须实现
2. 重写 `onLayoutChildren()` — 布局所有子 View
3. 重写 `canScrollVertically()` + `scrollVerticallyBy()` — 支持垂直滚动
4. 在滚动时正确回收和复用 View

### 3.2 完整实现

````java
/**
 * 流式布局 LayoutManager
 * 子 View 从左到右排列，超出宽度自动换行，支持垂直滚动
 */
public class FlowLayoutManager extends RecyclerView.LayoutManager {

    // 垂直方向已滚动的偏移量
    private int mVerticalOffset = 0;

    // 所有 item 的布局信息（相对于内容区域顶部）
    private final SparseArray<Rect> mItemRects = new SparseArray<>();

    // 内容总高度
    private int mTotalHeight = 0;

    @Override
    public RecyclerView.LayoutParams generateDefaultLayoutParams() {
        return new RecyclerView.LayoutParams(
            RecyclerView.LayoutParams.WRAP_CONTENT,
            RecyclerView.LayoutParams.WRAP_CONTENT
        );
    }

    @Override
    public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
        if (state.getItemCount() == 0) {
            detachAndScrapAttachedViews(recycler);
            return;
        }

        // 将现有子 View 全部 detach 到 Scrap
        detachAndScrapAttachedViews(recycler);

        // 计算所有 item 的位置
        calculateChildPositions(recycler, state);

        // 填充可见区域
        fillVisibleViews(recycler, state);
    }

    /**
     * 预计算所有 item 的布局位置
     */
    private void calculateChildPositions(RecyclerView.Recycler recycler, RecyclerView.State state) {
        mItemRects.clear();

        int curLineX = getPaddingLeft();
        int curLineTop = getPaddingTop();
        int lineMaxHeight = 0;

        for (int i = 0; i < state.getItemCount(); i++) {
            View view = recycler.getViewForPosition(i);
            addView(view);
            measureChildWithMargins(view, 0, 0);

            int itemWidth = getDecoratedMeasuredWidth(view);
            int itemHeight = getDecoratedMeasuredHeight(view);

            // 需要换行
            if (curLineX + itemWidth > getWidth() - getPaddingRight() && curLineX > getPaddingLeft()) {
                curLineTop += lineMaxHeight;
                curLineX = getPaddingLeft();
                lineMaxHeight = 0;
            }

            // 记录该 item 的布局矩形
            Rect rect = new Rect(curLineX, curLineTop, curLineX + itemWidth, curLineTop + itemHeight);
            mItemRects.put(i, rect);

            curLineX += itemWidth;
            lineMaxHeight = Math.max(lineMaxHeight, itemHeight);

            // 计算完位置后先回收，后续 fillVisibleViews 会按需取出
            detachAndScrapView(view, recycler);
        }

        mTotalHeight = curLineTop + lineMaxHeight + getPaddingBottom();
    }

    /**
     * 只填充当前可见区域内的 View
     */
    private void fillVisibleViews(RecyclerView.Recycler recycler, RecyclerView.State state) {
        int visibleTop = mVerticalOffset;
        int visibleBottom = mVerticalOffset + getVerticalSpace();
        Rect visibleRect = new Rect(getPaddingLeft(), visibleTop, getWidth() - getPaddingRight(), visibleBottom);

        for (int i = 0; i < state.getItemCount(); i++) {
            Rect rect = mItemRects.get(i);
            if (rect == null) continue;
            if (Rect.intersects(visibleRect, rect)) {
                View view = recycler.getViewForPosition(i);
                addView(view);
                measureChildWithMargins(view, 0, 0);
                layoutDecorated(
                    view,
                    rect.left,
                    rect.top - mVerticalOffset,
                    rect.right,
                    rect.bottom - mVerticalOffset
                );
            }
        }
    }

    @Override
    public boolean canScrollVertically() { return true; }

    @Override
    public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
        if (getChildCount() == 0 || dy == 0) return 0;

        int maxScrollY = mTotalHeight - getVerticalSpace();
        if (maxScrollY <= 0) return 0;

        // 限制滚动范围
        int realDy;
        if (mVerticalOffset + dy < 0) {
            realDy = -mVerticalOffset;
        } else if (mVerticalOffset + dy > maxScrollY) {
            realDy = maxScrollY - mVerticalOffset;
        } else {
            realDy = dy;
        }

        mVerticalOffset += realDy;

        // 回收不可见的 View，填充新可见的 View
        detachAndScrapAttachedViews(recycler);
        fillVisibleViews(recycler, state);

        return realDy;
    }

    private int getVerticalSpace() { return getHeight() - getPaddingTop() - getPaddingBottom(); }
}
````
### 3.3 使用方式

````java
RecyclerView recyclerView = findViewById(R.id.recyclerView);
recyclerView.setLayoutManager(new FlowLayoutManager());
recyclerView.setAdapter(new TagAdapter(tagList)); // 自定义 Adapter
````
### 3.4 优化方向

上面的实现是一个可工作的基础版本，生产环境中还需考虑：

1. **增量回收**：`scrollVerticallyBy` 中不要全部 detach 再重新 fill，而是只回收滑出屏幕的、只添加滑入屏幕的
2. **支持 ItemDecoration**：在计算位置时考虑 `getItemOffsets()`
3. **支持 scrollToPosition**：重写 `scrollToPosition()` 和 `smoothScrollToPosition()`
4. **支持动画**：正确处理 `supportsPredictiveItemAnimations()`

增量回收优化的核心思路：

````java
@Override
public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
    // ... 计算 realDy ...

    mVerticalOffset += realDy;
    offsetChildrenVertical(-realDy); // 先整体偏移

    // 回收滑出屏幕的 View
    for (int i = getChildCount() - 1; i >= 0; i--) {
        View child = getChildAt(i);
        if (child == null) continue;
        if (getDecoratedBottom(child) < 0 || getDecoratedTop(child) > getHeight()) {
            removeAndRecycleView(child, recycler);
        }
    }

    // 填充新进入屏幕的 View
    fillVisibleViews(recycler, state);

    return realDy;
}
````
---

## 四、面试题

### Q1：RecyclerView 的四级缓存分别是什么？各自的特点？

**A**：

| 缓存级别 | 存储对象 | 匹配方式 | 是否需要 bind | 默认容量 |
|---|---|---|---|---|
| Scrap（mAttachedScrap） | 当前屏幕上被 detach 的 View | position | 否 | 无限制 |
| Cache（mCachedViews） | 刚滑出屏幕的 View | position | 否 | 2 |
| ViewCacheExtension | 开发者自定义 | 自定义 | 视实现 | 自定义 |
| RecycledViewPool | 按 viewType 回收的 View | viewType | 是 | 每种类型 5 个 |

核心区别：Scrap 和 Cache 按 **position** 匹配，数据未变可直接使用；Pool 按 **viewType** 匹配，需要重新绑定数据。

### Q2：RecyclerView 和 ListView 的缓存机制有什么区别？

**A**：

- [[ListView]] 只有两级缓存：ActiveViews（类似 Scrap）和 ScrapViews（类似 Pool），且 ScrapViews 取出后必须重新 `getView()`
- [[RecyclerView]] 有四级缓存，其中 mCachedViews 是 ListView 没有的——它按 position 缓存，命中时无需重新 bind，这在来回滑动场景下性能优势明显
- RecyclerView 的 RecycledViewPool 可以跨多个 RecyclerView 共享，ListView 不支持
- RecyclerView 强制使用 [[ViewHolder]] 模式，ListView 中 ViewHolder 是可选的最佳实践

### Q3：自定义 LayoutManager 必须重写哪些方法？

**A**：

**必须重写**：
- `generateDefaultLayoutParams()`：返回子 View 的默认 LayoutParams
- `onLayoutChildren()`：执行子 View 的布局

**按需重写**：
- `canScrollHorizontally()` / `canScrollVertically()`：声明是否支持滚动
- `scrollHorizontallyBy()` / `scrollVerticallyBy()`：处理滚动逻辑
- `scrollToPosition()` / `smoothScrollToPosition()`：支持程序化滚动
- `onAdapterChanged()`：Adapter 切换时的处理
- `supportsPredictiveItemAnimations()`：支持预测性动画

### Q4：RecyclerView 的 onLayoutChildren 什么时候会被调用？

**A**：

1. **首次布局**：RecyclerView 第一次 layout 时
2. **Adapter 数据变化**：调用 `notifyXxx()` 系列方法后
3. **Adapter 替换**：调用 `setAdapter()` 时
4. **RecyclerView 尺寸变化**：如屏幕旋转导致宽高改变
5. **调用 `requestLayout()`**：手动触发重新布局

注意：普通滚动 **不会** 触发 `onLayoutChildren()`，滚动由 `scrollVerticallyBy()` 处理。

### Q5：RecycledViewPool 在什么场景下可以共享？如何共享？

**A**：

**典型场景**：
- [[ViewPager2]] + 多个 Fragment，每个 Fragment 中的 RecyclerView 使用相同的 viewType
- 嵌套 RecyclerView（外层垂直列表，内层水平列表），内层的多个 RecyclerView 共享 Pool

**共享方式**：

````java
// 创建共享 Pool
RecyclerView.RecycledViewPool sharedPool = new RecyclerView.RecycledViewPool();
sharedPool.setMaxRecycledViews(VIEW_TYPE_ITEM, 20); // 可调整容量

// 多个 RecyclerView 设置同一个 Pool
recyclerView1.setRecycledViewPool(sharedPool);
recyclerView2.setRecycledViewPool(sharedPool);
````
共享 Pool 可以减少 `onCreateViewHolder()` 的调用次数，提升嵌套列表的滑动流畅度。

### Q6：为什么 RecyclerView 的 Item 动画比 ListView 好？

**A**：

- RecyclerView 内置了 [[ItemAnimator]] 机制（默认 `DefaultItemAnimator`），支持 add/remove/move/change 四种动画
- `notifyItemInserted()` / `notifyItemRemoved()` 等精确通知方法，让 RecyclerView 知道具体哪些 item 发生了变化，从而播放对应动画
- ListView 只有 `notifyDataSetChanged()`，无法区分变化类型，无法做精确动画
- RecyclerView 的 `supportsPredictiveItemAnimations()` 允许 LayoutManager 在动画前后分别布局，实现更自然的过渡效果（如新 item 从屏幕外滑入）



---

## 五、实战与踩坑

### 5.1 notifyDataSetChanged vs DiffUtil

#### 问题

`notifyDataSetChanged()` 是最暴力的刷新方式：

- 所有 ViewHolder 被标记为 invalid，全部需要重新 bind
- mCachedViews 中的缓存全部失效，被移入 RecycledViewPool
- **无法触发 Item 动画**（因为不知道具体哪些 item 变了）
- 大数据量时会造成明显卡顿

#### 正确做法：使用 DiffUtil

[[DiffUtil]] 通过 Myers 差分算法计算新旧列表的最小变更集，自动调用精确的 `notifyItemXxx()` 方法：

````java
public class TagDiffCallback extends DiffUtil.Callback {

    private final List<Tag> oldList;
    private final List<Tag> newList;

    public TagDiffCallback(List<Tag> oldList, List<Tag> newList) {
        this.oldList = oldList;
        this.newList = newList;
    }

    @Override public int getOldListSize() { return oldList.size(); }
    @Override public int getNewListSize() { return newList.size(); }

    @Override
    public boolean areItemsTheSame(int oldPos, int newPos) {
        return oldList.get(oldPos).getId() == newList.get(newPos).getId();
    }

    @Override
    public boolean areContentsTheSame(int oldPos, int newPos) {
        return oldList.get(oldPos).equals(newList.get(newPos));
    }
}

// 使用
DiffUtil.DiffResult diffResult = DiffUtil.calculateDiff(new TagDiffCallback(oldList, newList));
adapter.updateData(newList);
diffResult.dispatchUpdatesTo(adapter);
````
更推荐使用 `ListAdapter`（内置异步 DiffUtil）：

````java
public class TagAdapter extends ListAdapter<Tag, TagViewHolder> {

    private static final DiffUtil.ItemCallback<Tag> TAG_DIFF_CALLBACK = new DiffUtil.ItemCallback<Tag>() {
        @Override public boolean areItemsTheSame(@NonNull Tag old, @NonNull Tag newItem) { return old.getId() == newItem.getId(); }
        @Override public boolean areContentsTheSame(@NonNull Tag old, @NonNull Tag newItem) { return old.equals(newItem); }
    };

    public TagAdapter() { super(TAG_DIFF_CALLBACK); }

    @NonNull @Override
    public TagViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
            .inflate(R.layout.item_tag, parent, false);
        return new TagViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull TagViewHolder holder, int position) {
        holder.bind(getItem(position));
    }
}

// 更新数据只需一行
adapter.submitList(newList);
````
### 5.2 RecyclerView 嵌套性能优化

嵌套 RecyclerView（如外层垂直列表 + 内层水平列表）是常见场景，但容易出现性能问题。

#### 坑 1：内层 RecyclerView 反复创建

**问题**：外层每次 `onBindViewHolder` 都给内层设置新的 LayoutManager 和 Adapter，导致内层 RecyclerView 的缓存全部失效。

**解决**：

````java
// 在 ViewHolder 中初始化，而不是在 onBind 中
public class OuterViewHolder extends RecyclerView.ViewHolder {
    public final RecyclerView innerRecyclerView;

    public OuterViewHolder(View itemView) {
        super(itemView);
        innerRecyclerView = itemView.findViewById(R.id.innerRv);
        innerRecyclerView.setLayoutManager(new LinearLayoutManager(
            itemView.getContext(), LinearLayoutManager.HORIZONTAL, false
        ));
        innerRecyclerView.setAdapter(new InnerAdapter());
        // 设置共享 Pool
        innerRecyclerView.setRecycledViewPool(sharedPool);
    }
}

// onBind 中只更新数据
@Override
public void onBindViewHolder(OuterViewHolder holder, int position) {
    ((InnerAdapter) holder.innerRecyclerView.getAdapter()).submitList(data.get(position).getItems());
}
````
#### 坑 2：滑动冲突

**问题**：内外层 RecyclerView 滑动方向相同时，手势冲突。

**解决**：内外层使用不同滑动方向（最常见），或通过 `NestedScrollingChild` / `NestedScrollingParent` 协调。对于同向嵌套，设置内层 `isNestedScrollingEnabled = false`。

#### 坑 3：预取优化

[[LinearLayoutManager]] 默认开启了 **预取（Prefetch）** 机制，在滚动时提前在 RenderThread 空闲期创建即将可见的 ViewHolder。对于嵌套场景，设置内层的预取 item 数：

````java
((LinearLayoutManager) innerRecyclerView.getLayoutManager()).setInitialPrefetchItemCount(4);
````
### 5.3 ItemDecoration 踩坑

[[ItemDecoration]] 用于给 item 添加装饰（分割线、间距、背景等），常见问题：

#### 坑 1：getItemOffsets 影响布局计算

`getItemOffsets()` 返回的 Rect 会被加到 item 的测量尺寸中。自定义 LayoutManager 中调用 `getDecoratedMeasuredWidth()` 得到的是 **item 宽度 + 左右 offset**，而不是 item 本身的宽度。

````java
// 自定义 ItemDecoration
public class SpaceDecoration extends RecyclerView.ItemDecoration {
    private final int space;
    public SpaceDecoration(int space) { this.space = space; }

    @Override
    public void getItemOffsets(Rect outRect, View view,
            RecyclerView parent, RecyclerView.State state) {
        outRect.set(space, space, space, space);
    }
}
````
在自定义 LayoutManager 中，使用 `getDecoratedXxx` 系列方法来获取包含 decoration 的尺寸：

````java
// ✅ 正确：包含 ItemDecoration 的尺寸
int width = getDecoratedMeasuredWidth(view);
int height = getDecoratedMeasuredHeight(view);

// ❌ 错误：不包含 ItemDecoration
int width2 = view.getMeasuredWidth();
int height2 = view.getMeasuredHeight();
````
#### 坑 2：onDraw vs onDrawOver

- `onDraw()`：在 item 之前绘制（item 会覆盖在上面）→ 适合绘制背景
- `onDrawOver()`：在 item 之后绘制（覆盖在 item 上面）→ 适合绘制悬浮效果（如吸顶分组头）

````java
public class StickyHeaderDecoration extends RecyclerView.ItemDecoration {
    // 绘制在 item 上层，实现吸顶效果
    @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
        View topChild = parent.getChildAt(0);
        if (topChild == null) return;
        int position = parent.getChildAdapterPosition(topChild);
        // 绘制当前分组的 header...
    }
}
````
#### 坑 3：多个 ItemDecoration 叠加

RecyclerView 支持添加多个 ItemDecoration，它们的 `getItemOffsets` 会 **累加**，`onDraw` / `onDrawOver` 会按添加顺序依次调用。注意不要重复添加：

````java
// ❌ 错误：在 onBindViewHolder 或 onResume 中重复添加
recyclerView.addItemDecoration(new SpaceDecoration(8));

// ✅ 正确：只添加一次，或先移除再添加
if (recyclerView.getItemDecorationCount() == 0) {
    recyclerView.addItemDecoration(new SpaceDecoration(8));
}
````
---

## 参考

- [[RecyclerView]] 官方文档
- [[ViewHolder]] 复用模式
- [[DiffUtil]] 差量更新
- [[ItemAnimator]] 动画机制
- [[NestedScrollView]] 嵌套滚动


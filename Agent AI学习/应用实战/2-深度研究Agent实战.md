# 深度研究 Agent 实战

> 深度研究 Agent 是一种能够自主规划、搜索、总结并生成结构化报告的多智能体系统，将传统 1-2 小时的调研工作压缩到分钟级别。

相关主题：[[1-Agent应用实战案例]] | [[../Agent设计模式/1-ReAct与推理策略]]

---

## 一、为什么需要深度研究 Agent

### 1.1 传统研究的痛点

在信息爆炸时代，人工研究面临三个核心痛点：

- **信息过载**：搜索引擎返回成千上万条结果，需要逐个点开链接阅读，大量时间花在筛选无关内容上。
- **缺少结构**：即使找到了相关信息，这些信息往往是碎片化的，缺少系统性组织，需要手动梳理逻辑关系。
- **重复劳动**：每次研究新主题都要重复"搜索 - 阅读 - 总结 - 整理"的过程，效率极低。

### 1.2 深度研究 Agent 的核心价值

深度研究 Agent 不仅仅是一个搜索工具，而是一个能够自主规划、执行和总结的研究助手。其核心价值体现在四个方面：

1. **节省时间**：将 1-2 小时的研究工作压缩到 5-10 分钟
2. **提高质量**：系统化的研究流程，避免遗漏重要信息
3. **可追溯**：记录所有搜索结果和来源，方便验证和引用
4. **可扩展**：可以轻松添加新的搜索引擎、数据源和分析工具

> 🔑 深度研究 Agent 的本质是将"研究"这个复杂的人类认知任务，转化为"规划 → 执行 → 整结"的自动化流程。这是 Agent 范式在知识密集型场景中的典型应用。

---

## 二、TODO 驱动研究范式

### 2.1 核心思想

TODO 驱动的研究范式将复杂的研究主题分解为多个子任务（TODO），逐个执行并整合结果。与直接向搜索引擎提问不同，这种范式先"规划"再"执行"，确保研究的系统性和完整性。

以研究"某开源组织是什么"为例，对比两种方式：

```
# 传统搜索方式
用户输入 → 搜索引擎返回 10-20 个链接 → 逐个阅读 → 碎片化信息

# TODO 驱动方式
用户输入 → 系统规划：
  ├─ TODO 1：基本信息（组织定位）
  ├─ TODO 2：核心项目（技术内容）
  ├─ TODO 3：社区文化（价值观）
  └─ TODO 4：影响力（社会贡献）
→ 逐个搜索 + 总结 → 结构化报告
```

### 2.2 三阶段研究流程

TODO 驱动的研究流程分为三个阶段，每个阶段有专门的 Agent 负责：

**阶段 1：规划（Planning）**

Planner Agent 接收研究主题，将其分解为 3-5 个子任务。每个子任务包含三个字段：

```json
{
  "title": "任务标题",
  "intent": "研究意图（为什么要研究这个）",
  "query": "搜索查询（用于搜索引擎的查询字符串）"
}
```

一个好的规划应该满足：覆盖全面（涵盖主题的所有重要方面）、逻辑清晰（子任务之间有递进或并列关系）、查询精准（搜索词能有效命中相关内容）、数量适中（3-5 个，太少覆盖不全，太多会冗余）。

> 🔑 规划阶段的 Prompt 中应包含当前日期，以帮助 Agent 生成时效性更强的搜索查询。

**阶段 2：执行（Execution）**

对每个子任务，执行器按以下步骤操作：

1. 使用搜索引擎执行搜索，获取标题、URL、摘要
2. 调用 Summarizer Agent 总结搜索结果，提取核心观点和关键数据
3. 为每个观点添加来源引用（如 `[1]`、`[2]` 标记）
4. 记录总结和来源到笔记系统

**阶段 3：报告（Reporting）**

Writer Agent 整合所有子任务的总结，生成最终报告。报告结构通常包含：

```markdown
# 研究主题

## 概述
（2-3 段简要介绍研究主题和报告结构）

## 1. 子任务一标题
（详细分析内容）

## 2. 子任务二标题
（详细分析内容）

...

## 总结
（1-2 段总结主要发现）

## 参考文献
（按子任务分组列出所有来源 URL）
```

---

## 三、三 Agent 顺序协作设计

### 3.1 职责划分

深度研究系统采用三个专门 Agent 的设计，每个 Agent 专注于一个特定任务：

| Agent | 职责 | 输入 | 输出 |
|-------|------|------|------|
| TODO Planner | 将研究主题分解为子任务 | 研究主题 + 当前日期 | JSON 格式的子任务列表 |
| Task Summarizer | 总结搜索结果，提取关键信息 | 子任务信息 + 搜索结果 | Markdown 格式总结 + 来源引用 |
| Report Writer | 整合所有总结，生成最终报告 | 研究主题 + 所有子任务总结 | Markdown 格式研究报告 |

### 3.2 顺序协作模式

三个 Agent 之间是顺序协作关系，核心协调器（Orchestrator）负责调度：

```python
# 核心协调器伪代码
def run(research_topic):
    # 1. 规划阶段
    todo_list = planner.plan_todo_list(research_topic)

    # 2. 执行阶段（逐个子任务）
    task_summaries = []
    for task in todo_list:
        search_results = search_service.search(task.query)
        summary = summarizer.summarize_task(task, search_results)
        task_summaries.append((task, summary))

    # 3. 报告阶段
    report = reporter.generate_report(research_topic, task_summaries)
    return report
```

顺序协作模式的特点：

1. **线性流程**：Agent 按照固定的顺序执行
2. **明确的输入输出**：每个 Agent 的输入来自上一个 Agent 的输出
3. **无并发**：同一时间只有一个 Agent 在工作

### 3.3 与并行协作的对比

| 维度 | 顺序协作（本方案） | 并行协作 |
|------|-------------------|---------|
| 适用场景 | 阶段间有依赖关系 | 子任务相互独立 |
| 实现复杂度 | 低，线性流程易理解和调试 | 高，需要处理并发和结果合并 |
| 错误处理 | 简单，单点故障易定位 | 复杂，需要处理部分失败 |
| 效率 | 较低，串行执行 | 较高，可同时处理多个子任务 |
| 可观测性 | 强，进度清晰可追踪 | 弱，并行时进度展示更复杂 |

> 🔑 选择协作模式时，应根据任务间的依赖关系决定。深度研究的三个阶段天然存在数据依赖（规划结果决定执行内容，执行结果决定报告内容），因此顺序协作是更自然的选择。执行阶段的多个子任务理论上可以并行，但为了进度可观测性和错误处理的简单性，串行执行也是合理的权衡。

---

## 四、Prompt 工程实践

### 4.1 Planner Agent Prompt 设计

Planner 的 Prompt 是整个系统最关键的 Prompt，它决定了研究的质量和覆盖度：

```python
planner_prompt = """
你是一个研究规划专家。你的任务是将用户的研究主题分解为3-5个子任务。

当前日期：{current_date}
研究主题：{research_topic}

请分析这个研究主题，将其分解为3-5个子任务。每个子任务应该：
1. 涵盖主题的一个重要方面
2. 有明确的研究目标
3. 可以通过搜索引擎找到相关资料

请以JSON格式返回子任务列表，每个子任务包含：
- title：任务标题（简洁明了）
- intent：任务意图（为什么要研究这个）
- query：搜索查询（用于搜索引擎的查询字符串）

请确保：
1. 子任务数量在3-5个之间
2. 子任务之间有逻辑关系（如从基础到应用，从现状到趋势）
3. 搜索查询能够准确找到相关资料
4. 只返回JSON，不要包含其他文本
"""
```

关键设计点：

- 包含**当前日期**，帮助 Agent 生成时效性查询
- 明确要求 **JSON 格式**输出，便于程序解析
- 通过**示例**帮助 Agent 理解期望输出格式
- 强调**子任务数量、逻辑关系**等约束条件

### 4.2 Summarizer Agent Prompt 设计

Summarizer 的 Prompt 需要明确输出格式和引用规范：

```python
summarizer_prompt = """
你是一个任务总结专家。你的任务是总结搜索结果，提取关键信息。

任务标题：{task_title}
任务意图：{task_intent}
搜索查询：{task_query}

搜索结果：
{search_results}

请仔细阅读以上搜索结果，提取关键信息，并以Markdown格式返回总结。

总结应该包含：
1. **核心观点**：搜索结果中的核心观点和结论
2. **关键数据**：重要的数字、日期、名称等
3. **来源引用**：为每个观点添加来源引用（使用[1]、[2]等标记）

请确保：
1. 总结简洁明了，避免冗余
2. 保留重要的细节和数据
3. 为每个观点添加来源引用
4. 使用Markdown格式
"""
```

> 🔑 来源引用是深度研究 Agent 区别于普通问答 Agent 的关键特征。Prompt 中必须强调"为每个观点添加来源引用"，否则 Agent 容易忽略引用。

### 4.3 Writer Agent Prompt 设计

Writer 的 Prompt 侧重于报告结构和信息整合：

```python
writer_prompt = """
你是一个报告撰写专家。你的任务是整合所有子任务的总结，生成一份结构化的研究报告。

研究主题：{research_topic}

子任务总结：
{task_summaries}

报告应该包含：
1. **标题**：研究主题
2. **概述**：简要介绍研究主题和报告结构（2-3段）
3. **各个子任务的详细分析**：按照逻辑顺序组织（使用二级标题）
4. **总结**：总结研究的主要发现（1-2段）
5. **参考文献**：所有来源引用（按照子任务分组）

请确保：
1. 报告结构清晰，逻辑连贯
2. 消除重复的信息
3. 保留所有来源引用
4. 使用Markdown格式
"""
```

---

## 五、搜索工具工程实践

### 5.1 多搜索引擎集成

深度研究系统应支持多种搜索引擎，以适应不同的使用场景：

| 搜索引擎 | 特点 | 适用场景 |
|----------|------|---------|
| Tavily | AI 优化，返回结构化结果 | 通用研究 |
| DuckDuckGo | 免费，无需 API Key | 开发测试、预算有限 |
| Perplexity | 自带 AI 摘要 | 需要快速概览 |
| SearXNG | 自托管，隐私保护 | 企业内网、隐私敏感 |

通过配置化设计，用户可以切换搜索引擎而无需修改代码：

```python
class SearchAPI(str, Enum):
    TAVILY = "tavily"
    DUCKDUCKGO = "duckduckgo"
    PERPLEXITY = "perplexity"
    SEARXNG = "searxng"
    ADVANCED = "advanced"  # 组合多个引擎
```

### 5.2 结果去重与 Token 限制

搜索结果可能包含重复内容和过长文本，需要后处理：

```python
def deduplicate_sources(sources: list) -> list:
    """基于 URL 去重"""
    seen_urls = set()
    unique = []
    for source in sources:
        if source["url"] not in seen_urls:
            seen_urls.add(source["url"])
            unique.append(source)
    return unique

def limit_source_tokens(source: dict, max_tokens: int = 2000) -> dict:
    """限制单个来源的 Token 数量（粗略估算：1 Token ≈ 4 字符）"""
    snippet = source["snippet"]
    max_chars = max_tokens * 4
    if len(snippet) > max_chars:
        snippet = snippet[:max_chars] + "..."
    return {**source, "snippet": snippet}
```

### 5.3 搜索结果缓存

为提高效率和降低成本，应实现搜索结果缓存：

```python
import hashlib
import json
from pathlib import Path

class SearchService:
    def __init__(self, config):
        self.cache_dir = Path("./cache/search")
        self.cache_dir.mkdir(parents=True, exist_ok=True)

    def search(self, query: str, max_results: int = 5, use_cache: bool = True):
        # 生成缓存键
        cache_key = hashlib.md5(f"{query}_{max_results}".encode()).hexdigest()
        cache_file = self.cache_dir / f"{cache_key}.json"

        # 尝试从缓存读取
        if use_cache and cache_file.exists():
            with open(cache_file, "r", encoding="utf-8") as f:
                return json.load(f)

        # 执行搜索
        results = self._execute_search(query, max_results)

        # 保存到缓存
        if use_cache and results:
            with open(cache_file, "w", encoding="utf-8") as f:
                json.dump(results, f, ensure_ascii=False, indent=2)

        return results
```

---

## 六、SSE 实时进度推送

### 6.1 前后端架构

深度研究任务通常耗时 1-3 分钟，用户需要实时了解研究进度。SSE（Server-Sent Events）是实现这一需求的理想方案：

```
客户端 ──POST /research/stream──→ 服务端
客户端 ←──text/event-stream──── 服务端（持续推送进度）
```

### 6.2 进度事件设计

服务端在研究的不同阶段推送不同类型的事件：

```python
async def research_stream(topic: str):
    # 1. 规划阶段
    yield sse_event("progress", stage="planning", percentage=10, text="正在规划研究任务...")
    todo_items = await planning_service.plan(topic)
    yield sse_event("plan", data=[item.dict() for item in todo_items])

    # 2. 执行阶段
    for idx, task in enumerate(todo_items):
        pct = 10 + (idx / len(todo_items)) * 70
        yield sse_event("progress", stage="executing", percentage=pct,
                        text=f"正在研究任务{idx+1}/{len(todo_items)}：{task.title}")
        results = await search_service.search(task.query)
        summary = await summarizer.summarize(task, results)
        yield sse_event("task_summary", task_id=task.id, summary=summary)

    # 3. 报告阶段
    yield sse_event("progress", stage="reporting", percentage=90, text="正在生成最终报告...")
    report = await reporter.generate(topic, task_summaries)
    yield sse_event("report", data=report)
    yield sse_event("progress", stage="completed", percentage=100, text="研究完成！")
```

SSE 事件类型设计：

| 事件类型 | 数据 | 用途 |
|----------|------|------|
| `progress` | stage, percentage, text | 更新进度条和状态文本 |
| `plan` | 子任务列表 | 展示规划结果 |
| `task_summary` | task_id, summary | 追加子任务总结 |
| `report` | 最终报告内容 | 展示完整报告 |
| `error` | 错误信息 | 展示错误提示 |

> 🔑 SSE 比 WebSocket 更适合这种"服务端单向推送"的场景。它基于 HTTP 协议，实现简单，自动重连，且天然支持文本流式传输。

---

## 七、鲁棒性处理

### 7.1 JSON 解析与验证

LLM 返回的 JSON 可能包含额外文本或格式错误，需要健壮的解析逻辑：

```python
import re
import json

def extract_json_from_response(response: str) -> list:
    """从 LLM 响应中提取 JSON 数组"""
    # 策略 1：正则提取 JSON 数组
    json_match = re.search(r'\[.*\]', response, re.DOTALL)
    if json_match:
        try:
            return json.loads(json_match.group(0))
        except json.JSONDecodeError:
            pass

    # 策略 2：尝试直接解析整个响应
    try:
        return json.loads(response)
    except json.JSONDecodeError:
        raise ValueError("无法从响应中提取 JSON")
```

常见问题及应对：

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 包含额外文本 | LLM 在 JSON 前后添加说明 | 正则表达式提取 JSON 部分 |
| 格式错误 | 缺少引号、逗号 | 多策略解析 + 容错 |
| 字段缺失 | LLM 遗漏必需字段 | 字段验证 + 默认值填充 |

### 7.2 重试机制

搜索和 LLM 调用可能因网络或限流失败，需要重试机制：

```python
import time

def search_with_retry(query: str, max_retries: int = 3) -> list:
    """带重试的搜索"""
    for attempt in range(max_retries):
        try:
            return search_service.search(query)
        except Exception as e:
            if attempt < max_retries - 1:
                wait = 2 ** attempt  # 指数退避
                time.sleep(wait)
            else:
                logger.error(f"搜索失败（已重试 {max_retries} 次）：{e}")
                return []  # 返回空结果，不中断整个流程
```

> 🔑 深度研究系统应具备"降级"能力：单个子任务的搜索失败不应中断整个研究流程。返回空结果并继续执行后续任务，是比直接报错更好的策略。

### 7.3 规划质量评估

规划阶段可以增加质量评估环节，确保子任务列表的质量：

```python
def evaluate_plan(todo_items: list) -> dict:
    """评估规划质量"""
    score = 100
    suggestions = []

    # 检查数量
    if len(todo_items) < 3:
        score -= 20
        suggestions.append("子任务数量过少，可能遗漏重要信息")
    elif len(todo_items) > 5:
        score -= 10
        suggestions.append("子任务数量过多，可能存在冗余")

    # 检查查询质量
    for task in todo_items:
        if len(task["query"].split()) < 2:
            score -= 10
            suggestions.append(f"任务「{task['title']}」的查询过于简单")

    return {"score": score, "suggestions": suggestions}
```

---

## 八、设计模式总结

### 8.1 架构分层

深度研究系统的四层架构设计：

```
┌─────────────────────────────────────┐
│          前端展示层                   │
│   进度条 · 任务列表 · Markdown 报告    │
├─────────────────────────────────────┤
│          API 服务层                   │
│   SSE 端点 · 请求路由 · 错误处理       │
├─────────────────────────────────────┤
│          智能体编排层                  │
│   Planner · Summarizer · Writer      │
├─────────────────────────────────────┤
│          工具与外部服务层              │
│   搜索引擎 · LLM · 笔记存储           │
└─────────────────────────────────────┘
```

### 8.2 可复用的设计原则

1. **单职责 Agent**：每个 Agent 只负责一个特定任务，Prompt 更精准，调试更容易
2. **TODO 驱动分解**：将复杂任务分解为可独立执行的子任务，是 Agent 处理开放性问题的通用范式
3. **中间结果持久化**：每个阶段的输出都应保存，支持断点续传和结果审计
4. **配置化外部依赖**：搜索引擎、LLM 等外部服务通过配置切换，不硬编码
5. **降级容错**：单点失败不中断整体流程，返回空结果继续执行

---

## 面试题精选

**1. 深度研究 Agent 为什么采用三 Agent 顺序协作，而不是一个全能 Agent？**

单个 Agent 处理复杂研究任务会导致 Prompt 过长、职责不清、难以调试。三个专门 Agent 各司其职，每个 Agent 的 Prompt 更精准、更短，输出质量更高。同时，顺序协作使得每个阶段的中间结果可检查、可持久化，便于调试和断点续传。

**2. TODO 驱动的研究范式与 ReAct 范式有什么区别？**

ReAct 是"思考 - 行动 - 观察"的循环，适合需要动态决策的场景。TODO 驱动是"规划 - 执行 - 整合"的线性流程，适合目标明确、步骤可预知的研究任务。TODO 驱动的优势在于可控性强、进度可追踪；ReAct 的优势在于灵活性高、能应对意外情况。两者可以结合使用，例如在执行阶段用 ReAct 处理单个子任务。

**3. 如何保证 LLM 返回的 JSON 格式正确？**

三层策略：（1）Prompt 中明确要求"只返回 JSON，不要包含其他文本"并给出示例；（2）使用正则表达式提取 JSON 部分，处理 LLM 在 JSON 前后添加说明文字的情况；（3）字段验证，确保必需字段存在且类型正确。还可以考虑使用结构化输出（Structured Output）功能，让 LLM 直接输出符合 Schema 的 JSON。

**4. 搜索结果去重的策略有哪些？**

常见策略：（1）URL 精确去重，相同 URL 只保留一条；（2）域名去重，同一域名限制最多 N 条结果；（3）内容相似度去重，使用文本相似度算法（如余弦相似度）去除高度相似的结果。URL 去重成本最低，适合作为基础策略；内容相似度去重效果最好，但计算成本较高。

**5. 为什么选择 SSE 而不是 WebSocket 实现实时进度推送？**

SSE 基于 HTTP，是单向（服务端到客户端）的文本流式传输，实现简单，天然支持自动重连。深度研究场景下，数据流方向是单向的（服务端推送进度），不需要双向通信，SSE 完全满足需求且比 WebSocket 更轻量。WebSocket 适合需要双向实时通信的场景（如聊天、协同编辑）。

**6. 如果某个子任务的搜索全部失败，系统应该怎么处理？**

应该采用"降级"策略：记录错误日志，跳过该子任务，继续执行后续子任务。在最终报告中标注该部分"因搜索失败未能获取数据"。这样单个子任务的失败不会导致整个研究流程中断。同时可以在规划阶段增加重试机制（指数退避），在执行前多尝试几次。

**7. 如何评估深度研究 Agent 生成报告的质量？**

可从四个维度评估：（1）覆盖度，报告是否涵盖了研究主题的所有重要方面；（2）准确性，关键数据和事实是否正确；（3）引用质量，每个观点是否有来源引用，引用是否真实有效；（4）结构清晰度，报告是否有逻辑连贯的组织结构。可以设计自动化评测脚本，通过关键词命中率、来源有效率等指标进行量化评估。

**8. 搜索结果缓存的设计需要注意什么？**

关键考虑点：（1）缓存键应包含查询内容、搜索引擎和结果数量，避免不同参数的结果混淆；（2）缓存需要过期机制，研究类数据时效性较强，建议 24 小时过期；（3）缓存存储用文件系统即可（JSON 文件），无需引入 Redis 等额外依赖；（4）开发调试时应支持禁用缓存（`use_cache=False`），确保获取最新数据。

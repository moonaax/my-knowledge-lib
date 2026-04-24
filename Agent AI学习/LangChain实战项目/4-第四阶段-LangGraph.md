# 第四阶段：LangGraph 多步骤 Agent

> 日期：2026-04-23 | 代码：`~/lib/langchain-project/`

相关主题：[[../LangChain专题/6-LangGraph编排]] | [[../LangChain专题/4-Tool与Agent]]

---

## 一、为什么需要 LangGraph — AgentExecutor 的问题

> 🔑 AgentExecutor 是一个"黑盒"循环，开发者无法控制内部执行流程。

### 1.1 AgentExecutor 的局限

第三阶段用 `create_tool_calling_agent` + `AgentExecutor` 实现了工具调用 Agent，但随着需求变复杂，问题逐渐暴露：

| 痛点 | 说明 |
|------|------|
| **黑盒执行** | 内部循环逻辑固定，无法插入自定义步骤（如日志、校验） |
| **难以条件分支** | 只有"调用工具 or 结束"两种路径，无法做复杂路由 |
| **不支持循环控制** | 无法精确控制重试、自纠错等循环逻辑 |
| **无状态持久化** | 每次调用无状态，无法断点恢复 |
| **多 Agent 协作困难** | 没有原生的多 Agent 编排能力 |

### 1.2 LangGraph 的设计思想

> **把 Agent 的执行流程建模为一个有向图，状态在节点之间流转，开发者完全控制图的拓扑结构。**

````
AgentExecutor（旧）：
  用户输入 → [黑盒循环: LLM → 工具 → LLM → ...] → 输出

LangGraph（新）：
  用户输入 → [START] → agent节点 → 条件路由 → tools节点 → agent节点 → [END]
                 ↑                      ↓
                 └──────── 循环 ────────┘
````

## 二、核心概念：State、Node、Edge

> 🔑 理解这四个概念就理解了整个 LangGraph 框架。

### 2.1 State（状态）

State 是图中所有节点共享的数据容器，使用 Python 的 `TypedDict` 定义：

````python
from typing import TypedDict, Annotated
from operator import add

# LangGraph 内置的 MessagesState，等价于：
# class MessagesState(TypedDict):
#     messages: Annotated[list[BaseMessage], add_messages]

# 也可以自定义 State
class MyState(TypedDict):
    query: str                        # 覆盖模式：直接替换
    logs: Annotated[list[str], add]   # 追加模式：新值追加到列表
````

**两种更新策略：**

| 模式 | 语法 | 行为 | 适用场景 |
|------|------|------|---------|
| 覆盖（默认） | `query: str` | 新值替换旧值 | 当前状态标记、最终结果 |
| 追加 | `Annotated[list, add]` | 新值追加到列表末尾 | 消息历史、执行日志 |

> 🔑 本项目使用 `MessagesState`（内置），消息列表自动追加，无需手动管理。

### 2.2 Node（节点）

节点是实际执行逻辑的地方，一个节点就是一个 Python 函数：

````python
# 节点函数规范：接收 State，返回部分更新字典
def agent_node(state: MessagesState) -> dict:
    """LLM 推理节点"""
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}  # 只返回要更新的字段

def tool_node(state: MessagesState) -> dict:
    """工具执行节点 — LangGraph 内置 ToolNode 自动处理"""
    # ToolNode 自动：
    # 1. 从最后一条 AIMessage 提取 tool_calls
    # 2. 调用对应工具
    # 3. 返回 ToolMessage 列表
    pass
````

**关键规范：**
- 接收 `state`，读取需要的字段
- 返回 `{}` 字典，只包含要更新的字段
- 不要直接修改 `state` 对象
- 未返回的字段保持不变

### 2.3 Edge（边）

边连接节点，定义执行顺序。分为两种：

| 类型 | 方法 | 行为 |
|------|------|------|
| 普通边 | `add_edge(A, B)` | A 执行完必定执行 B |
| 条件边 | `add_conditional_edges(A, route_fn)` | 根据路由函数动态选择下一个节点 |

````python
# 普通边：固定路由
graph.add_edge(START, "agent")        # 图从 agent 开始
graph.add_edge("tools", "agent")      # 工具执行完回到 agent（循环）

# 条件边：动态路由
def should_continue(state: MessagesState) -> str:
    """根据 LLM 输出决定走向"""
    last_msg = state["messages"][-1]
    if hasattr(last_msg, "tool_calls") and last_msg.tool_calls:
        return "tools"    # 有工具调用 → 执行工具
    return END            # 无工具调用 → 结束

graph.add_conditional_edges("agent", should_continue, {
    "tools": "tools",
    END: END,
})
````

### 2.4 Graph（图）

图是由节点和边组成的有向图，编译后成为可执行的 Runnable：

````python
from langgraph.graph import StateGraph, START, END, MessagesState

graph = StateGraph(MessagesState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "agent")

# 编译 = 把图定义转化为可执行对象
app = graph.compile(checkpointer=MemorySaver())

# 编译后就是标准的 Runnable，支持 invoke / stream
result = app.invoke({"messages": [HumanMessage(content="你好")]})
````

## 三、本项目的 LangGraph 实现

### 3.1 图结构

````
START → agent(LLM 推理) ──┬── 有 tool_calls → tools(ToolNode) → agent（循环）
                          └── 无 tool_calls → END
````

执行流程：

````
用户: "北京天气怎么样"
  ↓
[agent] LLM 推理 → 返回 AIMessage(tool_calls=[{name: "search_weather", args: {city: "北京"}}])
  ↓
[should_continue] 检测到 tool_calls → 路由到 "tools"
  ↓
[tools] ToolNode 自动调用 search_weather("北京") → 返回 ToolMessage("晴天，25°C")
  ↓
[agent] LLM 推理（带工具结果）→ 返回 AIMessage("北京今天晴天，25°C...")
  ↓
[should_continue] 无 tool_calls → 路由到 END
  ↓
输出: "北京今天晴天，25°C..."
````

### 3.2 完整代码解读 (`graph_agent.py`)

````python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage
from langgraph.graph import StateGraph, START, END, MessagesState
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver
from tools import all_tools

load_dotenv()

# 【1】LLM 初始化 — 复用 DeepSeek 配置
llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    temperature=0.7,
)

# 【2】绑定工具 — 让 LLM 知道有哪些工具可调用
# bind_tools 后，LLM 的输出可能包含 tool_calls 字段
llm_with_tools = llm.bind_tools(all_tools)

# 【3】ToolNode — LangGraph 内置工具执行节点
# 自动处理 AIMessage 中的 tool_calls，调用对应工具
tool_node = ToolNode(all_tools)

# 【4】agent 节点 — LLM 推理
def agent_node(state: MessagesState) -> dict:
    """接收消息历史，返回 AIMessage（可能包含 tool_calls）"""
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

# 【5】条件路由 — 根据 LLM 输出决定走向
def should_continue(state: MessagesState) -> str:
    last_msg = state["messages"][-1]
    if hasattr(last_msg, "tool_calls") and last_msg.tool_calls:
        return "tools"   # 有工具调用 → 执行工具
    return END           # 无工具调用 → 结束

# 【6】构建图
graph = StateGraph(MessagesState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.add_edge(START, "agent")                                    # 入口
graph.add_conditional_edges("agent", should_continue, {           # 条件路由
    "tools": "tools",
    END: END,
})
graph.add_edge("tools", "agent")                                  # 循环边

# 【7】编译 — 加入 MemorySaver 支持多轮对话
memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# 【8】使用
result = app.invoke(
    {"messages": [HumanMessage(content="北京天气怎么样")]},
    config={"configurable": {"thread_id": "default"}},
)
print(result["messages"][-1].content)  # AI 的最终回答
````

### 3.3 对比 AgentExecutor vs LangGraph

| 维度 | AgentExecutor（旧） | LangGraph（新） |
|------|---------------------|-----------------|
| 执行模型 | 黑盒 while 循环 | 可视化有向图 |
| 状态管理 | 无持久化 | MemorySaver 每步存档 |
| 控制粒度 | 只能配 max_iterations | 精确控制每个节点和边 |
| 扩展性 | 难以插入自定义步骤 | 新增节点/边即可扩展 |
| 多轮对话 | RunnableWithMessageHistory | checkpointer + thread_id |
| 调试能力 | 只能 verbose=True | stream 逐节点输出 |

> 🔑 打个比方：AgentExecutor = 坐出租车（跟司机说去哪，到了就到了）；LangGraph = 自己开车（知道每一步怎么走，可以随时掉头、换路、加站点）。

## 四、服务端实现 (`server.py`)

### 4.1 新增 LangGraph 端点

| 代码 | 知识点 | 对应文档 |
|------|--------|---------|
| `@app.post("/graph_chat")` | 双端点共存 — 旧 `/chat` 保留，新 `/graph_chat` 并行 | [[../LangChain专题/7-生产化部署]] |
| `graph_app.astream_events(..., version="v2")` | LangGraph 流式 — 与 AgentExecutor 使用相同的事件协议 | [[../LangChain专题/3-LCEL表达式语言]] |
| `configurable: {"thread_id": session_id}` | 会话隔离 — LangGraph 用 thread_id 区分会话 | [[../LangChain专题/6-LangGraph编排]] |
| `graph_memory.delete_thread(...)` | 清除历史 — 删除指定会话的所有检查点 | [[../LangChain专题/6-LangGraph编排]] |

### 4.2 流式推送协议

两种模式使用**相同的 SSE 协议**，前端无需区分：

````
event: tool_start    → {"tool": "search_weather", "input": {"city": "北京"}}
event: tool_end      → {"tool": "search_weather", "output": "晴天，25°C"}
data: 北京今天...     → LLM 最终回答的逐字输出
data: [DONE]         → 流结束
````

## 五、前端适配 (`App.tsx`)

### 5.1 模式切换

| 改动 | 知识点 |
|------|--------|
| `useLangGraph` 状态 | React 状态管理 — 控制当前使用哪个端点 |
| 顶栏切换按钮 | `⚡ Agent` / `🔗 LangGraph` — 点击切换模式 |
| API 端点动态切换 | `useLangGraph ? '/graph_chat' : '/chat'` — 根据模式选择端点 |
| 底部状态栏 | 显示当前模式名称，方便确认 |

### 5.2 终端版模式选择 (`chat.py`)

| 改动 | 知识点 |
|------|--------|
| `run_agent_executor()` / `run_langgraph_agent()` | 模式分离 — 两种实现独立封装，互不干扰 |
| 运行时选择模式 | 对比体验 — 同一工具集，不同执行引擎 |
| `llm.bind_tools(tools)` + `ToolNode(tools)` | LangGraph 工具集成 — 比 AgentExecutor 更显式 |

## 六、自纠错循环（进阶）

> 🔑 工具调用失败时，不是把错误丢给 LLM 碰运气，而是主动注入纠错提示，引导 LLM 换策略重试。

### 6.1 为什么需要自纠错

基础 ReAct 图的问题：工具调用失败后，LLM 收到的只是一条原始错误信息，可能：
- 重复调用同样的工具（死循环）
- 直接放弃回答（"抱歉我无法回答"）
- 不理解错误原因（参数格式错误但不知道怎么改）

自纠错的核心：**在工具失败和 LLM 之间插入一个"纠错节点"，把错误信息转化为结构化的重试提示。**

### 6.2 扩展 State

```python
# 默认 MessagesState 只有 messages 字段
# 自纠错需要额外追踪重试次数和错误信息
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add]  # 消息历史（追加模式）
    retry_count: int                               # 当前重试次数（覆盖模式）
    last_error: str                                # 上次工具失败的错误信息
```

### 6.3 corrector 节点

```python
MAX_RETRIES = 3

def corrector_node(state: AgentState) -> dict:
    """自纠错：分析失败原因，注入提示引导 LLM 换策略"""
    last_error = state["messages"][-1].content
    retry_count = state.get("retry_count", 0) + 1

    correction_prompt = f"""⚠️ 工具调用失败（第 {retry_count} 次重试）：
{last_error}

请分析失败原因并换一种方式尝试：
- 如果是参数错误，请修正参数后重试
- 如果是工具选择错误，请换一个更合适的工具
- 如果所有工具都无法解决，请直接用你的知识回答，并说明工具不可用"""

    return {
        "messages": [SystemMessage(content=correction_prompt)],
        "retry_count": retry_count,
        "last_error": last_error,
    }
```

### 6.4 条件路由（工具执行后）

```python
def check_tool_result(state: AgentState) -> str:
    """工具执行后路由：失败 → corrector，成功 → agent"""
    last_msg = state["messages"][-1]
    content = last_msg.content if hasattr(last_msg, "content") else ""
    retry_count = state.get("retry_count", 0)
    is_error = any(kw in content for kw in ["错误", "error", "未找到", "失败", "无法"])
    if is_error and retry_count < MAX_RETRIES:
        return "corrector"
    return "agent"
```

### 6.5 自纠错图结构

````
START → agent(LLM推理) ──有 tool_calls──→ tools ──失败──→ corrector → agent（纠错循环）
                                             │
                                             └──成功──→ agent（正常循环）
                                             
                                         无 tool_calls → END
````

### 6.6 重试策略对比

| 策略 | 实现 | 优点 | 缺点 |
|------|------|------|------|
| 无自纠错 | tools → agent | 简单 | 可能死循环或放弃 |
| 固定重试 | 重试 N 次 | 可控 | 不分析原因 |
| **纠错提示** | corrector 节点 | LLM 理解失败原因，主动换策略 | 额外一次 LLM 调用 |
| 人类介入 | interrupt_before | 最可靠 | 需要人工参与 |

### 6.7 前端适配

服务端新增 `tool_retry` SSE 事件，前端需要处理：

| 改动 | 说明 |
|------|------|
| `ContentBlock` 类型 | 新增 `{ type: 'tool_retry'; message: string }` |
| `parseSSEEvent` | 解析 `event: tool_retry` 事件 |
| `RetryCard` 组件 | 橙色毛玻璃卡片，显示"自纠错重试"提示 |
| 状态栏 | 显示 "LangGraph + 自纠错" |
| 模式切换按钮 | 文案更新为 "🔗 LangGraph + 自纠错" |

````tsx
// RetryCard 组件（橙色毛玻璃风格，与 ToolCard 一致）
function RetryCard({ block, v }) {
  return (
    <div style={{
      background: 'rgba(249,115,22,0.06)',
      border: '1px solid rgba(249,115,22,0.2)',
      borderRadius: 10, padding: '8px 12px',
      backdropFilter: 'blur(20px)',
    }}>
      <span>🔄</span>
      <span style={{ fontWeight: 600, color: '#f97316' }}>自纠错重试</span>
      <span>{block.message}</span>
    </div>
  );
}
````

## 七、Plan-and-Execute（先规划再执行）

> 🔑 ReAct 是"走一步看一步"，Plan-and-Execute 是"先想好再动手"。适合复杂多步骤任务。

### 7.1 为什么需要 Plan-and-Execute

ReAct 模式的问题：对于复杂任务（如"帮我查北京和上海的天气，然后比较哪个城市更适合出行"），LLM 可能：
- 遗漏某个步骤（只查了北京没查上海）
- 步骤顺序不合理（先比较再查询）
- 无法回溯调整（发现遗漏后不知道怎么补）

Plan-and-Execute 的核心：**先让 LLM 制定完整计划，再逐步执行，执行过程中可动态调整。**

### 7.2 与 ReAct 的对比

| 维度 | ReAct | Plan-and-Execute |
|------|-------|------------------|
| 执行策略 | 走一步看一步 | 先规划再执行 |
| 适用场景 | 简单问答、单步任务 | 复杂多步骤任务 |
| 状态管理 | messages | messages + plan + past_steps + response |
| 图结构 | agent ↔ tools 循环 | planner → executor → replanner 循环 |
| 可控性 | 低（全靠 LLM 自主决策） | 高（计划可见、可调整） |

### 7.3 State 设计

```python
class PlanExecuteState(TypedDict):
    messages: Annotated[list[BaseMessage], add]  # 消息历史
    plan: list[str]                                # 计划步骤列表
    current_step: int                              # 当前执行到第几步
    past_steps: list[dict]                         # 已执行步骤 [{step, result}]
    response: str                                  # 最终回答（非空时表示完成）
```

### 7.4 三个核心节点

**Planner（规划节点）**：
```python
def planner_node(state: PlanExecuteState) -> dict:
    # 用 LLM 把用户问题拆解为步骤列表
    # 输出格式：["步骤1: xxx", "步骤2: xxx", ...]
    response = llm.invoke([SystemMessage(content=PLANNER_PROMPT), ...])
    plan = json.loads(response.content)
    return {"plan": plan, "current_step": 0, "past_steps": [], "response": ""}
```

**Executor（执行节点）**：
```python
def executor_node(state: PlanExecuteState) -> dict:
    # 执行当前步骤，支持工具调用
    current_step = state["plan"][state["current_step"]]
    response = llm_with_tools.invoke([SystemMessage(content=prompt), ...])
    # 如果 LLM 要求调用工具，执行工具并获取结果
    if response.tool_calls:
        result = execute_tools(response.tool_calls)
    return {"past_steps": [...], "current_step": current_idx + 1}
```

**Re-planner（重新规划节点）**：
```python
def replanner_node(state: PlanExecuteState) -> dict:
    # 根据已执行结果，决定下一步
    # 三种走向：continue（继续）/ replan（调整计划）/ finish（输出最终回答）
    decision = json.loads(response.content)
    if decision["action"] == "finish":
        return {"response": decision["response"]}
    elif decision["action"] == "replan":
        return {"plan": decision["new_plan"], "current_step": 0}
    return {}  # continue
```

### 7.5 图结构

````
START → planner(规划) → executor(执行) → replanner(评估) ──继续──→ executor（循环）
                                            │
                                            └──完成──→ finish(输出回答) → END
````

### 7.6 前端适配

| 组件 | 说明 |
|------|------|
| `PlanCard` | 显示计划步骤列表，带执行进度（✅ 完成 / ▶️ 当前 / ○ 待执行） |
| `PlanStepCard` | 显示某个步骤的执行结果 |
| `plan_start` SSE 事件 | 规划完成时推送计划步骤列表 |
| `plan_step` SSE 事件 | 某个步骤执行完成时推送结果 |
| 模式切换 | 三档循环：⚡ Agent → 🔗 LangGraph → 📋 Plan |

## 八、当前架构

````
第四阶段架构（全链路）：
  后端（三种模式共存）：
    AgentExecutor 模式: create_tool_calling_agent → AgentExecutor → astream_events
    LangGraph 模式:     StateGraph(agent → tools → corrector → agent) → astream_events
    Plan-and-Execute:   StateGraph(planner → executor → replanner) → astream_events
    三种模式共享 tools.py 工具定义

  前端：
    SSE 事件: tool_start / tool_end / tool_retry / plan_start / plan_step / token / done
    UI 组件: ToolCard / RetryCard / PlanCard / PlanStepCard
    模式切换: ⚡ Agent ↔ 🔗 LangGraph + 自纠错 ↔ 📋 Plan-and-Execute
````

## 九、踩坑记录

1. **ToolMessage is not JSON serializable**：LangGraph 的 `astream_events` 在 `on_tool_end` 事件中，`output` 字段是 `ToolMessage` 对象而非字符串，`json.dumps` 无法序列化。解决：`str(output)` 转换后再序列化
2. **HuggingFace 连接超时**：LangGraph 模式下调用 `knowledge_search` 工具时，`HuggingFaceEmbeddings` 尝试联网检查更新导致超时。解决：启动时加 `HF_ENDPOINT=https://hf-mirror.com HF_HUB_OFFLINE=1` 环境变量

# LangGraph 编排 — 有状态 Agent 工作流框架

> LangChain 专题第五篇 | 适用 langgraph >= 0.2 | Python 3.10+
>
> LangGraph 是 LangChain 团队推出的有状态、可循环的 Agent 编排框架。它将 Agent 的执行流程建模为**图（Graph）**，让开发者能精确控制每一步的逻辑、分支和循环，彻底解决了 AgentExecutor 的"黑盒"问题。

---

## 1. LangGraph 概述

### 1.1 为什么需要 LangGraph — AgentExecutor 的局限性

LangChain 早期提供的 `AgentExecutor` 是一个"万能"的 Agent 运行时，但随着应用复杂度提升，它的问题逐渐暴露：

| 痛点 | 说明 |
|------|------|
| **黑盒执行** | 内部循环逻辑固定，开发者无法插入自定义步骤 |
| **难以实现条件分支** | 只有"调用工具 or 结束"两种路径，无法做复杂路由 |
| **不支持循环控制** | 无法精确控制重试、自纠错等循环逻辑 |
| **无状态持久化** | 每次调用都是无状态的，无法断点恢复 |
| **多 Agent 协作困难** | 没有原生的多 Agent 编排能力 |
| **人机协作缺失** | 无法在执行中途暂停等待人类审批 |

LangGraph 的设计目标就是解决以上所有问题。它的核心思想是：

> **把 Agent 的执行流程建模为一个有向图，状态在节点之间流转，开发者完全控制图的拓扑结构。**

### 1.2 核心概念：State、Node、Edge、Graph

LangGraph 有四个核心概念，理解它们就理解了整个框架：

````
┌─────────────────────────────────────────────┐
│                   Graph                      │
│                                             │
│   ┌───────┐    Edge    ┌───────┐            │
│   │ Node A│ ─────────→ │ Node B│            │
│   └───────┘            └───────┘            │
│       │                    │                │
│       │  读取/更新 State    │  读取/更新 State │
│       ▼                    ▼                │
│   ┌─────────────────────────────────┐       │
│   │           State                  │       │
│   │  { messages: [...], count: 0 }   │       │
│   └─────────────────────────────────┘       │
└─────────────────────────────────────────────┘
````
- **State（状态）**：一个 TypedDict，是图中所有节点共享的数据容器。每个节点读取 State、处理后返回更新。
- **Node（节点）**：一个 Python 函数，接收当前 State，返回 State 的部分更新（字典）。节点是实际执行逻辑的地方。
- **Edge（边）**：连接节点的有向边，定义执行顺序。分为普通边（固定路由）和条件边（动态路由）。
- **Graph（图）**：由节点和边组成的有向图，编译后成为可执行的 Runnable。

### 1.3 与 LangChain LCEL 的关系和区别

| 维度 | LCEL (LangChain Expression Language) | LangGraph |
|------|--------------------------------------|-----------|
| **数据流** | 线性管道，数据从左到右流动 | 图结构，支持分支、循环、回溯 |
| **状态管理** | 无状态，每步输出作为下步输入 | 有全局 State，所有节点共享 |
| **循环** | 不支持 | 原生支持 |
| **持久化** | 不支持 | 内置 Checkpointer |
| **人机协作** | 不支持 | 内置 interrupt 机制 |
| **适用场景** | 简单的 prompt → LLM → parser 链 | 复杂 Agent、多步推理、多 Agent |

两者不是替代关系，而是互补：LCEL 适合构建单个节点内部的处理链，LangGraph 负责节点之间的编排。

````python
# LCEL：线性链，适合简单场景
chain = prompt | llm | parser

# LangGraph：图结构，适合复杂 Agent
graph = StateGraph(State)
graph.add_node("think", think_node)
graph.add_node("act", act_node)
graph.add_conditional_edges("think", route_fn)
````
安装：

````bash
pip install langgraph langchain-openai
````
---

## 2. StateGraph 基础

### 2.1 状态定义（TypedDict + Annotated）

State 是 LangGraph 的核心数据结构，使用 Python 的 `TypedDict` 定义：

````python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph
from operator import add

# 最简单的状态定义
class SimpleState(TypedDict):
    query: str          # 用户查询
    result: str         # 处理结果
    step_count: int     # 步骤计数

# 使用 Annotated 指定状态更新策略
class ChatState(TypedDict):
    messages: Annotated[list, add]  # 追加模式：新消息追加到列表
    current_tool: str               # 覆盖模式（默认）：直接替换
````
关键规则：
- 不加 `Annotated` 的字段默认是**覆盖模式**（新值替换旧值）
- 使用 `Annotated[list, add]` 表示**追加模式**（新值追加到列表末尾）
- `add` 是 `operator.add`，对列表等同于 `old_list + new_list`

### 2.2 状态更新策略（覆盖 vs 追加）

````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, END

class DemoState(TypedDict):
    name: str                        # 覆盖模式
    logs: Annotated[list[str], add]  # 追加模式

def node_a(state: DemoState) -> dict:
    return {
        "name": "Alice",                # 覆盖：name 变为 "Alice"
        "logs": ["node_a executed"]     # 追加：logs 追加一条
    }

def node_b(state: DemoState) -> dict:
    return {
        "name": "Bob",                  # 覆盖：name 变为 "Bob"
        "logs": ["node_b executed"]     # 追加：logs 再追加一条
    }

graph = StateGraph(DemoState)
graph.add_node("a", node_a)
graph.add_node("b", node_b)
graph.add_edge("a", "b")
graph.set_entry_point("a")
graph.set_finish_point("b")
app = graph.compile()

result = app.invoke({"name": "", "logs": []})
print(result)
# {'name': 'Bob', 'logs': ['node_a executed', 'node_b executed']}
# name 被覆盖为最后一次赋值，logs 则累积了两条记录
````
### 2.3 节点函数的编写规范

节点函数必须遵循以下规范：

````python
# ✅ 正确：接收 State，返回部分更新字典
def my_node(state: MyState) -> dict:
    current_value = state["some_field"]  # 读取当前状态
    new_value = process(current_value)    # 处理逻辑
    return {"some_field": new_value}      # 返回要更新的字段

# ✅ 正确：可以只返回部分字段（未返回的字段保持不变）
def partial_update(state: MyState) -> dict:
    return {"result": "done"}  # 只更新 result，其他字段不变

# ❌ 错误：不要返回完整的 State 对象
def wrong_node(state: MyState) -> MyState:
    state["field"] = "value"  # 不要直接修改 state
    return state              # 不要返回完整 state

# ✅ 异步节点也支持
async def async_node(state: MyState) -> dict:
    result = await some_async_operation()
    return {"result": result}
````
---

## 3. 图的构建

### 3.1 add_node — 添加节点

````python
from langgraph.graph import StateGraph

graph = StateGraph(MyState)

# 方式一：函数名作为节点名
graph.add_node("process", process_fn)

# 方式二：可以是任何 Callable（函数、lambda、Runnable）
graph.add_node("llm_call", my_chain)  # LCEL chain 也可以作为节点
````
### 3.2 add_edge — 普通边

普通边表示固定的执行顺序，A 执行完必定执行 B：

````python
graph.add_edge("node_a", "node_b")  # A → B
graph.add_edge("node_b", "node_c")  # B → C
````
### 3.3 add_conditional_edges — 条件路由

条件边是 LangGraph 最强大的特性之一，根据当前状态动态决定下一步：

````python
from langgraph.graph import END

# 路由函数：接收 State，返回下一个节点的名称
def route_decision(state: MyState) -> str:
    if state["needs_tool"]:
        return "use_tool"
    elif state["needs_review"]:
        return "human_review"
    else:
        return END  # 结束

# 添加条件边
graph.add_conditional_edges(
    "decision_node",       # 源节点
    route_decision,        # 路由函数
    {                      # 路由映射（可选，用于可视化和校验）
        "use_tool": "tool_node",
        "human_review": "review_node",
        END: END,
    }
)
````
### 3.4 set_entry_point / set_finish_point

````python
# 设置入口节点（图从这里开始执行）
graph.set_entry_point("first_node")

# 设置结束节点（执行到这里图结束）
graph.set_finish_point("last_node")

# 也可以用 add_edge 连接到 END 来结束
from langgraph.graph import END
graph.add_edge("last_node", END)

# 入口也可以用 START
from langgraph.graph import START
graph.add_edge(START, "first_node")
````
### 3.5 compile 编译

编译将图定义转化为可执行的 Runnable 对象：

````python
# 基础编译
app = graph.compile()

# 带检查点的编译（支持持久化）
from langgraph.checkpoint.memory import MemorySaver
memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# 带人机协作的编译
app = graph.compile(
    checkpointer=memory,
    interrupt_before=["human_review_node"]  # 在该节点前暂停
)

# 编译后就是标准的 Runnable，支持 invoke / stream / batch
result = app.invoke({"query": "hello"})

# 流式输出（逐节点输出）
for event in app.stream({"query": "hello"}):
    print(event)
````
### 3.6 完整构建示例

````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END

class WorkflowState(TypedDict):
    input: str
    steps: Annotated[list[str], add]
    output: str

def step_one(state: WorkflowState) -> dict:
    return {"steps": ["step_one: 收到输入 -> " + state["input"]]}

def step_two(state: WorkflowState) -> dict:
    return {"steps": ["step_two: 处理中..."]}

def step_three(state: WorkflowState) -> dict:
    return {
        "steps": ["step_three: 生成输出"],
        "output": f"处理完成: {state['input']}"
    }

# 构建图
graph = StateGraph(WorkflowState)
graph.add_node("step1", step_one)
graph.add_node("step2", step_two)
graph.add_node("step3", step_three)

graph.add_edge(START, "step1")
graph.add_edge("step1", "step2")
graph.add_edge("step2", "step3")
graph.add_edge("step3", END)

app = graph.compile()
result = app.invoke({"input": "Hello LangGraph", "steps": [], "output": ""})
print(result["steps"])
# ['step_one: 收到输入 -> Hello LangGraph', 'step_two: 处理中...', 'step_three: 生成输出']
print(result["output"])
# '处理完成: Hello LangGraph'
````
---

## 4. 核心模式

### 4.1 线性流程 — A → B → C

最简单的模式，节点按固定顺序依次执行：

````
START → 提取 → 分析 → 总结 → END
````
````python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

class PipelineState(TypedDict):
    text: str
    keywords: str
    summary: str

def extract(state: PipelineState) -> dict:
    resp = llm.invoke(f"从以下文本中提取关键词：\n{state['text']}")
    return {"keywords": resp.content}

def summarize(state: PipelineState) -> dict:
    resp = llm.invoke(
        f"根据关键词 [{state['keywords']}] 对以下文本写一句话摘要：\n{state['text']}"
    )
    return {"summary": resp.content}

graph = StateGraph(PipelineState)
graph.add_node("extract", extract)
graph.add_node("summarize", summarize)
graph.add_edge(START, "extract")
graph.add_edge("extract", "summarize")
graph.add_edge("summarize", END)

app = graph.compile()
result = app.invoke({"text": "LangGraph是LangChain团队推出的Agent编排框架...", "keywords": "", "summary": ""})
print(result["summary"])
````
### 4.2 条件分支 — 根据状态选择路径

根据运行时状态动态选择执行路径：

````
                    ┌→ 简单回答 → END
START → 分类器 ─────┤
                    ├→ 工具调用 → END
                    └→ 拒绝回答 → END
````
````python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph, START, END

class RouterState(TypedDict):
    query: str
    category: str
    response: str

def classify(state: RouterState) -> dict:
    query = state["query"].lower()
    if "天气" in query or "计算" in query:
        return {"category": "tool"}
    elif any(w in query for w in ["你好", "介绍"]):
        return {"category": "chat"}
    else:
        return {"category": "reject"}

def chat_response(state: RouterState) -> dict:
    return {"response": f"你好！关于「{state['query']}」，我来为你解答..."}

def tool_response(state: RouterState) -> dict:
    return {"response": f"正在调用工具处理：{state['query']}"}

def reject_response(state: RouterState) -> dict:
    return {"response": "抱歉，我无法处理这个请求。"}

def route(state: RouterState) -> str:
    return state["category"]

graph = StateGraph(RouterState)
graph.add_node("classify", classify)
graph.add_node("chat", chat_response)
graph.add_node("tool", tool_response)
graph.add_node("reject", reject_response)

graph.add_edge(START, "classify")
graph.add_conditional_edges("classify", route, {
    "chat": "chat",
    "tool": "tool",
    "reject": "reject",
})
graph.add_edge("chat", END)
graph.add_edge("tool", END)
graph.add_edge("reject", END)

app = graph.compile()
print(app.invoke({"query": "今天天气怎么样", "category": "", "response": ""})["response"])
# '正在调用工具处理：今天天气怎么样'
````
### 4.3 循环 — Agent 推理循环

这是 LangGraph 最核心的模式，实现 ReAct 风格的"思考→行动→观察→再思考"循环：

````
                    ┌──────────────────────┐
                    │                      │
START → LLM 推理 ──┤── 需要工具 → 执行工具 ─┘
                    │
                    └── 完成 → END
````
````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage, BaseMessage

class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add]

llm = ChatOpenAI(model="gpt-4o-mini")

# 模拟工具
def search_tool(query: str) -> str:
    return f"搜索结果：LangGraph 是 LangChain 的 Agent 编排框架，发布于 2024 年。"

def agent_node(state: AgentState) -> dict:
    """LLM 推理节点"""
    response = llm.bind_tools([{
        "type": "function",
        "function": {
            "name": "search",
            "description": "搜索信息",
            "parameters": {"type": "object", "properties": {"query": {"type": "string"}}}
        }
    }]).invoke(state["messages"])
    return {"messages": [response]}

def tool_node(state: AgentState) -> dict:
    """工具执行节点"""
    last_msg = state["messages"][-1]
    results = []
    for call in last_msg.tool_calls:
        result = search_tool(call["args"].get("query", ""))
        results.append(ToolMessage(content=result, tool_call_id=call["id"]))
    return {"messages": results}

def should_continue(state: AgentState) -> str:
    """路由：有工具调用则继续，否则结束"""
    last_msg = state["messages"][-1]
    if hasattr(last_msg, "tool_calls") and last_msg.tool_calls:
        return "tools"
    return END

graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)

graph.add_edge(START, "agent")
graph.add_conditional_edges("agent", should_continue, {"tools": "tools", END: END})
graph.add_edge("tools", "agent")  # 工具执行完回到 agent，形成循环

app = graph.compile()
result = app.invoke({"messages": [HumanMessage(content="LangGraph 是什么？")]})
print(result["messages"][-1].content)
````
### 4.4 人机协作（Human-in-the-loop）

LangGraph 支持在执行过程中暂停，等待人类审批后继续：

````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver

class ApprovalState(TypedDict):
    request: str
    approved: bool
    logs: Annotated[list[str], add]

def prepare(state: ApprovalState) -> dict:
    return {"logs": [f"准备处理请求: {state['request']}"]}

def human_review(state: ApprovalState) -> dict:
    # 这个节点会被 interrupt，人类在外部更新 state
    return {"logs": ["等待人类审批..."]}

def execute(state: ApprovalState) -> dict:
    if state["approved"]:
        return {"logs": ["✅ 已批准，执行操作"]}
    return {"logs": ["❌ 已拒绝，取消操作"]}

graph = StateGraph(ApprovalState)
graph.add_node("prepare", prepare)
graph.add_node("review", human_review)
graph.add_node("execute", execute)

graph.add_edge(START, "prepare")
graph.add_edge("prepare", "review")
graph.add_edge("review", "execute")
graph.add_edge("execute", END)

# 关键：interrupt_before 让图在 review 节点前暂停
memory = MemorySaver()
app = graph.compile(checkpointer=memory, interrupt_before=["review"])

# 第一次调用：执行到 review 前暂停
config = {"configurable": {"thread_id": "approval-1"}}
result = app.invoke(
    {"request": "删除生产数据库", "approved": False, "logs": []},
    config=config
)
print("暂停状态:", result["logs"])
# ['准备处理请求: 删除生产数据库']

# 人类审批：更新状态
app.update_state(config, {"approved": True})

# 继续执行（传 None 表示从断点恢复）
result = app.invoke(None, config=config)
print("最终状态:", result["logs"])
# ['准备处理请求: 删除生产数据库', '等待人类审批...', '✅ 已批准，执行操作']
````
---

## 5. 内置组件

### 5.1 ToolNode — 工具执行节点

`ToolNode` 是 LangGraph 预构建的工具执行节点，自动处理 LLM 返回的 tool_calls：

````python
from langchain_core.tools import tool
from langgraph.prebuilt import ToolNode

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气"""
    weather_data = {"北京": "晴天 25°C", "上海": "多云 22°C"}
    return weather_data.get(city, f"{city}：暂无数据")

@tool
def calculator(expression: str) -> str:
    """计算数学表达式"""
    return str(eval(expression))

# 创建 ToolNode，传入工具列表
tools = [get_weather, calculator]
tool_node = ToolNode(tools)

# ToolNode 自动：
# 1. 从最后一条 AIMessage 中提取 tool_calls
# 2. 调用对应的工具
# 3. 返回 ToolMessage 列表
````
### 5.2 create_react_agent — 预构建的 ReAct Agent

这是 LangGraph 提供的开箱即用的 ReAct Agent，内部已经构建好了"LLM → 工具 → LLM"的循环图：

````python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def search(query: str) -> str:
    """搜索互联网信息"""
    return f"关于「{query}」的搜索结果：LangGraph 是一个 Agent 编排框架。"

@tool
def multiply(a: int, b: int) -> int:
    """计算两个数的乘积"""
    return a * b

llm = ChatOpenAI(model="gpt-4o-mini")
tools = [search, multiply]

# 一行代码创建完整的 ReAct Agent
agent = create_react_agent(llm, tools)

# 使用
result = agent.invoke({
    "messages": [("human", "LangGraph 是什么？帮我算一下 42 * 58")]
})
for msg in result["messages"]:
    print(f"{msg.type}: {msg.content[:100] if msg.content else '[tool_call]'}")
````
`create_react_agent` 内部构建的图结构：

````
START → agent(LLM) ──┬── 有 tool_calls → tools(ToolNode) → agent(LLM)
                      │                                        ↑ 循环
                      └── 无 tool_calls → END
````
### 5.3 MessagesState — 消息状态

`MessagesState` 是 LangGraph 预定义的状态类型，专为聊天场景设计：

````python
from langgraph.graph import MessagesState

# MessagesState 等价于：
# class MessagesState(TypedDict):
#     messages: Annotated[list[BaseMessage], add_messages]

# add_messages 比 operator.add 更智能：
# - 自动处理消息去重（基于 message.id）
# - 支持消息更新（相同 id 的消息会被替换而非追加）

from langgraph.graph import StateGraph, START, END

# 可以扩展 MessagesState 添加自定义字段
class MyState(MessagesState):
    current_tool: str
    iteration: int

graph = StateGraph(MyState)
# ... 添加节点和边
````
`add_messages` 的智能合并行为：

````python
from langgraph.graph import add_messages
from langchain_core.messages import AIMessage

# 普通追加
msgs = add_messages(
    [AIMessage(content="hello", id="1")],
    [AIMessage(content="world", id="2")]
)
# 结果：两条消息都保留

# 相同 id 则更新
msgs = add_messages(
    [AIMessage(content="hello", id="1")],
    [AIMessage(content="updated", id="1")]
)
# 结果：只有一条消息，content 为 "updated"
````
---

## 6. 状态持久化

LangGraph 的持久化机制基于 **Checkpointer**，在每个节点执行后自动保存状态快照，支持断点恢复和时间旅行。

### 6.1 MemorySaver — 内存检查点

最简单的持久化方式，数据存储在内存中（进程结束即丢失），适合开发调试：

````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage

class ConversationState(TypedDict):
    messages: Annotated[list[BaseMessage], add]

def chat_node(state: ConversationState) -> dict:
    last_msg = state["messages"][-1].content
    return {"messages": [AIMessage(content=f"你说的是：{last_msg}")]}

graph = StateGraph(ConversationState)
graph.add_node("chat", chat_node)
graph.add_edge(START, "chat")
graph.add_edge("chat", END)

# 使用 MemorySaver
memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# thread_id 区分不同会话
config = {"configurable": {"thread_id": "user-001"}}

# 第一轮对话
result = app.invoke({"messages": [HumanMessage(content="你好")]}, config=config)
print(result["messages"][-1].content)  # '你说的是：你好'

# 第二轮对话（自动携带历史）
result = app.invoke({"messages": [HumanMessage(content="记住我叫小明")]}, config=config)
print(len(result["messages"]))  # 4（两轮对话共4条消息）
````
### 6.2 SqliteSaver — SQLite 持久化

生产环境推荐使用 SQLite 或 PostgreSQL 持久化，数据不会因进程重启丢失：

````python
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

# 方式一：文件持久化
with SqliteSaver.from_conn_string("checkpoints.db") as memory:
    app = graph.compile(checkpointer=memory)
    config = {"configurable": {"thread_id": "persistent-001"}}
    result = app.invoke({"messages": [HumanMessage(content="你好")]}, config=config)

# 方式二：使用已有连接
conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
memory = SqliteSaver(conn)
app = graph.compile(checkpointer=memory)

# PostgreSQL 持久化（生产推荐）
# pip install langgraph-checkpoint-postgres
# from langgraph.checkpoint.postgres import PostgresSaver
# memory = PostgresSaver.from_conn_string("postgresql://user:pass@localhost/db")
````
### 6.3 断点恢复与时间旅行

Checkpointer 不仅保存最新状态，还保存每一步的快照，支持"时间旅行"：

````python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
app = graph.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "time-travel-001"}}

# 正常执行
app.invoke({"messages": [HumanMessage(content="第一轮")]}, config=config)
app.invoke({"messages": [HumanMessage(content="第二轮")]}, config=config)

# 获取所有历史状态快照
states = list(app.get_state_history(config))
print(f"共有 {len(states)} 个状态快照")

# 每个快照包含：
# - values: 当时的完整状态
# - next: 下一步要执行的节点
# - config: 该快照的配置（含 checkpoint_id）
# - created_at: 创建时间
# - parent_config: 父快照配置

for i, state in enumerate(states):
    print(f"快照 {i}: next={state.next}, msgs={len(state.values.get('messages', []))}")

# 时间旅行：回到某个历史快照继续执行
old_config = states[-2].config  # 回到倒数第二个快照
result = app.invoke(
    {"messages": [HumanMessage(content="从历史分支继续")]},
    config=old_config
)
````
持久化架构总览：

````
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Graph 执行  │────→│ Checkpointer │────→│  存储后端        │
│  每步自动存  │     │  (接口层)     │     │  MemorySaver    │
│  储状态快照  │     │              │     │  SqliteSaver    │
│             │     │              │     │  PostgresSaver  │
└─────────────┘     └──────────────┘     └─────────────────┘
                           │
                    ┌──────┴──────┐
                    │ 断点恢复     │
                    │ 时间旅行     │
                    │ 会话隔离     │
                    └─────────────┘
````
---

## 7. Multi-Agent 编排

LangGraph 天然支持多 Agent 协作，以下是三种主流模式。

### 7.1 Supervisor 模式 — 主管分配任务

一个 Supervisor Agent 负责分析任务并分配给专业 Agent：

````
                    ┌→ 研究员 Agent ─┐
START → Supervisor ─┤                ├→ Supervisor → ... → END
                    └→ 写作者 Agent ─┘
````
````python
from typing import TypedDict, Annotated, Literal
from operator import add
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage

llm = ChatOpenAI(model="gpt-4o-mini")

class MultiAgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add]
    next_agent: str
    research_result: str
    final_output: str

def supervisor(state: MultiAgentState) -> dict:
    """主管：决定下一步交给谁"""
    last_msg = state["messages"][-1].content if state["messages"] else ""
    # 简化逻辑：如果已有研究结果，交给写作者；否则交给研究员
    if state.get("research_result"):
        return {"next_agent": "writer"}
    return {"next_agent": "researcher"}

def researcher(state: MultiAgentState) -> dict:
    """研究员 Agent"""
    query = state["messages"][0].content
    resp = llm.invoke(f"作为研究员，请调研以下主题并给出要点：{query}")
    return {
        "research_result": resp.content,
        "messages": [AIMessage(content=f"[研究员] {resp.content[:200]}")]
    }

def writer(state: MultiAgentState) -> dict:
    """写作者 Agent"""
    resp = llm.invoke(
        f"作为写作者，基于以下研究结果撰写一段总结：\n{state['research_result']}"
    )
    return {
        "final_output": resp.content,
        "messages": [AIMessage(content=f"[写作者] {resp.content[:200]}")]
    }

def route_supervisor(state: MultiAgentState) -> str:
    if state.get("final_output"):
        return END
    return state["next_agent"]

graph = StateGraph(MultiAgentState)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", researcher)
graph.add_node("writer", writer)

graph.add_edge(START, "supervisor")
graph.add_conditional_edges("supervisor", route_supervisor, {
    "researcher": "researcher",
    "writer": "writer",
    END: END,
})
# 每个 worker 完成后回到 supervisor
graph.add_edge("researcher", "supervisor")
graph.add_edge("writer", "supervisor")

app = graph.compile()
result = app.invoke({
    "messages": [HumanMessage(content="分析 LangGraph 的优势和应用场景")],
    "next_agent": "", "research_result": "", "final_output": ""
})
print(result["final_output"][:200])
````
### 7.2 Swarm 模式 — Agent 间直接交接

Agent 之间直接传递控制权，无需中央调度：

````
START → Agent A ──→ Agent B ──→ Agent C → END
         │                        ↑
         └────────────────────────┘
````
````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END

class SwarmState(TypedDict):
    task: str
    current_agent: str
    results: Annotated[list[str], add]
    done: bool

def agent_planner(state: SwarmState) -> dict:
    """规划 Agent：分析任务，决定交给谁"""
    return {
        "results": ["[Planner] 任务已分析，需要执行代码"],
        "current_agent": "coder"  # 直接指定下一个 agent
    }

def agent_coder(state: SwarmState) -> dict:
    """编码 Agent：编写代码"""
    return {
        "results": ["[Coder] 代码已编写完成"],
        "current_agent": "reviewer"
    }

def agent_reviewer(state: SwarmState) -> dict:
    """审查 Agent：审查代码"""
    return {
        "results": ["[Reviewer] 代码审查通过"],
        "done": True
    }

def swarm_route(state: SwarmState) -> str:
    if state.get("done"):
        return END
    return state["current_agent"]

graph = StateGraph(SwarmState)
graph.add_node("planner", agent_planner)
graph.add_node("coder", agent_coder)
graph.add_node("reviewer", agent_reviewer)

graph.add_edge(START, "planner")
for node in ["planner", "coder", "reviewer"]:
    graph.add_conditional_edges(node, swarm_route, {
        "planner": "planner",
        "coder": "coder",
        "reviewer": "reviewer",
        END: END,
    })

app = graph.compile()
result = app.invoke({
    "task": "写一个排序算法", "current_agent": "", "results": [], "done": False
})
print(result["results"])
# ['[Planner] 任务已分析，需要执行代码', '[Coder] 代码已编写完成', '[Reviewer] 代码审查通过']
````
### 7.3 层级 Agent — 嵌套子图

将复杂的 Agent 封装为子图，在父图中作为节点使用：

````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END

# ===== 子图：研究团队 =====
class ResearchState(TypedDict):
    topic: str
    findings: Annotated[list[str], add]

def search_node(state: ResearchState) -> dict:
    return {"findings": [f"搜索发现：{state['topic']} 相关论文 10 篇"]}

def analyze_node(state: ResearchState) -> dict:
    return {"findings": ["分析完成：主要趋势是..."]}

research_graph = StateGraph(ResearchState)
research_graph.add_node("search", search_node)
research_graph.add_node("analyze", analyze_node)
research_graph.add_edge(START, "search")
research_graph.add_edge("search", "analyze")
research_graph.add_edge("analyze", END)
research_subgraph = research_graph.compile()

# ===== 父图 =====
class MainState(TypedDict):
    topic: str
    findings: Annotated[list[str], add]
    report: str

def write_report(state: MainState) -> dict:
    findings_text = "\n".join(state["findings"])
    return {"report": f"研究报告：\n{findings_text}\n\n结论：研究完成。"}

main_graph = StateGraph(MainState)
# 子图作为节点嵌入父图
main_graph.add_node("research_team", research_subgraph)
main_graph.add_node("write_report", write_report)

main_graph.add_edge(START, "research_team")
main_graph.add_edge("research_team", "write_report")
main_graph.add_edge("write_report", END)

app = main_graph.compile()
result = app.invoke({"topic": "大语言模型", "findings": [], "report": ""})
print(result["report"])
````
---

## 8. 实战案例

### 8.1 Plan-and-Execute Agent

先制定计划，再逐步执行，执行过程中可以根据结果修正计划：

````
START → 制定计划 → 执行步骤 → 检查是否完成 ──┬── 未完成 → 执行步骤（循环）
                                              └── 完成 → 汇总结果 → END
````
````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

class PlanExecuteState(TypedDict):
    task: str
    plan: list[str]
    current_step: int
    results: Annotated[list[str], add]
    final_answer: str

def planner(state: PlanExecuteState) -> dict:
    """制定执行计划"""
    resp = llm.invoke(
        f"为以下任务制定 3 个执行步骤，每步一行，只输出步骤：\n{state['task']}"
    )
    steps = [s.strip() for s in resp.content.strip().split("\n") if s.strip()]
    return {"plan": steps, "current_step": 0}

def executor(state: PlanExecuteState) -> dict:
    """执行当前步骤"""
    step_idx = state["current_step"]
    step = state["plan"][step_idx]
    resp = llm.invoke(f"执行以下步骤并给出结果（简洁）：\n{step}")
    return {
        "results": [f"步骤{step_idx+1}: {resp.content[:200]}"],
        "current_step": step_idx + 1
    }

def check_done(state: PlanExecuteState) -> str:
    """检查是否所有步骤都执行完"""
    if state["current_step"] >= len(state["plan"]):
        return "summarize"
    return "executor"

def summarizer(state: PlanExecuteState) -> dict:
    """汇总所有结果"""
    all_results = "\n".join(state["results"])
    resp = llm.invoke(f"基于以下执行结果，给出最终总结：\n{all_results}")
    return {"final_answer": resp.content}

graph = StateGraph(PlanExecuteState)
graph.add_node("planner", planner)
graph.add_node("executor", executor)
graph.add_node("summarizer", summarizer)

graph.add_edge(START, "planner")
graph.add_edge("planner", "executor")
graph.add_conditional_edges("executor", check_done, {
    "executor": "executor",
    "summarize": "summarizer",
})
graph.add_edge("summarizer", END)

app = graph.compile()
result = app.invoke({
    "task": "分析 Python 和 Rust 在 Web 开发中的优劣势",
    "plan": [], "current_step": 0, "results": [], "final_answer": ""
})
print(result["final_answer"][:300])
````
### 8.2 带自纠错的代码生成 Agent

生成代码 → 检查语法 → 如果有错则修复 → 再检查，循环直到正确：

````
START → 生成代码 → 检查代码 ──┬── 有错误 → 修复代码 → 检查代码（循环）
                              └── 无错误 → END
````
````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
import ast

llm = ChatOpenAI(model="gpt-4o-mini")

class CodeGenState(TypedDict):
    requirement: str
    code: str
    error: str
    attempts: int
    logs: Annotated[list[str], add]

def generate_code(state: CodeGenState) -> dict:
    """生成代码"""
    if state.get("error"):
        # 修复模式
        prompt = f"修复以下 Python 代码的错误：\n错误：{state['error']}\n代码：\n{state['code']}\n只输出修复后的代码。"
    else:
        prompt = f"根据需求编写 Python 代码，只输出代码：\n{state['requirement']}"
    resp = llm.invoke(prompt)
    code = resp.content.strip().removeprefix("```python").removesuffix("```").strip()
    return {
        "code": code,
        "attempts": state.get("attempts", 0) + 1,
        "logs": [f"第 {state.get('attempts', 0) + 1} 次生成代码"]
    }

def check_code(state: CodeGenState) -> dict:
    """语法检查"""
    try:
        ast.parse(state["code"])
        return {"error": "", "logs": ["✅ 语法检查通过"]}
    except SyntaxError as e:
        return {"error": str(e), "logs": [f"❌ 语法错误: {e}"]}

def route_check(state: CodeGenState) -> str:
    if state["error"] and state["attempts"] < 3:
        return "generate"  # 有错误且未超过重试次数，继续修复
    return END

graph = StateGraph(CodeGenState)
graph.add_node("generate", generate_code)
graph.add_node("check", check_code)

graph.add_edge(START, "generate")
graph.add_edge("generate", "check")
graph.add_conditional_edges("check", route_check, {"generate": "generate", END: END})

app = graph.compile()
result = app.invoke({
    "requirement": "写一个函数，接收列表并返回去重后的排序结果",
    "code": "", "error": "", "attempts": 0, "logs": []
})
print("最终代码：")
print(result["code"])
print("\n执行日志：", result["logs"])
````
### 8.3 多 Agent 协作的研究助手

三个 Agent 协作完成研究任务：搜索员收集信息、分析师提炼观点、编辑生成报告：

````python
from typing import TypedDict, Annotated
from operator import add
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, BaseMessage

llm = ChatOpenAI(model="gpt-4o-mini")

class ResearchAssistantState(TypedDict):
    topic: str
    messages: Annotated[list[BaseMessage], add]
    raw_data: str
    analysis: str
    report: str
    phase: str

def searcher_agent(state: ResearchAssistantState) -> dict:
    """搜索员：收集原始信息"""
    resp = llm.invoke(
        f"作为研究搜索员，请收集关于「{state['topic']}」的关键信息，"
        f"列出 5 个要点。"
    )
    return {
        "raw_data": resp.content,
        "phase": "analyze",
        "messages": [AIMessage(content=f"[搜索员] 已收集 {state['topic']} 的信息")]
    }

def analyst_agent(state: ResearchAssistantState) -> dict:
    """分析师：提炼观点"""
    resp = llm.invoke(
        f"作为研究分析师，基于以下原始数据提炼 3 个核心观点：\n{state['raw_data']}"
    )
    return {
        "analysis": resp.content,
        "phase": "report",
        "messages": [AIMessage(content="[分析师] 分析完成")]
    }

def editor_agent(state: ResearchAssistantState) -> dict:
    """编辑：生成最终报告"""
    resp = llm.invoke(
        f"作为编辑，基于以下分析撰写一份简洁的研究报告（200字内）：\n{state['analysis']}"
    )
    return {
        "report": resp.content,
        "phase": "done",
        "messages": [AIMessage(content="[编辑] 报告已生成")]
    }

def route_phase(state: ResearchAssistantState) -> str:
    phase = state.get("phase", "search")
    return {"search": "searcher", "analyze": "analyst", "report": "editor", "done": END}.get(phase, END)

graph = StateGraph(ResearchAssistantState)
graph.add_node("searcher", searcher_agent)
graph.add_node("analyst", analyst_agent)
graph.add_node("editor", editor_agent)

graph.add_node("router", lambda state: {})  # 空节点做路由
graph.add_edge(START, "searcher")
graph.add_edge("searcher", "router")
graph.add_edge("analyst", "router")
graph.add_edge("editor", "router")
graph.add_conditional_edges("router", route_phase, {
    "searcher": "searcher",
    "analyst": "analyst",
    "editor": "editor",
    END: END,
})

app = graph.compile()
result = app.invoke({
    "topic": "2024年大语言模型发展趋势",
    "messages": [], "raw_data": "", "analysis": "", "report": "", "phase": "analyze"
})
print("=== 研究报告 ===")
print(result["report"])
````
---

## 9. 调试与可视化

### 9.1 graph.get_graph().draw_mermaid()

LangGraph 内置了图可视化能力，可以导出 Mermaid 格式的流程图：

````python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class MyState(TypedDict):
    value: str

def node_a(state): return {"value": "a"}
def node_b(state): return {"value": "b"}
def route(state): return "b" if state["value"] == "a" else END

graph = StateGraph(MyState)
graph.add_node("a", node_a)
graph.add_node("b", node_b)
graph.add_edge(START, "a")
graph.add_conditional_edges("a", route, {"b": "b", END: END})
graph.add_edge("b", END)
app = graph.compile()

# 方式一：输出 Mermaid 文本（可粘贴到 Mermaid Live Editor）
print(app.get_graph().draw_mermaid())
# 输出类似：
# graph TD
#     __start__ --> a
#     a -. b .-> b
#     a -. __end__ .-> __end__
#     b --> __end__

# 方式二：直接生成 PNG 图片（需要安装 graphviz）
# pip install pygraphviz
# app.get_graph().draw_mermaid_png(output_file_path="graph.png")

# 方式三：在 Jupyter Notebook 中直接显示
# from IPython.display import Image, display
# display(Image(app.get_graph().draw_mermaid_png()))
````
### 9.2 流式调试 — 观察每个节点的输出

````python
# stream 模式可以观察每个节点的执行过程
for event in app.stream({"value": ""}):
    # event 是 {节点名: 该节点返回的更新}
    for node_name, update in event.items():
        print(f"[{node_name}] -> {update}")

# 输出：
# [a] -> {'value': 'a'}
# [b] -> {'value': 'b'}
````
### 9.3 LangSmith 追踪

LangSmith 是 LangChain 官方的可观测性平台，与 LangGraph 深度集成：

````bash
# 设置环境变量即可自动追踪
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY="your-api-key"
export LANGCHAIN_PROJECT="my-langgraph-project"
````
````python
# 设置后，所有 LangGraph 执行自动上报到 LangSmith
# 可以在 LangSmith 控制台看到：
# - 完整的图执行轨迹
# - 每个节点的输入/输出
# - LLM 调用的 token 用量和延迟
# - 工具调用的参数和返回值

# 也可以在代码中手动添加 metadata
result = app.invoke(
    {"value": "test"},
    config={
        "metadata": {"user_id": "u123", "session": "s456"},
        "tags": ["production", "v2"],
    }
)
````
LangSmith 追踪界面展示的信息：

````
┌─ Run: my-langgraph-project ──────────────────────┐
│                                                    │
│  ▶ __start__          0ms                         │
│  ▶ node_a             120ms   tokens: 0           │
│    ├─ Input:  {"value": ""}                       │
│    └─ Output: {"value": "a"}                      │
│  ▶ node_b             85ms    tokens: 0           │
│    ├─ Input:  {"value": "a"}                      │
│    └─ Output: {"value": "b"}                      │
│  ▶ __end__            0ms                         │
│                                                    │
│  Total: 205ms | Status: Success                   │
└────────────────────────────────────────────────────┘
````
---

## 10. 面试题精选

### 题目 1：LangGraph 和 AgentExecutor 的核心区别是什么？

**答案：**

| 维度 | AgentExecutor | LangGraph |
|------|---------------|-----------|
| 执行模型 | 固定的 while 循环 | 自定义有向图 |
| 控制粒度 | 黑盒，只能配置参数 | 白盒，完全控制每个节点和边 |
| 分支能力 | 只有"调用工具/结束"两条路 | 任意条件分支 |
| 循环控制 | 只能设置 max_iterations | 精确控制循环条件和退出 |
| 状态管理 | 无持久化 | 内置 Checkpointer |
| 人机协作 | 不支持 | interrupt_before/after |
| 多 Agent | 不支持 | 原生支持（子图、Supervisor 等） |

核心区别在于：AgentExecutor 是一个**预定义的执行循环**，而 LangGraph 是一个**图编排框架**，让开发者自己定义执行拓扑。LangGraph 的设计哲学是"给开发者最大的控制权"。

---

### 题目 2：解释 LangGraph 中 State 的覆盖模式和追加模式，什么时候用哪种？

**答案：**

- **覆盖模式（默认）**：字段不加 `Annotated`，每次更新直接替换旧值。适用于：当前状态标记（如 `current_step`）、最终结果（如 `output`）、配置项等。

- **追加模式**：使用 `Annotated[list, operator.add]`，新值追加到列表末尾。适用于：消息历史（`messages`）、执行日志（`logs`）、中间结果收集等。

````python
class State(TypedDict):
    current_phase: str                    # 覆盖：只关心当前阶段
    messages: Annotated[list, add]        # 追加：保留完整对话历史
    final_answer: str                     # 覆盖：只要最终答案
    tool_results: Annotated[list, add]    # 追加：收集所有工具结果
````
选择原则：如果需要保留历史记录用追加，如果只关心最新值用覆盖。

---

### 题目 3：如何在 LangGraph 中实现人机协作（Human-in-the-loop）？

**答案：**

LangGraph 通过 `interrupt_before` 和 `interrupt_after` 实现人机协作，核心流程：

1. **编译时声明中断点**：`graph.compile(checkpointer=memory, interrupt_before=["review_node"])`
2. **首次调用**：图执行到中断节点前自动暂停，状态保存到 Checkpointer
3. **人类介入**：通过 `app.update_state(config, updates)` 更新状态（如审批结果）
4. **恢复执行**：调用 `app.invoke(None, config)` 从断点继续

关键要点：
- 必须配置 Checkpointer（否则无法保存断点状态）
- `interrupt_before` 在节点执行前暂停，`interrupt_after` 在节点执行后暂停
- 恢复时传 `None` 作为输入，表示从断点继续而非重新开始
- 通过 `thread_id` 区分不同的执行会话

---

### 题目 4：LangGraph 的 Checkpointer 机制是如何工作的？支持哪些存储后端？

**答案：**

Checkpointer 的工作机制：
1. 图的每个节点执行完毕后，自动将当前完整 State 保存为一个快照（checkpoint）
2. 每个快照包含：完整状态值、下一步节点、时间戳、父快照引用
3. 通过 `thread_id` 隔离不同会话的状态
4. 支持从任意历史快照恢复执行（时间旅行）

支持的存储后端：

| 后端 | 包名 | 适用场景 |
|------|------|---------|
| MemorySaver | langgraph（内置） | 开发调试，进程内临时存储 |
| SqliteSaver | langgraph-checkpoint-sqlite | 单机持久化，轻量级应用 |
| PostgresSaver | langgraph-checkpoint-postgres | 生产环境，多实例共享 |

时间旅行的实现：
````python
# 获取历史快照
states = list(app.get_state_history(config))
# 回到某个历史节点
old_config = states[3].config
result = app.invoke({"messages": [HumanMessage("新分支")]}, config=old_config)
````
---

### 题目 5：对比 Supervisor 模式和 Swarm 模式的优缺点，各适用什么场景？

**答案：**

| 维度 | Supervisor 模式 | Swarm 模式 |
|------|----------------|------------|
| 控制方式 | 中央调度，Supervisor 决定任务分配 | 去中心化，Agent 之间直接交接 |
| 优点 | 全局视角，易于监控和调试 | 灵活，Agent 自主性强 |
| 缺点 | Supervisor 是瓶颈和单点故障 | 难以全局优化，可能出现死循环 |
| 适用场景 | 任务分工明确、需要统一调度 | Agent 能力互补、流程动态变化 |
| 实现复杂度 | 中等（需要设计 Supervisor 逻辑） | 较高（需要每个 Agent 知道何时交接） |

选择建议：
- 如果任务可以清晰分解为子任务 → Supervisor
- 如果 Agent 之间需要频繁交互、流程不固定 → Swarm
- 如果系统规模大、层级多 → 层级 Agent（嵌套子图）

---

### 题目 6：LangGraph 中如何防止 Agent 陷入无限循环？

**答案：**

多种防护机制：

1. **递归限制**：编译时设置 `recursion_limit`
````python
app = graph.compile()
result = app.invoke(input, config={"recursion_limit": 25})  # 默认 25
````
2. **状态计数器**：在 State 中维护迭代次数
````python
def should_continue(state):
    if state["iterations"] >= 5:
        return END  # 强制退出
    return "next_node"
````
3. **超时控制**：结合 Python 的 asyncio 超时
````python
import asyncio
result = await asyncio.wait_for(app.ainvoke(input), timeout=60)
````
4. **条件边兜底**：确保条件路由函数总有一条通往 END 的路径

最佳实践是组合使用：recursion_limit 作为全局保险，状态计数器作为业务逻辑控制，超时作为最后防线。

---

### 题目 7：如何将一个 LangGraph 图嵌套为另一个图的子节点？有什么注意事项？

**答案：**

将子图编译后直接作为父图的节点：

````python
# 子图
sub_graph = StateGraph(SubState)
# ... 添加节点和边
sub_app = sub_graph.compile()

# 父图中使用子图
parent_graph = StateGraph(ParentState)
parent_graph.add_node("sub_task", sub_app)  # 编译后的子图作为节点
````
注意事项：
1. **状态兼容性**：子图的 State 字段必须是父图 State 的子集，或者字段名和类型完全匹配
2. **状态传递**：父图的 State 会自动传递给子图中匹配的字段，子图的输出也会自动合并回父图
3. **Checkpointer**：子图不需要单独配置 Checkpointer，会继承父图的配置
4. **可视化**：嵌套子图在可视化时会展开显示内部结构
5. **错误处理**：子图内部的错误会冒泡到父图，需要在父图层面处理

---

## 总结

LangGraph 的核心价值在于将 Agent 的执行流程从"黑盒循环"变为"可视化、可控制、可持久化的图"。掌握以下要点即可应对绝大多数场景：

| 概念 | 一句话总结 |
|------|-----------|
| State | 全局共享的数据容器，TypedDict 定义 |
| Node | 处理函数，读取 State 返回更新 |
| Edge | 节点间的连接，支持固定和条件路由 |
| Checkpointer | 每步自动存档，支持断点恢复和时间旅行 |
| interrupt | 人机协作的暂停/恢复机制 |
| 子图 | 复杂 Agent 的模块化封装 |

````
学习路径建议：
1. 先跑通线性流程（Section 4.1）
2. 加入条件分支（Section 4.2）
3. 实现 ReAct 循环（Section 4.3）
4. 加入持久化和人机协作（Section 4.4 + 6）
5. 尝试多 Agent 编排（Section 7）
6. 挑战实战案例（Section 8）
````
> 📚 官方文档：https://langchain-ai.github.io/langgraph/
> 📚 GitHub：https://github.com/langchain-ai/langgraph
> 📚 LangSmith：https://smith.langchain.com/

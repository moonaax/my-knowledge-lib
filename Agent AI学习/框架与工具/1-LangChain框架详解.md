# LangChain 框架详解

## 1. 概述

LangChain 是目前最流行的 LLM 应用开发框架，提供了构建 Agent、RAG、Chain 等应用的完整工具链。

### 核心理念

````
LangChain 的核心 = 组件化 + 可组合

Model I/O     → 统一的模型调用接口
Chains        → 将多个步骤串联
Agents        → 让 LLM 自主决策调用工具
Memory        → 对话记忆管理
Retrieval     → 检索增强（RAG）
````
### 生态全景

````
┌─────────────────────────────────────────┐
│              LangChain 生态              │
├──────────┬──────────┬───────────────────┤
│langchain │langchain │  LangSmith        │
│  -core   │-community│  (可观测性平台)    │
├──────────┴──────────┴───────────────────┤
│  LangGraph (有状态多 Agent 编排)         │
├─────────────────────────────────────────┤
│  LangServe (API 部署)                   │
└─────────────────────────────────────────┘
````
## 2. 安装与配置

> **包结构说明：** LangChain 拆分成了多个包——`langchain-core` 是核心抽象层（接口定义），`langchain` 是主框架，`langchain-openai` / `langchain-anthropic` 等是各模型提供商的适配器，`langchain-community` 是社区贡献的集成。按需安装可以避免引入不必要的依赖。

````bash
# 核心包
pip install langchain langchain-core

# 模型提供商（按需安装）
pip install langchain-openai      # OpenAI
pip install langchain-anthropic   # Claude
pip install langchain-community   # 社区集成

# 向量数据库
pip install chromadb faiss-cpu

# 完整安装
pip install langchain[all]
````
## 3. Model I/O — 模型调用

### 3.1 Chat Model

> **这是什么：** Chat Model 是 LangChain 对各种 LLM 的统一封装。不管底层是 OpenAI、Claude 还是本地模型，调用方式都一样——`invoke()`（单次调用）、`stream()`（流式输出）、`batch()`（批量调用）。换模型只需改一行初始化代码，业务逻辑完全不变。

````python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage

# 初始化模型
llm = ChatOpenAI(model="gpt-4o", temperature=0.7)

# 基础调用
response = llm.invoke([
    SystemMessage(content="你是一个 Python 专家"),
    HumanMessage(content="解释装饰器的原理")
])
print(response.content)

# 流式输出
for chunk in llm.stream("什么是 Agent?"):
    print(chunk.content, end="")

# 批量调用
results = llm.batch([
    [HumanMessage(content="1+1=?")],
    [HumanMessage(content="2+2=?")]
])
````

> **关键代码解读：**
> - `ChatOpenAI(model="gpt-4o", temperature=0.7)` — 初始化模型，`temperature` 控制随机性（0=确定性，1=更随机）
> - `invoke()` 接收消息列表，`SystemMessage` 设定角色，`HumanMessage` 是用户输入
> - `stream()` 返回生成器，逐 token 输出，适合实时展示
> - `batch()` 并行处理多个请求，比循环调用 invoke 更高效

### 3.2 Prompt Template

> **这是什么：** Prompt Template 把 prompt 模板化，用变量占位符替代硬编码的内容。好处是 prompt 可以复用、可以版本管理、可以和 Chain 组合。`|` 管道符把 prompt 和 llm 串起来，形成一个可调用的 chain。

````python
from langchain_core.prompts import ChatPromptTemplate

# 简单模板
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}专家"),
    ("human", "{question}")
])

# 使用模板
chain = prompt | llm
response = chain.invoke({"role": "Python", "question": "如何处理异常?"})

# 带 Few-Shot 的模板
from langchain_core.prompts import FewShotChatMessagePromptTemplate

examples = [
    {"input": "开心", "output": "happy"},
    {"input": "难过", "output": "sad"},
]

few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=ChatPromptTemplate.from_messages([
        ("human", "{input}"),
        ("ai", "{output}")
    ]),
    examples=examples,
)
````

> **关键代码解读：**
> - `ChatPromptTemplate.from_messages()` — 定义消息模板列表，`{role}` 和 `{question}` 是变量占位符
> - `chain = prompt | llm` — LCEL 语法，`|` 把 prompt 和 llm 串联，调用时自动先渲染模板再调用模型
> - `FewShotChatMessagePromptTemplate` — Few-Shot 模板，给 LLM 几个示例让它学会输出格式，比纯文字描述更有效

### 3.3 Output Parser

> **这是什么：** LLM 输出的是自由文本，但我们通常需要结构化数据（JSON、列表等）。Output Parser 用 Pydantic 模型定义期望的输出结构，自动在 prompt 中注入格式说明，并把 LLM 的文本输出解析为 Python 对象。

````python
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field

# 定义输出结构
class CodeReview(BaseModel):
    issues: list[str] = Field(description="发现的问题列表")
    score: int = Field(description="代码质量评分 1-10")
    suggestion: str = Field(description="改进建议")

parser = JsonOutputParser(pydantic_object=CodeReview)

prompt = ChatPromptTemplate.from_messages([
    ("system", "Review 代码并按指定格式输出。\n{format_instructions}"),
    ("human", "```python\n{code}\n```")
])

chain = prompt | llm | parser
result = chain.invoke({
    "code": "def add(a,b): return a+b",
    "format_instructions": parser.get_format_instructions()
})
# result = {"issues": [...], "score": 8, "suggestion": "..."}
````

> **关键代码解读：**
> - `JsonOutputParser(pydantic_object=CodeReview)` — 根据 Pydantic 模型自动生成格式说明（告诉 LLM 输出什么 JSON 结构）
> - `parser.get_format_instructions()` — 生成类似"请按以下 JSON 格式输出..."的指令文本
> - `chain = prompt | llm | parser` — 三步串联：渲染模板 → 调用模型 → 解析输出，最终 `result` 直接是 Python 字典
>
> **实际价值：** 没有 Output Parser 时，你需要手动写正则或 json.loads 来解析 LLM 输出，还要处理格式不对的情况。Parser 把这些都封装好了。

## 4. LCEL — LangChain 表达式语言

LCEL 是 LangChain 的核心编排方式，使用 `|` 管道符组合组件：

> **为什么需要 LCEL：** 传统方式是手动写 `result = parser.parse(llm.invoke(prompt.format(...)))`，嵌套调用又丑又难维护。LCEL 用 `|` 管道符让代码像 Unix 管道一样直观，而且自动获得流式输出、异步调用、批量处理、重试等能力，无需额外编码。

````python
from langchain_core.runnables import RunnablePassthrough, RunnableLambda

# 基础链
chain = prompt | llm | parser

# 并行执行
from langchain_core.runnables import RunnableParallel

parallel_chain = RunnableParallel(
    summary=prompt_summary | llm,
    translation=prompt_translate | llm,
)
result = parallel_chain.invoke({"text": "..."})
# result = {"summary": "...", "translation": "..."}

# 条件分支
from langchain_core.runnables import RunnableBranch

branch = RunnableBranch(
    (lambda x: "代码" in x["question"], code_chain),
    (lambda x: "数据" in x["question"], data_chain),
    default_chain  # 默认分支
)

# 带重试和回退
chain_with_fallback = llm.with_fallbacks([backup_llm])
chain_with_retry = llm.with_retry(stop_after_attempt=3)
````

> **关键代码解读：**
> - `RunnableParallel` — 并行执行多个 chain，结果合并为字典。比如同时做摘要和翻译，耗时取决于最慢的那个而非两者之和
> - `RunnableBranch` — 条件分支，根据输入内容路由到不同的 chain，类似 if-else
> - `with_fallbacks([backup_llm])` — 主模型失败时自动切换到备用模型，提高可用性
> - `with_retry(stop_after_attempt=3)` — 自动重试，应对网络抖动或 API 限流

## 5. Agent — 智能体

### 5.1 基础 Agent

> **Agent 和 Chain 的区别：** Chain 是预定义的固定流程（A→B→C），Agent 是让 LLM 自主决定下一步做什么——它会根据用户问题判断需要调用哪个工具、调用几次、以什么顺序调用。这就是"智能体"的核心：自主决策能力。

````python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.tools import tool

# 定义工具
@tool
def search(query: str) -> str:
    """搜索互联网获取信息"""
    # 实际实现搜索逻辑
    return f"搜索结果: {query}"

@tool
def calculator(expression: str) -> str:
    """计算数学表达式"""
    return str(eval(expression))

# 创建 Agent
tools = [search, calculator]
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手，可以使用工具回答问题。"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}")
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 运行
result = executor.invoke({"input": "北京今天的天气怎么样?"})
````

> **关键代码解读：**
> - `@tool` 装饰器 — 把普通函数变成 Agent 可调用的工具，函数的 docstring 会作为工具描述传给 LLM（所以描述要写清楚）
> - `create_tool_calling_agent()` — 创建一个能调用工具的 Agent，它会根据用户问题自动决定是否需要调用工具
> - `AgentExecutor` — Agent 的运行时，负责执行 Agent 的决策循环：思考 → 调用工具 → 观察结果 → 继续思考...
> - `{agent_scratchpad}` — Agent 的"草稿纸"，存放中间的思考过程和工具调用结果

### 5.2 自定义工具

> **什么时候需要自定义工具：** `@tool` 适合简单函数，`StructuredTool` 适合需要多个参数、参数校验的复杂工具。工具的 `description` 至关重要——LLM 完全靠描述来判断何时调用这个工具。

````python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    max_results: int = Field(default=5, description="最大结果数")

def search_func(query: str, max_results: int = 5) -> str:
    # 实现搜索逻辑
    return f"找到 {max_results} 条关于 '{query}' 的结果"

search_tool = StructuredTool.from_function(
    func=search_func,
    name="web_search",
    description="搜索互联网获取最新信息",
    args_schema=SearchInput
)
````

> **关键代码解读：**
> - `StructuredTool.from_function()` — 从普通函数创建工具，`args_schema` 用 Pydantic 模型定义参数结构
> - `SearchInput` — 定义了 `query`（必填）和 `max_results`（可选，默认 5）两个参数，`Field(description=...)` 告诉 LLM 每个参数的含义

## 6. LangGraph — 有状态 Agent 编排

LangGraph 是 LangChain 团队推出的有状态、可循环的 Agent 编排框架：

> **为什么需要 LangGraph：** LCEL 的 Chain 是线性的（A→B→C），不支持循环和条件跳转。但真实的 Agent 需要"思考→执行→检查→不满意就重来"这种循环流程。LangGraph 用有向图（节点 + 边）来编排，支持循环、分支、状态持久化，是构建复杂 Agent 的首选。

````python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

# 定义状态
class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    next_step: str

# 定义节点
def analyze(state: AgentState) -> AgentState:
    # 分析任务
    response = llm.invoke(state["messages"])
    return {"messages": [response], "next_step": "execute"}

def execute(state: AgentState) -> AgentState:
    # 执行操作
    return {"messages": ["执行完成"], "next_step": "review"}

def review(state: AgentState) -> AgentState:
    # 审查结果
    return {"messages": ["审查通过"], "next_step": "end"}

# 构建图
graph = StateGraph(AgentState)
graph.add_node("analyze", analyze)
graph.add_node("execute", execute)
graph.add_node("review", review)

graph.set_entry_point("analyze")
graph.add_edge("analyze", "execute")
graph.add_edge("execute", "review")
graph.add_edge("review", END)

# 编译并运行
app = graph.compile()
result = app.invoke({"messages": ["请帮我分析这段代码"], "next_step": ""})
````

> **关键代码解读：**
> - `StateGraph(AgentState)` — 创建一个以 `AgentState` 为状态的有向图
> - `add_node("name", func)` — 添加节点，每个节点是一个处理函数，接收状态、返回更新后的状态
> - `add_edge("A", "B")` — 添加边，定义节点之间的执行顺序
> - `set_entry_point()` — 设置入口节点
> - `graph.compile()` — 编译图为可执行的应用
>
> **和 LCEL Chain 的对比：** Chain 是 `prompt | llm | parser`（线性管道），LangGraph 是节点+边的图结构，可以有循环（如 analyze→execute→review→analyze）和条件分支。

## 7. Memory — 记忆管理

> **这是什么：** LangChain 的记忆模块让 Chain/Agent 能"记住"之前的对话。核心是 `RunnableWithMessageHistory`——它包装一个 chain，自动在每次调用前加载历史消息、调用后保存新消息。通过 `session_id` 区分不同用户/会话。

````python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# 会话存储
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

# 带记忆的链
chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history"
)

# 使用（同一 session_id 共享记忆）
chain_with_memory.invoke(
    {"input": "我叫张三"},
    config={"configurable": {"session_id": "user_001"}}
)
chain_with_memory.invoke(
    {"input": "我叫什么名字?"},  # 能记住上一轮
    config={"configurable": {"session_id": "user_001"}}
)
````

> **关键代码解读：**
> - `InMemoryChatMessageHistory()` — 内存中的消息存储，生产环境可换成 Redis、数据库等
> - `get_session_history(session_id)` — 根据 session_id 获取对应的历史记录，不同用户的记忆互相隔离
> - `RunnableWithMessageHistory` — 包装 chain，自动注入历史消息。`input_messages_key` 指定输入字段名，`history_messages_key` 指定历史消息在 prompt 中的占位符
> - 两次调用用同一个 `session_id`，第二次问"我叫什么"时能从历史中找到答案

## 8. LangSmith — 可观测性

> **为什么需要可观测性：** Agent 的行为是不确定的——它可能调用了错误的工具、陷入了循环、或者中间某步推理出了问题。没有追踪工具，调试 Agent 就像黑盒。LangSmith 记录每一步的输入输出、耗时、Token 用量，让你能"回放"Agent 的完整思考过程。

````python
# 设置环境变量启用 LangSmith 追踪
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-key"
os.environ["LANGCHAIN_PROJECT"] = "my-agent-project"

# 之后所有 LangChain 调用都会自动记录到 LangSmith
# 可以在 LangSmith 控制台查看:
# - 每次调用的输入/输出
# - Token 使用量和延迟
# - 错误和异常
# - 完整的调用链路
````

> **关键代码解读：** 只需设置 3 个环境变量就能启用追踪，零代码侵入。之后所有 LangChain 调用（Chain、Agent、Tool）都会自动上报到 LangSmith 控制台，可以看到完整的调用树、每个节点的输入输出、Token 消耗和延迟。

## 9. 最佳实践

### 项目结构

````
my-agent/
├── agents/          # Agent 定义
│   ├── base.py
│   └── code_agent.py
├── tools/           # 工具定义
│   ├── search.py
│   └── code_exec.py
├── prompts/         # Prompt 模板
│   └── templates.py
├── chains/          # Chain 定义
│   └── review_chain.py
├── config.py        # 配置
└── main.py          # 入口
````
### 关键建议

1. **从简单开始**：先用 Chain，复杂了再用 Agent
2. **工具描述要精确**：LLM 根据描述决定何时调用工具
3. **开启 LangSmith**：调试 Agent 必备
4. **控制 Agent 循环**：设置 `max_iterations` 防止无限循环
5. **错误处理**：工具调用可能失败，要有降级方案

---

## 面试题精选

### Q1: LangChain 的核心组件有哪些？各自的作用？
**答：** Model I/O（统一模型调用接口）、Chains（多步骤串联）、Agents（LLM 自主决策调用工具）、Memory（对话记忆管理）、Retrieval（RAG 检索增强）。核心理念是组件化 + 可组合。

### Q2: LCEL（LangChain Expression Language）是什么？有什么优势？
**答：** LCEL 用 `|` 管道符组合组件（如 prompt | llm | parser），支持流式输出、异步调用、批量处理、并行执行、重试和回退。优势是代码简洁、自动获得这些能力而无需额外编码。

### Q3: LangChain 和 LangGraph 的关系是什么？什么时候用 LangGraph？
**答：** LangGraph 是 LangChain 团队推出的有状态 Agent 编排框架。当需要循环执行、条件分支、状态持久化、多 Agent 协作时用 LangGraph。简单的线性 Chain 用 LangChain LCEL 就够了。

### Q4: LangChain 中如何实现带记忆的多轮对话？
**答：** 使用 RunnableWithMessageHistory 包装 Chain，配合 ChatMessageHistory 存储对话历史。通过 session_id 区分不同会话，每次调用自动加载历史消息并追加新消息。

### Q5: LangSmith 在 Agent 开发中有什么作用？
**答：** LangSmith 提供完整的调用链路追踪，可以看到每步的输入输出、Token 使用量、延迟、错误详情。对于调试 Agent 的推理过程和工具调用链路至关重要，是生产环境必备的可观测性工具。

### Q6: 使用 LangChain 开发 Agent 有哪些最佳实践？
**答：** 从简单 Chain 开始，复杂了再用 Agent；工具描述要精确（LLM 靠描述决定调用时机）；开启 LangSmith 追踪；设置 max_iterations 防止无限循环；工具调用要有错误处理和降级方案。

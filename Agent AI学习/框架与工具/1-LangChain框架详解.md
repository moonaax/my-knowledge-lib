# LangChain 框架详解

## 1. 概述

LangChain 是目前最流行的 LLM 应用开发框架，提供了构建 Agent、RAG、Chain 等应用的完整工具链。

### 核心理念

```
LangChain 的核心 = 组件化 + 可组合

Model I/O     → 统一的模型调用接口
Chains        → 将多个步骤串联
Agents        → 让 LLM 自主决策调用工具
Memory        → 对话记忆管理
Retrieval     → 检索增强（RAG）
```

### 生态全景

```
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
```

## 2. 安装与配置

```bash
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
```

## 3. Model I/O — 模型调用

### 3.1 Chat Model

```python
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
```

### 3.2 Prompt Template

```python
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
```

### 3.3 Output Parser

```python
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
```

## 4. LCEL — LangChain 表达式语言

LCEL 是 LangChain 的核心编排方式，使用 `|` 管道符组合组件：

```python
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
```

## 5. Agent — 智能体

### 5.1 基础 Agent

```python
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
```

### 5.2 自定义工具

```python
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
```

## 6. LangGraph — 有状态 Agent 编排

LangGraph 是 LangChain 团队推出的有状态、可循环的 Agent 编排框架：

```python
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
```

## 7. Memory — 记忆管理

```python
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
```

## 8. LangSmith — 可观测性

```python
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
```

## 9. 最佳实践

### 项目结构

```
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
```

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

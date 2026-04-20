# Tool 与 Agent — LangChain 工具定义与智能体构建

> LangChain 专题第三篇。本文全面讲解如何在 LangChain 中定义工具、构建 Agent，并通过实战案例掌握智能体开发的核心技能。

## Tool 工具体系概述

### Agent 为什么需要工具

LLM 本身只能生成文本，无法直接与外部世界交互。它不能搜索互联网、不能执行代码、不能读写文件、不能调用 API。**工具（Tool）是 LLM 连接真实世界的桥梁**。

```
┌─────────────────────────────────────────────────┐
│                   Agent                          │
│                                                 │
│   ┌───────┐    ┌──────────┐    ┌──────────┐   │
│   │  LLM  │───▶│ 决策引擎  │───▶│ 工具调用  │   │
│   └───────┘    └──────────┘    └──────────┘   │
│                                     │           │
└─────────────────────────────────────┼───────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                  ▼
              ┌──────────┐    ┌──────────┐      ┌──────────┐
              │ 搜索引擎  │    │ 数据库    │      │ 计算器    │
              └──────────┘    └──────────┘      └──────────┘
```

核心思想：
- LLM 负责**推理和决策**（选择用哪个工具、传什么参数）
- Tool 负责**执行动作**（搜索、计算、API 调用等）
- Agent 是将两者结合的**编排框架**

### 工具在 LangChain 中的角色

在 LangChain 中，Tool 是一个标准化的接口，包含：

| 属性 | 说明 |
|------|------|
| `name` | 工具名称，LLM 通过名称引用工具 |
| `description` | 工具描述，LLM 根据描述决定何时使用 |
| `args_schema` | 输入参数的 Pydantic Schema |
| `func` | 实际执行的函数 |

```python
from langchain_core.tools import tool

@tool
def multiply(a: int, b: int) -> int:
    """将两个数字相乘。当用户需要计算乘法时使用此工具。"""
    return a * b

# 查看工具的元信息
print(f"名称: {multiply.name}")          # multiply
print(f"描述: {multiply.description}")    # 将两个数字相乘...
print(f"参数: {multiply.args}")           # {'a': {'type': 'integer'}, 'b': {'type': 'integer'}}
```

## 工具定义方式

LangChain 提供三种定义工具的方式，适用于不同复杂度的场景。

### @tool 装饰器 — 最简方式

最快捷的工具定义方式，适合简单函数：

```python
from langchain_core.tools import tool

@tool
def search_weather(city: str) -> str:
    """查询指定城市的天气信息。输入城市名称，返回当前天气。"""
    # 模拟天气查询
    weather_data = {
        "北京": "晴天，25°C",
        "上海": "多云，22°C",
        "广州": "小雨，28°C",
    }
    return weather_data.get(city, f"未找到 {city} 的天气信息")

# 直接调用
result = search_weather.invoke({"city": "北京"})
print(result)  # 晴天，25°C
```

支持返回工件（artifact）的工具：

```python
from langchain_core.tools import tool

@tool(response_format="content_and_artifact")
def generate_report(topic: str) -> tuple[str, dict]:
    """生成指定主题的报告摘要和完整数据。"""
    data = {"topic": topic, "pages": 10, "charts": 3}
    summary = f"已生成关于「{topic}」的报告，共 10 页 3 张图表"
    return summary, data
```

### StructuredTool.from_function — 带 Schema 的工具

当需要更精细地控制参数验证时使用：

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    max_results: int = Field(default=5, description="最大返回结果数", ge=1, le=20)

def search_documents(query: str, max_results: int = 5) -> str:
    """在知识库中搜索文档。"""
    return f"搜索「{query}」，找到 {max_results} 条结果"

search_tool = StructuredTool.from_function(
    func=search_documents,
    name="knowledge_search",
    description="在内部知识库中搜索相关文档。当用户询问公司内部信息时使用。",
    args_schema=SearchInput,
)

result = search_tool.invoke({"query": "年度报告", "max_results": 3})
print(result)
```

### BaseTool 子类 — 完全自定义

适合需要维护状态、异步执行或复杂逻辑的场景：

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Optional, Type
import httpx

class APIQueryInput(BaseModel):
    endpoint: str = Field(description="API 端点路径")
    method: str = Field(default="GET", description="HTTP 方法")

class APIQueryTool(BaseTool):
    name: str = "api_query"
    description: str = "调用内部 REST API。用于获取业务系统数据。"
    args_schema: Type[BaseModel] = APIQueryInput
    base_url: str = "https://api.example.com"

    def _run(self, endpoint: str, method: str = "GET") -> str:
        """同步执行"""
        url = f"{self.base_url}/{endpoint}"
        # 实际项目中发起 HTTP 请求
        return f"[模拟] {method} {url} -> 200 OK"

    async def _arun(self, endpoint: str, method: str = "GET") -> str:
        """异步执行"""
        url = f"{self.base_url}/{endpoint}"
        async with httpx.AsyncClient() as client:
            resp = await client.request(method, url)
            return resp.text

api_tool = APIQueryTool()
print(api_tool.invoke({"endpoint": "users/1", "method": "GET"}))
```

### 工具描述的重要性

**LLM 完全依赖工具的 `name` 和 `description` 来决定使用哪个工具**。描述写得好坏直接决定 Agent 的表现。

| 描述质量 | 示例 | 效果 |
|---------|------|------|
| ❌ 差 | `"搜索工具"` | LLM 不知道何时该用 |
| ⚠️ 一般 | `"搜索互联网"` | 模糊，可能误用 |
| ✅ 好 | `"搜索互联网获取实时信息。当用户询问最新新闻、实时数据或你不确定的事实时使用此工具。输入为搜索关键词。"` | 精确触发 |

编写描述的原则：
1. **说明用途**：这个工具做什么
2. **说明时机**：什么情况下应该使用
3. **说明输入**：需要什么参数
4. **说明限制**：不能做什么（可选）

```python
@tool
def calculator(expression: str) -> str:
    """安全地计算数学表达式并返回结果。

    当用户需要进行数学计算时使用此工具，包括加减乘除、幂运算、开方等。
    输入应为合法的 Python 数学表达式，如 "2 + 3 * 4" 或 "math.sqrt(16)"。
    注意：不支持赋值语句和函数定义。
    """
    import math
    allowed_names = {"math": math, "abs": abs, "round": round}
    return str(eval(expression, {"__builtins__": {}}, allowed_names))
```

## 内置工具与社区工具

LangChain 生态提供了大量开箱即用的工具，覆盖搜索、代码执行、文件操作等常见场景。

### 搜索工具

#### Tavily Search（推荐）

Tavily 是专为 AI Agent 设计的搜索引擎，返回结构化结果：

```python
# pip install langchain-community tavily-python
import os
os.environ["TAVILY_API_KEY"] = "your-api-key"

from langchain_community.tools.tavily_search import TavilySearchResults

search = TavilySearchResults(
    max_results=3,
    search_depth="advanced",  # "basic" 或 "advanced"
    include_answer=True,
)

results = search.invoke("LangChain 最新版本是什么")
for r in results:
    print(f"- {r['url']}: {r['content'][:100]}")
```

#### SerpAPI

Google 搜索的 API 封装：

```python
# pip install google-search-results
os.environ["SERPAPI_API_KEY"] = "your-api-key"

from langchain_community.utilities import SerpAPIWrapper
from langchain_core.tools import Tool

serpapi = SerpAPIWrapper()
search_tool = Tool(
    name="google_search",
    func=serpapi.run,
    description="通过 Google 搜索获取实时互联网信息。输入为搜索关键词。",
)

print(search_tool.invoke("Python 3.12 新特性"))
```

### 代码执行工具

```python
# pip install langchain-experimental
from langchain_experimental.tools import PythonREPLTool

python_repl = PythonREPLTool()

# Agent 可以用它执行 Python 代码
result = python_repl.invoke("print(sum(range(1, 101)))")
print(result)  # 5050
```

> ⚠️ **安全警告**：代码执行工具有安全风险，生产环境应使用沙箱（如 Docker 容器）隔离执行。

### 文件操作工具

```python
from langchain_community.tools.file_management import (
    ReadFileTool,
    WriteFileTool,
    ListDirectoryTool,
)

# 初始化文件工具
read_tool = ReadFileTool()
write_tool = WriteFileTool()
list_tool = ListDirectoryTool()

# 使用示例
files = list_tool.invoke({"dir_path": "."})
content = read_tool.invoke({"file_path": "README.md"})
```

### 如何发现和使用社区工具

```python
# 1. 通过 langchain-community 包发现工具
# pip install langchain-community

# 2. 查看所有可用工具集成
# 文档：https://python.langchain.com/docs/integrations/tools/

# 3. 常用工具包一览
from langchain_community.tools import (
    WikipediaQueryRun,        # 维基百科
    ArxivQueryRun,            # 学术论文
    DuckDuckGoSearchRun,      # DuckDuckGo 搜索（免费）
    ShellTool,                # Shell 命令
)

# 4. 工具包（Toolkit）— 一组相关工具的集合
from langchain_community.agent_toolkits import SQLDatabaseToolkit
# toolkit = SQLDatabaseToolkit(db=db, llm=llm)
# tools = toolkit.get_tools()
```

社区工具使用模式：

```
pip install langchain-community  # 基础包
pip install <specific-package>   # 特定工具的依赖

from langchain_community.tools import XxxTool
tool = XxxTool(api_key="...")
```

## Agent 核心概念

### Agent 的推理循环：思考 → 行动 → 观察

Agent 不是一次性生成答案，而是通过**迭代循环**逐步解决问题：

```
┌──────────────────────────────────────────────────────┐
│                  Agent 推理循环                        │
│                                                      │
│   ┌────────┐    ┌────────┐    ┌────────┐           │
│   │ Thought │──▶│ Action │──▶│Observe │──┐         │
│   │  思考   │    │  行动   │    │  观察  │  │         │
│   └────────┘    └────────┘    └────────┘  │         │
│        ▲                                   │         │
│        └───────────────────────────────────┘         │
│                                                      │
│   循环直到：得出最终答案 或 达到最大迭代次数            │
└──────────────────────────────────────────────────────┘
```

一个完整的推理过程示例：

```
用户问题：北京今天的气温是多少摄氏度？换算成华氏度是多少？

[第 1 轮]
Thought: 我需要先查询北京的天气获取气温
Action: search_weather(city="北京")
Observation: 晴天，25°C

[第 2 轮]
Thought: 气温是 25°C，我需要换算成华氏度。公式是 F = C × 9/5 + 32
Action: calculator(expression="25 * 9 / 5 + 32")
Observation: 77.0

[第 3 轮]
Thought: 我已经得到了所有信息，可以回答用户了
Final Answer: 北京今天气温 25°C，换算成华氏度是 77°F。
```

### ReAct 模式在 LangChain 中的实现

ReAct（Reasoning + Acting）是最经典的 Agent 模式，由 Yao et al. 2022 提出。核心思想是让 LLM 交替进行**推理**和**行动**。

LangChain 中 ReAct 的实现依赖特定的 Prompt 模板：

```python
# ReAct Prompt 的核心结构
REACT_PROMPT = """Answer the following questions as best you can.
You have access to the following tools:

{tools}

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought: {agent_scratchpad}"""
```

### AgentExecutor 的工作原理

`AgentExecutor` 是 Agent 的运行时引擎，负责：

1. **调用 Agent** 获取下一步动作
2. **执行工具** 并收集结果
3. **将结果反馈** 给 Agent
4. **判断终止条件**（最终答案 / 最大迭代 / 超时）

```
AgentExecutor 内部流程：

while True:
    ├── agent.plan(input, intermediate_steps)
    │       │
    │       ├── 返回 AgentAction → 执行工具
    │       │       ├── tool.run(action.tool_input)
    │       │       └── 将结果加入 intermediate_steps
    │       │
    │       └── 返回 AgentFinish → 返回最终结果
    │
    └── 检查：是否超过 max_iterations？→ 强制停止
```

关键参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_iterations` | 15 | 最大循环次数，防止无限循环 |
| `max_execution_time` | None | 最大执行时间（秒） |
| `early_stopping_method` | "force" | 达到限制时的处理方式 |
| `handle_parsing_errors` | False | 是否自动处理输出解析错误 |

## Agent 构建

### create_tool_calling_agent — 基于 Function Calling 的 Agent

这是**推荐的现代方式**，利用 LLM 原生的 Function Calling 能力（OpenAI、Anthropic、Google 等均支持）：

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.agents import create_tool_calling_agent, AgentExecutor

# 1. 定义工具
@tool
def get_word_length(word: str) -> int:
    """获取单词的字符长度。"""
    return len(word)

@tool
def add(a: int, b: int) -> int:
    """计算两个整数的和。"""
    return a + b

tools = [get_word_length, add]

# 2. 创建 LLM
llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 3. 定义 Prompt（必须包含 agent_scratchpad）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有用的助手。尽可能使用工具来回答问题。"),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

# 4. 创建 Agent
agent = create_tool_calling_agent(llm, tools, prompt)

# 5. 用 AgentExecutor 运行
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = agent_executor.invoke({"input": "hello 这个单词有几个字母？再加上 3 等于多少？"})
print(result["output"])
```

Function Calling 方式的优势：
- LLM 原生支持，参数解析更准确
- 支持并行工具调用（一次调用多个工具）
- 不需要复杂的 Prompt 工程

### create_react_agent — ReAct Agent

基于文本解析的经典方式，兼容性更广：

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain.agents import create_react_agent, AgentExecutor
from langchain import hub

# 1. 定义工具
@tool
def search(query: str) -> str:
    """搜索互联网获取信息。"""
    return f"搜索结果：关于「{query}」的最新信息..."

@tool
def calculator(expression: str) -> str:
    """计算数学表达式。输入合法的数学表达式。"""
    return str(eval(expression))

tools = [search, calculator]

# 2. 从 LangChain Hub 拉取标准 ReAct Prompt
prompt = hub.pull("hwchase17/react")

# 3. 创建 ReAct Agent
llm = ChatOpenAI(model="gpt-4o", temperature=0)
agent = create_react_agent(llm, tools, prompt)

# 4. 运行
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
result = agent_executor.invoke({"input": "2024年世界人口大约多少？除以中国人口大约是多少倍？"})
print(result["output"])
```

### AgentExecutor 配置详解

```python
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,

    # --- 核心配置 ---
    verbose=True,                    # 打印推理过程
    max_iterations=10,               # 最大迭代次数（防止无限循环）
    max_execution_time=60,           # 最大执行时间（秒）
    early_stopping_method="generate",# "force"=直接停止 / "generate"=让LLM总结

    # --- 错误处理 ---
    handle_parsing_errors=True,      # 自动处理解析错误
    # 也可以传入自定义处理函数：
    # handle_parsing_errors=lambda e: f"解析错误，请重试: {e}",

    # --- 返回配置 ---
    return_intermediate_steps=True,  # 返回中间步骤（调试用）
)

result = agent_executor.invoke({"input": "你的问题"})

# 查看中间步骤
if "intermediate_steps" in result:
    for step in result["intermediate_steps"]:
        action, observation = step
        print(f"工具: {action.tool}, 输入: {action.tool_input}")
        print(f"结果: {observation}\n")
```

`early_stopping_method` 对比：

| 方法 | 行为 | 适用场景 |
|------|------|---------|
| `"force"` | 直接返回 "Agent stopped" | 严格控制成本 |
| `"generate"` | 让 LLM 基于已有信息生成最终答案 | 用户体验优先 |

## Agent 调试

### verbose 模式查看推理过程

最简单的调试方式，打印 Agent 每一步的思考和行动：

```python
# 方式 1：AgentExecutor 级别
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 方式 2：全局设置
from langchain.globals import set_verbose, set_debug
set_verbose(True)   # 打印关键步骤
set_debug(True)     # 打印所有细节（非常详细）
```

verbose 输出示例：

```
> Entering new AgentExecutor chain...

Invoking: `search_weather` with `{'city': '北京'}`
responded: 我需要查询北京的天气信息。

晴天，25°C

Invoking: `calculator` with `{'expression': '25 * 9 / 5 + 32'}`
responded: 现在我需要将摄氏度转换为华氏度。

77.0

北京今天气温 25°C，换算成华氏度是 77°F。

> Finished chain.
```

### LangSmith 追踪 Agent 链路

LangSmith 是 LangChain 官方的可观测性平台，提供完整的调用链追踪：

```python
import os

# 1. 配置 LangSmith（注册后获取 API Key）
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-langsmith-api-key"
os.environ["LANGCHAIN_PROJECT"] = "my-agent-project"

# 2. 正常运行 Agent，所有调用自动上报
result = agent_executor.invoke({"input": "你的问题"})

# 3. 在 LangSmith 控制台查看：
#    - 完整调用链（每个 LLM 调用、工具调用）
#    - Token 用量和延迟
#    - 输入输出详情
#    - 错误堆栈
```

LangSmith 追踪视图中可以看到：

```
AgentExecutor (3.2s, $0.012)
├── ChatOpenAI (1.1s) → 决定调用 search_weather
├── search_weather (0.3s) → "晴天，25°C"
├── ChatOpenAI (0.9s) → 决定调用 calculator
├── calculator (0.01s) → "77.0"
└── ChatOpenAI (0.8s) → 生成最终答案
```

### 常见问题排查

#### 问题 1：工具调用失败

```python
# 症状：Agent 报错 "Tool not found" 或参数错误

# 排查步骤：
# 1. 检查工具名称是否与 Agent 注册的一致
print([t.name for t in tools])

# 2. 检查参数 schema 是否正确
print(my_tool.args)

# 3. 添加错误处理
@tool
def safe_search(query: str) -> str:
    """搜索工具（带错误处理）"""
    try:
        return do_search(query)
    except Exception as e:
        return f"搜索失败: {str(e)}，请尝试其他关键词"
```

#### 问题 2：无限循环

```python
# 症状：Agent 反复调用同一个工具，不收敛

# 解决方案：
# 1. 设置 max_iterations
agent_executor = AgentExecutor(agent=agent, tools=tools, max_iterations=5)

# 2. 改进工具描述，明确告诉 LLM 何时停止
@tool
def search(query: str) -> str:
    """搜索信息。如果搜索结果已经足够回答问题，不要再次搜索。"""
    return "..."

# 3. 在 system prompt 中加入约束
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是助手。如果已有足够信息，直接回答，不要重复使用工具。"),
    ...
])
```

#### 问题 3：输出格式错误（ReAct Agent）

```python
# 症状：OutputParserException - Could not parse LLM output

# 解决方案：
# 1. 启用 handle_parsing_errors
agent_executor = AgentExecutor(
    agent=agent, tools=tools,
    handle_parsing_errors="输出格式错误，请严格按照 Thought/Action/Action Input 格式重试。"
)

# 2. 使用 Function Calling Agent 替代 ReAct（推荐）
# Function Calling 由 LLM 原生支持，格式错误概率极低
```

## Callback 回调系统

Callback 是 LangChain 的事件钩子系统，可以在 LLM 调用、工具执行、链运行等各个阶段插入自定义逻辑。

### 内置 Callback Handler

```python
from langchain_core.callbacks import StdOutCallbackHandler
from langchain_community.callbacks import get_openai_callback

# 1. 标准输出 Handler（等同于 verbose=True）
handler = StdOutCallbackHandler()
result = agent_executor.invoke(
    {"input": "问题"},
    config={"callbacks": [handler]}
)

# 2. Token 计费统计
with get_openai_callback() as cb:
    result = agent_executor.invoke({"input": "你的问题"})
    print(f"总 Token: {cb.total_tokens}")
    print(f"总费用: ${cb.total_cost:.4f}")
    print(f"请求次数: {cb.successful_requests}")
```

### 自定义 Callback

```python
from langchain_core.callbacks import BaseCallbackHandler
from typing import Any, Dict, List
from langchain_core.agents import AgentAction, AgentFinish

class MyAgentCallback(BaseCallbackHandler):
    """自定义 Agent 回调，记录每一步操作。"""

    def on_agent_action(self, action: AgentAction, **kwargs) -> None:
        """Agent 决定执行工具时触发"""
        print(f"🔧 调用工具: {action.tool}")
        print(f"   输入: {action.tool_input}")

    def on_agent_finish(self, finish: AgentFinish, **kwargs) -> None:
        """Agent 完成时触发"""
        print(f"✅ 最终答案: {finish.return_values.get('output', '')[:100]}")

    def on_tool_start(self, serialized: Dict, input_str: str, **kwargs) -> None:
        """工具开始执行时触发"""
        print(f"⏳ 工具执行中...")

    def on_tool_end(self, output: str, **kwargs) -> None:
        """工具执行完成时触发"""
        print(f"📋 工具返回: {output[:200]}")

    def on_tool_error(self, error: BaseException, **kwargs) -> None:
        """工具执行出错时触发"""
        print(f"❌ 工具错误: {error}")

    def on_llm_start(self, serialized: Dict, prompts: List[str], **kwargs) -> None:
        """LLM 调用开始"""
        print(f"🧠 LLM 思考中...")

# 使用自定义 Callback
callback = MyAgentCallback()
result = agent_executor.invoke(
    {"input": "北京天气如何？"},
    config={"callbacks": [callback]}
)
```

### 流式输出中的 Callback

实现 Agent 推理过程的实时流式输出：

```python
from langchain_core.callbacks import BaseCallbackHandler
import sys

class StreamingCallback(BaseCallbackHandler):
    """流式输出 Handler，逐 token 打印 LLM 生成内容。"""

    def on_llm_new_token(self, token: str, **kwargs) -> None:
        """每生成一个 token 时触发"""
        sys.stdout.write(token)
        sys.stdout.flush()

    def on_agent_action(self, action: AgentAction, **kwargs) -> None:
        print(f"\n\n🔧 正在使用工具: {action.tool}\n")

# 配合流式 LLM 使用
llm = ChatOpenAI(model="gpt-4o", temperature=0, streaming=True)

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools)

# 使用 stream 方法获取流式事件
for event in agent_executor.stream(
    {"input": "你的问题"},
    config={"callbacks": [StreamingCallback()]}
):
    # event 包含每一步的详细信息
    if "output" in event:
        print(f"\n最终答案: {event['output']}")
```

使用 `astream_events` 获取更细粒度的事件流：

```python
async def stream_agent():
    async for event in agent_executor.astream_events(
        {"input": "你的问题"},
        version="v2",
    ):
        kind = event["event"]
        if kind == "on_chat_model_stream":
            # LLM 生成的每个 token
            content = event["data"]["chunk"].content
            if content:
                print(content, end="", flush=True)
        elif kind == "on_tool_start":
            print(f"\n🔧 调用: {event['name']}")
        elif kind == "on_tool_end":
            print(f"📋 结果: {event['data']['output'][:100]}")

import asyncio
asyncio.run(stream_agent())
```

## 实战案例 — 构建一个完整的多工具 Agent

构建一个「研究助手 Agent」，具备搜索、计算、文件写入能力：

```python
"""
研究助手 Agent — 完整实战案例
功能：搜索信息、数学计算、生成并保存报告
"""
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.callbacks import BaseCallbackHandler
from datetime import datetime
import json

# ============ 1. 定义工具集 ============

@tool
def web_search(query: str) -> str:
    """搜索互联网获取实时信息。当需要查找最新数据、新闻或不确定的事实时使用。"""
    # 实际项目中接入 Tavily/SerpAPI
    mock_results = {
        "python": "Python 3.12 发布于 2024 年，新增类型参数语法和性能改进。",
        "langchain": "LangChain 是最流行的 LLM 应用框架，支持 Agent、RAG 等模式。",
        "ai": "2024 年 AI 领域重大进展包括 GPT-4o、Claude 3.5、Gemini 等。",
    }
    for key, value in mock_results.items():
        if key in query.lower():
            return value
    return f"关于「{query}」的搜索结果：暂无相关信息，请尝试其他关键词。"

@tool
def calculator(expression: str) -> str:
    """安全计算数学表达式。支持加减乘除、幂运算、百分比等。
    输入示例: "100 * 1.15", "2**10", "(500 - 200) / 3"
    """
    import math
    allowed = {"math": math, "abs": abs, "round": round, "pow": pow}
    try:
        result = eval(expression, {"__builtins__": {}}, allowed)
        return f"计算结果: {result}"
    except Exception as e:
        return f"计算错误: {e}。请检查表达式格式。"

@tool
def save_report(title: str, content: str) -> str:
    """将研究报告保存为文件。title 为报告标题，content 为报告正文（Markdown 格式）。"""
    filename = f"report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.md"
    full_content = f"# {title}\n\n生成时间: {datetime.now().strftime('%Y-%m-%d %H:%M')}\n\n{content}"
    # 实际写入文件
    with open(filename, "w", encoding="utf-8") as f:
        f.write(full_content)
    return f"报告已保存为 {filename}（{len(full_content)} 字符）"

@tool
def get_current_time() -> str:
    """获取当前日期和时间。"""
    return datetime.now().strftime("%Y年%m月%d日 %H:%M:%S")

# ============ 2. 自定义 Callback ============

class ResearchCallback(BaseCallbackHandler):
    def __init__(self):
        self.steps = []

    def on_agent_action(self, action, **kwargs):
        self.steps.append({"tool": action.tool, "input": str(action.tool_input)})
        print(f"  📍 Step {len(self.steps)}: 使用 {action.tool}")

    def on_agent_finish(self, finish, **kwargs):
        print(f"  ✅ 完成！共执行 {len(self.steps)} 步")

# ============ 3. 构建 Agent ============

tools = [web_search, calculator, save_report, get_current_time]

llm = ChatOpenAI(model="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个专业的研究助手。你的工作流程：
1. 理解用户的研究需求
2. 使用搜索工具收集信息
3. 使用计算器进行必要的数据分析
4. 整理信息并生成结构化报告
5. 将报告保存为文件

注意：
- 搜索时使用精确的关键词
- 计算时确保表达式正确
- 报告使用 Markdown 格式，包含标题、要点和总结"""),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

agent = create_tool_calling_agent(llm, tools, prompt)

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=10,
    handle_parsing_errors=True,
    return_intermediate_steps=True,
)

# ============ 4. 运行 Agent ============

if __name__ == "__main__":
    callback = ResearchCallback()

    result = agent_executor.invoke(
        {"input": "帮我研究一下 LangChain 框架，计算如果一个团队 5 人每人每天节省 2 小时，一年能节省多少工时，然后生成一份简要报告。"},
        config={"callbacks": [callback]}
    )

    print("\n" + "=" * 50)
    print("最终输出:", result["output"])
    print(f"\n中间步骤数: {len(result['intermediate_steps'])}")
```

运行效果：

```
  📍 Step 1: 使用 web_search
  📍 Step 2: 使用 calculator
  📍 Step 3: 使用 save_report
  ✅ 完成！共执行 3 步

==================================================
最终输出: 研究报告已生成并保存。LangChain 是最流行的 LLM 应用框架...
5 人团队每年可节省 3,650 小时工时（5人 × 2小时 × 365天）。
```

## 最佳实践

### 工具设计原则

| 原则 | 说明 | 示例 |
|------|------|------|
| 单一职责 | 每个工具只做一件事 | ❌ `search_and_save` → ✅ `search` + `save` |
| 描述精确 | 描述要告诉 LLM 何时用、怎么用 | 包含用途、时机、输入格式、限制 |
| 容错设计 | 工具内部处理异常，返回有意义的错误信息 | `return f"错误: {e}，请尝试..."` |
| 输出简洁 | 返回 LLM 能理解的简洁文本 | 避免返回巨大 JSON，做好摘要 |
| 参数明确 | 用 Pydantic Field 描述每个参数 | `Field(description="城市名称，如'北京'")` |

### Agent 设计原则

```python
# 1. 工具数量控制在 3-8 个（太多会降低选择准确率）
tools = [search, calculator, save_file]  # ✅ 精简

# 2. System Prompt 明确工作流程
system_prompt = """你是 XX 助手。工作流程：
1. 先理解用户需求
2. 必要时搜索信息
3. 直接回答，不要过度使用工具"""

# 3. 设置合理的安全边界
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=8,          # 防止无限循环
    max_execution_time=120,    # 2 分钟超时
    handle_parsing_errors=True,# 自动恢复
)

# 4. 生产环境必须加入错误处理
try:
    result = agent_executor.invoke({"input": user_input})
except Exception as e:
    result = {"output": f"抱歉，处理过程中出现错误: {str(e)}"}
```

### 性能优化

```python
# 1. 使用缓存避免重复调用
from langchain.cache import InMemoryCache
from langchain.globals import set_llm_cache
set_llm_cache(InMemoryCache())

# 2. 异步并发执行（多个独立工具调用）
import asyncio

async def run_tools_parallel(tools_with_inputs):
    tasks = [tool.ainvoke(inp) for tool, inp in tools_with_inputs]
    return await asyncio.gather(*tasks)

# 3. 选择合适的模型
# - 简单任务：gpt-4o-mini（快、便宜）
# - 复杂推理：gpt-4o（准确）
# - 超长上下文：claude-3.5-sonnet

# 4. 工具返回值精简
@tool
def search_db(query: str) -> str:
    """搜索数据库"""
    results = db.query(query)  # 可能返回大量数据
    # ✅ 只返回摘要，不要返回原始大数据
    return f"找到 {len(results)} 条结果:\n" + "\n".join(
        f"- {r['title']}" for r in results[:5]
    )
```

### 安全注意事项

1. **代码执行工具**：必须在沙箱中运行，限制文件系统和网络访问
2. **用户输入**：对传入工具的参数做校验和清洗
3. **API Key**：使用环境变量，不要硬编码
4. **权限控制**：不同用户可访问不同工具集
5. **日志审计**：记录所有工具调用，便于追溯

## 面试题精选

### 题目 1：Tool 和 Chain 的区别是什么？

**答案：**

| 维度 | Tool | Chain |
|------|------|-------|
| 定义 | 单个可调用的功能单元 | 多个组件的有序组合 |
| 调用方 | 由 Agent（LLM）动态决定是否调用 | 由开发者预定义调用顺序 |
| 灵活性 | 高，Agent 自主选择 | 低，固定流程 |
| 适用场景 | 不确定需要哪些步骤 | 步骤明确固定 |

Tool 是 Agent 的"手脚"，Chain 是预编排的"流水线"。Agent 通过推理决定用哪个 Tool，而 Chain 的执行顺序在编码时就确定了。

### 题目 2：如何让 Agent 更准确地选择工具？

**答案：**

1. **优化工具描述**：描述中明确说明用途、触发条件、输入格式
2. **减少工具数量**：控制在 3-8 个，避免选择困难
3. **工具名称语义化**：`calculate_math` 比 `tool_1` 好
4. **System Prompt 引导**：在 prompt 中说明工具使用策略
5. **使用 Function Calling**：比 ReAct 文本解析更准确
6. **Few-shot 示例**：在 prompt 中给出工具使用示例

```python
# 好的工具描述示例
@tool
def search_company_docs(query: str) -> str:
    """在公司内部知识库中搜索文档。
    使用场景：当用户询问公司政策、产品文档、内部流程时使用。
    不适用：通用知识问题（用 web_search 代替）。
    输入：搜索关键词，建议 2-5 个词。"""
    ...
```

### 题目 3：Agent 陷入无限循环怎么办？

**答案：**

无限循环的常见原因和解决方案：

| 原因 | 解决方案 |
|------|---------|
| 工具返回信息不足，Agent 反复尝试 | 改进工具返回值，提供更完整的信息 |
| 工具描述不清，Agent 不知道何时停止 | 在描述中加入"如果结果足够就停止" |
| LLM 能力不足，无法正确推理 | 换用更强的模型（如 gpt-4o） |
| Prompt 缺少终止引导 | 加入"如果已有足够信息，直接回答" |

防御措施：
```python
agent_executor = AgentExecutor(
    agent=agent, tools=tools,
    max_iterations=8,           # 硬性上限
    max_execution_time=60,      # 时间上限
    early_stopping_method="generate",  # 超限时让 LLM 总结
)
```

### 题目 4：ReAct Agent 和 Function Calling Agent 的区别？

**答案：**

| 维度 | ReAct Agent | Function Calling Agent |
|------|-------------|----------------------|
| 实现方式 | 文本解析（正则匹配 Thought/Action） | LLM 原生 API（结构化 JSON） |
| 准确性 | 较低，依赖 LLM 遵循格式 | 高，LLM 原生支持 |
| 兼容性 | 任何 LLM 都可用 | 需要支持 Function Calling 的 LLM |
| 并行调用 | 不支持 | 支持（一次调用多个工具） |
| 调试难度 | 较高（格式错误常见） | 低 |
| 推荐度 | 兼容场景 | ⭐ 首选方案 |

**结论**：如果使用的 LLM 支持 Function Calling（OpenAI、Anthropic、Google 等），优先使用 `create_tool_calling_agent`。

### 题目 5：如何在生产环境中监控 Agent？

**答案：**

生产环境 Agent 监控的关键维度：

1. **可观测性**：使用 LangSmith 或自建追踪系统
   - 记录每次 LLM 调用的输入输出
   - 记录工具调用链路和耗时
   - 记录 Token 消耗和成本

2. **Callback 埋点**：
```python
class ProductionCallback(BaseCallbackHandler):
    def on_agent_action(self, action, **kwargs):
        metrics.increment("agent.tool_call", tags={"tool": action.tool})

    def on_agent_finish(self, finish, **kwargs):
        metrics.timing("agent.duration", time.time() - self.start_time)

    def on_tool_error(self, error, **kwargs):
        alerting.send(f"工具执行失败: {error}")
```

3. **关键指标**：
   - 平均迭代次数（正常应 < 5）
   - 工具调用成功率
   - 端到端延迟
   - Token 成本 / 请求
   - 超时和错误率

4. **告警规则**：
   - 单次请求迭代 > 8 次 → 告警
   - 工具错误率 > 5% → 告警
   - 平均延迟 > 30s → 告警

### 题目 6：Callback 系统的设计模式是什么？有哪些应用场景？

**答案：**

LangChain 的 Callback 系统采用**观察者模式（Observer Pattern）**：

- **Subject**：LLM、Tool、Chain、Agent 等组件
- **Observer**：各种 CallbackHandler
- **事件**：on_llm_start、on_tool_end、on_agent_action 等

应用场景：

| 场景 | 实现方式 |
|------|---------|
| 日志记录 | 自定义 Handler 写入日志系统 |
| 流式输出 | `on_llm_new_token` 逐 token 推送前端 |
| 成本统计 | 累计 token 数量计算费用 |
| 性能监控 | 记录各环节耗时 |
| 错误告警 | `on_tool_error` 触发告警 |
| 用户反馈 | 记录中间步骤供用户查看 |

Callback 的传递方式：
```python
# 方式 1：构造时传入（全局生效）
llm = ChatOpenAI(callbacks=[handler])

# 方式 2：运行时传入（单次生效，推荐）
result = agent_executor.invoke(
    {"input": "..."},
    config={"callbacks": [handler]}
)
```

### 题目 7：如何设计一个可靠的多工具 Agent 系统？

**答案：**

设计要点：

1. **工具分层**：
   - 信息获取层：搜索、数据库查询
   - 计算处理层：数学计算、数据分析
   - 行动执行层：发邮件、写文件、调 API

2. **错误恢复**：每个工具内部 try-catch，返回有意义的错误信息让 Agent 可以调整策略

3. **权限隔离**：根据用户角色动态加载工具集

4. **幂等设计**：写操作工具要支持幂等，防止重复执行

5. **超时熔断**：设置合理的 max_iterations 和 max_execution_time

```python
# 生产级 Agent 架构
def create_production_agent(user_role: str):
    # 根据角色加载工具
    tools = load_tools_for_role(user_role)

    agent = create_tool_calling_agent(llm, tools, prompt)

    return AgentExecutor(
        agent=agent,
        tools=tools,
        max_iterations=8,
        max_execution_time=120,
        handle_parsing_errors=True,
        callbacks=[MonitoringCallback(), CostTrackingCallback()],
    )
```

---

> 📚 **延伸阅读**
> - [LangChain 官方文档 - Agents](https://python.langchain.com/docs/concepts/agents/)
> - [ReAct 论文](https://arxiv.org/abs/2210.03629)
> - [LangSmith 文档](https://docs.smith.langchain.com/)
> - [LangChain Hub - Prompt 模板](https://smith.langchain.com/hub)

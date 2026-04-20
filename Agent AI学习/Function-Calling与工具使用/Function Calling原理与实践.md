# Function Calling 原理与实践

## 1. 什么是 Function Calling

Function Calling 是 LLM 根据用户意图，自动决定调用哪个函数、传什么参数的能力。它是 Agent 使用工具的基础。

````
传统方式: 用户 → 开发者写 if/else 判断意图 → 调用函数
FC 方式:  用户 → LLM 自动识别意图和参数 → 调用函数
````
### 工作流程

````
1. 开发者定义可用函数（名称、描述、参数 Schema）
2. 用户发送消息
3. LLM 分析消息，决定是否需要调用函数
4. 如果需要，LLM 输出函数名和参数（JSON）
5. 开发者执行函数，获取结果
6. 将结果返回给 LLM
7. LLM 基于结果生成最终回答
````
## 2. OpenAI Function Calling

### 2.1 基础用法

````python
from openai import OpenAI

client = OpenAI()

# 定义工具
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的当前天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如 '北京'、'上海'"
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "温度单位"
                    }
                },
                "required": ["city"]
            }
        }
    }
]

# 发送请求
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "北京今天天气怎么样?"}],
    tools=tools,
    tool_choice="auto"  # auto/none/required/指定函数
)

message = response.choices[0].message

# 检查是否需要调用函数
if message.tool_calls:
    for tool_call in message.tool_calls:
        print(f"函数: {tool_call.function.name}")
        print(f"参数: {tool_call.function.arguments}")
        # 输出: 函数: get_weather, 参数: {"city": "北京"}
````
### 2.2 完整调用流程

````python
import json

def get_weather(city: str, unit: str = "celsius") -> str:
    """实际的天气查询函数"""
    # 模拟 API 调用
    return json.dumps({"city": city, "temp": 25, "condition": "晴"})

def run_conversation(user_message: str):
    messages = [{"role": "user", "content": user_message}]
    
    # 第一次调用: LLM 决定是否调用函数
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools
    )
    
    message = response.choices[0].message
    
    if not message.tool_calls:
        return message.content  # 不需要调用函数，直接返回
    
    # 执行函数调用
    messages.append(message)  # 添加 assistant 的回复
    
    for tool_call in message.tool_calls:
        func_name = tool_call.function.name
        func_args = json.loads(tool_call.function.arguments)
        
        # 调用实际函数
        result = get_weather(**func_args)
        
        # 将函数结果添加到消息
        messages.append({
            "role": "tool",
            "tool_call_id": tool_call.id,
            "content": result
        })
    
    # 第二次调用: LLM 基于函数结果生成回答
    final_response = client.chat.completions.create(
        model="gpt-4o",
        messages=messages,
        tools=tools
    )
    
    return final_response.choices[0].message.content

print(run_conversation("北京和上海今天哪个更热?"))
````
### 2.3 并行函数调用

LLM 可以一次返回多个函数调用：

````python
# 用户: "北京和上海今天哪个更热?"
# LLM 会返回两个 tool_calls:
#   1. get_weather(city="北京")
#   2. get_weather(city="上海")
# 可以并行执行这两个调用
````
### 2.4 tool_choice 控制

````python
# auto: LLM 自动决定是否调用（默认）
tool_choice="auto"

# none: 禁止调用函数
tool_choice="none"

# required: 必须调用某个函数
tool_choice="required"

# 指定调用某个函数
tool_choice={"type": "function", "function": {"name": "get_weather"}}
````
## 3. 函数定义最佳实践

### 3.1 描述要精确

````python
# ❌ 差的描述
{
    "name": "search",
    "description": "搜索",
    "parameters": {"query": {"type": "string"}}
}

# ✅ 好的描述
{
    "name": "search_knowledge_base",
    "description": "在内部知识库中搜索技术文档。适用于查找项目文档、API 文档、最佳实践等内部资料。不适用于搜索互联网信息。",
    "parameters": {
        "query": {
            "type": "string",
            "description": "搜索关键词，建议使用具体的技术术语"
        },
        "category": {
            "type": "string",
            "enum": ["api", "guide", "faq", "architecture"],
            "description": "文档类别，用于缩小搜索范围"
        }
    }
}
````
### 3.2 参数设计原则

````
1. 必填参数要少 — 降低 LLM 出错概率
2. 用 enum 约束取值 — 避免无效参数
3. description 要具体 — LLM 靠描述理解参数含义
4. 提供默认值 — 减少 LLM 需要推断的信息
5. 参数名要语义化 — city 比 c 好
````
### 3.3 错误处理

````python
def safe_tool_call(func, **kwargs):
    """安全的工具调用包装"""
    try:
        result = func(**kwargs)
        return {"status": "success", "data": result}
    except ValueError as e:
        return {"status": "error", "message": f"参数错误: {e}"}
    except TimeoutError:
        return {"status": "error", "message": "调用超时，请稍后重试"}
    except Exception as e:
        return {"status": "error", "message": f"未知错误: {e}"}
````
## 4. 自定义工具开发

### 4.1 LangChain 工具

````python
from langchain_core.tools import tool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    max_results: int = Field(default=5, description="最大结果数")

@tool(args_schema=SearchInput)
def search_docs(query: str, max_results: int = 5) -> str:
    """在知识库中搜索文档。用于查找技术文档和最佳实践。"""
    # 实现搜索逻辑
    results = vector_store.similarity_search(query, k=max_results)
    return "\n".join([r.page_content for r in results])
````
### 4.2 异步工具

````python
@tool
async def fetch_url(url: str) -> str:
    """获取网页内容"""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.text()
````
### 4.3 工具组合

````python
# 将多个工具组合为一个工具包
class DatabaseToolkit:
    def __init__(self, db_url: str):
        self.engine = create_engine(db_url)
    
    @tool
    def query_db(self, sql: str) -> str:
        """执行 SQL 查询"""
        return pd.read_sql(sql, self.engine).to_string()
    
    @tool
    def list_tables(self) -> str:
        """列出所有数据表"""
        return str(inspect(self.engine).get_table_names())
    
    def get_tools(self):
        return [self.query_db, self.list_tables]
````
## 5. MCP（Model Context Protocol）

### 5.1 什么是 MCP

MCP 是 Anthropic 提出的开放协议，标准化了 LLM 应用与外部工具/数据源的连接方式。

````
传统方式: 每个 LLM 框架有自己的工具定义格式
MCP 方式: 统一的协议，工具一次开发，到处使用

类比: MCP 之于 AI 工具 = USB 之于外设
````
### 5.2 MCP 架构

````
┌──────────┐     MCP 协议     ┌──────────────┐
│ MCP Host │ ◄──────────────► │  MCP Server  │
│ (AI App) │                  │  (工具提供方)  │
└──────────┘                  └──────────────┘
     │                              │
  LLM 应用                    工具/数据源
  (Claude, etc.)              (DB, API, FS...)
````
### 5.3 MCP Server 开发

````python
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("my-tools")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="query_database",
            description="查询数据库",
            inputSchema={
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "SQL 查询语句"}
                },
                "required": ["sql"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "query_database":
        result = execute_sql(arguments["sql"])
        return [TextContent(type="text", text=str(result))]

# 运行 MCP Server
if __name__ == "__main__":
    import asyncio
    from mcp.server.stdio import stdio_server
    asyncio.run(stdio_server(server))
````
## 6. 安全考虑

````
1. 输入验证 — 不信任 LLM 生成的参数
2. 权限控制 — 工具应有最小权限
3. 速率限制 — 防止 Agent 过度调用
4. 审计日志 — 记录所有工具调用
5. 沙箱执行 — 代码执行类工具必须隔离
6. 敏感操作确认 — 删除/修改操作需人工确认
````
---

## 面试题精选

### Q1: Function Calling 的完整工作流程是怎样的？
**答：** 开发者定义函数 Schema（名称、描述、参数）→ 用户发消息 → LLM 分析意图决定是否调用函数 → 输出函数名和参数 JSON → 开发者执行函数获取结果 → 结果返回给 LLM → LLM 基于结果生成最终回答。至少需要两次 LLM 调用。

### Q2: 函数描述（description）为什么对 Function Calling 至关重要？
**答：** LLM 完全依赖 description 来理解函数的用途和使用时机。描述不精确会导致该调用时不调用、不该调用时误调用、参数生成错误。好的描述应说明功能、适用场景和不适用场景。

### Q3: tool_choice 的几种模式分别是什么？什么时候用哪种？
**答：** auto（LLM 自动决定，默认）、none（禁止调用）、required（必须调用某个函数）、指定函数名（强制调用特定函数）。大多数场景用 auto，需要强制执行某操作时用 required 或指定函数。

### Q4: MCP（Model Context Protocol）是什么？它解决了什么问题？
**答：** MCP 是 Anthropic 提出的开放协议，标准化了 LLM 应用与外部工具的连接方式。它解决了每个框架各自定义工具格式的碎片化问题，类似 USB 之于外设——工具一次开发，到处使用。

### Q5: Function Calling 的安全风险有哪些？如何防护？
**答：** 风险包括：LLM 生成恶意参数（如 SQL 注入）、过度调用导致资源耗尽、敏感操作未经确认。防护措施：输入验证不信任 LLM 参数、工具最小权限、速率限制、敏感操作需人工确认、代码执行类工具必须沙箱隔离。

### Q6: 并行函数调用是什么？有什么好处？
**答：** LLM 可以一次返回多个 tool_calls（如同时查北京和上海天气），开发者可以并行执行这些调用再一起返回结果。好处是减少 LLM 调用轮次、降低延迟，适合多个独立查询的场景。

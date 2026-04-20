# LangChain Model I/O 详解

> 本文面向有 Python 基础但未接触过 LangChain 的开发者，系统讲解 Model I/O 模块的核心概念与实战用法。

## Model I/O 概述

### 什么是 Model I/O

Model I/O 是 LangChain 中与大语言模型（LLM）交互的核心抽象层。它解决的核心问题是：**如何优雅地构造输入、调用模型、解析输出**。

在 LangChain 的整体架构中，Model I/O 处于最基础的位置：

```
┌─────────────────────────────────────────────────────┐
│                   LangChain 应用层                     │
│         (Agents / Chains / RAG Pipeline)             │
├─────────────────────────────────────────────────────┤
│                   Model I/O 层                        │
│  ┌──────────────┐  ┌───────────┐  ┌──────────────┐  │
│  │Prompt Template│  │ Chat Model │  │Output Parser │  │
│  │  (构造输入)    │  │ (调用模型)  │  │ (解析输出)   │  │
│  └──────┬───────┘  └─────┬─────┘  └──────┬───────┘  │
│         │                │                │          │
│         └────────────────┼────────────────┘          │
│                          │                           │
│              prompt | llm | parser                    │
├─────────────────────────────────────────────────────┤
│              基础设施层 (LangChain Core)               │
│      (Runnable Protocol / Callback / Serialization)  │
└─────────────────────────────────────────────────────┘
```

### 三大组件的职责

| 组件 | 职责 | 输入 | 输出 |
|------|------|------|------|
| Prompt Template | 将用户变量填充为结构化的消息列表 | `dict` | `ChatPromptValue` |
| Chat Model | 调用 LLM API 获取响应 | `Messages` | `AIMessage` |
| Output Parser | 将模型原始输出转换为结构化数据 | `AIMessage` | `str / dict / Pydantic Model` |

三者通过 LCEL（LangChain Expression Language）的管道操作符 `|` 串联：

```python
chain = prompt | llm | parser
result = chain.invoke({"topic": "AI"})
```

### 安装依赖

```bash
pip install langchain langchain-openai langchain-anthropic langchain-community
```

---

## Chat Model

Chat Model 是 Model I/O 的核心，负责与大语言模型进行实际通信。LangChain 中所有 Chat Model 都继承自 `BaseChatModel`，提供统一的调用接口。

### 初始化与配置

#### 基础初始化

```python
from langchain_openai import ChatOpenAI

# 最简初始化（使用环境变量 OPENAI_API_KEY）
llm = ChatOpenAI()

# 完整配置
llm = ChatOpenAI(
    model="gpt-4o",           # 模型名称
    temperature=0.7,          # 创造性：0=确定性，1=高随机性
    max_tokens=2048,          # 最大输出 token 数
    timeout=30,               # 请求超时（秒）
    max_retries=2,            # 失败重试次数
    api_key="sk-xxx",         # API Key（推荐用环境变量）
    base_url="https://api.openai.com/v1",  # 自定义端点
)
```

#### 关键参数说明

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `model` | str | `"gpt-3.5-turbo"` | 模型标识符 |
| `temperature` | float | `0.7` | 控制输出随机性，0~2 |
| `max_tokens` | int | None | 限制输出长度，None 表示模型默认 |
| `top_p` | float | `1.0` | 核采样参数，与 temperature 二选一调节 |
| `frequency_penalty` | float | `0.0` | 降低重复词频率，-2~2 |
| `presence_penalty` | float | `0.0` | 鼓励新话题，-2~2 |
| `stop` | list | None | 停止生成的标记列表 |

#### temperature 选择指南

```
temperature = 0.0  → 代码生成、数据提取、事实问答（需要确定性）
temperature = 0.3  → 摘要、翻译（需要一致性但允许微调）
temperature = 0.7  → 通用对话、创意写作（平衡创造性和连贯性）
temperature = 1.0+ → 头脑风暴、诗歌创作（高度创造性）
```

### 基础调用方式

LangChain 的所有 Runnable（包括 Chat Model）都支持以下调用接口：

#### invoke — 单次同步调用

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 直接传字符串
response = llm.invoke("用一句话解释什么是机器学习")
print(response.content)
# 输出: 机器学习是让计算机通过数据自动学习规律并做出预测的技术。

# 传消息列表
response = llm.invoke([HumanMessage(content="你好")])
print(response.content)
```

#### stream — 流式输出

```python
llm = ChatOpenAI(model="gpt-4o", temperature=0)

for chunk in llm.stream("写一首关于春天的五言绝句"):
    print(chunk.content, end="", flush=True)
# 逐字输出，适合实时展示
```

#### batch — 批量调用

```python
llm = ChatOpenAI(model="gpt-4o", temperature=0)

questions = [
    "Python 的 GIL 是什么？",
    "什么是协程？",
    "解释 SOLID 原则",
]

# 并发调用，提高吞吐量
responses = llm.batch(questions, config={"max_concurrency": 3})
for q, r in zip(questions, responses):
    print(f"Q: {q}\nA: {r.content}\n")
```

#### ainvoke — 异步调用

```python
import asyncio
from langchain_openai import ChatOpenAI

async def main():
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    
    # 异步单次调用
    response = await llm.ainvoke("异步编程的优势是什么？")
    print(response.content)
    
    # 异步流式
    async for chunk in llm.astream("讲个笑话"):
        print(chunk.content, end="", flush=True)

asyncio.run(main())
```

### 多模型支持

LangChain 的核心优势之一是统一接口，切换模型只需更换类名：

#### OpenAI

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o", temperature=0)
response = llm.invoke("Hello!")
```

#### Anthropic (Claude)

```python
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(model="claude-3-5-sonnet-20241022", temperature=0)
response = llm.invoke("Hello!")
```

#### 本地模型（Ollama）

```python
from langchain_community.chat_models import ChatOllama

# 需要先安装并运行 Ollama：ollama run llama3
llm = ChatOllama(model="llama3", temperature=0)
response = llm.invoke("Hello!")
```

#### 统一接口的威力

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

def ask_question(llm, question: str) -> str:
    """无论底层模型是什么，调用方式完全一致"""
    response = llm.invoke(question)
    return response.content

# 同一函数，不同模型
openai_llm = ChatOpenAI(model="gpt-4o")
claude_llm = ChatAnthropic(model="claude-3-5-sonnet-20241022")

print(ask_question(openai_llm, "1+1=?"))
print(ask_question(claude_llm, "1+1=?"))
```

### Message 类型详解

Chat Model 的输入输出都是 Message 对象。LangChain 定义了以下核心消息类型：

```
┌─────────────────────────────────────────────┐
│              Message 类型体系                 │
├─────────────────────────────────────────────┤
│  SystemMessage   → 系统指令，设定角色和规则    │
│  HumanMessage    → 用户输入                  │
│  AIMessage       → 模型响应                  │
│  ToolMessage     → 工具调用结果              │
│  FunctionMessage → (已废弃，用 ToolMessage)   │
├─────────────────────────────────────────────┤
│  每条消息包含:                                │
│    - content: str | list  (文本或多模态内容)  │
│    - additional_kwargs: dict (额外元数据)     │
│    - response_metadata: dict (响应元信息)     │
└─────────────────────────────────────────────┘
```

#### 完整示例

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import (
    SystemMessage,
    HumanMessage,
    AIMessage,
    ToolMessage,
)

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 多轮对话
messages = [
    SystemMessage(content="你是一个专业的Python导师，回答简洁明了。"),
    HumanMessage(content="什么是装饰器？"),
    AIMessage(content="装饰器是一个接收函数并返回新函数的高阶函数，用于在不修改原函数代码的情况下扩展其功能。"),
    HumanMessage(content="给我一个最简单的例子"),
]

response = llm.invoke(messages)
print(response.content)
```

#### ToolMessage 用法（Function Calling 场景）

```python
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage

# 模拟工具调用流程
messages = [
    HumanMessage(content="北京今天天气怎么样？"),
    # AI 决定调用工具
    AIMessage(
        content="",
        tool_calls=[{
            "id": "call_abc123",
            "name": "get_weather",
            "args": {"city": "北京"}
        }]
    ),
    # 工具返回结果
    ToolMessage(
        content='{"temp": 22, "condition": "晴"}',
        tool_call_id="call_abc123"  # 必须与 tool_calls 中的 id 对应
    ),
]

llm = ChatOpenAI(model="gpt-4o", temperature=0)
response = llm.invoke(messages)
print(response.content)
# 输出: 北京今天天气晴朗，气温22°C。
```

#### 多模态消息（图片输入）

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage

llm = ChatOpenAI(model="gpt-4o", temperature=0)

message = HumanMessage(
    content=[
        {"type": "text", "text": "描述这张图片的内容"},
        {"type": "image_url", "image_url": {"url": "https://example.com/cat.jpg"}},
    ]
)

response = llm.invoke([message])
print(response.content)
```

---

## Prompt Template

Prompt Template 负责将用户的变量输入转换为结构化的消息列表，是 Model I/O 的"输入构造器"。

### 为什么需要 Prompt Template

直接拼接字符串的问题：

```python
# ❌ 不推荐：硬编码、难复用、易出错
prompt = f"你是{role}，请用{style}风格回答：{question}"
```

使用 Prompt Template 的优势：
- **可复用**：同一模板可填充不同变量
- **可组合**：多个模板可以嵌套组合
- **类型安全**：自动校验变量是否齐全
- **序列化**：可保存为文件、从文件加载

### ChatPromptTemplate 基础用法

```python
from langchain_core.prompts import ChatPromptTemplate

# 方式一：from_messages（推荐，最灵活）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{domain}领域的专家，回答风格{style}。"),
    ("human", "{question}"),
])

# 填充变量，生成消息列表
messages = prompt.invoke({
    "domain": "机器学习",
    "style": "简洁",
    "question": "什么是过拟合？"
})
print(messages.to_messages())
# [SystemMessage(content='你是一个机器学习领域的专家，回答风格简洁。'),
#  HumanMessage(content='什么是过拟合？')]
```

```python
# 方式二：from_template（简单场景）
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template(
    "将以下文本翻译成{language}：\n\n{text}"
)

messages = prompt.invoke({"language": "英文", "text": "你好世界"})
print(messages.to_messages())
# [HumanMessage(content='将以下文本翻译成英文：\n\n你好世界')]
```

#### 查看模板变量

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是{role}"),
    ("human", "{question}"),
])

print(prompt.input_variables)  # ['role', 'question']
```

### MessagesPlaceholder 动态消息

`MessagesPlaceholder` 允许在模板中插入动态数量的消息，常用于：
- 多轮对话历史
- 动态 few-shot 示例
- Agent 的中间步骤

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的助手。"),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{input}"),
])

# 模拟多轮对话
messages = prompt.invoke({
    "chat_history": [
        HumanMessage(content="我叫张三"),
        AIMessage(content="你好张三！有什么可以帮你的？"),
    ],
    "input": "我叫什么名字？"
})

for msg in messages.to_messages():
    print(f"{msg.__class__.__name__}: {msg.content}")
# SystemMessage: 你是一个有帮助的助手。
# HumanMessage: 我叫张三
# AIMessage: 你好张三！有什么可以帮你的？
# HumanMessage: 我叫什么名字？
```

#### optional 参数

```python
# 当 chat_history 可能为空时，设置 optional=True
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是助手。"),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
])

# 不传 chat_history 也不会报错
messages = prompt.invoke({"input": "你好"})
```

### FewShotChatMessagePromptTemplate 少样本提示

Few-shot prompting 通过提供示例来引导模型输出格式和风格：

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    FewShotChatMessagePromptTemplate,
)

# 定义示例
examples = [
    {"input": "happy", "output": "sad"},
    {"input": "tall", "output": "short"},
    {"input": "sunny", "output": "cloudy"},
]

# 定义每个示例的格式模板
example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

# 构建 few-shot 模板
few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)

# 组合到完整 prompt
final_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个反义词生成器。用户给出一个词，你给出它的反义词。"),
    few_shot_prompt,
    ("human", "{input}"),
])

messages = final_prompt.invoke({"input": "big"})
for msg in messages.to_messages():
    print(f"{msg.__class__.__name__}: {msg.content}")
```

#### 动态选择示例（基于相似度）

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    FewShotChatMessagePromptTemplate,
)
from langchain_core.example_selectors import SemanticSimilarityExampleSelector
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

examples = [
    {"input": "happy", "output": "sad"},
    {"input": "tall", "output": "short"},
    {"input": "energetic", "output": "lethargic"},
    {"input": "sunny", "output": "gloomy"},
    {"input": "windy", "output": "calm"},
]

# 基于语义相似度选择最相关的示例
example_selector = SemanticSimilarityExampleSelector.from_examples(
    examples,
    OpenAIEmbeddings(),
    FAISS,
    k=2,  # 选择 2 个最相似的示例
)

example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    example_selector=example_selector,
)

final_prompt = ChatPromptTemplate.from_messages([
    ("system", "给出反义词。"),
    few_shot_prompt,
    ("human", "{input}"),
])

# "warm" 会自动匹配到与天气/温度相关的示例
messages = final_prompt.invoke({"input": "warm"})
```

### 从文件加载模板、模板组合与复用

#### 从 YAML 文件加载

```yaml
# prompts/translator.yaml
_type: prompt
input_variables:
  - language
  - text
template: "将以下内容翻译成{language}：\n\n{text}"
```

```python
from langchain_core.prompts import load_prompt

prompt = load_prompt("prompts/translator.yaml")
result = prompt.invoke({"language": "日语", "text": "你好世界"})
```

#### 模板组合（Pipeline）

```python
from langchain_core.prompts import ChatPromptTemplate, PipelinePromptTemplate

# 子模板：角色设定
role_template = ChatPromptTemplate.from_template(
    "你是一个{role}，擅长{skill}。"
)

# 子模板：输出格式
format_template = ChatPromptTemplate.from_template(
    "请按以下格式输出：{format_instructions}"
)

# 主模板
full_template = ChatPromptTemplate.from_template(
    "{role_prompt}\n{format_prompt}\n\n用户问题：{question}"
)

# 组合
pipeline_prompt = PipelinePromptTemplate(
    final_prompt=full_template,
    pipeline_prompts=[
        ("role_prompt", role_template),
        ("format_prompt", format_template),
    ],
)

result = pipeline_prompt.invoke({
    "role": "数据分析师",
    "skill": "SQL和Python",
    "format_instructions": "先给结论，再给分析过程",
    "question": "如何优化慢查询？",
})
```

### Partial Prompt（部分填充）

当某些变量在创建时就已知，可以提前填充：

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是{company}的客服，今天是{date}。"),
    ("human", "{question}"),
])

# 部分填充：提前绑定 company
partial_prompt = prompt.partial(company="小米")

# 后续只需传剩余变量
messages = partial_prompt.invoke({
    "date": "2024-12-01",
    "question": "退货政策是什么？"
})
```

#### 使用函数动态填充

```python
from datetime import datetime
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "当前时间：{current_time}。你是助手。"),
    ("human", "{question}"),
])

# 用函数动态生成值
partial_prompt = prompt.partial(
    current_time=lambda: datetime.now().strftime("%Y-%m-%d %H:%M")
)

messages = partial_prompt.invoke({"question": "现在几点？"})
```

---

## Output Parser

Output Parser 负责将模型的原始文本输出转换为程序可用的结构化数据。它是 Model I/O 的"输出解析器"。

```
模型原始输出 (str)
       │
       ▼
┌─────────────────┐
│  Output Parser  │
├─────────────────┤
│ • 格式指令注入   │ ← get_format_instructions()
│ • 文本解析      │ ← parse(text)
│ • 错误修复      │ ← (OutputFixingParser)
└────────┬────────┘
         │
         ▼
结构化数据 (str / dict / Pydantic Model)
```

### StrOutputParser — 字符串输出

最简单的 Parser，将 `AIMessage` 转换为纯字符串：

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template("用一句话解释{concept}")
llm = ChatOpenAI(model="gpt-4o", temperature=0)
parser = StrOutputParser()

# 组成链
chain = prompt | llm | parser

# 返回 str 而非 AIMessage
result = chain.invoke({"concept": "递归"})
print(type(result))  # <class 'str'>
print(result)        # "递归是函数调用自身来解决问题的编程技巧。"
```

### JsonOutputParser — JSON 结构化输出

将模型输出解析为 Python 字典：

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.prompts import ChatPromptTemplate

parser = JsonOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个信息提取助手。{format_instructions}"),
    ("human", "从以下文本中提取人名和年龄：{text}"),
])

# 注入格式指令
prompt = prompt.partial(format_instructions=parser.get_format_instructions())

llm = ChatOpenAI(model="gpt-4o", temperature=0)
chain = prompt | llm | parser

result = chain.invoke({"text": "张三今年25岁，他的朋友李四28岁。"})
print(result)
# {'people': [{'name': '张三', 'age': 25}, {'name': '李四', 'age': 28}]}
print(type(result))  # <class 'dict'>
```

#### 指定 JSON Schema（使用 Pydantic）

```python
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field

class Person(BaseModel):
    name: str = Field(description="人名")
    age: int = Field(description="年龄")

class PeopleList(BaseModel):
    people: list[Person] = Field(description="人员列表")

parser = JsonOutputParser(pydantic_object=PeopleList)
print(parser.get_format_instructions())
# 会生成包含 JSON Schema 的格式指令
```

### PydanticOutputParser — Pydantic 模型验证

比 JsonOutputParser 更强大，直接返回 Pydantic 对象，带类型验证：

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

# 定义输出结构
class MovieReview(BaseModel):
    title: str = Field(description="电影名称")
    rating: float = Field(description="评分，1-10")
    summary: str = Field(description="一句话评价")
    recommend: bool = Field(description="是否推荐")

parser = PydanticOutputParser(pydantic_object=MovieReview)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是电影评论家。根据用户描述生成结构化评价。\n{format_instructions}"),
    ("human", "{movie_description}"),
])

prompt = prompt.partial(format_instructions=parser.get_format_instructions())

llm = ChatOpenAI(model="gpt-4o", temperature=0)
chain = prompt | llm | parser

result = chain.invoke({
    "movie_description": "一部关于盗梦的科幻电影，诺兰导演，莱昂纳多主演"
})

print(type(result))       # <class 'MovieReview'>
print(result.title)       # "盗梦空间"
print(result.rating)      # 9.2
print(result.recommend)   # True
```

### 自定义 Parser

继承 `BaseOutputParser` 实现自定义解析逻辑：

```python
from langchain_core.output_parsers import BaseOutputParser
from typing import List

class CommaSeparatedListParser(BaseOutputParser[List[str]]):
    """将逗号分隔的文本解析为列表"""
    
    @property
    def _type(self) -> str:
        return "comma_separated_list"
    
    def get_format_instructions(self) -> str:
        return "请用逗号分隔的格式输出，例如：item1, item2, item3"
    
    def parse(self, text: str) -> List[str]:
        return [item.strip() for item in text.split(",")]

# 使用
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

parser = CommaSeparatedListParser()

prompt = ChatPromptTemplate.from_template(
    "列出5种{category}。{format_instructions}"
)
prompt = prompt.partial(format_instructions=parser.get_format_instructions())

llm = ChatOpenAI(model="gpt-4o", temperature=0)
chain = prompt | llm | parser

result = chain.invoke({"category": "编程语言"})
print(result)  # ['Python', 'JavaScript', 'Java', 'Go', 'Rust']
```

#### 使用 RunnableLambda 快速自定义

```python
from langchain_core.runnables import RunnableLambda

# 更轻量的方式：用 lambda 做简单转换
upper_parser = RunnableLambda(lambda x: x.content.upper())

chain = prompt | llm | upper_parser
```

### 输出修复（OutputFixingParser）

当模型输出格式不符合预期时，`OutputFixingParser` 会自动调用 LLM 修复：

```python
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import PydanticOutputParser
from langchain.output_parsers import OutputFixingParser
from pydantic import BaseModel, Field

class Action(BaseModel):
    action: str = Field(description="要执行的动作")
    action_input: str = Field(description="动作的输入参数")

# 基础 parser
base_parser = PydanticOutputParser(pydantic_object=Action)

# 包装为自动修复 parser
fixing_parser = OutputFixingParser.from_llm(
    parser=base_parser,
    llm=ChatOpenAI(model="gpt-4o", temperature=0),
)

# 即使模型输出格式有误，也能自动修复
bad_output = '{"action": "search", "action_input": }' # 无效 JSON
result = fixing_parser.parse(bad_output)
print(result)  # Action(action='search', action_input='')
```

#### RetryOutputParser — 带重试的修复

```python
from langchain.output_parsers import RetryOutputParser
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

retry_parser = RetryOutputParser.from_llm(
    parser=base_parser,
    llm=ChatOpenAI(model="gpt-4o", temperature=0),
    max_retries=3,  # 最多重试 3 次
)
```

---

## 三者组合实战

### 实战一：智能翻译器

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class TranslationResult(BaseModel):
    original: str = Field(description="原文")
    translated: str = Field(description="译文")
    language_from: str = Field(description="源语言")
    language_to: str = Field(description="目标语言")
    confidence: float = Field(description="翻译置信度，0-1")

parser = PydanticOutputParser(pydantic_object=TranslationResult)

prompt = ChatPromptTemplate.from_messages([
    ("system", """你是专业翻译。将用户输入翻译成{target_language}。
{format_instructions}"""),
    ("human", "{text}"),
])
prompt = prompt.partial(format_instructions=parser.get_format_instructions())

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 组装链
translation_chain = prompt | llm | parser

# 调用
result = translation_chain.invoke({
    "target_language": "英文",
    "text": "机器学习是人工智能的一个重要分支"
})

print(f"原文: {result.original}")
print(f"译文: {result.translated}")
print(f"置信度: {result.confidence}")
```

### 实战二：多步骤处理链

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.runnables import RunnablePassthrough

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 第一步：提取关键信息
extract_prompt = ChatPromptTemplate.from_template(
    "从以下文本中提取关键信息，以JSON格式输出（包含topic、key_points数组、sentiment字段）：\n\n{text}"
)

# 第二步：基于提取结果生成摘要
summary_prompt = ChatPromptTemplate.from_template(
    "基于以下结构化信息，生成一段50字以内的中文摘要：\n\n{extracted_info}"
)

# 组装多步骤链
extract_chain = extract_prompt | llm | StrOutputParser()
summary_chain = summary_prompt | llm | StrOutputParser()

# 串联
full_chain = (
    {"extracted_info": extract_chain, "text": RunnablePassthrough()}
    | summary_chain
)

result = full_chain.invoke({
    "text": """
    OpenAI 发布了 GPT-5.4 模型，支持文本、图像和音频的多模态输入。
    该模型在速度和成本上都有显著改进，响应延迟降低了50%。
    开发者社区对此反应积极，认为这将推动AI应用的普及。
    """
})
print(result)
```

### 实战三：带对话历史的问答系统

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.output_parsers import StrOutputParser
from langchain_core.messages import HumanMessage, AIMessage

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个知识渊博的助手。基于对话历史回答问题，如果不知道就说不知道。"),
    MessagesPlaceholder(variable_name="history"),
    ("human", "{question}"),
])

llm = ChatOpenAI(model="gpt-4o", temperature=0)
chain = prompt | llm | StrOutputParser()

# 模拟多轮对话
history = []

def chat(question: str) -> str:
    response = chain.invoke({"history": history, "question": question})
    # 更新历史
    history.append(HumanMessage(content=question))
    history.append(AIMessage(content=response))
    return response

print(chat("Python是什么时候发明的？"))
print(chat("它的创造者是谁？"))  # 能理解"它"指 Python
print(chat("他现在在哪个公司？"))  # 能理解"他"指创造者
```

### 实战四：流式输出 + 结构化解析

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser

prompt = ChatPromptTemplate.from_template(
    "生成3个关于{topic}的创意想法，以JSON数组格式输出，每个包含title和description字段。"
)

llm = ChatOpenAI(model="gpt-4o", temperature=0.7, streaming=True)
parser = JsonOutputParser()

chain = prompt | llm | parser

# 流式获取部分 JSON（JsonOutputParser 支持流式解析）
async def stream_ideas():
    async for chunk in chain.astream({"topic": "智能家居"}):
        print(chunk)  # 每次输出当前已解析的部分结果

import asyncio
asyncio.run(stream_ideas())
```

---

## 最佳实践与常见坑

### 最佳实践

#### 1. 始终使用 LCEL 管道语法

```python
# ✅ 推荐：声明式、可组合、自动支持 stream/batch/async
chain = prompt | llm | parser

# ❌ 不推荐：命令式、难以复用
messages = prompt.invoke({"question": "hi"})
response = llm.invoke(messages)
result = parser.invoke(response)
```

#### 2. 环境变量管理 API Key

```python
# ✅ 推荐
import os
os.environ["OPENAI_API_KEY"] = "sk-xxx"  # 或用 .env 文件
llm = ChatOpenAI()

# ❌ 不推荐：硬编码在代码中
llm = ChatOpenAI(api_key="sk-xxx")
```

#### 3. 为不同任务选择合适的 temperature

```python
# 数据提取、分类 → 确定性输出
extract_llm = ChatOpenAI(temperature=0)

# 创意生成 → 多样性输出
creative_llm = ChatOpenAI(temperature=0.9)
```

#### 4. 使用 with_structured_output 替代手动 Parser（LangChain 0.2+）

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

class Joke(BaseModel):
    setup: str = Field(description="笑话的铺垫")
    punchline: str = Field(description="笑话的笑点")

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# 直接让模型输出结构化数据，无需手动 Parser
structured_llm = llm.with_structured_output(Joke)
result = structured_llm.invoke("讲一个程序员笑话")
print(result.setup)
print(result.punchline)
```

#### 5. 错误处理与 Fallback

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# 主模型失败时自动切换到备用模型
llm = ChatOpenAI(model="gpt-4o").with_fallbacks([
    ChatAnthropic(model="claude-3-5-sonnet-20241022"),
    ChatOpenAI(model="gpt-3.5-turbo"),
])
```

### 常见坑

#### 坑 1：忘记安装对应的模型包

```bash
# 每个模型提供商有独立的包
pip install langchain-openai      # OpenAI
pip install langchain-anthropic   # Anthropic
pip install langchain-community   # 社区模型（Ollama 等）
```

#### 坑 2：Prompt 变量名与 invoke 参数不匹配

```python
prompt = ChatPromptTemplate.from_template("你好{name}")

# ❌ 报错：KeyError
chain.invoke({"username": "张三"})

# ✅ 正确
chain.invoke({"name": "张三"})

# 调试技巧：检查 input_variables
print(prompt.input_variables)  # ['name']
```

#### 坑 3：流式输出时使用了不支持流式的 Parser

```python
# ❌ PydanticOutputParser 不支持流式（需要完整 JSON）
chain = prompt | llm | PydanticOutputParser(pydantic_object=MyModel)
for chunk in chain.stream({"input": "hi"}):  # 可能报错
    print(chunk)

# ✅ StrOutputParser 和 JsonOutputParser 支持流式
chain = prompt | llm | StrOutputParser()
for chunk in chain.stream({"input": "hi"}):
    print(chunk, end="")
```

#### 坑 4：MessagesPlaceholder 忘记传值

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是助手"),
    MessagesPlaceholder("history"),  # 非 optional
    ("human", "{input}"),
])

# ❌ 报错：缺少 history
chain.invoke({"input": "hi"})

# ✅ 传空列表
chain.invoke({"input": "hi", "history": []})

# ✅ 或设置 optional=True
MessagesPlaceholder("history", optional=True)
```

#### 坑 5：混淆 Chat Model 和 LLM

```python
# LangChain 中有两种模型接口：
# - ChatModel（推荐）：输入/输出是 Message
# - LLM（旧版）：输入/输出是纯字符串

# ✅ 使用 ChatOpenAI（ChatModel）
from langchain_openai import ChatOpenAI

# ❌ 避免使用 OpenAI（旧版 LLM 接口，已废弃）
# from langchain_openai import OpenAI  # 不推荐
```

#### 坑 6：异步环境中使用同步方法

```python
# ❌ 在 async 函数中用 invoke 会阻塞事件循环
async def handler():
    result = llm.invoke("hi")  # 阻塞！

# ✅ 使用 ainvoke
async def handler():
    result = await llm.ainvoke("hi")  # 非阻塞
```

---

## 面试题精选

### 题目 1：LangChain 中 Chat Model 和 LLM 的区别是什么？

**答案：**

| 维度 | Chat Model | LLM |
|------|-----------|-----|
| 输入 | `List[BaseMessage]` | `str` |
| 输出 | `AIMessage` | `str` |
| 代表类 | `ChatOpenAI` | `OpenAI`（已废弃） |
| 适用场景 | 多轮对话、角色设定、工具调用 | 简单文本补全 |
| 当前状态 | **推荐使用** | 逐步废弃 |

Chat Model 是 LangChain 当前推荐的标准接口。它通过 Message 类型系统支持更丰富的交互模式（系统指令、多轮对话、工具调用），而旧版 LLM 接口仅支持简单的字符串输入输出。所有新开发都应使用 Chat Model。

---

### 题目 2：解释 LCEL 管道操作符 `|` 的工作原理

**答案：**

LCEL（LangChain Expression Language）的 `|` 操作符本质是 Python 的 `__or__` 魔术方法重载。它将多个 `Runnable` 对象串联为 `RunnableSequence`。

```python
chain = prompt | llm | parser
# 等价于
chain = RunnableSequence(first=prompt, middle=[llm], last=parser)
```

执行流程：
1. `prompt.invoke(input_dict)` → 返回 `ChatPromptValue`
2. `llm.invoke(prompt_value)` → 返回 `AIMessage`
3. `parser.invoke(ai_message)` → 返回最终结果

核心优势：
- **自动类型适配**：每个 Runnable 的输出自动作为下一个的输入
- **统一接口**：整条链自动获得 `invoke/stream/batch/ainvoke` 能力
- **可观测性**：内置 callback 支持，方便调试和监控
- **可序列化**：整条链可以保存和加载

---

### 题目 3：Output Parser 的 `get_format_instructions()` 有什么作用？如何工作？

**答案：**

`get_format_instructions()` 生成一段文本指令，告诉 LLM 应该以什么格式输出。这段指令通常被注入到 Prompt 的 system message 中。

工作流程：
```
1. parser.get_format_instructions() 
   → 生成格式说明文本（如 "请输出 JSON，schema 如下..."）

2. prompt.partial(format_instructions=instructions) 
   → 将格式说明注入 Prompt

3. LLM 看到格式要求后，按指定格式输出

4. parser.parse(llm_output) 
   → 将文本解析为结构化数据
```

例如 `PydanticOutputParser` 会生成类似这样的指令：
```
The output should be formatted as a JSON instance that conforms to the JSON schema below.
{"properties": {"name": {"type": "string"}, "age": {"type": "integer"}}}
```

注意：这依赖 LLM 的"遵循指令"能力，不是 100% 可靠的，所以才有 `OutputFixingParser` 作为兜底。

---

### 题目 4：`MessagesPlaceholder` 和普通变量 `{variable}` 有什么区别？

**答案：**

| 维度 | `{variable}` | `MessagesPlaceholder` |
|------|-------------|----------------------|
| 填充内容 | 字符串 | `List[BaseMessage]` |
| 消息数量 | 固定 1 条 | 动态 0~N 条 |
| 典型用途 | 用户输入、参数 | 对话历史、动态示例 |
| 位置 | 嵌入在某条消息内部 | 独立占据消息位置 |

```python
# {variable} — 填充到单条消息的内容中
("human", "翻译成{language}：{text}")

# MessagesPlaceholder — 展开为多条独立消息
MessagesPlaceholder("chat_history")
# 可以展开为：
# HumanMessage("你好")
# AIMessage("你好！")
# HumanMessage("天气如何")
# AIMessage("今天晴天")
```

`MessagesPlaceholder` 的核心价值是让 Prompt 模板能够处理**不确定数量**的消息，这在多轮对话、Agent 中间步骤等场景中不可或缺。

---

### 题目 5：如何实现模型 Fallback？在什么场景下需要？

**答案：**

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

llm = ChatOpenAI(model="gpt-4o").with_fallbacks([
    ChatOpenAI(model="gpt-4o-mini"),
    ChatAnthropic(model="claude-3-5-sonnet-20241022"),
])
```

当主模型调用失败（网络超时、API 限流、服务不可用）时，自动按顺序尝试备用模型。

适用场景：
1. **高可用系统**：生产环境不能因单一 API 故障而中断
2. **成本优化**：先尝试便宜模型，失败再用贵的
3. **速率限制**：主模型触发限流时自动切换
4. **灰度测试**：新模型不稳定时用旧模型兜底

Fallback 机制是 LangChain Runnable 协议的内置能力，任何 Runnable 都可以使用。

---

### 题目 6：`with_structured_output` 和 `PydanticOutputParser` 有什么区别？推荐哪个？

**答案：**

| 维度 | `with_structured_output` | `PydanticOutputParser` |
|------|-------------------------|----------------------|
| 实现方式 | 利用模型原生的 JSON Mode / Tool Calling | 在 Prompt 中注入格式指令 |
| 可靠性 | 高（模型层面保证格式） | 中（依赖模型遵循指令） |
| 兼容性 | 需要模型支持（GPT-5.x、Claude 4.x） | 任何模型都可用 |
| 使用复杂度 | 低（一行代码） | 中（需要手动注入 format_instructions） |
| 流式支持 | 支持 | 不支持 |

```python
# with_structured_output（推荐，LangChain 0.2+）
structured_llm = llm.with_structured_output(MyModel)
result = structured_llm.invoke("...")

# PydanticOutputParser（兼容性方案）
parser = PydanticOutputParser(pydantic_object=MyModel)
chain = prompt.partial(format_instructions=parser.get_format_instructions()) | llm | parser
```

**推荐**：优先使用 `with_structured_output`，它利用模型原生能力（如 OpenAI 的 JSON Mode 或 Function Calling），格式正确率更高。只有当模型不支持时才退回到 `PydanticOutputParser`。

---

### 题目 7：LangChain 中 `invoke`、`stream`、`batch`、`ainvoke` 的区别和使用场景？

**答案：**

| 方法 | 同步/异步 | 输入 | 输出 | 场景 |
|------|----------|------|------|------|
| `invoke` | 同步 | 单个输入 | 单个输出 | 普通单次调用 |
| `stream` | 同步 | 单个输入 | 迭代器（逐块） | 实时展示、打字机效果 |
| `batch` | 同步 | 输入列表 | 输出列表 | 批量处理、并发提速 |
| `ainvoke` | 异步 | 单个输入 | 单个输出 | Web 服务、高并发 |
| `astream` | 异步 | 单个输入 | 异步迭代器 | 异步流式（SSE） |
| `abatch` | 异步 | 输入列表 | 输出列表 | 异步批量 |

选择原则：
- **脚本/CLI** → `invoke`
- **Web API 返回流式** → `astream`
- **批量数据处理** → `batch`（可设 `max_concurrency`）
- **FastAPI/异步框架** → `ainvoke` / `astream`

这些方法是 Runnable 协议的一部分，所有 LangChain 组件（Prompt、Model、Parser、Chain）都自动支持，无需额外实现。

---

## 总结

Model I/O 是 LangChain 的基石模块，掌握它意味着掌握了与 LLM 交互的标准范式：

```
用户需求 → Prompt Template（构造输入）→ Chat Model（调用模型）→ Output Parser（解析输出）→ 结构化结果
```

核心要点回顾：
1. **Chat Model** 提供统一接口，一套代码适配所有模型
2. **Prompt Template** 让提示词可复用、可组合、可测试
3. **Output Parser** 将非结构化文本转为程序可用的数据
4. **LCEL 管道** 用 `|` 将三者优雅串联，自动获得 stream/batch/async 能力
5. **优先使用 `with_structured_output`** 获取结构化输出（LangChain 0.2+）

下一篇我们将深入 **Chain 与 LCEL**，探索更复杂的组合模式。

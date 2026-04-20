# LCEL 表达式语言（LangChain Expression Language）

> LangChain 专题第二篇 | 面向有 Python 基础的开发者 | 涵盖原理、组件、实战与面试

---

## LCEL 概述

### 什么是 LCEL

LCEL（LangChain Expression Language）是 LangChain 提供的一种声明式编排语言，用于将多个组件（LLM、Prompt、Parser、Retriever 等）组合成复杂的处理链。它的核心思想是：**用管道符 `|` 将组件串联，像搭积木一样构建 AI 应用**。

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 一个最简单的 LCEL 链
chain = ChatPromptTemplate.from_template("讲一个关于{topic}的笑话") | ChatOpenAI() | StrOutputParser()
result = chain.invoke({"topic": "程序员"})
print(result)
```

### 为什么需要 LCEL

| 痛点 | 传统 Chain 方式 | LCEL 方式 |
|------|---------------|-----------|
| 流式支持 | 需要手动实现 | 自动支持 stream |
| 异步支持 | 需要单独写异步版本 | 自动提供 ainvoke/astream |
| 批处理 | 手动循环 | 内置 batch 并发 |
| 重试/回退 | 自己写 try-except | with_retry/with_fallbacks |
| 可观测性 | 手动打日志 | 自动集成 LangSmith |
| 类型安全 | 运行时才报错 | 输入输出类型可推断 |
| 组合方式 | 继承、嵌套类 | 管道符声明式组合 |

### 与传统 Chain 的对比

```python
# ❌ 传统方式（LangChain 0.1 之前）— 已废弃
from langchain.chains import LLMChain
chain = LLMChain(llm=llm, prompt=prompt, output_parser=parser)

# ✅ LCEL 方式 — 推荐
chain = prompt | llm | parser
```

LCEL 的本质是一套 **Runnable 协议**，所有组件只要实现了这个协议，就能通过 `|` 自由组合。

---

## Runnable 协议

### 统一接口

每个 LCEL 组件都是一个 `Runnable`，提供以下标准方法：

| 方法 | 说明 | 适用场景 |
|------|------|---------|
| `invoke(input)` | 同步调用，单条输入 | 最常用 |
| `stream(input)` | 同步流式，逐 token 返回 | 实时输出 |
| `batch(inputs)` | 批量并发处理 | 批量任务 |
| `ainvoke(input)` | 异步调用 | Web 服务 |
| `astream(input)` | 异步流式 | 异步实时输出 |
| `astream_events(input)` | 异步事件流 | 细粒度监控 |

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("用一句话解释{concept}")
llm = ChatOpenAI(model="gpt-4o-mini")
parser = StrOutputParser()
chain = prompt | llm | parser

# invoke — 单条调用
result = chain.invoke({"concept": "量子计算"})

# stream — 流式输出
for chunk in chain.stream({"concept": "量子计算"}):
    print(chunk, end="", flush=True)

# batch — 批量处理（自动并发）
results = chain.batch([
    {"concept": "量子计算"},
    {"concept": "区块链"},
    {"concept": "神经网络"},
], config={"max_concurrency": 3})

# ainvoke — 异步调用
import asyncio
result = asyncio.run(chain.ainvoke({"concept": "量子计算"}))
```

### RunnableSequence — 管道符串联

`|` 操作符创建的是 `RunnableSequence`，它将多个 Runnable 按顺序串联：

```
输入 → [Runnable A] → [Runnable B] → [Runnable C] → 输出
```

```python
from langchain_core.runnables import RunnableSequence

# 以下两种写法等价
chain1 = prompt | llm | parser
chain2 = RunnableSequence(first=prompt, middle=[llm], last=parser)

# 链可以继续组合
chain_a = prompt | llm
chain_b = chain_a | parser  # 在已有链上追加
```

**关键规则**：前一个 Runnable 的输出类型必须匹配后一个 Runnable 的输入类型。

### 输入输出类型推断

每个 Runnable 都有 `input_schema` 和 `output_schema`：

```python
chain = prompt | llm | parser

# 查看输入 schema（基于 Pydantic）
print(chain.input_schema.schema())
# {'properties': {'concept': {'title': 'Concept', 'type': 'string'}}, 'required': ['concept']}

# 查看输出 schema
print(chain.output_schema.schema())
# {'title': 'StrOutputParserOutput', 'type': 'string'}
```

这使得 LangServe 能自动生成 API 文档和 Playground。

---

## 核心组件

### RunnablePassthrough — 透传输入

将输入原封不动传递到下一步，常用于在 `RunnableParallel` 中保留原始输入：

```python
from langchain_core.runnables import RunnablePassthrough, RunnableParallel

# 场景：RAG 中同时传递 question 和 context
chain = RunnableParallel(
    context=retriever,                    # 检索文档
    question=RunnablePassthrough()        # 透传原始问题
) | prompt | llm | parser

result = chain.invoke("什么是 LCEL？")
```

`RunnablePassthrough.assign()` 可以在透传的同时添加新字段：

```python
from langchain_core.runnables import RunnablePassthrough

# 输入 {"question": "xxx"} → 输出 {"question": "xxx", "context": "检索结果"}
chain = RunnablePassthrough.assign(
    context=lambda x: retriever.invoke(x["question"])
)
```

### RunnableLambda — 自定义函数包装

将任意 Python 函数包装为 Runnable，使其能参与 LCEL 链：

```python
from langchain_core.runnables import RunnableLambda

# 方式一：显式包装
def word_count(text: str) -> dict:
    return {"text": text, "count": len(text.split())}

count_runnable = RunnableLambda(word_count)

# 方式二：装饰器（推荐）
from langchain_core.runnables import chain as chain_decorator

@chain_decorator
def format_output(input: dict) -> str:
    return f"文本共 {input['count']} 个词：{input['text'][:50]}..."

# 组合使用
full_chain = prompt | llm | parser | count_runnable | format_output
```

**支持异步函数**：

```python
async def async_process(text: str) -> str:
    # 模拟异步操作
    await asyncio.sleep(0.1)
    return text.upper()

async_runnable = RunnableLambda(async_process)
```

### RunnableParallel — 并行执行

同时执行多个 Runnable，将结果合并为字典：

```python
from langchain_core.runnables import RunnableParallel

# 并行生成摘要和关键词
parallel_chain = RunnableParallel(
    summary=summary_prompt | llm | parser,
    keywords=keyword_prompt | llm | parser,
    sentiment=sentiment_prompt | llm | parser,
)

# 输入会被广播到每个分支
result = parallel_chain.invoke({"text": "今天天气真好，适合出去玩"})
# result = {"summary": "...", "keywords": "...", "sentiment": "..."}
```

**简写方式** — 直接传字典：

```python
# 以下两种写法等价
chain = RunnableParallel(a=runnable_a, b=runnable_b)
chain = {"a": runnable_a, "b": runnable_b}  # 字典自动转为 RunnableParallel
```

### RunnableBranch — 条件分支路由

根据条件选择不同的执行路径：

```python
from langchain_core.runnables import RunnableBranch

# 根据问题类型路由到不同链
branch = RunnableBranch(
    # (条件函数, 对应的 Runnable)
    (lambda x: "代码" in x["question"], code_chain),
    (lambda x: "数学" in x["question"], math_chain),
    # 最后一个参数是默认分支（无条件）
    general_chain,
)

result = branch.invoke({"question": "写一段排序代码"})
# → 走 code_chain
```

**更灵活的路由方式** — 用 `RunnableLambda` 实现动态路由：

```python
from langchain_core.runnables import RunnableLambda

def route(input: dict):
    if input["topic"] == "code":
        return code_chain
    elif input["topic"] == "math":
        return math_chain
    return general_chain

chain = classify_prompt | llm | parser | RunnableLambda(route)
```

### RunnableConfig — 配置传递

在链的执行过程中传递配置信息（如 callbacks、tags、metadata）：

```python
from langchain_core.runnables import RunnableConfig

# 调用时传入配置
result = chain.invoke(
    {"question": "什么是 LCEL？"},
    config=RunnableConfig(
        tags=["production", "user-query"],
        metadata={"user_id": "u123", "session_id": "s456"},
        max_concurrency=5,
        run_name="my_rag_chain",
    )
)
```

在自定义 Runnable 中获取配置：

```python
from langchain_core.runnables import RunnableConfig, RunnableLambda

def my_func(input: str, config: RunnableConfig) -> str:
    # 函数签名中加 config 参数即可自动注入
    user_id = config.get("metadata", {}).get("user_id", "unknown")
    return f"[{user_id}] {input}"

chain = RunnableLambda(my_func)
```

---

## 高级特性

### with_fallbacks — 降级回退

当主链失败时，自动切换到备用链：

```python
from langchain_openai import ChatOpenAI

# 主模型失败时回退到备用模型
llm_with_fallback = ChatOpenAI(model="gpt-4o").with_fallbacks(
    [ChatOpenAI(model="gpt-4o-mini"), ChatOpenAI(model="gpt-3.5-turbo")]
)

# 整条链也可以设置 fallback
main_chain = prompt | ChatOpenAI(model="gpt-4o") | parser
fallback_chain = prompt | ChatOpenAI(model="gpt-4o-mini") | parser

chain_with_fallback = main_chain.with_fallbacks([fallback_chain])
result = chain_with_fallback.invoke({"topic": "AI"})
```

**指定触发回退的异常类型**：

```python
from openai import RateLimitError, APITimeoutError

llm_safe = ChatOpenAI(model="gpt-4o").with_fallbacks(
    [ChatOpenAI(model="gpt-4o-mini")],
    exceptions_to_handle=(RateLimitError, APITimeoutError),
)
```

### with_retry — 重试机制

遇到临时错误时自动重试：

```python
from langchain_openai import ChatOpenAI

# 最多重试 3 次，指数退避
llm_retry = ChatOpenAI(model="gpt-4o").with_retry(
    stop_after_attempt=3,
    wait_exponential_jitter=True,  # 指数退避 + 随机抖动
)

# 指定哪些异常触发重试
from openai import RateLimitError

llm_retry = ChatOpenAI(model="gpt-4o").with_retry(
    retry_if_exception_type=(RateLimitError,),
    stop_after_attempt=5,
)
```

**组合使用 retry + fallback**：

```python
# 先重试，重试失败后再回退
llm_robust = (
    ChatOpenAI(model="gpt-4o")
    .with_retry(stop_after_attempt=2)
    .with_fallbacks([ChatOpenAI(model="gpt-4o-mini")])
)
```

### with_config — 运行时配置

为 Runnable 预设配置：

```python
from langchain_openai import ChatOpenAI

# 预设 tags 和 metadata
llm_configured = ChatOpenAI(model="gpt-4o").with_config(
    tags=["important"],
    metadata={"purpose": "summarization"},
    run_name="summary_llm",
)

# configurable_fields — 允许运行时动态修改参数
from langchain_core.runnables import ConfigurableField

llm = ChatOpenAI(model="gpt-4o").configurable_fields(
    model_name=ConfigurableField(id="model", name="模型名称")
)

# 运行时切换模型
result = llm.with_config(configurable={"model": "gpt-4o-mini"}).invoke("你好")
```

### bind — 绑定参数

为 Runnable 预绑定部分参数（常用于给 LLM 绑定 tools 或 stop 词）：

```python
from langchain_openai import ChatOpenAI

# 绑定 stop 词
llm_stop = ChatOpenAI().bind(stop=["\n\n"])

# 绑定工具调用
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """获取城市天气"""
    return f"{city}今天晴，25°C"

llm_with_tools = ChatOpenAI(model="gpt-4o").bind_tools([get_weather])

# 在链中使用
chain = prompt | llm_with_tools
result = chain.invoke({"question": "北京天气怎么样？"})
```

### assign — 动态添加字段

在数据流经过程中动态计算并添加新字段：

```python
from langchain_core.runnables import RunnablePassthrough

# 输入: {"question": "什么是LCEL？"}
# 输出: {"question": "什么是LCEL？", "context": "检索到的文档...", "length": 5}
chain = RunnablePassthrough.assign(
    context=lambda x: retriever.invoke(x["question"]),
    length=lambda x: len(x["question"]),
)

# assign 可以链式调用
chain = (
    RunnablePassthrough.assign(context=retriever_chain)
    | RunnablePassthrough.assign(answer=rag_chain)
)
```

**与 RunnableParallel 的区别**：
- `RunnableParallel`：输出只包含并行分支的结果
- `assign`：保留原始输入，并追加新字段

---

## 流式处理

### stream 基础用法

LCEL 链天然支持流式输出，无需额外代码：

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_template("写一首关于{topic}的诗")
    | ChatOpenAI(model="gpt-4o-mini", streaming=True)
    | StrOutputParser()
)

# 同步流式
for chunk in chain.stream({"topic": "春天"}):
    print(chunk, end="", flush=True)

# 异步流式（FastAPI 场景）
import asyncio

async def stream_response():
    async for chunk in chain.astream({"topic": "春天"}):
        print(chunk, end="", flush=True)

asyncio.run(stream_response())
```

**流式传播规则**：
- 链中每个组件如果支持流式，就会逐步传递
- 不支持流式的组件（如 Retriever）会等待完成后一次性传递
- `StrOutputParser` 会将 `AIMessageChunk` 转为字符串 chunk

### astream_events — 事件流

获取链执行过程中每个步骤的详细事件：

```python
import asyncio
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = (
    ChatPromptTemplate.from_template("解释{concept}")
    | ChatOpenAI(model="gpt-4o-mini")
    | StrOutputParser()
)

async def watch_events():
    async for event in chain.astream_events(
        {"concept": "LCEL"}, version="v2"
    ):
        kind = event["event"]
        if kind == "on_chat_model_stream":
            # LLM 输出的每个 token
            print(event["data"]["chunk"].content, end="")
        elif kind == "on_chain_start":
            print(f"\n--- 链开始: {event['name']} ---")
        elif kind == "on_chain_end":
            print(f"\n--- 链结束: {event['name']} ---")

asyncio.run(watch_events())
```

**常见事件类型**：

| 事件 | 触发时机 |
|------|---------|
| `on_chain_start` | 链/子链开始执行 |
| `on_chain_end` | 链/子链执行完成 |
| `on_chat_model_start` | LLM 开始调用 |
| `on_chat_model_stream` | LLM 输出一个 token |
| `on_chat_model_end` | LLM 调用完成 |
| `on_retriever_start` | 检索器开始 |
| `on_retriever_end` | 检索器完成 |
| `on_tool_start` | 工具调用开始 |
| `on_tool_end` | 工具调用完成 |

### 流式 + 结构化输出

使用 `JsonOutputParser` 实现流式结构化输出：

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template(
    "分析以下文本的情感，返回 JSON 格式 {{\"sentiment\": str, \"score\": float, \"reason\": str}}\n\n文本：{text}"
)

chain = prompt | ChatOpenAI(model="gpt-4o-mini") | JsonOutputParser()

# 流式输出部分 JSON（逐步构建完整对象）
for chunk in chain.stream({"text": "今天心情特别好！"}):
    print(chunk)
    # 第一次: {}
    # 第二次: {"sentiment": ""}
    # 第三次: {"sentiment": "positive"}
    # ...逐步完善
```

---

## 复杂编排实战

### 多步骤 RAG 链

一个完整的 RAG 链：查询改写 → 检索 → 重排 → 生成：

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from langchain_community.vectorstores import FAISS

# 准备组件
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(
    ["LCEL是LangChain的表达式语言", "Runnable是LCEL的核心协议", "管道符|用于串联组件"],
    embeddings
)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
llm = ChatOpenAI(model="gpt-4o-mini")

# Step 1: 查询改写
rewrite_prompt = ChatPromptTemplate.from_template(
    "将以下问题改写为更适合检索的形式，只输出改写后的问题：\n{question}"
)
rewrite_chain = rewrite_prompt | llm | StrOutputParser()

# Step 2: 检索 + 格式化
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

# Step 3: 生成答案
rag_prompt = ChatPromptTemplate.from_template("""
基于以下上下文回答问题。如果上下文中没有相关信息，请说"我不确定"。

上下文：
{context}

问题：{question}

答案：""")

# 完整 RAG 链
rag_chain = (
    # 改写查询并保留原始问题
    RunnablePassthrough.assign(rewritten=rewrite_chain)
    # 用改写后的查询检索
    | RunnablePassthrough.assign(
        context=lambda x: format_docs(retriever.invoke(x["rewritten"]))
    )
    # 生成答案
    | rag_prompt
    | llm
    | StrOutputParser()
)

result = rag_chain.invoke({"question": "LCEL 是什么？"})
print(result)
```

### 带条件路由的对话链

根据用户意图分类，路由到不同的处理链：

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableBranch, RunnableLambda

llm = ChatOpenAI(model="gpt-4o-mini")

# 意图分类链
classify_prompt = ChatPromptTemplate.from_template(
    "将以下用户消息分类为：code/math/chat 中的一个，只输出类别。\n消息：{input}"
)
classify_chain = classify_prompt | llm | StrOutputParser()

# 各专业链
code_prompt = ChatPromptTemplate.from_template("你是编程专家。请回答：{input}")
math_prompt = ChatPromptTemplate.from_template("你是数学专家。请回答：{input}")
chat_prompt = ChatPromptTemplate.from_template("你是友好的助手。请回答：{input}")

code_chain = code_prompt | llm | StrOutputParser()
math_chain = math_prompt | llm | StrOutputParser()
chat_chain = chat_prompt | llm | StrOutputParser()

# 路由逻辑
def route_by_intent(input_dict: dict) -> str:
    intent = classify_chain.invoke(input_dict).strip().lower()
    if "code" in intent:
        return code_chain.invoke(input_dict)
    elif "math" in intent:
        return math_chain.invoke(input_dict)
    return chat_chain.invoke(input_dict)

# 完整链
routed_chain = RunnableLambda(route_by_intent)

# 使用
print(routed_chain.invoke({"input": "用Python写一个快速排序"}))
print(routed_chain.invoke({"input": "计算 sin(π/4) 的值"}))
print(routed_chain.invoke({"input": "今天天气怎么样"}))
```

### 并行处理 + 结果合并

同时从多个角度分析文本，最后合并结果：

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableParallel, RunnableLambda

llm = ChatOpenAI(model="gpt-4o-mini")

# 三个并行分析链
summary_chain = (
    ChatPromptTemplate.from_template("用一句话总结：{text}")
    | llm | StrOutputParser()
)
keyword_chain = (
    ChatPromptTemplate.from_template("提取3个关键词，逗号分隔：{text}")
    | llm | StrOutputParser()
)
sentiment_chain = (
    ChatPromptTemplate.from_template("判断情感倾向(正面/负面/中性)：{text}")
    | llm | StrOutputParser()
)

# 并行执行
parallel = RunnableParallel(
    summary=summary_chain,
    keywords=keyword_chain,
    sentiment=sentiment_chain,
)

# 合并结果
def merge_results(results: dict) -> str:
    return f"""
📝 摘要：{results['summary']}
🏷️ 关键词：{results['keywords']}
💭 情感：{results['sentiment']}
""".strip()

# 完整链：并行分析 → 合并
analysis_chain = parallel | RunnableLambda(merge_results)

result = analysis_chain.invoke({"text": "LangChain的LCEL让AI应用开发变得简单高效，社区反响热烈。"})
print(result)
```

---

## 调试技巧

### 打印中间结果

方法一：插入 `RunnableLambda` 打印：

```python
from langchain_core.runnables import RunnableLambda

def debug_print(x, label="DEBUG"):
    print(f"[{label}] {type(x).__name__}: {str(x)[:200]}")
    return x  # 必须返回原值，不影响链

chain = (
    prompt
    | RunnableLambda(lambda x: debug_print(x, "PROMPT_OUT"))
    | llm
    | RunnableLambda(lambda x: debug_print(x, "LLM_OUT"))
    | parser
)
```

方法二：使用 `chain.pick()` 选择性输出：

```python
# 只获取链中某个步骤的输出
partial_chain = prompt | llm
result = partial_chain.invoke({"topic": "AI"})
print(result)  # 查看 LLM 原始输出（AIMessage）
```

### LangSmith 追踪

LangSmith 是 LangChain 官方的可观测性平台，LCEL 链自动集成：

```python
import os

# 设置环境变量即可启用追踪
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-api-key"
os.environ["LANGCHAIN_PROJECT"] = "my-project"

# 之后所有 chain.invoke() 都会自动上报到 LangSmith
chain = prompt | llm | parser
result = chain.invoke({"topic": "AI"})
# 在 LangSmith 控制台可以看到：
# - 每个步骤的输入输出
# - 耗时统计
# - Token 用量
# - 错误堆栈
```

**手动添加追踪信息**：

```python
result = chain.invoke(
    {"topic": "AI"},
    config={
        "run_name": "production_query",
        "tags": ["v2", "production"],
        "metadata": {"user_id": "u123"},
    }
)
```

---

## 最佳实践与常见坑

### 最佳实践

**1. 优先使用声明式组合，避免在 RunnableLambda 中写复杂逻辑**

```python
# ❌ 不推荐：把所有逻辑塞进一个函数
chain = RunnableLambda(lambda x: llm.invoke(prompt.format(**x)))

# ✅ 推荐：声明式组合
chain = prompt | llm | parser
```

**2. 给链命名，方便调试**

```python
chain = (prompt | llm | parser).with_config(run_name="summary_chain")
```

**3. 合理使用 batch 的并发控制**

```python
# 避免触发 API 限流
results = chain.batch(
    inputs,
    config={"max_concurrency": 5}  # 限制并发数
)
```

**4. 错误处理三件套**

```python
robust_chain = (
    prompt
    | llm.with_retry(stop_after_attempt=3)
         .with_fallbacks([backup_llm])
    | parser
)
```

**5. 流式优先**

```python
# Web 场景始终使用流式，提升用户体验
async def handle_request(question: str):
    async for chunk in chain.astream({"question": question}):
        yield chunk
```

### 常见坑

**坑 1：RunnableParallel 中的输入类型**

```python
# ❌ 错误：parallel 的每个分支接收的是同一个输入
parallel = RunnableParallel(
    a=chain_expecting_str,   # 期望 str
    b=chain_expecting_dict,  # 期望 dict
)
# 输入只能是一种类型！

# ✅ 正确：用 lambda 适配
parallel = RunnableParallel(
    a=RunnableLambda(lambda x: x["text"]) | chain_expecting_str,
    b=chain_expecting_dict,
)
```

**坑 2：忘记 RunnableLambda 必须返回值**

```python
# ❌ 错误：没有 return，下游收到 None
def process(x):
    print(x)  # 忘记 return

# ✅ 正确
def process(x):
    print(x)
    return x
```

**坑 3：流式场景下 parser 的选择**

```python
# ❌ PydanticOutputParser 不支持流式
from langchain_core.output_parsers import PydanticOutputParser
chain = prompt | llm | PydanticOutputParser(...)  # stream 时会等全部完成

# ✅ JsonOutputParser 支持流式（逐步构建 JSON）
from langchain_core.output_parsers import JsonOutputParser
chain = prompt | llm | JsonOutputParser()  # stream 时逐步输出部分结果
```

**坑 4：异步环境中混用同步调用**

```python
# ❌ 在 async 函数中用 invoke（会阻塞事件循环）
async def handler():
    result = chain.invoke(input)  # 阻塞！

# ✅ 使用 ainvoke
async def handler():
    result = await chain.ainvoke(input)
```

**坑 5：assign 和 RunnableParallel 的区别搞混**

```python
# RunnableParallel — 输出只有并行分支的结果
RunnableParallel(a=fn_a, b=fn_b).invoke({"x": 1})
# → {"a": ..., "b": ...}  ← 原始输入 x 丢失了！

# assign — 保留原始输入 + 追加新字段
RunnablePassthrough.assign(a=fn_a, b=fn_b).invoke({"x": 1})
# → {"x": 1, "a": ..., "b": ...}  ← 原始输入保留
```

---

## 面试题精选

### 题目 1：LCEL 中的 `|` 操作符本质是什么？

**答案**：

`|` 操作符是 Python 的 `__or__` 魔术方法重载。当两个 Runnable 对象使用 `|` 连接时，会创建一个 `RunnableSequence` 实例。

```python
# a | b | c 等价于：
RunnableSequence(first=a, middle=[b], last=c)
```

`RunnableSequence` 本身也是一个 Runnable，因此可以继续与其他 Runnable 组合，形成嵌套结构。执行时，数据从左到右依次流过每个组件，前一个的输出作为后一个的输入。

---

### 题目 2：RunnablePassthrough 和 RunnableParallel 有什么区别？什么时候用 assign？

**答案**：

| 组件 | 作用 | 输出 |
|------|------|------|
| `RunnablePassthrough` | 透传输入不做修改 | 与输入相同 |
| `RunnableParallel` | 并行执行多个分支 | 只包含各分支结果的字典 |
| `RunnablePassthrough.assign()` | 透传输入 + 追加新字段 | 原始输入 + 新字段 |

典型场景：
- RAG 中需要同时传递 `question`（原始）和 `context`（计算得到）→ 用 `assign`
- 需要从多个角度并行分析同一输入 → 用 `RunnableParallel`
- 在链中某个位置不需要变换，只是占位 → 用 `RunnablePassthrough`

```python
# assign 示例：保留 question，追加 context
chain = RunnablePassthrough.assign(
    context=lambda x: retriever.invoke(x["question"])
)
# 输入 {"question": "hi"} → 输出 {"question": "hi", "context": [...]}
```

---

### 题目 3：如何让 LCEL 链支持流式输出？哪些组件会阻断流式？

**答案**：

LCEL 链默认支持流式，只需调用 `chain.stream(input)` 或 `chain.astream(input)`。流式传播遵循以下规则：

1. **支持流式的组件**：LLM（ChatModel）、StrOutputParser、JsonOutputParser — 逐 token/chunk 传递
2. **阻断流式的组件**：Retriever、RunnableLambda（普通函数）、PydanticOutputParser — 必须等待完成后一次性输出
3. **RunnableParallel**：所有分支并行执行，每个分支独立流式，但整体需要等最慢的分支

要让自定义函数支持流式，需要实现生成器：

```python
from langchain_core.runnables import RunnableLambda

def streaming_func(input):
    for char in input:
        yield char  # 生成器自动支持流式

chain = RunnableLambda(streaming_func)
```

---

### 题目 4：with_retry 和 with_fallbacks 的执行顺序是什么？如何组合使用？

**答案**：

- `with_retry`：对同一个 Runnable 重复尝试，适合临时性错误（网络超时、限流）
- `with_fallbacks`：当前 Runnable 彻底失败后，切换到备选 Runnable

**组合顺序很重要**：

```python
# 推荐：先 retry 再 fallback
llm = (
    ChatOpenAI(model="gpt-4o")
    .with_retry(stop_after_attempt=3)      # 先重试 3 次
    .with_fallbacks([ChatOpenAI(model="gpt-4o-mini")])  # 都失败了再降级
)
```

执行流程：
```
gpt-4o 第1次 → 失败 → gpt-4o 第2次 → 失败 → gpt-4o 第3次 → 失败 → gpt-4o-mini
```

如果反过来（先 fallback 再 retry），则 retry 会作用在整个 fallback 链上，语义不同。

---

### 题目 5：如何在 LCEL 链中实现条件分支？有几种方式？

**答案**：

至少有三种方式：

**方式一：RunnableBranch（声明式）**

```python
from langchain_core.runnables import RunnableBranch

branch = RunnableBranch(
    (lambda x: x["type"] == "code", code_chain),
    (lambda x: x["type"] == "math", math_chain),
    default_chain,  # 最后一个是默认
)
```

**方式二：RunnableLambda 动态路由**

```python
def route(input):
    if input["type"] == "code":
        return code_chain.invoke(input)
    return default_chain.invoke(input)

chain = RunnableLambda(route)
```

**方式三：返回 Runnable 的函数（自动执行）**

```python
def pick_chain(input):
    if input["type"] == "code":
        return code_chain  # 返回 Runnable 本身
    return default_chain

# LangChain 会自动对返回的 Runnable 调用 invoke
chain = classify_chain | RunnableLambda(pick_chain)
```

推荐使用方式一（简单场景）或方式三（复杂场景），方式二在流式场景下有局限。

---

### 题目 6：解释 LCEL 中 configurable_fields 和 configurable_alternatives 的区别

**答案**：

两者都用于运行时动态配置，但粒度不同：

**configurable_fields** — 修改组件的某个参数：

```python
from langchain_core.runnables import ConfigurableField

llm = ChatOpenAI(model="gpt-4o", temperature=0.7).configurable_fields(
    temperature=ConfigurableField(id="temp", name="温度"),
    model_name=ConfigurableField(id="model", name="模型"),
)

# 运行时修改
result = llm.with_config(configurable={"temp": 0.1, "model": "gpt-4o-mini"}).invoke("hi")
```

**configurable_alternatives** — 整体替换为另一个组件：

```python
from langchain_core.runnables import ConfigurableField
from langchain_anthropic import ChatAnthropic

llm = ChatOpenAI(model="gpt-4o").configurable_alternatives(
    ConfigurableField(id="llm_provider"),
    anthropic=ChatAnthropic(model="claude-3-sonnet-20240229"),
    default_key="openai",
)

# 运行时切换到 Anthropic
result = llm.with_config(configurable={"llm_provider": "anthropic"}).invoke("hi")
```

核心区别：`configurable_fields` 是微调参数，`configurable_alternatives` 是整体替换组件。

---

### 题目 7：LCEL 链的 batch 方法内部是如何实现并发的？如何控制并发度？

**答案**：

`batch` 方法内部使用 Python 的 `concurrent.futures.ThreadPoolExecutor`（同步）或 `asyncio.gather`（异步）实现并发。

```python
# 默认无并发限制
results = chain.batch([input1, input2, input3])

# 控制并发度
results = chain.batch(
    [input1, input2, ..., input100],
    config={"max_concurrency": 10}  # 最多同时执行 10 个
)
```

内部实现逻辑：
1. 将输入列表分成大小为 `max_concurrency` 的批次
2. 每个批次内的请求并发执行
3. 等待当前批次全部完成后，执行下一批次
4. 收集所有结果，按原始顺序返回

注意：`batch` 中每个输入独立执行完整的链，不会共享中间状态。

---

## 总结

LCEL 是 LangChain 的核心编排机制，掌握它的关键在于：

```
理解 Runnable 协议 → 熟悉核心组件 → 掌握组合模式 → 实战复杂链
```

| 层次 | 内容 | 重要度 |
|------|------|--------|
| 基础 | invoke/stream/batch、管道符 `\|` | ⭐⭐⭐⭐⭐ |
| 核心 | Passthrough、Lambda、Parallel、Branch | ⭐⭐⭐⭐⭐ |
| 进阶 | retry、fallbacks、bind、assign | ⭐⭐⭐⭐ |
| 高级 | astream_events、configurable | ⭐⭐⭐ |

记住一个原则：**能用声明式组合解决的，就不要写命令式代码**。LCEL 的声明式风格不仅代码更简洁，还能自动获得流式、异步、可观测性等能力。

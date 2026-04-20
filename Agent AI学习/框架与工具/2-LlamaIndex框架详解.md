# LlamaIndex 框架详解

## 1. 概述

LlamaIndex（原 GPT Index）是一个专注于**数据连接和检索**的 LLM 框架。如果说 LangChain 是"瑞士军刀"，LlamaIndex 就是"数据检索专家"。

### 核心定位

````
LangChain  → 通用 LLM 应用框架（Agent、Chain、工具编排）
LlamaIndex → 数据索引和检索框架（RAG、知识库、数据查询）

两者互补，可以一起使用
````
### 核心能力

````
数据连接 → 支持 100+ 数据源（PDF、数据库、API、网页...）
数据索引 → 多种索引策略（向量、关键词、树形、知识图谱）
查询引擎 → 智能检索 + LLM 生成回答
Agent    → 基于数据的智能 Agent
````
## 2. 安装与配置

> **包结构说明：** 和 LangChain 类似，LlamaIndex 也拆分成了多个包。`llama-index` 是主包，LLM、Embedding、向量数据库、文件 Reader 等都是独立的子包，按需安装。这样核心包很轻量，不会引入一堆用不到的依赖。

````bash
# 核心包
pip install llama-index

# 按需安装
pip install llama-index-llms-openai
pip install llama-index-embeddings-openai
pip install llama-index-vector-stores-chroma
pip install llama-index-readers-file  # PDF、DOCX 等
````
````python
import os
os.environ["OPENAI_API_KEY"] = "your-key"
````
## 3. 快速开始 — 5 分钟构建知识库问答

> **这段代码做了什么：** 只用 3 行核心代码就构建了一个完整的 RAG 知识库问答系统——加载文档、构建向量索引、查询。LlamaIndex 把文档分块、Embedding、向量存储、检索、LLM 生成回答这些步骤全部封装好了，开箱即用。

````python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 1. 加载文档
documents = SimpleDirectoryReader("./data").load_data()

# 2. 构建索引
index = VectorStoreIndex.from_documents(documents)

# 3. 查询
query_engine = index.as_query_engine()
response = query_engine.query("这个项目的主要功能是什么?")
print(response)
````

> **关键代码解读：**
> - `SimpleDirectoryReader("./data")` — 自动识别目录下所有文件类型（PDF、MD、TXT 等），加载为 Document 对象
> - `VectorStoreIndex.from_documents()` — 自动完成：文档分块 → Embedding → 存入向量索引
> - `index.as_query_engine()` — 把索引包装为查询引擎，调用 `query()` 时自动完成：问题 Embedding → 向量检索 → 将检索结果 + 问题一起传给 LLM 生成回答
>
> **对比 LangChain：** 同样的功能在 LangChain 中需要手动组装 TextSplitter、Embeddings、VectorStore、RetrievalQA 等多个组件。LlamaIndex 的优势就是把 RAG 流程高度封装。

## 4. 数据连接 — Data Connectors

### 4.1 内置 Reader

> **Reader 是什么：** Reader 负责把各种格式的数据源转换为 LlamaIndex 统一的 `Document` 对象。`SimpleDirectoryReader` 是最常用的，能自动识别文件类型。

````python
from llama_index.core import SimpleDirectoryReader

# 自动识别文件类型（PDF、DOCX、TXT、MD、CSV...）
documents = SimpleDirectoryReader(
    input_dir="./docs",
    recursive=True,           # 递归子目录
    required_exts=[".pdf", ".md"],  # 过滤文件类型
).load_data()
````
### 4.2 LlamaHub — 社区数据连接器

> **LlamaHub 是什么：** 社区维护的数据连接器市场，支持 100+ 数据源——数据库、网页、Notion、Slack、GitHub 等。安装对应的 reader 包就能用，接口统一都是 `load_data()`。

````python
# 从数据库加载
from llama_index.readers.database import DatabaseReader

reader = DatabaseReader(uri="postgresql://user:pass@localhost/db")
documents = reader.load_data(query="SELECT * FROM articles")

# 从网页加载
from llama_index.readers.web import SimpleWebPageReader

reader = SimpleWebPageReader()
documents = reader.load_data(urls=["https://example.com/doc"])

# 从 Notion 加载
from llama_index.readers.notion import NotionPageReader

reader = NotionPageReader(integration_token="your-token")
documents = reader.load_data(page_ids=["page-id"])
````

> **关键代码解读：** 三个 Reader 的用法完全一致——初始化 → `load_data()` → 得到 Document 列表。不管数据来自 PostgreSQL、网页还是 Notion，后续的索引和查询代码都不需要改。

### 4.3 文档处理

> **为什么要分块：** LLM 的上下文窗口有限，不可能把整篇文档塞进去。分块（Chunking）把长文档切成小段，每段独立做 Embedding 和检索。`chunk_overlap` 是块之间的重叠部分，防止关键信息恰好被切断。

````python
from llama_index.core.node_parser import SentenceSplitter

# 文档分块
parser = SentenceSplitter(
    chunk_size=1024,      # 每块最大 token 数
    chunk_overlap=200,    # 块之间重叠 token 数
)
nodes = parser.get_nodes_from_documents(documents)
````

> **关键代码解读：**
> - `SentenceSplitter` — 按句子边界分块，比简单按字符数切割更智能，不会把一句话切成两半
> - `chunk_size=1024` — 每块最大 1024 个 token（不是字符）
> - `chunk_overlap=200` — 相邻块重叠 200 token，确保跨块的信息不丢失

## 5. 索引类型

> **索引是 LlamaIndex 的核心：** 不同的索引类型决定了不同的检索策略。大多数场景用 VectorStoreIndex（向量语义检索）就够了，但特定场景下其他索引类型可能更合适。

### 5.1 VectorStoreIndex（最常用）

> **这是什么：** 把文档块转成向量存储，查询时通过向量相似度检索最相关的文档块。默认存在内存中，也可以持久化到 ChromaDB、Pinecone 等向量数据库。

````python
from llama_index.core import VectorStoreIndex

# 内存向量索引
index = VectorStoreIndex.from_documents(documents)

# 持久化到 ChromaDB
import chromadb
from llama_index.vector_stores.chroma import ChromaVectorStore
from llama_index.core import StorageContext

chroma_client = chromadb.PersistentClient(path="./chroma_db")
collection = chroma_client.get_or_create_collection("my_docs")
vector_store = ChromaVectorStore(chroma_collection=collection)

storage_context = StorageContext.from_defaults(vector_store=vector_store)
index = VectorStoreIndex.from_documents(
    documents, storage_context=storage_context
)
````

> **关键代码解读：**
> - 内存索引：`VectorStoreIndex.from_documents(documents)` 一行搞定，适合开发调试
> - 持久化索引：通过 `StorageContext` 指定存储后端（这里是 ChromaDB），数据写入磁盘，重启后不丢失
> - `ChromaVectorStore` 是 LlamaIndex 对 ChromaDB 的适配器，让 LlamaIndex 的索引能透明地使用 ChromaDB 存储

### 5.2 SummaryIndex

> **和 VectorStoreIndex 的区别：** VectorStoreIndex 只检索最相关的 Top-K 块，SummaryIndex 会遍历所有文档块生成摘要。适合"总结整篇文档"这类需要全局信息的场景，但速度更慢、Token 消耗更大。

````python
from llama_index.core import SummaryIndex

# 适合需要全文摘要的场景
index = SummaryIndex.from_documents(documents)
query_engine = index.as_query_engine(response_mode="tree_summarize")
````
### 5.3 KeywordTableIndex

> **适用场景：** 基于关键词的精确匹配检索，不依赖 Embedding。当用户的查询包含明确的关键词（如 API 名称、错误码）时，关键词索引比向量检索更精准。可以和 VectorStoreIndex 组合使用，互补长短。

````python
from llama_index.core import KeywordTableIndex

# 基于关键词的索引，适合精确匹配
index = KeywordTableIndex.from_documents(documents)
````
## 6. 查询引擎

### 6.1 基础查询

> **查询引擎做了什么：** 调用 `query()` 时，引擎自动完成：问题 Embedding → 在索引中检索 Top-K 相关文档块 → 把检索结果和问题一起传给 LLM → 生成回答。`response.source_nodes` 可以看到回答引用了哪些原始文档，方便溯源验证。

````python
query_engine = index.as_query_engine(
    similarity_top_k=5,          # 检索 top 5 相关文档
    response_mode="compact",     # 响应模式
)

response = query_engine.query("如何优化启动速度?")
print(response)                  # 回答
print(response.source_nodes)     # 来源文档
````
### 6.2 响应模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| `compact` | 将检索结果压缩后一次性给 LLM | 默认，通用 |
| `refine` | 逐个文档迭代优化回答 | 需要精确回答 |
| `tree_summarize` | 树形递归摘要 | 长文档摘要 |
| `simple_summarize` | 简单拼接后摘要 | 快速摘要 |
| `no_text` | 只返回检索结果，不生成回答 | 只需检索 |

### 6.3 Chat Engine（多轮对话）

> **和 Query Engine 的区别：** Query Engine 每次查询独立，不记得之前问过什么。Chat Engine 维护对话历史，能理解"它"指的是什么、"继续"是继续什么。`condense_plus_context` 模式会先把多轮对话压缩为一个独立的查询，再去检索，效果最好。

````python
chat_engine = index.as_chat_engine(chat_mode="condense_plus_context")

response = chat_engine.chat("这个项目用了什么技术栈?")
print(response)

response = chat_engine.chat("它的性能怎么样?")  # 自动关联上下文
print(response)
````
## 7. Agent 模式

> **LlamaIndex 也能做 Agent：** 不只是检索，LlamaIndex 的 Agent 可以把查询引擎包装为工具，让 LLM 自主决定何时查知识库、何时用其他工具。`ReActAgent` 使用 ReAct（Reasoning + Acting）模式——先推理需要什么信息，再调用工具获取，循环直到能回答问题。

````python
from llama_index.core.agent import ReActAgent
from llama_index.core.tools import QueryEngineTool, FunctionTool

# 将查询引擎包装为工具
query_tool = QueryEngineTool.from_defaults(
    query_engine=query_engine,
    name="knowledge_base",
    description="查询项目知识库获取技术文档信息"
)

# 自定义工具
def calculate(expression: str) -> str:
    """计算数学表达式"""
    return str(eval(expression))

calc_tool = FunctionTool.from_defaults(fn=calculate)

# 创建 Agent
agent = ReActAgent.from_tools(
    [query_tool, calc_tool],
    llm=llm,
    verbose=True
)

response = agent.chat("项目中有多少个模块?每个模块平均多少行代码?")
````

> **关键代码解读：**
> - `QueryEngineTool.from_defaults()` — 把查询引擎包装为 Agent 可调用的工具，`description` 告诉 LLM 这个工具能做什么
> - `FunctionTool.from_defaults()` — 把普通 Python 函数包装为工具
> - `ReActAgent.from_tools()` — 创建 ReAct Agent，它会交替进行"思考"和"行动"，直到得出最终答案
> - `verbose=True` — 打印 Agent 的思考过程，方便调试

## 8. 高级特性

### 8.1 子问题查询

> **解决什么问题：** 用户问"对比 Android 和后端的架构设计异同"，这个问题涉及两个领域，单个索引回答不了。SubQuestionQueryEngine 会自动把它拆成"Android 的架构设计是什么"和"后端的架构设计是什么"两个子问题，分别路由到对应的索引查询，最后合并结果。

````python
from llama_index.core.query_engine import SubQuestionQueryEngine
from llama_index.core.tools import QueryEngineTool

# 多个索引，每个覆盖不同领域
tools = [
    QueryEngineTool.from_defaults(android_engine, name="android", description="Android 开发知识"),
    QueryEngineTool.from_defaults(backend_engine, name="backend", description="后端开发知识"),
]

# 自动拆分复杂问题为子问题
engine = SubQuestionQueryEngine.from_defaults(query_engine_tools=tools)
response = engine.query("对比 Android 和后端的架构设计有什么异同?")
````
### 8.2 路由查询

> **和子问题查询的区别：** 子问题查询是"拆分问题，多路查询，合并结果"。路由查询是"判断问题属于哪个领域，只路由到一个最合适的引擎"。适合问题明确属于某个领域的场景，比如用户问 Android 问题就路由到 Android 索引。

````python
from llama_index.core.query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector

# 根据问题自动路由到合适的引擎
router_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(),
    query_engine_tools=tools
)
````
## 9. LlamaIndex vs LangChain 选型

| 场景 | 推荐 |
|------|------|
| 构建 RAG / 知识库问答 | LlamaIndex |
| 构建通用 Agent | LangChain |
| 复杂数据处理和索引 | LlamaIndex |
| 工具编排和 Chain | LangChain |
| 多 Agent 协作 | LangChain (LangGraph) |
| 两者结合 | LlamaIndex 做检索 + LangChain 做编排 |

---

## 面试题精选

### Q1: LlamaIndex 和 LangChain 的定位有什么区别？
**答：** LlamaIndex 专注于数据索引和检索（RAG 专家），LangChain 是通用 LLM 应用框架（瑞士军刀）。构建 RAG/知识库问答优先选 LlamaIndex，构建通用 Agent 选 LangChain，两者可以结合使用。

### Q2: LlamaIndex 支持哪些索引类型？各自适用什么场景？
**答：** VectorStoreIndex（向量检索，最常用）、SummaryIndex（全文摘要）、KeywordTableIndex（关键词精确匹配）。还支持知识图谱索引。大多数 RAG 场景用 VectorStoreIndex 即可。

### Q3: LlamaIndex 的响应模式（response_mode）有哪些？怎么选？
**答：** compact（压缩后一次给 LLM，默认通用）、refine（逐文档迭代优化，需要精确回答）、tree_summarize（树形递归摘要，长文档摘要）、no_text（只返回检索结果不生成回答）。

### Q4: SubQuestionQueryEngine 解决了什么问题？
**答：** 它能自动将复杂问题拆分为多个子问题，分别路由到不同的索引/引擎查询，最后合并结果。适合跨领域的对比分析类问题，如"对比 Android 和后端的架构设计异同"。

### Q5: LlamaIndex 的 Chat Engine 和 Query Engine 有什么区别？
**答：** Query Engine 是单轮问答，每次查询独立。Chat Engine 支持多轮对话，自动维护对话历史并将历史上下文融入检索和生成过程，能理解代词指代和上下文关联。

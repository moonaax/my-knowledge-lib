# LlamaIndex 框架详解

## 1. 概述

LlamaIndex（原 GPT Index）是一个专注于**数据连接和检索**的 LLM 框架。如果说 LangChain 是"瑞士军刀"，LlamaIndex 就是"数据检索专家"。

### 核心定位

```
LangChain  → 通用 LLM 应用框架（Agent、Chain、工具编排）
LlamaIndex → 数据索引和检索框架（RAG、知识库、数据查询）

两者互补，可以一起使用
```

### 核心能力

```
数据连接 → 支持 100+ 数据源（PDF、数据库、API、网页...）
数据索引 → 多种索引策略（向量、关键词、树形、知识图谱）
查询引擎 → 智能检索 + LLM 生成回答
Agent    → 基于数据的智能 Agent
```

## 2. 安装与配置

```bash
# 核心包
pip install llama-index

# 按需安装
pip install llama-index-llms-openai
pip install llama-index-embeddings-openai
pip install llama-index-vector-stores-chroma
pip install llama-index-readers-file  # PDF、DOCX 等
```

```python
import os
os.environ["OPENAI_API_KEY"] = "your-key"
```

## 3. 快速开始 — 5 分钟构建知识库问答

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# 1. 加载文档
documents = SimpleDirectoryReader("./data").load_data()

# 2. 构建索引
index = VectorStoreIndex.from_documents(documents)

# 3. 查询
query_engine = index.as_query_engine()
response = query_engine.query("这个项目的主要功能是什么?")
print(response)
```

## 4. 数据连接 — Data Connectors

### 4.1 内置 Reader

```python
from llama_index.core import SimpleDirectoryReader

# 自动识别文件类型（PDF、DOCX、TXT、MD、CSV...）
documents = SimpleDirectoryReader(
    input_dir="./docs",
    recursive=True,           # 递归子目录
    required_exts=[".pdf", ".md"],  # 过滤文件类型
).load_data()
```

### 4.2 LlamaHub — 社区数据连接器

```python
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
```

### 4.3 文档处理

```python
from llama_index.core.node_parser import SentenceSplitter

# 文档分块
parser = SentenceSplitter(
    chunk_size=1024,      # 每块最大 token 数
    chunk_overlap=200,    # 块之间重叠 token 数
)
nodes = parser.get_nodes_from_documents(documents)
```

## 5. 索引类型

### 5.1 VectorStoreIndex（最常用）

```python
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
```

### 5.2 SummaryIndex

```python
from llama_index.core import SummaryIndex

# 适合需要全文摘要的场景
index = SummaryIndex.from_documents(documents)
query_engine = index.as_query_engine(response_mode="tree_summarize")
```

### 5.3 KeywordTableIndex

```python
from llama_index.core import KeywordTableIndex

# 基于关键词的索引，适合精确匹配
index = KeywordTableIndex.from_documents(documents)
```

## 6. 查询引擎

### 6.1 基础查询

```python
query_engine = index.as_query_engine(
    similarity_top_k=5,          # 检索 top 5 相关文档
    response_mode="compact",     # 响应模式
)

response = query_engine.query("如何优化启动速度?")
print(response)                  # 回答
print(response.source_nodes)     # 来源文档
```

### 6.2 响应模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| `compact` | 将检索结果压缩后一次性给 LLM | 默认，通用 |
| `refine` | 逐个文档迭代优化回答 | 需要精确回答 |
| `tree_summarize` | 树形递归摘要 | 长文档摘要 |
| `simple_summarize` | 简单拼接后摘要 | 快速摘要 |
| `no_text` | 只返回检索结果，不生成回答 | 只需检索 |

### 6.3 Chat Engine（多轮对话）

```python
chat_engine = index.as_chat_engine(chat_mode="condense_plus_context")

response = chat_engine.chat("这个项目用了什么技术栈?")
print(response)

response = chat_engine.chat("它的性能怎么样?")  # 自动关联上下文
print(response)
```

## 7. Agent 模式

```python
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
```

## 8. 高级特性

### 8.1 子问题查询

```python
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
```

### 8.2 路由查询

```python
from llama_index.core.query_engine import RouterQueryEngine
from llama_index.core.selectors import LLMSingleSelector

# 根据问题自动路由到合适的引擎
router_engine = RouterQueryEngine(
    selector=LLMSingleSelector.from_defaults(),
    query_engine_tools=tools
)
```

## 9. LlamaIndex vs LangChain 选型

| 场景 | 推荐 |
|------|------|
| 构建 RAG / 知识库问答 | LlamaIndex |
| 构建通用 Agent | LangChain |
| 复杂数据处理和索引 | LlamaIndex |
| 工具编排和 Chain | LangChain |
| 多 Agent 协作 | LangChain (LangGraph) |
| 两者结合 | LlamaIndex 做检索 + LangChain 做编排 |

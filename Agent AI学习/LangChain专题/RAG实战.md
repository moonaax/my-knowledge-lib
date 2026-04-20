# RAG 实战 — 用 LangChain 构建检索增强生成系统

> LangChain 专题第四篇 | 面向有 Python 基础的开发者
> 
> RAG（Retrieval-Augmented Generation）是当前大模型应用最核心的架构模式之一。本文将从零开始，带你用 LangChain 构建一个完整的 RAG 系统。

---

## 1. RAG 在 LangChain 中的全景

### 什么是 RAG

RAG（检索增强生成）的核心思想：**不依赖模型记忆所有知识，而是在生成回答前，先从外部知识库中检索相关信息，再将检索结果作为上下文提供给 LLM 生成答案。**

这解决了 LLM 的三大痛点：
- **知识过时**：训练数据有截止日期
- **幻觉问题**：模型可能编造不存在的信息
- **领域知识缺失**：通用模型不了解企业私有数据

### 架构全景图

```
┌─────────────────────────────────────────────────────────────┐
│                      RAG 系统架构                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────── 离线索引阶段 ────────────┐                    │
│  │                                      │                    │
│  │  Documents → Loader → Splitter → Embedding → VectorStore │
│  │                                      │                    │
│  └──────────────────────────────────────┘                    │
│                                                             │
│  ┌──────────── 在线查询阶段 ────────────┐                    │
│  │                                      │                    │
│  │  Query → Embedding → Retriever → Context → LLM → Answer │
│  │                                      │                    │
│  └──────────────────────────────────────┘                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件关系

| 组件 | 职责 | LangChain 类 |
|------|------|-------------|
| Document Loader | 加载各种格式的文档 | `TextLoader`, `PyPDFLoader`, ... |
| Text Splitter | 将文档切分为合适大小的块 | `RecursiveCharacterTextSplitter` |
| Embedding Model | 将文本转换为向量表示 | `OpenAIEmbeddings`, `HuggingFaceEmbeddings` |
| Vector Store | 存储和检索向量 | `Chroma`, `FAISS` |
| Retriever | 根据查询检索相关文档 | `VectorStoreRetriever` |
| LLM/ChatModel | 基于上下文生成回答 | `ChatOpenAI` |
| Chain | 串联以上组件 | `RetrievalQA`, LCEL |

### 环境准备

```python
# 安装核心依赖
pip install langchain langchain-openai langchain-community langchain-chroma
pip install chromadb faiss-cpu pypdf unstructured
pip install sentence-transformers rank_bm25

# 设置环境变量
import os
os.environ["OPENAI_API_KEY"] = "your-api-key"
```

---

## 2. Document Loader — 文档加载

Document Loader 负责将各种格式的数据源加载为 LangChain 统一的 `Document` 对象。

### Document 对象结构

每个 Document 包含两个核心属性：

```python
from langchain_core.documents import Document

doc = Document(
    page_content="这是文档的实际文本内容",
    metadata={
        "source": "file.pdf",
        "page": 0,
        "author": "张三",
        # 任意自定义元数据
    }
)

print(doc.page_content)  # 文本内容
print(doc.metadata)      # 元数据字典
```

### 文本文件加载

```python
from langchain_community.document_loaders import TextLoader

# 加载单个文本文件
loader = TextLoader("./data/example.txt", encoding="utf-8")
docs = loader.load()

print(f"加载了 {len(docs)} 个文档")
print(f"内容预览: {docs[0].page_content[:100]}")
print(f"元数据: {docs[0].metadata}")
```

### PDF 文件加载

```python
from langchain_community.document_loaders import PyPDFLoader

# 每页作为一个 Document
loader = PyPDFLoader("./data/paper.pdf")
pages = loader.load()

print(f"PDF 共 {len(pages)} 页")
for page in pages[:3]:
    print(f"第 {page.metadata['page']} 页: {page.page_content[:50]}...")
```

### Markdown 文件加载

```python
from langchain_community.document_loaders import UnstructuredMarkdownLoader

loader = UnstructuredMarkdownLoader("./data/readme.md")
docs = loader.load()

# 如果需要按标题分块
from langchain_community.document_loaders import MarkdownHeaderTextSplitter

headers_to_split_on = [
    ("#", "Header 1"),
    ("##", "Header 2"),
    ("###", "Header 3"),
]
splitter = MarkdownHeaderTextSplitter(headers_to_split_on=headers_to_split_on)
splits = splitter.split_text(docs[0].page_content)
```

### 网页加载

```python
from langchain_community.document_loaders import WebBaseLoader

# 加载单个网页
loader = WebBaseLoader("https://docs.python.org/3/tutorial/index.html")
docs = loader.load()

# 加载多个网页
loader = WebBaseLoader([
    "https://example.com/page1",
    "https://example.com/page2",
])
docs = loader.load()
```

### CSV 和 JSON 结构化数据加载

```python
from langchain_community.document_loaders import CSVLoader, JSONLoader

# CSV 加载 — 每行作为一个 Document
loader = CSVLoader(
    file_path="./data/products.csv",
    csv_args={"delimiter": ","},
    source_column="product_name",  # 用某列作为 source 元数据
)
docs = loader.load()

# JSON 加载 — 使用 jq schema 提取内容
loader = JSONLoader(
    file_path="./data/articles.json",
    jq_schema=".articles[]",           # jq 表达式定位数据
    content_key="content",             # 哪个字段作为 page_content
    metadata_func=lambda record, metadata: {
        **metadata,
        "title": record.get("title"),
        "author": record.get("author"),
    }
)
docs = loader.load()
```

### 自定义 Loader

```python
from langchain_core.document_loaders import BaseLoader
from langchain_core.documents import Document
from typing import Iterator

class DatabaseLoader(BaseLoader):
    """从数据库加载文档的自定义 Loader"""
    
    def __init__(self, connection_string: str, query: str):
        self.connection_string = connection_string
        self.query = query
    
    def lazy_load(self) -> Iterator[Document]:
        """惰性加载，适合大数据量场景"""
        import sqlite3
        conn = sqlite3.connect(self.connection_string)
        cursor = conn.execute(self.query)
        
        for row in cursor:
            yield Document(
                page_content=row[1],  # 假设第二列是内容
                metadata={"id": row[0], "source": "database"}
            )
        conn.close()

# 使用
loader = DatabaseLoader("./data/knowledge.db", "SELECT id, content FROM articles")
docs = loader.load()
```


---

## 3. Text Splitter — 文本分块

文本分块是 RAG 中最关键的环节之一。分块质量直接影响检索效果和最终回答质量。

### 为什么需要分块

- LLM 上下文窗口有限，不能把整篇文档塞进去
- 向量检索需要细粒度的文本单元才能精确匹配
- 太大的块噪声多，太小的块缺乏上下文

### RecursiveCharacterTextSplitter — 最常用

递归字符分割器按照一组分隔符的优先级递归地分割文本，尽量保持语义完整性。

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,        # 每块最大字符数
    chunk_overlap=50,      # 块之间的重叠字符数
    length_function=len,   # 计算长度的函数
    separators=["\n\n", "\n", "。", "！", "？", ".", " ", ""],  # 中文优化
)

# 分割文档列表
docs = text_splitter.split_documents(loaded_docs)

# 分割纯文本
texts = text_splitter.split_text("你的长文本内容...")

print(f"分割后共 {len(docs)} 个块")
for i, doc in enumerate(docs[:3]):
    print(f"块 {i}: {len(doc.page_content)} 字符")
    print(f"  内容: {doc.page_content[:80]}...")
```

默认分隔符优先级：`["\n\n", "\n", " ", ""]`

工作原理：
1. 先尝试用 `\n\n`（段落）分割
2. 如果某段仍超过 chunk_size，用 `\n`（行）继续分割
3. 依次递归，直到满足大小要求

### 按 Token 分块

当需要精确控制 Token 数量时（如适配模型上下文窗口）：

```python
from langchain.text_splitter import TokenTextSplitter

# 基于 tiktoken（OpenAI 的分词器）
splitter = TokenTextSplitter(
    encoding_name="cl100k_base",  # GPT-4 使用的编码
    chunk_size=256,               # 最大 token 数
    chunk_overlap=20,             # 重叠 token 数
)

chunks = splitter.split_text(long_text)
```

### 按语义分块

利用 Embedding 模型判断语义边界，在语义变化较大的位置切分：

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# 基于语义相似度的分块
semantic_splitter = SemanticChunker(
    embeddings,
    breakpoint_threshold_type="percentile",  # 断点阈值类型
    breakpoint_threshold_amount=95,          # 百分位阈值
)

chunks = semantic_splitter.split_text(long_text)
```

### chunk_size 和 chunk_overlap 的选择策略

| 场景 | chunk_size | chunk_overlap | 说明 |
|------|-----------|---------------|------|
| 精确问答（FAQ） | 200-500 | 20-50 | 小块更精确 |
| 文档摘要 | 1000-2000 | 100-200 | 大块保留更多上下文 |
| 代码分析 | 500-1000 | 50-100 | 按函数/类分割 |
| 对话系统 | 300-800 | 50-100 | 平衡精确度和上下文 |

**经验法则**：
- `chunk_overlap` 通常为 `chunk_size` 的 10%-20%
- 中文文本 chunk_size 可以比英文小（中文信息密度更高）
- 先用默认值跑通，再根据检索效果调优

### 代码文件的特殊分块

```python
from langchain.text_splitter import (
    RecursiveCharacterTextSplitter,
    Language,
)

# LangChain 内置了多种编程语言的分隔符
python_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.PYTHON,
    chunk_size=500,
    chunk_overlap=50,
)

# 支持的语言
print(RecursiveCharacterTextSplitter.get_separators_for_language(Language.PYTHON))
# ['\nclass ', '\ndef ', '\n\tdef ', '\n\n', '\n', ' ', '']

js_splitter = RecursiveCharacterTextSplitter.from_language(
    language=Language.JS,
    chunk_size=500,
    chunk_overlap=50,
)

# 分割 Python 代码
python_code = """
class Calculator:
    def __init__(self):
        self.result = 0
    
    def add(self, x, y):
        return x + y
    
    def multiply(self, x, y):
        return x * y

def main():
    calc = Calculator()
    print(calc.add(1, 2))
"""

chunks = python_splitter.split_text(python_code)
for i, chunk in enumerate(chunks):
    print(f"--- 块 {i} ---")
    print(chunk)
```


---

## 4. Embedding 模型 — 文本向量化

Embedding 模型将文本转换为高维向量，使得语义相似的文本在向量空间中距离更近。这是向量检索的基础。

### OpenAI Embedding

```python
from langchain_openai import OpenAIEmbeddings

# 默认使用 text-embedding-ada-002
embeddings = OpenAIEmbeddings()

# 使用更新的模型
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# 嵌入单个文本
vector = embeddings.embed_query("什么是机器学习？")
print(f"向量维度: {len(vector)}")  # 1536

# 批量嵌入文档
texts = ["文档1的内容", "文档2的内容", "文档3的内容"]
vectors = embeddings.embed_documents(texts)
print(f"嵌入了 {len(vectors)} 个文档")
```

### 开源 Embedding — HuggingFace

```python
from langchain_community.embeddings import HuggingFaceEmbeddings

# 使用 BGE 中文模型（推荐中文场景）
embeddings = HuggingFaceEmbeddings(
    model_name="BAAI/bge-base-zh-v1.5",
    model_kwargs={"device": "cpu"},       # 或 "cuda"
    encode_kwargs={"normalize_embeddings": True},  # 归一化
)

# 使用 M3E 模型（中文通用）
embeddings = HuggingFaceEmbeddings(
    model_name="moka-ai/m3e-base",
)

vector = embeddings.embed_query("LangChain 是什么？")
print(f"向量维度: {len(vector)}")  # 768
```

### Embedding 模型选型对比

| 模型 | 维度 | 语言 | 特点 | 适用场景 |
|------|------|------|------|---------|
| text-embedding-3-small | 1536 | 多语言 | 性价比高，API 调用 | 快速原型、生产环境 |
| text-embedding-3-large | 3072 | 多语言 | 效果最好，成本较高 | 对精度要求极高 |
| bge-base-zh-v1.5 | 768 | 中文 | 中文效果优秀，本地部署 | 中文知识库 |
| bge-large-zh-v1.5 | 1024 | 中文 | 中文 SOTA | 中文高精度场景 |
| m3e-base | 768 | 中文 | 社区流行，效果不错 | 中文通用场景 |
| multilingual-e5-large | 1024 | 多语言 | 多语言效果好 | 多语言混合场景 |

**选型建议**：
- 快速验证 → OpenAI text-embedding-3-small
- 中文生产环境 → BGE-large-zh
- 离线/隐私要求 → 本地部署 HuggingFace 模型
- 成本敏感 → m3e-base 或 bge-base

### 计算相似度

```python
import numpy as np

def cosine_similarity(v1, v2):
    return np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))

q = embeddings.embed_query("Python 编程")
d1 = embeddings.embed_query("Python 是一种编程语言")
d2 = embeddings.embed_query("今天天气很好")

print(f"相关文档相似度: {cosine_similarity(q, d1):.4f}")  # 高
print(f"无关文档相似度: {cosine_similarity(q, d2):.4f}")  # 低
```


---

## 5. VectorStore — 向量存储

VectorStore 负责存储文档向量并提供高效的相似度检索能力。

### Chroma — 轻量级本地方案

Chroma 是一个开源的嵌入式向量数据库，适合开发和中小规模应用。

```python
from langchain_chroma import Chroma
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()

# 从文档创建向量存储
vectorstore = Chroma.from_documents(
    documents=split_docs,
    embedding=embeddings,
    collection_name="my_knowledge_base",
    persist_directory="./chroma_db",  # 持久化目录
)

# 从文本创建
vectorstore = Chroma.from_texts(
    texts=["文本1", "文本2", "文本3"],
    embedding=embeddings,
    metadatas=[{"source": "a"}, {"source": "b"}, {"source": "c"}],
    persist_directory="./chroma_db",
)

# 加载已有的向量存储
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=embeddings,
    collection_name="my_knowledge_base",
)

# 添加新文档
vectorstore.add_documents(new_docs)
vectorstore.add_texts(["新文本"], metadatas=[{"source": "new"}])
```

### FAISS — 高性能检索

FAISS（Facebook AI Similarity Search）适合大规模向量检索，性能极高。

```python
from langchain_community.vectorstores import FAISS

# 创建 FAISS 向量存储
vectorstore = FAISS.from_documents(split_docs, embeddings)

# 持久化到本地
vectorstore.save_local("./faiss_index")

# 加载
vectorstore = FAISS.load_local(
    "./faiss_index",
    embeddings,
    allow_dangerous_deserialization=True,
)

# 合并两个 FAISS 索引
vectorstore1 = FAISS.from_texts(["文本A", "文本B"], embeddings)
vectorstore2 = FAISS.from_texts(["文本C", "文本D"], embeddings)
vectorstore1.merge_from(vectorstore2)
```

### 相似度搜索 vs MMR 搜索

```python
# 1. 基础相似度搜索 — 返回最相似的 k 个文档
results = vectorstore.similarity_search(
    query="什么是深度学习？",
    k=4,
)
for doc in results:
    print(f"[{doc.metadata.get('source')}] {doc.page_content[:80]}...")

# 2. 带分数的相似度搜索
results_with_scores = vectorstore.similarity_search_with_score(
    query="什么是深度学习？",
    k=4,
)
for doc, score in results_with_scores:
    print(f"分数: {score:.4f} | {doc.page_content[:60]}...")

# 3. MMR 搜索（最大边际相关性）— 兼顾相关性和多样性
results = vectorstore.max_marginal_relevance_search(
    query="什么是深度学习？",
    k=4,              # 最终返回数量
    fetch_k=20,       # 初始候选数量
    lambda_mult=0.5,  # 0=最大多样性, 1=最大相关性
)
```

**MMR vs 普通相似度搜索**：

| 特性 | similarity_search | MMR search |
|------|------------------|------------|
| 算法 | 纯余弦相似度排序 | 相关性 + 多样性平衡 |
| 结果 | 可能有重复/冗余 | 结果更多样化 |
| 适用 | 精确匹配场景 | 需要覆盖多个方面 |
| 性能 | 更快 | 稍慢（需要额外计算） |

### 带过滤条件的搜索

```python
# Chroma 支持元数据过滤
results = vectorstore.similarity_search(
    query="机器学习算法",
    k=4,
    filter={"source": "textbook.pdf"},  # 只搜索特定来源
)

# FAISS 通过 filter 函数过滤
results = vectorstore.similarity_search(
    query="机器学习算法",
    k=4,
    filter={"category": "ml"},
)
```


---

## 6. Retriever — 检索器

Retriever 是 LangChain 中检索的抽象接口，所有 Retriever 都实现了 `get_relevant_documents(query)` 方法。

### VectorStoreRetriever 基础用法

```python
# 从 VectorStore 创建 Retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",  # "similarity" | "mmr" | "similarity_score_threshold"
    search_kwargs={
        "k": 4,                # 返回文档数
        # "score_threshold": 0.5,  # 仅 similarity_score_threshold 模式
        # "fetch_k": 20,           # 仅 mmr 模式
    }
)

# 使用
docs = retriever.invoke("什么是 RAG？")
for doc in docs:
    print(doc.page_content[:100])
```

### MultiQueryRetriever — 多查询检索

用 LLM 将原始查询改写为多个不同角度的查询，分别检索后合并去重，提高召回率。

```python
from langchain.retrievers.multi_query import MultiQueryRetriever
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    llm=llm,
)

# 内部会生成多个查询变体，分别检索后合并
docs = multi_retriever.invoke("LangChain 如何实现 RAG？")
print(f"检索到 {len(docs)} 个去重文档")
```

工作原理：
```
原始查询: "LangChain 如何实现 RAG？"
    ↓ LLM 改写
查询1: "LangChain RAG 架构是什么？"
查询2: "如何用 LangChain 构建检索增强生成系统？"
查询3: "LangChain 中 RAG 的实现步骤有哪些？"
    ↓ 分别检索
结果合并去重 → 最终文档集
```

### ContextualCompressionRetriever — 上下文压缩

检索到的文档可能包含大量无关内容。压缩检索器会用 LLM 提取与查询最相关的部分。

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

# 使用 LLM 提取相关内容
compressor = LLMChainExtractor.from_llm(llm)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 6}),
)

docs = compression_retriever.invoke("RAG 的优势是什么？")
# 返回的文档内容已被压缩，只保留与查询相关的部分
for doc in docs:
    print(f"压缩后: {doc.page_content[:100]}...")
```

### EnsembleRetriever — 混合检索（向量 + BM25）

结合稠密向量检索（语义匹配）和稀疏检索（关键词匹配），取长补短。

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# BM25 关键词检索器
bm25_retriever = BM25Retriever.from_documents(split_docs)
bm25_retriever.k = 4

# 向量检索器
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# 混合检索 — 加权融合
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6],  # BM25 权重 40%, 向量权重 60%
)

docs = ensemble_retriever.invoke("Python 装饰器的用法")
print(f"混合检索返回 {len(docs)} 个文档")
```

**为什么需要混合检索？**
- 向量检索擅长语义匹配（"汽车" ≈ "轿车"）
- BM25 擅长精确关键词匹配（专有名词、代码符号）
- 混合使用可以覆盖更多场景

### ParentDocumentRetriever — 父文档检索

解决"小块检索精确但上下文不足"的问题：用小块做检索，返回对应的大块（父文档）。

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain.text_splitter import RecursiveCharacterTextSplitter

# 父文档分割器（大块）
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)

# 子文档分割器（小块，用于检索）
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400, chunk_overlap=50)

# 存储父文档的 store
store = InMemoryStore()

parent_retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

# 添加文档（会自动分割并建立父子关系）
parent_retriever.add_documents(original_docs)

# 检索时：用小块匹配，返回对应的大块
docs = parent_retriever.invoke("什么是注意力机制？")
# 返回的是包含更多上下文的父文档块
```

工作原理：
```
原始文档 → 父分割器 → 大块（2000字符）→ 存入 docstore
                         ↓
                    子分割器 → 小块（400字符）→ 存入 vectorstore
                    
检索时：query → 匹配小块 → 找到对应父块 → 返回父块
```


---

## 7. RAG Chain 构建

将以上组件串联起来，构建完整的 RAG 问答链。

### 基础 RAG Chain（LCEL 方式）

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
retriever = vectorstore.as_retriever(search_kwargs={"k": 4})

# RAG Prompt 模板
template = """基于以下上下文回答用户的问题。如果上下文中没有相关信息，请说"我无法从已有资料中找到答案"。

上下文：
{context}

问题：{question}

回答："""

prompt = ChatPromptTemplate.from_template(template)

# 辅助函数：将文档列表格式化为字符串
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

# 构建 RAG Chain（LCEL 管道语法）
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 使用
answer = rag_chain.invoke("什么是 RAG？它有什么优势？")
print(answer)
```

### 带 Source 引用的 RAG

让模型在回答时标注信息来源，增强可信度。

```python
from langchain_core.runnables import RunnableParallel

# 同时返回答案和源文档
def format_docs_with_source(docs):
    formatted = []
    for i, doc in enumerate(docs):
        source = doc.metadata.get("source", "未知")
        formatted.append(f"[来源{i+1}: {source}]\n{doc.page_content}")
    return "\n\n".join(formatted)

template_with_source = """基于以下上下文回答问题。请在回答中引用来源编号。

上下文：
{context}

问题：{question}

请回答问题并标注引用来源（如 [来源1]）："""

prompt_with_source = ChatPromptTemplate.from_template(template_with_source)

# 构建带源引用的 Chain
rag_chain_with_source = (
    RunnableParallel(
        context=retriever | format_docs_with_source,
        question=RunnablePassthrough(),
        source_documents=retriever,  # 同时保留原始文档
    )
    | RunnableParallel(
        answer=prompt_with_source | llm | StrOutputParser(),
        sources=lambda x: [doc.metadata for doc in x["source_documents"]],
    )
)

result = rag_chain_with_source.invoke("RAG 的核心组件有哪些？")
print("回答:", result["answer"])
print("来源:", result["sources"])
```

### 对话式 RAG（带历史记忆）

支持多轮对话，能理解上下文中的指代关系。

```python
from langchain_core.prompts import MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage
from langchain.chains import create_history_aware_retriever, create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain

# 1. 构建历史感知检索器 — 将对话历史融入检索查询
contextualize_q_prompt = ChatPromptTemplate.from_messages([
    ("system", "根据对话历史和最新问题，重新表述一个独立的检索查询。"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

history_aware_retriever = create_history_aware_retriever(
    llm, retriever, contextualize_q_prompt
)

# 2. 构建问答链
qa_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个知识库助手。基于以下上下文回答问题。\n\n{context}"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
])

question_answer_chain = create_stuff_documents_chain(llm, qa_prompt)

# 3. 组合为完整的对话式 RAG Chain
conversational_rag_chain = create_retrieval_chain(
    history_aware_retriever, question_answer_chain
)

# 使用 — 多轮对话
chat_history = []

# 第一轮
response = conversational_rag_chain.invoke({
    "input": "什么是 LangChain？",
    "chat_history": chat_history,
})
print("A1:", response["answer"])
chat_history.extend([
    HumanMessage(content="什么是 LangChain？"),
    AIMessage(content=response["answer"]),
])

# 第二轮 — "它"指代 LangChain
response = conversational_rag_chain.invoke({
    "input": "它支持哪些向量数据库？",
    "chat_history": chat_history,
})
print("A2:", response["answer"])
```

### 多文档 RAG

处理多个不同来源的文档，支持按来源路由检索。

```python
from langchain_core.runnables import RunnableLambda

# 为不同文档集创建独立的向量存储
tech_vectorstore = Chroma.from_documents(tech_docs, embeddings, collection_name="tech")
business_vectorstore = Chroma.from_documents(biz_docs, embeddings, collection_name="biz")

tech_retriever = tech_vectorstore.as_retriever(search_kwargs={"k": 3})
business_retriever = business_vectorstore.as_retriever(search_kwargs={"k": 3})

# 路由函数 — 根据问题类型选择检索器
def route_retriever(query: str):
    """简单的关键词路由"""
    tech_keywords = ["代码", "API", "技术", "实现", "bug", "错误"]
    if any(kw in query for kw in tech_keywords):
        return tech_retriever.invoke(query)
    else:
        return business_retriever.invoke(query)

# 或者用 EnsembleRetriever 合并多个来源
from langchain.retrievers import EnsembleRetriever

multi_source_retriever = EnsembleRetriever(
    retrievers=[tech_retriever, business_retriever],
    weights=[0.5, 0.5],
)

# 构建多文档 RAG Chain
multi_rag_chain = (
    {"context": multi_source_retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)
```


---

## 8. RAG 评估与优化

RAG 系统的效果取决于检索和生成两个环节，需要分别评估和优化。

### 检索质量评估

```python
from typing import List

def evaluate_retrieval(
    queries: List[str],
    expected_docs: List[List[str]],  # 每个查询的期望文档ID
    retriever,
    k: int = 4,
) -> dict:
    """评估检索质量"""
    total_recall = 0
    total_precision = 0
    total_mrr = 0
    
    for query, expected in zip(queries, expected_docs):
        retrieved = retriever.invoke(query)
        retrieved_ids = [doc.metadata.get("id", "") for doc in retrieved[:k]]
        
        # Recall@K: 期望文档中有多少被检索到
        hits = len(set(retrieved_ids) & set(expected))
        recall = hits / len(expected) if expected else 0
        total_recall += recall
        
        # Precision@K: 检索结果中有多少是相关的
        precision = hits / k
        total_precision += precision
        
        # MRR: 第一个相关文档的排名倒数
        for i, doc_id in enumerate(retrieved_ids):
            if doc_id in expected:
                total_mrr += 1 / (i + 1)
                break
    
    n = len(queries)
    return {
        "recall@k": total_recall / n,
        "precision@k": total_precision / n,
        "mrr": total_mrr / n,
    }

# 使用示例
eval_queries = ["什么是 RAG？", "向量数据库有哪些？"]
eval_expected = [["doc_001", "doc_002"], ["doc_010", "doc_011"]]
metrics = evaluate_retrieval(eval_queries, eval_expected, retriever)
print(f"Recall@4: {metrics['recall@k']:.3f}")
print(f"MRR: {metrics['mrr']:.3f}")
```

### 生成质量评估

```python
from langchain_openai import ChatOpenAI

eval_llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

def evaluate_answer(question: str, answer: str, context: str, reference: str) -> dict:
    """用 LLM 评估生成质量"""
    
    # 忠实度评估：答案是否基于上下文
    faithfulness_prompt = f"""评估以下回答是否忠实于给定的上下文（不包含上下文之外的信息）。
    
上下文: {context}
回答: {answer}

评分（1-5）并简要说明原因。格式：分数|原因"""
    
    # 相关性评估：答案是否回答了问题
    relevance_prompt = f"""评估以下回答是否充分回答了问题。

问题: {question}
回答: {answer}

评分（1-5）并简要说明原因。格式：分数|原因"""
    
    faithfulness = eval_llm.invoke(faithfulness_prompt).content
    relevance = eval_llm.invoke(relevance_prompt).content
    
    return {
        "faithfulness": faithfulness,
        "relevance": relevance,
    }
```

### 常见问题与优化策略

| 问题 | 原因 | 优化策略 |
|------|------|---------|
| 检索不到相关文档 | 查询与文档表述差异大 | 使用 MultiQueryRetriever；优化 chunk 策略 |
| 检索到但答案不对 | 上下文噪声太多 | 使用 ContextualCompression；减小 chunk_size |
| 回答包含幻觉 | Prompt 约束不够 | 强化 Prompt 中"仅基于上下文回答"的指令 |
| 无法处理多跳问题 | 单次检索不够 | 使用迭代检索或 Agent 方式 |
| 专有名词检索差 | 向量检索对精确匹配弱 | 混合检索（向量 + BM25） |
| 长文档上下文丢失 | 分块切断了语义 | 增大 chunk_overlap；使用 ParentDocumentRetriever |

### 优化清单

```
┌─────────────────────────────────────────────────────┐
│              RAG 优化路线图                           │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Level 1: 基础优化                                   │
│  ├── 调整 chunk_size / chunk_overlap                │
│  ├── 选择合适的 Embedding 模型                       │
│  └── 优化 Prompt 模板                               │
│                                                     │
│  Level 2: 检索优化                                   │
│  ├── 混合检索（向量 + BM25）                         │
│  ├── MultiQuery 多角度检索                           │
│  ├── 重排序（Reranking）                            │
│  └── 元数据过滤                                     │
│                                                     │
│  Level 3: 高级优化                                   │
│  ├── 查询路由（按类型分发）                           │
│  ├── 自适应检索（判断是否需要检索）                    │
│  ├── 迭代检索（多轮检索补充信息）                     │
│  └── 知识图谱增强                                   │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Reranking 重排序示例

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder

# 使用 Cross-Encoder 重排序
model = HuggingFaceCrossEncoder(model_name="BAAI/bge-reranker-base")
compressor = CrossEncoderReranker(model=model, top_n=3)

reranking_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 10}),
)

# 先粗检索 10 个，再精排取 top 3
docs = reranking_retriever.invoke("深度学习的反向传播算法")
```


---

## 9. 完整实战 — 从零构建知识库问答系统

下面我们将所有组件串联，构建一个完整的知识库问答系统。

### 项目结构

```
rag_project/
├── data/                  # 知识库文档
│   ├── docs/
│   └── ...
├── vectorstore/           # 持久化向量存储
├── rag_system.py          # 核心系统代码
└── requirements.txt
```

### 完整代码

```python
"""
rag_system.py — 完整的知识库问答系统
"""
import os
from pathlib import Path
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma
from langchain_community.document_loaders import (
    TextLoader, PyPDFLoader, UnstructuredMarkdownLoader, DirectoryLoader
)
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.messages import HumanMessage, AIMessage
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain.chains import create_history_aware_retriever, create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain


class RAGSystem:
    """知识库问答系统"""
    
    def __init__(
        self,
        persist_dir: str = "./vectorstore",
        model_name: str = "gpt-4o-mini",
        embedding_model: str = "text-embedding-3-small",
        chunk_size: int = 500,
        chunk_overlap: int = 50,
    ):
        self.persist_dir = persist_dir
        self.embeddings = OpenAIEmbeddings(model=embedding_model)
        self.llm = ChatOpenAI(model=model_name, temperature=0)
        self.text_splitter = RecursiveCharacterTextSplitter(
            chunk_size=chunk_size,
            chunk_overlap=chunk_overlap,
            separators=["\n\n", "\n", "。", "！", "？", ".", " ", ""],
        )
        self.vectorstore = None
        self.chat_history = []
        
        # 尝试加载已有向量存储
        if Path(persist_dir).exists():
            self.vectorstore = Chroma(
                persist_directory=persist_dir,
                embedding_function=self.embeddings,
            )
            print(f"已加载向量存储，共 {self.vectorstore._collection.count()} 个文档块")
    
    def load_documents(self, data_dir: str) -> list:
        """加载目录下所有支持的文档"""
        docs = []
        data_path = Path(data_dir)
        
        # 加载不同格式的文件
        loaders = {
            "*.txt": TextLoader,
            "*.md": UnstructuredMarkdownLoader,
            "*.pdf": PyPDFLoader,
        }
        
        for pattern, loader_cls in loaders.items():
            for file_path in data_path.rglob(pattern):
                try:
                    loader = loader_cls(str(file_path), encoding="utf-8") \
                        if loader_cls == TextLoader else loader_cls(str(file_path))
                    file_docs = loader.load()
                    # 添加文件名到元数据
                    for doc in file_docs:
                        doc.metadata["filename"] = file_path.name
                    docs.extend(file_docs)
                    print(f"  已加载: {file_path.name} ({len(file_docs)} 页)")
                except Exception as e:
                    print(f"  加载失败: {file_path.name} - {e}")
        
        print(f"\n共加载 {len(docs)} 个文档")
        return docs
    
    def build_index(self, data_dir: str):
        """构建向量索引"""
        print("=== 开始构建知识库索引 ===\n")
        
        # 1. 加载文档
        docs = self.load_documents(data_dir)
        if not docs:
            print("未找到任何文档！")
            return
        
        # 2. 分块
        splits = self.text_splitter.split_documents(docs)
        print(f"分块完成，共 {len(splits)} 个文档块")
        
        # 3. 构建向量存储
        self.vectorstore = Chroma.from_documents(
            documents=splits,
            embedding=self.embeddings,
            persist_directory=self.persist_dir,
        )
        print(f"向量索引构建完成，已持久化到 {self.persist_dir}")
    
    def get_retriever(self, search_type="mmr", k=4):
        """获取检索器"""
        if not self.vectorstore:
            raise ValueError("请先构建索引（调用 build_index）")
        return self.vectorstore.as_retriever(
            search_type=search_type,
            search_kwargs={"k": k, "fetch_k": 20},
        )
    
    def query(self, question: str) -> str:
        """单轮问答"""
        retriever = self.get_retriever()
        
        template = """你是一个知识库助手。请基于以下上下文回答用户的问题。
如果上下文中没有相关信息，请明确告知用户。回答要准确、简洁。

上下文：
{context}

问题：{question}

回答："""
        
        prompt = ChatPromptTemplate.from_template(template)
        
        def format_docs(docs):
            return "\n\n---\n\n".join(
                f"[来源: {d.metadata.get('filename', '未知')}]\n{d.page_content}"
                for d in docs
            )
        
        chain = (
            {"context": retriever | format_docs, "question": RunnablePassthrough()}
            | prompt
            | self.llm
            | StrOutputParser()
        )
        
        return chain.invoke(question)
    
    def chat(self, question: str) -> str:
        """多轮对话问答"""
        retriever = self.get_retriever()
        
        # 历史感知检索
        contextualize_prompt = ChatPromptTemplate.from_messages([
            ("system", "根据对话历史，将用户最新问题改写为独立的检索查询。如果问题已经独立，直接返回原问题。"),
            MessagesPlaceholder("chat_history"),
            ("human", "{input}"),
        ])
        
        history_retriever = create_history_aware_retriever(
            self.llm, retriever, contextualize_prompt
        )
        
        # 问答
        qa_prompt = ChatPromptTemplate.from_messages([
            ("system", "你是知识库助手。基于上下文回答问题，不确定时说明。\n\n上下文：\n{context}"),
            MessagesPlaceholder("chat_history"),
            ("human", "{input}"),
        ])
        
        qa_chain = create_stuff_documents_chain(self.llm, qa_prompt)
        rag_chain = create_retrieval_chain(history_retriever, qa_chain)
        
        response = rag_chain.invoke({
            "input": question,
            "chat_history": self.chat_history,
        })
        
        # 更新历史
        self.chat_history.extend([
            HumanMessage(content=question),
            AIMessage(content=response["answer"]),
        ])
        
        return response["answer"]
    
    def clear_history(self):
        """清除对话历史"""
        self.chat_history = []


# ===== 使用示例 =====
if __name__ == "__main__":
    # 初始化系统
    rag = RAGSystem(
        persist_dir="./vectorstore",
        chunk_size=500,
        chunk_overlap=50,
    )
    
    # 构建索引（首次运行）
    rag.build_index("./data/docs")
    
    # 单轮问答
    answer = rag.query("什么是机器学习？")
    print(f"Q: 什么是机器学习？\nA: {answer}\n")
    
    # 多轮对话
    print("--- 多轮对话模式 ---")
    a1 = rag.chat("介绍一下深度学习")
    print(f"A1: {a1}\n")
    
    a2 = rag.chat("它和传统机器学习有什么区别？")  # "它"指代深度学习
    print(f"A2: {a2}\n")
    
    a3 = rag.chat("有哪些常见的应用场景？")
    print(f"A3: {a3}\n")
```

### 运行效果

```bash
$ python rag_system.py

=== 开始构建知识库索引 ===

  已加载: ml_intro.txt (1 页)
  已加载: deep_learning.pdf (15 页)
  已加载: nlp_guide.md (1 页)

共加载 17 个文档
分块完成，共 89 个文档块
向量索引构建完成，已持久化到 ./vectorstore

Q: 什么是机器学习？
A: 机器学习是人工智能的一个分支，它使计算机系统能够从数据中学习和改进，
   而无需进行明确的编程。通过识别数据中的模式，机器学习算法可以做出预测
   或决策... [来源: ml_intro.txt]
```


---

## 10. 面试题精选

### Q1: RAG 和 Fine-tuning 的区别是什么？各自适用什么场景？

**答案**：

| 维度 | RAG | Fine-tuning |
|------|-----|-------------|
| 知识更新 | 实时更新，修改文档即可 | 需要重新训练 |
| 成本 | 低（只需向量数据库） | 高（GPU 训练） |
| 幻觉控制 | 好（有明确来源） | 较差 |
| 领域适应 | 适合知识密集型任务 | 适合风格/格式适应 |
| 可解释性 | 高（可追溯来源） | 低 |

**适用场景**：
- RAG：企业知识库问答、客服系统、文档检索、需要实时更新的场景
- Fine-tuning：特定写作风格、专业术语理解、输出格式控制
- 最佳实践：两者结合 — Fine-tune 模型理解领域语言 + RAG 提供具体知识

### Q2: chunk_size 设置过大或过小分别有什么问题？如何选择？

**答案**：

**过大的问题**：
- 向量表示模糊，检索精度下降
- 包含过多无关信息，增加噪声
- 可能超出 Embedding 模型的最大输入长度

**过小的问题**：
- 缺乏上下文，语义不完整
- 检索到的片段可能无法独立回答问题
- 索引数量膨胀，存储和检索成本增加

**选择策略**：
1. 根据文档类型：FAQ → 200-400，技术文档 → 500-1000，法律合同 → 1000-2000
2. 根据问题类型：精确事实 → 小块，综合分析 → 大块
3. 实验验证：准备测试集，对比不同 chunk_size 的检索效果
4. 考虑使用 ParentDocumentRetriever：小块检索 + 大块返回

### Q3: 如何解决 RAG 系统中的"检索到了但回答错误"问题？

**答案**：

这通常是"Lost in the Middle"问题或上下文噪声问题。解决方案：

1. **减少噪声**：
   - 使用 ContextualCompressionRetriever 压缩无关内容
   - 添加 Reranker 重排序，确保最相关的文档排在前面
   - 减小 k 值，只保留最相关的 2-3 个文档

2. **优化 Prompt**：
   - 明确指示"仅基于上下文回答"
   - 要求模型标注信息来源
   - 添加"如果上下文不足以回答，请说明"的指令

3. **改进检索**：
   - 使用 MMR 搜索增加结果多样性
   - 混合检索（向量 + BM25）覆盖不同匹配模式
   - 元数据过滤缩小搜索范围

4. **架构优化**：
   - 将长上下文中最相关的内容放在开头和结尾（避免 Lost in the Middle）
   - 使用 Map-Reduce 方式分别处理每个文档再汇总

### Q4: 向量检索和关键词检索（BM25）各有什么优缺点？为什么要混合使用？

**答案**：

**向量检索（Dense Retrieval）**：
- 优点：理解语义（"汽车" ≈ "轿车" ≈ "车辆"），跨语言能力
- 缺点：对精确关键词匹配弱，对专有名词/代码/数字不敏感

**BM25（Sparse Retrieval）**：
- 优点：精确关键词匹配强，对专有名词友好，无需训练
- 缺点：无法理解同义词，对表述差异敏感

**混合使用的原因**：
```
查询: "Python GIL 是什么？"

向量检索可能返回: "Python 的全局解释器锁限制了多线程并行..."  ✓ 语义相关
BM25 可能返回: "GIL (Global Interpreter Lock) 是 CPython 的..." ✓ 精确匹配

两者互补，覆盖更全面
```

实现方式：EnsembleRetriever 加权融合，通常向量权重 0.5-0.7，BM25 权重 0.3-0.5。

### Q5: 如何评估一个 RAG 系统的效果？有哪些关键指标？

**答案**：

RAG 评估分为两个维度：

**检索质量指标**：
| 指标 | 含义 | 计算方式 |
|------|------|---------|
| Recall@K | 相关文档被检索到的比例 | 命中数 / 总相关文档数 |
| Precision@K | 检索结果中相关文档的比例 | 命中数 / K |
| MRR | 第一个相关结果的排名 | 1 / 首个相关文档排名 |
| NDCG | 考虑排序的检索质量 | 归一化折损累积增益 |

**生成质量指标**：
| 指标 | 含义 | 评估方式 |
|------|------|---------|
| Faithfulness | 答案是否忠实于上下文 | LLM 评估 / 人工标注 |
| Relevance | 答案是否回答了问题 | LLM 评估 / 人工标注 |
| Completeness | 答案是否完整 | 与参考答案对比 |
| Hallucination | 是否包含编造信息 | 检查答案中的事实是否有上下文支撑 |

**评估工具**：
- RAGAS：专门的 RAG 评估框架
- LangSmith：LangChain 官方的追踪和评估平台
- 自建评估：用 GPT-4 作为评判者

### Q6: 生产环境中 RAG 系统需要注意哪些问题？

**答案**：

1. **性能优化**：
   - Embedding 计算缓存，避免重复计算
   - 向量数据库选型（小规模用 Chroma，大规模用 Milvus/Pinecone）
   - 异步检索，减少延迟

2. **数据管理**：
   - 增量更新机制（新文档入库、旧文档删除）
   - 文档版本管理
   - 元数据标准化

3. **安全性**：
   - 权限控制（不同用户只能检索授权文档）
   - 输入过滤（防止 Prompt 注入）
   - 输出审核（防止泄露敏感信息）

4. **可观测性**：
   - 记录每次检索的查询、结果、耗时
   - 用户反馈收集（点赞/点踩）
   - 定期评估检索质量

5. **容错处理**：
   - 检索为空时的兜底策略
   - LLM 调用失败的重试机制
   - 超时控制

### Q7: 什么是 Hypothetical Document Embedding (HyDE)？它如何改善检索效果？

**答案**：

HyDE 的核心思想：**先让 LLM 生成一个假设性的答案文档，再用这个假设文档去检索，而不是直接用原始查询检索。**

```python
from langchain.chains import HypotheticalDocumentEmbedder

hyde_embeddings = HypotheticalDocumentEmbedder.from_llm(
    llm=ChatOpenAI(model="gpt-4o-mini"),
    base_embeddings=OpenAIEmbeddings(),
    prompt_key="web_search",  # 内置的 prompt 模板
)

# 使用 HyDE embedding 进行检索
vectorstore = Chroma.from_documents(docs, hyde_embeddings)
```

**原理**：
```
原始查询: "量子计算的优势"
    ↓ LLM 生成假设文档
假设文档: "量子计算相比经典计算具有指数级加速优势，特别是在..."
    ↓ Embedding
假设文档向量 → 与知识库中的真实文档向量更接近
    ↓ 检索
返回真正相关的文档
```

**优势**：查询通常很短且抽象，而文档是详细的描述。HyDE 将短查询"扩展"为类似文档的形式，缩小了查询和文档之间的表示差距。

---

## 总结

本文完整介绍了用 LangChain 构建 RAG 系统的全流程：

```
文档加载 → 文本分块 → 向量化 → 存储 → 检索 → 生成
```

关键要点：
1. **分块策略**是 RAG 效果的基础，需要根据场景调优
2. **混合检索**（向量 + BM25）通常优于单一检索方式
3. **Reranking** 可以显著提升检索精度
4. **对话式 RAG** 需要处理好历史上下文和查询改写
5. **评估驱动优化** — 建立评估体系，用数据指导优化方向

下一步学习建议：
- 尝试不同的 Embedding 模型和 chunk 策略，对比效果
- 学习 LangSmith 进行 RAG 系统的追踪和调试
- 探索 Agent + RAG 的结合，实现更复杂的问答场景


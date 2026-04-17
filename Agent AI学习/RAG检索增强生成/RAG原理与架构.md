# RAG 原理与架构

## 1. 什么是 RAG

RAG（Retrieval-Augmented Generation，检索增强生成）是一种让 LLM 基于外部知识回答问题的技术。它解决了 LLM 的两个核心问题：

- **知识过时**：LLM 训练数据有截止日期
- **幻觉问题**：LLM 可能编造不存在的信息

### 核心思路

```
传统 LLM:  问题 → LLM → 回答（可能不准确）

RAG:       问题 → 检索相关文档 → 文档 + 问题 → LLM → 回答（基于事实）
```

## 2. RAG 完整流程

```
┌──────────────────────────────────────────────────┐
│                   离线阶段（索引）                  │
│                                                    │
│  文档 → 分块 → Embedding → 存入向量数据库           │
│                                                    │
├──────────────────────────────────────────────────┤
│                   在线阶段（查询）                  │
│                                                    │
│  用户问题 → Embedding → 向量检索 → Top-K 文档       │
│                    ↓                               │
│  Prompt = 系统指令 + 检索到的文档 + 用户问题         │
│                    ↓                               │
│               LLM 生成回答                          │
└──────────────────────────────────────────────────┘
```

## 3. 基础实现

### 3.1 最简 RAG（使用 LangChain）

```python
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import DirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# 1. 加载文档
loader = DirectoryLoader("./docs", glob="**/*.md")
documents = loader.load()

# 2. 文档分块
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=["\n## ", "\n### ", "\n\n", "\n", " "]
)
chunks = splitter.split_documents(documents)

# 3. 创建向量索引
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(chunks, embeddings, persist_directory="./db")

# 4. 构建检索链
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

prompt = ChatPromptTemplate.from_template("""
基于以下参考文档回答问题。如果文档中没有相关信息，请明确说明。

参考文档:
{context}

问题: {question}

回答:""")

llm = ChatOpenAI(model="gpt-4o", temperature=0)

chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# 5. 查询
answer = chain.invoke("如何优化 Android 启动速度?")
print(answer)
```

### 3.2 使用 LlamaIndex 实现

```python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

documents = SimpleDirectoryReader("./docs").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine(similarity_top_k=5)

response = query_engine.query("如何优化 Android 启动速度?")
print(response)
print("来源:", [n.metadata for n in response.source_nodes])
```

## 4. 文档分块策略

分块质量直接影响检索效果，是 RAG 最关键的环节之一。

### 4.1 常用分块方法

| 方法 | 原理 | 适用场景 |
|------|------|---------|
| 固定大小 | 按字符/Token 数切分 | 通用，简单快速 |
| 递归分割 | 按分隔符层级切分 | Markdown、代码 |
| 语义分块 | 按语义相似度切分 | 需要高质量分块 |
| 文档结构 | 按标题/段落切分 | 结构化文档 |

### 4.2 递归分割（推荐）

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=200,
    separators=[
        "\n## ",    # 先按二级标题分
        "\n### ",   # 再按三级标题
        "\n\n",     # 再按段落
        "\n",       # 再按行
        " ",        # 最后按空格
    ]
)
```

### 4.3 语义分块

```python
from langchain_experimental.text_splitter import SemanticChunker

splitter = SemanticChunker(
    embeddings=OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95
)
# 当相邻句子的语义相似度低于阈值时切分
chunks = splitter.split_documents(documents)
```

### 4.4 分块参数调优

```
chunk_size 太小 → 上下文不完整，检索到碎片信息
chunk_size 太大 → 包含无关信息，降低精度
chunk_overlap 太小 → 可能切断关键信息
chunk_overlap 太大 → 冗余增加，成本上升

推荐起始值:
- chunk_size: 500-1000 tokens
- chunk_overlap: 10%-20% of chunk_size
- 根据实际效果迭代调整
```

## 5. 检索策略

### 5.1 基础向量检索

```python
# 相似度检索
results = vectorstore.similarity_search("启动优化", k=5)

# 带分数的检索
results = vectorstore.similarity_search_with_score("启动优化", k=5)
for doc, score in results:
    print(f"[{score:.3f}] {doc.page_content[:100]}")
```

### 5.2 混合检索（向量 + 关键词）

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# BM25 关键词检索
bm25_retriever = BM25Retriever.from_documents(chunks)
bm25_retriever.k = 5

# 向量检索
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# 混合检索（加权融合）
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]  # 关键词 40%，向量 60%
)

results = ensemble_retriever.invoke("Binder 机制原理")
```

### 5.3 元数据过滤

```python
# 添加元数据
for chunk in chunks:
    chunk.metadata["category"] = "性能优化"
    chunk.metadata["source"] = "启动优化.md"

# 检索时过滤
results = vectorstore.similarity_search(
    "冷启动优化",
    k=5,
    filter={"category": "性能优化"}
)
```

## 6. 查询优化

### 6.1 查询改写

```python
# 让 LLM 优化用户的查询
rewrite_prompt = ChatPromptTemplate.from_template("""
将以下用户问题改写为更适合向量检索的查询。
生成 3 个不同角度的查询。

原始问题: {question}

改写查询（每行一个）:""")

rewrite_chain = rewrite_prompt | llm | StrOutputParser()
queries = rewrite_chain.invoke({"question": "app 打开慢怎么办"})
# 输出:
# 1. Android 应用启动速度优化方案
# 2. 冷启动耗时分析与优化
# 3. Application onCreate 优化策略
```

### 6.2 HyDE（假设性文档嵌入）

```python
# 先让 LLM 生成一个"假设性回答"，用这个回答去检索
hyde_prompt = ChatPromptTemplate.from_template("""
请回答以下问题（即使不确定也请尝试回答）:
{question}
""")

# 用假设性回答的 embedding 去检索，往往比直接用问题检索效果更好
hypothetical_answer = (hyde_prompt | llm | StrOutputParser()).invoke(
    {"question": "如何优化冷启动"}
)
results = vectorstore.similarity_search(hypothetical_answer, k=5)
```

### 6.3 重排序（Reranking）

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain_cohere import CohereRerank

# 先粗检索 20 条，再用 Reranker 精排取 Top 5
reranker = CohereRerank(model="rerank-v3.5", top_n=5)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20})
)

results = compression_retriever.invoke("Binder 通信机制")
```

## 7. 评估 RAG 效果

### 7.1 关键指标

| 指标 | 含义 | 评估方法 |
|------|------|---------|
| 检索召回率 | 相关文档是否被检索到 | 人工标注 + 自动评估 |
| 检索精度 | 检索结果中相关文档的比例 | Top-K 精度 |
| 回答准确性 | 生成的回答是否正确 | LLM-as-Judge |
| 回答忠实度 | 回答是否基于检索到的文档 | 幻觉检测 |

### 7.2 使用 RAGAS 评估

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

# 准备评估数据
eval_data = {
    "question": ["如何优化冷启动?"],
    "answer": [rag_answer],
    "contexts": [[doc.page_content for doc in retrieved_docs]],
    "ground_truth": ["通过延迟初始化、异步加载等方式优化..."]
}

result = evaluate(
    dataset=eval_data,
    metrics=[faithfulness, answer_relevancy, context_precision]
)
print(result)
```

## 8. 生产环境架构

```
                    ┌─────────────┐
                    │   用户请求    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  API Gateway │
                    └──────┬──────┘
                           │
              ┌────────────▼────────────┐
              │      RAG Service        │
              │  ┌──────────────────┐   │
              │  │ 查询改写 + 路由   │   │
              │  └────────┬─────────┘   │
              │           │             │
              │  ┌────────▼─────────┐   │
              │  │  混合检索引擎     │   │
              │  │  (向量 + BM25)   │   │
              │  └────────┬─────────┘   │
              │           │             │
              │  ┌────────▼─────────┐   │
              │  │  Reranker 重排序  │   │
              │  └────────┬─────────┘   │
              │           │             │
              │  ┌────────▼─────────┐   │
              │  │  LLM 生成回答     │   │
              │  └──────────────────┘   │
              └─────────────────────────┘
                     │           │
          ┌──────────▼──┐  ┌────▼────────┐
          │ 向量数据库   │  │ 文档存储     │
          │ (Milvus/    │  │ (S3/MinIO)  │
          │  Qdrant)    │  │             │
          └─────────────┘  └─────────────┘
```

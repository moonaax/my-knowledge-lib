# RAG 原理与架构


> **本文定位：** RAG 是目前最实用的 LLM 应用技术——让 LLM 基于你的私有数据回答问题，而不是瞎编。本文从原理到实现，覆盖分块策略、检索策略、查询优化、效果评估的完整知识链。
## 1. 什么是 RAG

RAG（Retrieval-Augmented Generation，检索增强生成）是一种让 LLM 基于外部知识回答问题的技术。它解决了 LLM 的两个核心问题：

- **知识过时**：LLM 训练数据有截止日期
- **幻觉问题**：LLM 可能编造不存在的信息

### 核心思路

````
传统 LLM:  问题 → LLM → 回答（可能不准确）

RAG:       问题 → 检索相关文档 → 文档 + 问题 → LLM → 回答（基于事实）
````
## 2. RAG 完整流程

````
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
````

> **从流程图到代码：** 下面分别用 LangChain 和 LlamaIndex 实现基础 RAG。LangChain 版本更灵活（每步可定制），LlamaIndex 版本更简洁（高度封装）。
## 3. 基础实现

### 3.1 最简 RAG（使用 LangChain）


> **思路：** 用 LangChain 实现最基础的 RAG，5 个步骤：加载文档 → 分块 → 创建向量索引 → 构建检索链 → 查询。这是所有 RAG 系统的骨架。
````python
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

> **关键代码解读（LangChain 版）：**
> - `RecursiveCharacterTextSplitter` — 按层级分隔符递归分块，优先在标题处切割
> - `Chroma.from_documents()` — 自动完成 Embedding + 存储
> - `create_stuff_documents_chain` — 把检索到的文档"塞入"prompt
> - `create_retrieval_chain` — 组装完整的 RAG 链：检索 → 拼接 → 生成
> - `response["answer"]` 是 LLM 的回答，`response["context"]` 是检索到的原始文档
````
### 3.2 使用 LlamaIndex 实现

````python
from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

documents = SimpleDirectoryReader("./docs").load_data()
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine(similarity_top_k=5)

response = query_engine.query("如何优化 Android 启动速度?")
print(response)
print("来源:", [n.metadata for n in response.source_nodes])

> **关键代码解读（LlamaIndex 版）：** 只需 3 行核心代码——加载文档、构建索引、查询。LlamaIndex 把分块、Embedding、存储、检索、生成全部封装好了，比 LangChain 版更简洁。
````

> **分块是 RAG 中最容易被忽视但影响最大的环节：** 块太大导致检索不精确（噪声多），块太小导致上下文不完整。分块策略直接决定了检索质量的上限。
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

````python
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
````
### 4.3 语义分块

````python
from langchain_experimental.text_splitter import SemanticChunker

splitter = SemanticChunker(
    embeddings=OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",
    breakpoint_threshold_amount=95
)
# 当相邻句子的语义相似度低于阈值时切分
chunks = splitter.split_documents(documents)

> **关键代码解读（语义分块）：** 不按固定大小切割，而是计算相邻句子的语义相似度，当相似度低于阈值时切分。这样每个块内的内容语义连贯，不会把相关内容切断。
````
### 4.4 分块参数调优

````
chunk_size 太小 → 上下文不完整，检索到碎片信息
chunk_size 太大 → 包含无关信息，降低精度
chunk_overlap 太小 → 可能切断关键信息
chunk_overlap 太大 → 冗余增加，成本上升

推荐起始值:
- chunk_size: 500-1000 tokens
- chunk_overlap: 10%-20% of chunk_size
- 根据实际效果迭代调整
````

> **检索策略决定了 RAG 的质量上限：** 分块决定了"有什么可以检索"，检索策略决定了"能不能找到最相关的"。从基础向量检索到混合检索，精度逐步提升。
## 5. 检索策略

### 5.1 基础向量检索

````python
# 相似度检索
results = vectorstore.similarity_search("启动优化", k=5)

# 带分数的检索
results = vectorstore.similarity_search_with_score("启动优化", k=5)
for doc, score in results:
    print(f"[{score:.3f}] {doc.page_content[:100]}")
````
### 5.2 混合检索（向量 + 关键词）


> **思路：** 纯向量检索可能漏掉包含精确关键词的文档（如 API 名称、错误码）。混合检索结合 BM25 关键词检索和向量语义检索，取长补短。
````python
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

> **关键代码解读（混合检索）：**
> - BM25 基于关键词匹配（精确但不理解语义），向量检索基于语义相似度（理解语义但可能漏掉关键词）
> - `EnsembleRetriever` 加权融合两者结果，`weights=[0.5, 0.5]` 表示各占一半权重
> - 混合检索兼顾了精确匹配和语义理解，效果通常优于单一检索
````
### 5.3 元数据过滤

````python
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

> **关键代码解读（元数据过滤）：** 给文档块附加元数据（来源、类型、日期等），检索时可以按元数据过滤，缩小搜索范围。比如只搜索"API 文档"类型的内容。
````

> **检索到了不一定准，查询优化让检索更精准：** 用户的问题可能模糊、口语化，直接检索效果不好。查询改写、HyDE、重排序是三个最有效的优化手段。
## 6. 查询优化

### 6.1 查询改写

````python
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

> **关键代码解读（查询改写）：** 用户的原始问题可能模糊或口语化。让 LLM 把一个问题改写为多个更精确的查询，分别检索后合并结果，召回率显著提升。
````
### 6.2 HyDE（假设性文档嵌入）


> **HyDE 的巧妙之处：** 用户的问题和文档的表述方式往往不同（问题是疑问句，文档是陈述句）。HyDE 先让 LLM 生成一个假设性回答，用回答去检索，语义匹配度更高。
````python
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

> **关键代码解读（HyDE）：**
> - 先让 LLM 生成一个"假设性回答"（不需要准确，只需要语义接近）
> - 用这个假设性回答的 Embedding 去检索，而不是用原始问题
> - 原理：假设性回答和真实文档的语义更接近（都是"回答"的语气），比问题和文档的匹配度更高
````
### 6.3 重排序（Reranking）


> **思路：** 向量检索的 Top-K 结果不一定是最相关的（Embedding 模型有局限）。Reranker 用更强的模型（Cross-Encoder）对粗检索结果重新排序，显著提升精度。
````python
from langchain.retrievers import ContextualCompressionRetriever
from langchain_cohere import CohereRerank

# 先粗检索 20 条，再用 Reranker 精排取 Top 5
reranker = CohereRerank(model="rerank-v3.5", top_n=5)

compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=vectorstore.as_retriever(search_kwargs={"k": 20})
)

results = compression_retriever.invoke("Binder 通信机制")

> **关键代码解读（Reranking）：**
> - 先用向量检索粗筛 20 条（速度快但不够精确）
> - 再用 Cross-Encoder 重排序精选 Top 5（更精确但速度慢）
> - 这种"粗筛+精排"的两阶段策略在搜索引擎中非常常见
````

> **评估是 RAG 优化的基础：** 没有评估就不知道优化是否有效。RAG 的评估比普通 NLP 任务更复杂，因为要同时评估检索质量和生成质量。
## 7. 评估 RAG 效果

### 7.1 关键指标

| 指标 | 含义 | 评估方法 |
|------|------|---------|
| 检索召回率 | 相关文档是否被检索到 | 人工标注 + 自动评估 |
| 检索精度 | 检索结果中相关文档的比例 | Top-K 精度 |
| 回答准确性 | 生成的回答是否正确 | LLM-as-Judge |
| 回答忠实度 | 回答是否基于检索到的文档 | 幻觉检测 |

### 7.2 使用 RAGAS 评估


> **RAGAS 是什么：** 一个专门评估 RAG 系统的开源框架，自动计算忠实度、相关性等指标，不需要人工标注。
````python
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
````

> **生产环境比 Demo 复杂得多：** 需要考虑文档更新、索引增量构建、缓存、监控、多租户隔离等。下面的架构图展示了一个完整的生产级 RAG 系统。
## 8. 生产环境架构

````
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
````
---

## 面试题精选

### Q1: RAG 解决了 LLM 的哪两个核心问题？
**答：** 知识过时（训练数据有截止日期，无法获取最新信息）和幻觉问题（模型可能编造不存在的信息）。RAG 通过检索外部知识库，让 LLM 基于真实文档生成回答。

### Q2: 文档分块（Chunking）的 chunk_size 和 chunk_overlap 如何选择？
**答：** chunk_size 太小会丢失上下文，太大会引入噪声，推荐 500-1000 tokens 起步。chunk_overlap 建议为 chunk_size 的 10%-20%，防止关键信息被切断。需要根据实际检索效果迭代调优。

### Q3: 混合检索（向量 + BM25）相比纯向量检索有什么优势？
**答：** 向量检索擅长语义匹配但可能忽略关键词精确匹配；BM25 擅长关键词匹配但不理解语义。混合检索结合两者优势，通过加权融合提高召回率，尤其对专业术语和实体名称的检索效果更好。

### Q4: 什么是 HyDE？它为什么能提升检索效果？
**答：** HyDE（假设性文档嵌入）先让 LLM 生成一个假设性回答，再用这个回答的 embedding 去检索。因为假设性回答的语义表示比短问题更接近实际文档的表示，所以检索效果往往更好。

### Q5: 如何评估一个 RAG 系统的效果？有哪些关键指标？
**答：** 检索层面看召回率（相关文档是否被检索到）和精度（检索结果中相关文档比例）；生成层面看准确性（回答是否正确）和忠实度（回答是否基于检索文档而非编造）。可用 RAGAS 等框架自动评估。

### Q6: Reranking 在 RAG 中起什么作用？为什么不直接用 Top-K？
**答：** 向量检索的 Top-K 是粗排，可能包含语义相关但实际不相关的结果。Reranker（如 Cohere Rerank）用交叉编码器对 query-document 对做精排，能更准确地判断相关性。通常先粗检索 20 条再精排取 Top 5。

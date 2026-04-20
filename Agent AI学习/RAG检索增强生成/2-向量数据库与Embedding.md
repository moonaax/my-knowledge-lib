# 向量数据库与 Embedding

## 1. Embedding 基础

Embedding 是将文本转换为高维向量的过程，使得语义相近的文本在向量空间中距离更近。

````
"如何优化启动速度" → [0.12, -0.34, 0.56, ..., 0.78]  (1536维)
"冷启动优化方案"   → [0.11, -0.33, 0.55, ..., 0.77]  (距离近 → 语义相似)
"今天天气不错"     → [0.89, 0.12, -0.67, ..., 0.23]  (距离远 → 语义不同)
````
## 2. Embedding 模型选型

### 2.1 主流模型对比

| 模型 | 维度 | 最大长度 | 中文支持 | 部署方式 |
|------|------|---------|---------|---------|
| text-embedding-3-small | 1536 | 8191 | 一般 | API |
| text-embedding-3-large | 3072 | 8191 | 一般 | API |
| bge-large-zh-v1.5 | 1024 | 512 | 优秀 | 本地/API |
| bge-m3 | 1024 | 8192 | 优秀 | 本地/API |
| jina-embeddings-v3 | 1024 | 8192 | 好 | API |
| cohere-embed-v3 | 1024 | 512 | 好 | API |

### 2.2 选型建议

````
通用场景（英文为主）  → text-embedding-3-small（性价比高）
中文场景             → bge-large-zh 或 bge-m3
需要长文本           → bge-m3 或 jina-embeddings-v3
本地部署             → bge 系列（HuggingFace 开源）
多语言              → bge-m3（支持 100+ 语言）
````
### 2.3 使用示例

````python
# OpenAI Embedding
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vector = embeddings.embed_query("AI Agent 是什么")

# 本地 BGE 模型
from langchain_huggingface import HuggingFaceEmbeddings

embeddings = HuggingFaceEmbeddings(
    model_name="BAAI/bge-large-zh-v1.5",
    model_kwargs={"device": "cuda"},
    encode_kwargs={"normalize_embeddings": True}
)
````
## 3. 向量数据库

### 3.1 主流向量数据库对比

| 数据库 | 类型 | 特点 | 适用场景 |
|--------|------|------|---------|
| Chroma | 嵌入式 | 轻量、易用、Python 原生 | 开发/小规模 |
| FAISS | 库 | Meta 出品、极快 | 大规模离线检索 |
| Milvus | 分布式 | 高性能、可扩展 | 生产环境 |
| Qdrant | 独立服务 | Rust 实现、高性能 | 生产环境 |
| Pinecone | 云服务 | 全托管、免运维 | 快速上线 |
| Weaviate | 独立服务 | 支持混合检索 | 需要多模态 |
| pgvector | PG 扩展 | 复用 PostgreSQL | 已有 PG 基础设施 |

### 3.2 Chroma（开发首选）

````python
import chromadb
from langchain_community.vectorstores import Chroma

# 内存模式（开发测试）
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings
)

# 持久化模式
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db",
    collection_name="my_knowledge"
)

# 后续加载
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=embeddings,
    collection_name="my_knowledge"
)

# 增量添加
vectorstore.add_documents(new_chunks)

# 删除
vectorstore.delete(ids=["doc_id_1", "doc_id_2"])
````
### 3.3 FAISS（高性能检索）

````python
from langchain_community.vectorstores import FAISS

# 创建索引
vectorstore = FAISS.from_documents(chunks, embeddings)

# 保存/加载
vectorstore.save_local("./faiss_index")
vectorstore = FAISS.load_local("./faiss_index", embeddings,
                                allow_dangerous_deserialization=True)

# 合并索引
vectorstore.merge_from(another_vectorstore)
````
### 3.4 Milvus（生产环境）

````python
from langchain_milvus import Milvus

# 连接 Milvus
vectorstore = Milvus.from_documents(
    documents=chunks,
    embedding=embeddings,
    connection_args={"host": "localhost", "port": "19530"},
    collection_name="knowledge_base"
)

# 带元数据过滤的检索
results = vectorstore.similarity_search(
    "启动优化",
    k=5,
    expr='category == "性能优化"'  # Milvus 过滤表达式
)
````
## 4. 相似度计算

### 4.1 常用距离度量

````python
import numpy as np

# 余弦相似度（最常用）
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# 欧氏距离
def euclidean_distance(a, b):
    return np.linalg.norm(np.array(a) - np.array(b))

# 内积（向量已归一化时等价于余弦相似度）
def inner_product(a, b):
    return np.dot(a, b)
````
### 4.2 选择建议

| 度量 | 特点 | 适用场景 |
|------|------|---------|
| 余弦相似度 | 不受向量长度影响 | 文本相似度（默认选择） |
| 欧氏距离 | 考虑绝对距离 | 需要考虑向量大小时 |
| 内积 | 最快，需归一化 | 已归一化的向量 |

## 5. 索引优化

### 5.1 ANN 索引类型

````
精确搜索 (Flat)
  - 100% 准确，但慢
  - 适合: < 10万条数据

IVF (Inverted File Index)
  - 先聚类，再在类内搜索
  - 适合: 10万 - 1000万

HNSW (Hierarchical Navigable Small World)
  - 图结构，高召回率
  - 适合: 需要高召回的场景（推荐）

PQ (Product Quantization)
  - 向量压缩，节省内存
  - 适合: 超大规模、内存受限
````
### 5.2 FAISS 索引选择

````python
import faiss

dimension = 1536
n_vectors = 1000000  # 100万条

# 小数据量: Flat（精确）
index = faiss.IndexFlatIP(dimension)

# 中等数据量: IVF + Flat
nlist = 100  # 聚类数
index = faiss.IndexIVFFlat(
    faiss.IndexFlatIP(dimension), dimension, nlist
)
index.train(vectors)  # 需要训练
index.nprobe = 10     # 搜索时探测的聚类数

# 大数据量: HNSW
index = faiss.IndexHNSWFlat(dimension, 32)  # 32 = 连接数
````
## 6. 实践建议

### 6.1 Embedding 质量提升

````
1. 添加指令前缀（BGE 模型推荐）
   查询: "为这个句子生成表示以用于检索相关文章: " + query
   文档: 直接使用原文

2. 文档预处理
   - 去除无意义的页眉页脚
   - 保留文档结构信息（标题层级）
   - 为每个 chunk 添加上下文摘要

3. 定期更新索引
   - 增量更新而非全量重建
   - 记录文档版本，避免重复索引
````
### 6.2 检索效果调优清单

````
□ chunk_size 是否合适？（试试 500、1000、1500）
□ chunk_overlap 是否足够？（建议 10-20%）
□ 是否使用了混合检索？（向量 + BM25）
□ 是否添加了 Reranker？
□ 是否做了查询改写？
□ 元数据过滤是否有效？
□ Embedding 模型是否适合你的语言/领域？
□ Top-K 值是否合理？（太少遗漏，太多噪声）
````
---

## 面试题精选

### Q1: 余弦相似度和欧氏距离有什么区别？RAG 场景下用哪个？
**答：** 余弦相似度衡量方向相似性，不受向量长度影响；欧氏距离衡量绝对距离，受长度影响。RAG 场景下推荐余弦相似度，因为文本 embedding 的语义相似性主要体现在方向上而非绝对大小。

### Q2: HNSW 和 IVF 索引各自的原理和适用场景是什么？
**答：** IVF 先将向量聚类，检索时只在相关聚类内搜索，适合 10 万-1000 万级数据。HNSW 构建多层图结构，通过图遍历快速找到近邻，召回率更高，适合对召回率要求高的场景。HNSW 是目前最推荐的索引类型。

### Q3: 中文场景下 Embedding 模型怎么选？
**答：** 中文场景推荐 bge-large-zh（智源出品，中文效果最佳之一）或 bge-m3（支持 8K 长文本和 100+ 语言）。如果需要 API 调用可选 OpenAI text-embedding-3-small，但中文效果不如 bge 系列。

### Q4: Chroma、FAISS、Milvus 分别适合什么场景？
**答：** Chroma 轻量易用，适合开发和小规模场景；FAISS 是 Meta 出品的库，极快但无服务化能力，适合大规模离线检索；Milvus 是分布式向量数据库，支持高可用和水平扩展，适合生产环境。

### Q5: 如何提升 Embedding 检索的质量？
**答：** 关键手段包括：为 chunk 添加上下文前缀（文档标题、章节信息）、使用指令前缀（BGE 模型推荐）、调优 chunk_size 和 overlap、使用混合检索（向量+BM25）、添加 Reranker 精排、做好文档预处理（去噪、保留结构）。

### Q6: 向量数据库的增量更新怎么做？需要注意什么？
**答：** 增量更新而非全量重建，通过文档 ID 或哈希值判断是否已索引。需要注意：记录文档版本避免重复索引、删除已失效文档的向量、更新后验证检索效果未退化。

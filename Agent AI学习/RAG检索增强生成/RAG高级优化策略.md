# RAG 高级优化策略

## 1. 概述

基础 RAG 在实际应用中常遇到检索不准、回答不完整等问题。本文介绍一系列进阶优化策略。

```
基础 RAG 的常见问题:
❌ 检索到不相关的文档
❌ 关键信息被分块切断
❌ 用户问题模糊导致检索偏差
❌ 多文档信息无法有效整合
❌ 回答缺乏来源引用
```

## 2. 索引阶段优化

### 2.1 文档预处理

```python
def preprocess_document(doc: str) -> str:
    """文档清洗"""
    # 去除多余空白
    doc = re.sub(r'\n{3,}', '\n\n', doc)
    # 去除页眉页脚
    doc = remove_headers_footers(doc)
    # 统一格式
    doc = normalize_unicode(doc)
    return doc
```

### 2.2 层级索引（Parent-Child）

将文档同时按大块和小块索引，检索时用小块匹配，返回时用大块提供上下文：

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore

# 小块用于检索（精确匹配）
child_splitter = RecursiveCharacterTextSplitter(chunk_size=400)
# 大块用于返回（完整上下文）
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000)

store = InMemoryStore()

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,
    docstore=store,
    child_splitter=child_splitter,
    parent_splitter=parent_splitter,
)

retriever.add_documents(documents)
# 检索时: 用小块匹配 → 返回对应的大块
results = retriever.invoke("冷启动优化")
```

### 2.3 上下文增强分块

为每个 chunk 添加上下文信息：

```python
def add_context_to_chunks(chunks, document_title, section_title):
    """为每个 chunk 添加上下文前缀"""
    for chunk in chunks:
        context_prefix = f"文档: {document_title}\n章节: {section_title}\n\n"
        chunk.page_content = context_prefix + chunk.page_content
    return chunks
```

### 2.4 多表示索引

同一文档生成多种表示用于检索：

```python
from langchain.retrievers.multi_vector import MultiVectorRetriever

# 为每个文档生成摘要
summaries = [llm.invoke(f"用一句话总结: {doc.page_content}") for doc in chunks]

# 为每个文档生成假设性问题
questions = [llm.invoke(f"这段文档能回答什么问题: {doc.page_content}") for doc in chunks]

# 原文、摘要、问题都作为检索入口，指向同一个原文档
```

## 3. 检索阶段优化

### 3.1 查询改写（Query Rewriting）

```python
# 多角度查询
multi_query_prompt = ChatPromptTemplate.from_template("""
你是一个查询优化专家。请将以下问题从 3 个不同角度改写，以提高检索效果。

原始问题: {question}

改写要求:
1. 使用更专业的术语
2. 从不同角度描述同一个问题
3. 补充可能的上下文

改写结果（每行一个）:""")

# 用所有改写后的查询分别检索，合并去重
```

### 3.2 Step-Back Prompting

先让 LLM 思考更高层次的问题，再检索：

```python
step_back_prompt = ChatPromptTemplate.from_template("""
给定以下具体问题，请生成一个更通用的上位问题，有助于检索到更全面的信息。

具体问题: {question}
上位问题:""")

# 例: "Android 12 的冷启动优化" → "Android 应用启动性能优化方法"
# 同时用原始问题和上位问题检索，合并结果
```

### 3.3 自适应检索

根据问题类型动态调整检索策略：

```python
def adaptive_retrieve(question: str):
    # 判断问题类型
    question_type = classify_question(question)
    
    if question_type == "factual":
        # 事实性问题: 精确检索，少量结果
        return vectorstore.similarity_search(question, k=3)
    elif question_type == "analytical":
        # 分析性问题: 广泛检索，多角度
        return ensemble_retriever.invoke(question)  # 混合检索
    elif question_type == "comparative":
        # 对比性问题: 分别检索各个对比对象
        entities = extract_entities(question)
        results = []
        for entity in entities:
            results.extend(vectorstore.similarity_search(entity, k=3))
        return results
```

## 4. 生成阶段优化

### 4.1 Lost in the Middle 问题

LLM 倾向于关注输入的开头和结尾，中间的信息容易被忽略：

```python
def reorder_documents(docs):
    """将最相关的文档放在开头和结尾"""
    reordered = []
    for i, doc in enumerate(docs):
        if i % 2 == 0:
            reordered.insert(0, doc)  # 偶数位放开头
        else:
            reordered.append(doc)     # 奇数位放结尾
    return reordered
```

### 4.2 引用来源

```python
cite_prompt = ChatPromptTemplate.from_template("""
基于以下参考文档回答问题。回答时必须标注信息来源。

参考文档:
[1] {doc1}
[2] {doc2}
[3] {doc3}

问题: {question}

请在回答中使用 [1][2][3] 标注每个观点的来源。如果文档中没有相关信息，请明确说明。
""")
```

### 4.3 答案验证

```python
verify_prompt = ChatPromptTemplate.from_template("""
请验证以下回答是否完全基于提供的参考文档。

参考文档: {context}
问题: {question}
回答: {answer}

检查:
1. 回答中是否有文档中未提及的信息?（幻觉检测）
2. 回答是否完整覆盖了文档中的相关信息?
3. 回答是否准确引用了文档内容?

验证结果:""")
```

## 5. 高级 RAG 架构

### 5.1 Corrective RAG (CRAG)

检索后评估文档质量，不合格则触发补充检索：

```python
def corrective_rag(question: str):
    # 1. 初始检索
    docs = retriever.invoke(question)
    
    # 2. 评估检索质量
    relevance_scores = evaluate_relevance(question, docs)
    
    if max(relevance_scores) < 0.6:
        # 3a. 检索质量差 → 查询改写后重新检索
        rewritten = rewrite_query(question)
        docs = retriever.invoke(rewritten)
    
    if max(relevance_scores) < 0.3:
        # 3b. 完全不相关 → 使用网络搜索补充
        web_results = web_search(question)
        docs.extend(web_results)
    
    # 4. 生成回答
    return generate_answer(question, docs)
```

### 5.2 Self-RAG

让 LLM 自己决定是否需要检索，以及评估检索结果：

```
问题 → LLM 判断是否需要检索
         ├── 不需要 → 直接回答
         └── 需要 → 检索 → LLM 评估相关性
                            ├── 相关 → 生成回答 → 自我评估
                            └── 不相关 → 重新检索或放弃
```

### 5.3 Graph RAG

结合知识图谱增强检索：

```
传统 RAG: 文本 → 向量 → 相似度检索
Graph RAG: 文本 → 实体/关系提取 → 知识图谱 → 图检索 + 向量检索

优势:
- 能回答多跳推理问题
- 理解实体之间的关系
- 全局摘要能力更强
```

## 6. 评估与迭代

### 评估维度

```
检索质量:
  - Recall@K: Top-K 中包含正确文档的比例
  - MRR: 第一个正确文档的排名倒数
  - NDCG: 考虑排名位置的综合指标

生成质量:
  - Faithfulness: 回答是否忠于检索文档
  - Relevancy: 回答是否切题
  - Completeness: 回答是否完整

端到端:
  - 用户满意度
  - 任务完成率
```

### 迭代优化流程

```
1. 收集 Bad Case（回答不好的案例）
2. 分析原因:
   - 检索问题? → 优化分块/检索策略
   - 生成问题? → 优化 Prompt/模型
   - 数据问题? → 补充/清洗数据
3. 针对性优化
4. 回归测试（确保不影响已有好的 Case）
5. 重复迭代
```

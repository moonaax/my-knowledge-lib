# RAG 高级优化策略


> **本文定位：** 基础 RAG 能跑但效果一般。本文介绍从索引、检索、生成三个阶段的进阶优化策略，以及 CRAG、Self-RAG、Graph RAG 三种高级架构。这些是把 RAG 从"能用"提升到"好用"的关键。
## 1. 概述

基础 RAG 在实际应用中常遇到检索不准、回答不完整等问题。本文介绍一系列进阶优化策略。

````
基础 RAG 的常见问题:
❌ 检索到不相关的文档
❌ 关键信息被分块切断
❌ 用户问题模糊导致检索偏差
❌ 多文档信息无法有效整合
❌ 回答缺乏来源引用
````

> **索引阶段优化的目标：** 让文档被更好地"理解"和"组织"。文档预处理清理噪声，层级索引兼顾精度和上下文，多表示索引提升召回率。
## 2. 索引阶段优化

### 2.1 文档预处理


> **思路：** 文档预处理是 RAG 优化的第一步——清理噪声数据（页眉页脚、特殊字符、多余空白），让后续的分块和 Embedding 质量更高。
````python
def preprocess_document(doc: str) -> str:
    """文档清洗"""
    # 去除多余空白
    doc = re.sub(r'\n{3,}', '\n\n', doc)
    # 去除页眉页脚
    doc = remove_headers_footers(doc)
    # 统一格式
    doc = normalize_unicode(doc)
    return doc
````

> **层级索引解决了分块的两难问题：** 块太小检索精确但上下文不完整，块太大上下文完整但检索不精确。Parent-Child 方案用小块检索、返回大块，两全其美。
### 2.2 层级索引（Parent-Child）

将文档同时按大块和小块索引，检索时用小块匹配，返回时用大块提供上下文：

````python
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

> **关键代码解读（层级索引）：**
> - `child_splitter` 切小块（200 token）用于精确检索
> - `parent_splitter` 切大块（1000 token）用于返回完整上下文
> - 检索时用小块匹配，但返回小块所属的大块——兼顾检索精度和上下文完整性
````
### 2.3 上下文增强分块

为每个 chunk 添加上下文信息：

````python
def add_context_to_chunks(chunks, document_title, section_title):
    """为每个 chunk 添加上下文前缀"""
    for chunk in chunks:
        context_prefix = f"文档: {document_title}\n章节: {section_title}\n\n"
        chunk.page_content = context_prefix + chunk.page_content
    return chunks

> **关键代码解读（上下文增强分块）：** 在每个块的开头加上文档标题和章节标题作为上下文前缀。这样即使块被单独检索出来，也能知道它属于哪个文档的哪个章节，不会丢失上下文。
````
### 2.4 多表示索引

同一文档生成多种表示用于检索：

````python
from langchain.retrievers.multi_vector import MultiVectorRetriever

# 为每个文档生成摘要
summaries = [llm.invoke(f"用一句话总结: {doc.page_content}") for doc in chunks]

# 为每个文档生成假设性问题
questions = [llm.invoke(f"这段文档能回答什么问题: {doc.page_content}") for doc in chunks]

# 原文、摘要、问题都作为检索入口，指向同一个原文档

> **关键代码解读（多表示索引）：** 为同一个文档生成多种表示——原文、摘要、假设性问题。用户的问题可能和摘要更匹配，也可能和假设性问题更匹配。多入口检索显著提升召回率。
````

> **检索阶段优化的目标：** 让检索结果更精准、更全面。查询改写提升召回率，Step-Back 提升泛化能力，自适应检索减少噪声。
## 3. 检索阶段优化

### 3.1 查询改写（Query Rewriting）

````python
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

> **关键代码解读（查询改写）：** 一个问题改写为多个不同角度的查询，分别检索后合并去重。比如"如何优化启动速度"可以改写为"冷启动优化方案"、"Application onCreate 优化"等，每个角度能检索到不同的文档。
````

> **Step-Back Prompting 的巧妙之处：** 用户问的太具体时，可能检索不到（文档中没有完全匹配的内容）。退一步问更通用的问题，反而能找到更多相关文档。
### 3.2 Step-Back Prompting

先让 LLM 思考更高层次的问题，再检索：

````python
step_back_prompt = ChatPromptTemplate.from_template("""
给定以下具体问题，请生成一个更通用的上位问题，有助于检索到更全面的信息。

具体问题: {question}
上位问题:""")

# 例: "Android 12 的冷启动优化" → "Android 应用启动性能优化方法"
# 同时用原始问题和上位问题检索，合并结果

> **关键代码解读（Step-Back）：** 把具体问题抽象为更通用的上位问题。比如"Android 12 的冷启动优化"→"Android 应用启动性能优化方法"。上位问题能检索到更全面的文档，和原始问题的检索结果互补。
````
### 3.3 自适应检索

根据问题类型动态调整检索策略：


> **思路：** 不是所有问题都需要检索——闲聊、简单推理不需要。自适应检索先判断问题类型，只在需要时才触发检索，减少不必要的开销和噪声。
````python
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

> **关键代码解读（自适应检索）：**
> - 先让 LLM 判断问题类型：事实查询需要检索，闲聊/推理不需要
> - 避免对不需要检索的问题浪费检索资源
> - 这是一个简单的路由机制，可以扩展为更复杂的多路由策略
````

> **生成阶段的优化目标：** 检索到了好的文档，还要确保 LLM 能正确利用它们。Lost in the Middle、引用来源、答案验证是三个关键优化点。
## 4. 生成阶段优化


> **Lost in the Middle 是什么：** 研究发现 LLM 对输入中间位置的信息关注度最低。如果最相关的文档恰好在中间，LLM 可能会忽略它。解决方案是把最相关的文档放在开头和结尾。
### 4.1 Lost in the Middle 问题

LLM 倾向于关注输入的开头和结尾，中间的信息容易被忽略：

````python
def reorder_documents(docs):
    """将最相关的文档放在开头和结尾"""
    reordered = []
    for i, doc in enumerate(docs):
        if i % 2 == 0:
            reordered.insert(0, doc)  # 偶数位放开头
        else:
            reordered.append(doc)     # 奇数位放结尾
    return reordered

> **关键代码解读（Lost in the Middle）：**
> - 把最相关的文档放在开头和结尾，不太相关的放中间
> - 这种重排策略简单但有效，几乎零成本
````
### 4.2 引用来源


> **思路：** 让 LLM 在回答中标注每句话的信息来源（引用了哪个文档），用户可以点击来源验证准确性。这大大提升了用户对 RAG 系统的信任度。
````python
cite_prompt = ChatPromptTemplate.from_template("""
基于以下参考文档回答问题。回答时必须标注信息来源。

参考文档:
[1] {doc1}
[2] {doc2}
[3] {doc3}

问题: {question}

请在回答中使用 [1][2][3] 标注每个观点的来源。如果文档中没有相关信息，请明确说明。
""")
````
### 4.3 答案验证


> **思路：** LLM 可能在回答中编造检索文档中没有的信息（幻觉）。答案验证让另一个 LLM 调用检查回答是否忠实于检索结果。
````python
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

> **关键代码解读（答案验证）：**
> - 生成回答后，让 LLM 检查回答是否忠实于检索到的文档
> - 如果发现幻觉（回答中有文档未提及的信息），标记为不可信
> - 这是一个简单但有效的"自我检查"机制
````

> **高级 RAG 架构是对基础 RAG 的系统性升级：** CRAG 加了检索质量评估，Self-RAG 加了自我反思，Graph RAG 引入了知识图谱。根据场景选择合适的架构。
## 5. 高级 RAG 架构

### 5.1 Corrective RAG (CRAG)

检索后评估文档质量，不合格则触发补充检索：


> **思路：** 基础 RAG 盲目信任检索结果，但检索到的文档不一定相关。CRAG 在检索后加了一个"质量评估"环节——评估检索结果的相关性，不相关的丢弃，必要时触发 Web 搜索补充。
````python
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

> **关键代码解读（CRAG）：**
> - 检索后先用 LLM 评估每条结果的相关性（"相关"/"不相关"/"不确定"）
> - 相关的直接用，不相关的丢弃
> - 如果所有结果都不相关，触发 Web 搜索作为补充
> - 这种"检索→评估→补充"的模式显著减少了不相关信息对生成的干扰
````

> **Self-RAG 的核心：** Agent 自己判断是否需要检索、检索结果是否有用、生成的回答是否忠实于检索结果。通过多个"反思 token"实现自我质量控制。
### 5.2 Self-RAG

让 LLM 自己决定是否需要检索，以及评估检索结果：

````
问题 → LLM 判断是否需要检索
         ├── 不需要 → 直接回答
         └── 需要 → 检索 → LLM 评估相关性
                            ├── 相关 → 生成回答 → 自我评估
                            └── 不相关 → 重新检索或放弃
````

> **Graph RAG 的独特价值：** 传统向量 RAG 只能找到"语义相似"的文档块，Graph RAG 能沿着知识图谱的关系链推理，回答需要多跳推理的复杂问题（如"A 的老板的公司做什么业务"）。
### 5.3 Graph RAG

结合知识图谱增强检索：

````
传统 RAG: 文本 → 向量 → 相似度检索
Graph RAG: 文本 → 实体/关系提取 → 知识图谱 → 图检索 + 向量检索

优势:
- 能回答多跳推理问题
- 理解实体之间的关系
- 全局摘要能力更强
````

> **评估驱动优化：** 没有评估就不知道优化是否有效。建立评估基准（benchmark），每次优化后对比指标变化。
## 6. 评估与迭代

### 评估维度

````
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
````

> **RAG 优化是一个持续迭代的过程：** 不是一次性做完的。上线后持续收集 bad case，分析是检索问题还是生成问题，针对性优化，循环迭代。
### 迭代优化流程

````
1. 收集 Bad Case（回答不好的案例）
2. 分析原因:
   - 检索问题? → 优化分块/检索策略
   - 生成问题? → 优化 Prompt/模型
   - 数据问题? → 补充/清洗数据
3. 针对性优化
4. 回归测试（确保不影响已有好的 Case）
5. 重复迭代
````
---

## 面试题精选

### Q1: 什么是 Parent-Child 层级索引？它解决了什么问题？
**答：** 用小块（Child）做精确检索匹配，命中后返回对应的大块（Parent）提供完整上下文。它解决了小 chunk 检索精准但上下文不足、大 chunk 上下文完整但检索噪声大的矛盾。

### Q2: 什么是 Lost in the Middle 问题？如何缓解？
**答：** LLM 倾向于关注输入的开头和结尾，中间的信息容易被忽略。缓解方法是将最相关的文档放在开头和结尾位置，或者减少一次性输入的文档数量，也可以用 Map-Reduce 方式分批处理。

### Q3: Corrective RAG 和 Self-RAG 的核心思路分别是什么？
**答：** Corrective RAG 在检索后评估文档质量，质量差则改写查询重新检索或用网络搜索补充。Self-RAG 让 LLM 自己判断是否需要检索、评估检索结果相关性、并对生成的回答做自我评估，实现全流程自适应。

### Q4: 查询改写（Query Rewriting）有哪些常用策略？
**答：** 多角度改写（从不同角度描述同一问题）、Step-Back Prompting（生成更通用的上位问题）、HyDE（生成假设性回答用于检索）。这些策略可以弥补用户原始查询表述不精确的问题。

### Q5: RAG 系统上线后效果不好，你会怎么排查和优化？
**答：** 先收集 Bad Case 分析原因：如果是检索问题（没检索到相关文档），优化分块策略、检索方式或添加 Reranker；如果是生成问题（检索到了但回答不对），优化 Prompt 或换更强的模型；如果是数据问题，补充或清洗知识库。然后做回归测试确保不影响已有好的 Case。

### Q6: Graph RAG 相比传统向量 RAG 有什么优势？
**答：** Graph RAG 结合知识图谱，能理解实体之间的关系，支持多跳推理（如"A 的老板的公司在哪"），全局摘要能力更强。传统向量 RAG 只能做语义相似度匹配，难以处理需要关系推理的复杂问题。

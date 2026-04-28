# RAG 高级优化策略


> **本文定位：** 基础 RAG 能跑但效果一般。本文介绍从索引、检索、生成三个阶段的进阶优化策略，以及 CRAG、Self-RAG、Graph RAG 三种高级架构。这些是把 RAG 从"能用"提升到"好用"的关键。
## 1. 概述

基础 RAG 在实际应用中常遇到检索不准、回答不完整等问题。本文介绍一系列进阶优化策略。RAG 技术的发展经历了三个阶段：**朴素 RAG**（TF-IDF/BM25 关键词匹配）→ **高级 RAG**（稠密嵌入语义检索 + 查询优化）→ **模块化 RAG**（混合检索、MQE、自我反思，各模块可插拔组合）。本文的优化策略主要覆盖第二和第三阶段的技术。

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
`````

> **RAG 技术并非一步到位，而是经历了从简单到复杂的演进过程。** 理解这个演进脉络，有助于我们在实际项目中选择合适的技术方案——不必追求最前沿，够用就好。
## 7. RAG 技术发展历程

RAG 技术的发展可以划分为三个阶段，每个阶段都在前一阶段的基础上解决了关键痛点：

````
┌─────────────────────────────────────────────────────────────────────┐
│  第一阶段：朴素 RAG（Naive RAG, 2020-2021）                         │
│  ─────────────────────────────────────────                          │
│  检索方式：TF-IDF / BM25 等关键词匹配                               │
│  生成方式：检索文档直接拼接到 Prompt                                  │
│  优点：实现简单，无需训练                                             │
│  缺点：无法理解语义相似性，"同义不同词"会漏检                         │
├─────────────────────────────────────────────────────────────────────┤
│  第二阶段：高级 RAG（Advanced RAG, 2022-2023）                       │
│  ─────────────────────────────────────────                          │
│  检索方式：稠密嵌入（Dense Embedding）语义检索                        │
│  生成方式：查询重写、文档分块优化、重排序（Reranker）                  │
│  优点：能理解语义，检索精度大幅提升                                    │
│  缺点：单一向量检索仍有局限，复杂问题难以处理                          │
├─────────────────────────────────────────────────────────────────────┤
│  第三阶段：模块化 RAG（Modular RAG, 2023-至今）                      │
│  ─────────────────────────────────────────                          │
│  检索方式：混合检索、多查询扩展（MQE）、HyDE                          │
│  生成方式：思维链推理、自我反思与修正                                  │
│  优点：各模块可插拔组合，适应多样化场景                                │
│  趋势：与 Agent、记忆系统深度融合                                     │
└─────────────────────────────────────────────────────────────────────┘
````

> 🔑 **关键认知：** 当前主流的 RAG 系统处于第二到第三阶段之间。朴素 RAG 的 TF-IDF/BM25 并未被淘汰——它们与稠密检索组合成混合检索，反而成为模块化 RAG 的重要组成部分。

> **索引阶段的分块策略直接决定了检索质量的上限。** 前文介绍了 RecursiveCharacterTextSplitter，但在处理结构化文档（Markdown、技术文档）时，按文档结构分块比按字符数分块效果好得多。
## 8. Markdown 结构感知分块

### 8.1 为什么需要结构感知分块

`RecursiveCharacterTextSplitter` 按固定字符数切分，不理解文档结构。这会导致：
- 标题与正文被切到不同 chunk，丢失上下文
- 代码块被从中间截断
- 列表项被拆散

对于 Markdown 格式的文档，利用标题层次（`#`、`##`、`###`）进行语义分割，能显著提升 chunk 的语义完整性。

### 8.2 核心实现思路

````python
def split_markdown_by_headings(text: str) -> list:
    """根据标题层次分割 Markdown，保持语义完整性"""
    lines = text.splitlines()
    heading_stack = []  # 维护当前标题路径：["一级标题", "二级标题", ...]
    paragraphs = []
    buf = []

    def flush():
        """将缓冲区内容作为一个段落输出"""
        if not buf:
            return
        content = "\n".join(buf).strip()
        if content:
            paragraphs.append({
                "content": content,
                "heading_path": " > ".join(heading_stack) if heading_stack else None,
            })
        buf.clear()

    for line in lines:
        if line.strip().startswith("#"):
            flush()  # 遇到新标题，先输出之前积累的段落
            level = len(line) - len(line.lstrip('#'))
            title = line.lstrip('#').strip()
            # 维护标题栈：同级或更低级别的标题替换栈中对应位置
            heading_stack = heading_stack[:level - 1]
            heading_stack.append(title)
        elif line.strip() == "":
            flush()  # 空行作为段落分隔
        else:
            buf.append(line)

    flush()
    return paragraphs

# ── 段落合并为 chunk（按 Token 数控制大小）──
def chunk_paragraphs(paragraphs, chunk_tokens=500, overlap_tokens=50):
    """将段落按 Token 数合并为 chunk，支持重叠"""
    chunks, cur, cur_tokens = [], [], 0

    for p in paragraphs:
        p_tokens = approx_token_len(p["content"])
        if cur_tokens + p_tokens > chunk_tokens and cur:
            # 生成当前 chunk
            content = "\n\n".join(x["content"] for x in cur)
            heading_path = next(
                (x["heading_path"] for x in reversed(cur) if x.get("heading_path")),
                None
            )
            chunks.append({"content": content, "heading_path": heading_path})
            # 保留尾部作为重叠
            cur, cur_tokens = build_overlap(cur, overlap_tokens)

        cur.append(p)
        cur_tokens += p_tokens

    if cur:
        content = "\n\n".join(x["content"] for x in cur)
        heading_path = next(
            (x["heading_path"] for x in reversed(cur) if x.get("heading_path")),
            None
        )
        chunks.append({"content": content, "heading_path": heading_path})

    return chunks
````

> **关键代码解读（Markdown 结构感知分块）：**
> - `heading_stack` 维护当前的标题路径（如 "RAG优化 > 检索策略 > 混合检索"），每个 chunk 都携带这个路径作为元数据
> - 遇到新标题时自动 flush 缓冲区，保证标题和正文在同一个 chunk
> - 按 Token 数而非字符数控制 chunk 大小，更贴合 Embedding 模型的输入限制

### 8.3 中英文混合 Token 估算

分块需要精确控制 Token 数，但中英文混合文本的 Token 计算不同：

````python
def approx_token_len(text: str) -> int:
    """中英文混合 Token 估算"""
    # CJK 字符：每个字符约 1 token
    cjk_count = sum(1 for ch in text if '一' <= ch <= '鿿')
    # 非 CJK 字符：按空格分词估算
    non_cjk = text
    for ch in text:
        if '一' <= ch <= '鿿':
            non_cjk = non_cjk.replace(ch, ' ')
    non_cjk_tokens = len(non_cjk.split())
    return cjk_count + non_cjk_tokens
````

> 🔑 **为什么不用 tiktoken？** tiktoken 是 OpenAI 的 Token 计数器，对中文的计算不准确（一个汉字可能被拆成 2-3 个 token）。上面的估算方式更贴近实际的 Embedding 模型行为，且零依赖。

> **统一文档格式是模块化 RAG 的重要设计决策。** 现实中的知识库来源多样——PDF、Word、Excel、图片、音频——如果每种格式都用不同的处理管线，维护成本极高。统一转换为 Markdown 后，后续的分块、向量化、检索都可以复用同一套逻辑。
## 9. MarkItDown 统一文档转换

### 9.1 核心思路

MarkItDown 是微软开源的通用文档转换工具，能将几乎所有常见格式统一转换为 Markdown：

````
输入格式                     转换方式
────────────────────────────────────────
PDF                        → 结构化 Markdown（保留标题/表格）
Word (.docx)               → Markdown（保留格式）
Excel (.xlsx)              → Markdown 表格
PowerPoint (.pptx)         → Markdown（按页提取）
图片 (JPG/PNG)             → OCR → Markdown
音频 (MP3/WAV)             → 语音转录 → Markdown
代码文件                   → 代码块 Markdown
HTML / CSV / JSON / XML    → Markdown
````

### 9.2 使用示例

````python
from markitdown import MarkItDown

md = MarkItDown()

# 统一转换接口
result = md.convert("report.pdf")
markdown_text = result.text_content  # 结构化 Markdown

# 后续流程统一：Markdown → 分块 → 向量化 → 存储
chunks = split_markdown_by_headings(markdown_text)
index_chunks(chunks)
````

### 9.3 与分块的配合

MarkItDown 的最大价值在于：所有格式转换后都是标准 Markdown，可以直接用上一节的结构感知分块策略处理。这形成了一个清晰的管线：

````
任意格式文档 → MarkItDown → Markdown → 结构感知分块 → 向量化 → 检索
````

> 🔑 **PDF 增强处理：** 对于复杂 PDF（扫描件、多栏排版），MarkItDown 可能不够用。实践中建议对 PDF 做增强处理——先用 OCR 引擎（如 PaddleOCR）提取文本，再交给 MarkItDown 做格式化。

> **查询改写只改写了"一个"查询，而 MQE 会生成"多个"语义等价查询。** 这个看似微小的差异，在实际效果上差距显著——同一个问题的不同表述可能匹配到完全不同的文档子集。
## 10. 多查询扩展（MQE）

### 10.1 与查询改写的区别

前文 3.1 节的查询改写是将一个问题改写为多个角度的查询。MQE（Multi-Query Expansion）更进一步：用 LLM 生成多个**语义等价但表述不同**的查询，分别检索后合并去重。

核心区别：
- **查询改写**：侧重"角度变换"（专业术语 → 口语化 → 场景化）
- **MQE**：侧重"语义等价"（同一个意思的多种说法），且有标准化的扩展-检索-合并流程

### 10.2 实现流程

````python
def multi_query_expansion(query: str, n: int = 3) -> list:
    """MQE：用 LLM 生成 N 个语义等价查询"""
    prompt = f"""你是检索查询扩展助手。请将以下查询生成 {n} 个语义等价但表述不同的查询。
    要求：使用中文，简短，避免标点，每行一个。

    原始查询：{query}"""

    response = llm.invoke(prompt)
    expanded = [line.strip("- \t") for line in response.splitlines() if line.strip()]
    return expanded[:n] or [query]

def search_with_mqe(query: str, retriever, top_k: int = 8):
    """MQE 扩展检索：扩展 → 并行检索 → 合并去重"""
    # 1. 生成扩展查询
    expansions = [query] + multi_query_expansion(query, n=3)

    # 2. 每个查询独立检索
    seen = {}  # memory_id → best_score，用于去重
    per_query_k = max(top_k * 2, 20) // len(expansions)

    for q in expansions:
        results = retriever.similarity_search(q, k=per_query_k)
        for doc in results:
            doc_id = doc.metadata.get("id")
            score = doc.metadata.get("score", 0)
            if doc_id not in seen or score > seen[doc_id]["score"]:
                seen[doc_id] = {"doc": doc, "score": score}

    # 3. 按分数排序，取 top_k
    merged = sorted(seen.values(), key=lambda x: x["score"], reverse=True)
    return [item["doc"] for item in merged[:top_k]]
````

> **关键代码解读（MQE）：**
> - `multi_query_expansion` 用 LLM 生成 N 个语义等价查询（如 "如何学习Python" → "Python入门教程"、"Python学习方法"、"Python编程指南"）
> - 每个扩展查询独立检索，扩大候选池
> - 用 `seen` 字典去重，同一文档只保留最高分数
> - MQE 对模糊查询和专业术语查询效果尤其显著，召回率可提升 30%-50%

> 🔑 **MQE vs HyDE：** MQE 生成的是"不同说法的问题"，HyDE 生成的是"假设性的答案"。两者互补：MQE 解决用词多样性问题，HyDE 解决问题-文档语义鸿沟问题。对高召回率场景建议同时启用。

> **RAG 和记忆系统不是互斥的，而是互补的。** RAG 负责从外部知识库检索，记忆系统负责存储交互历史和学习积累。两者的协同能让 Agent 既"博学"（有知识库）又"记仇"（有记忆）。
## 11. RAG 与记忆系统的协同

### 11.1 为什么需要协同

单独的 RAG 系统有一个盲区：它只负责"查资料"，不记录"查过什么"。用户第二次问同样的问题，RAG 会重新检索，浪费资源且无法利用之前的回答质量反馈。

记忆系统（Memory System）解决了这个问题——它存储交互历史、学习经历和抽象知识，让 Agent 能"记住"过去的检索结果和用户反馈。

### 11.2 协同架构

````
用户提问
   │
   ├──→ 记忆系统检索：是否有相关的历史经验？
   │         │
   │         ├── 有 → 将历史经验注入 Prompt 作为参考
   │         └── 无 → 继续
   │
   ├──→ RAG 检索：从知识库中检索相关文档
   │
   ├──→ LLM 生成回答（融合检索结果 + 历史经验）
   │
   └──→ 将本次检索结果自动存入记忆系统
            ├── 工作记忆：当前对话上下文
            ├── 情景记忆：本次问答事件（带时间戳）
            └── 语义记忆：提取的知识点（持久化）
````

### 11.3 关键实现：检索结果自动存入记忆

````python
def rag_with_memory(question, rag_tool, memory_tool):
    """RAG 检索 + 记忆协同"""
    # 1. 先查记忆：是否有相关历史经验
    history = memory_tool.execute("search", query=question, limit=3)

    # 2. RAG 检索知识库
    answer = rag_tool.execute("ask", question=question, limit=5)

    # 3. 将检索结果自动存入记忆系统
    memory_tool.execute("add",
        content=f"问题: {question}\n回答摘要: {answer[:200]}",
        memory_type="episodic",     # 情景记忆：记录这次问答事件
        importance=0.7,
        event_type="qa_interaction"
    )

    # 4. 如果回答中有高价值知识点，存入语义记忆（持久化）
    if is_high_value(answer):
        memory_tool.execute("add",
            content=extract_knowledge(answer),
            memory_type="semantic",  # 语义记忆：抽象知识，长期保存
            importance=0.9
        )

    return answer
````

> **关键代码解读（RAG + 记忆协同）：**
> - 先查记忆再查 RAG，避免重复检索
> - 每次 RAG 检索的结果自动写入情景记忆，形成知识积累
> - 高价值知识点升级为语义记忆（持久化），不会被遗忘策略清理
> - 这种"检索→生成→记忆"的闭环让 Agent 越用越聪明

> 🔑 **记忆整合（Consolidation）机制：** 工作记忆中的临时信息，如果重要性超过阈值（如 0.7），会被自动"整合"为情景记忆或语义记忆。这模拟了人类大脑将短期记忆转化为长期记忆的过程。经过多次整合，Agent 的语义记忆会逐渐形成一个结构化的知识体系。

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

### Q7: RAG 技术经历了哪几个发展阶段？各阶段的核心特征是什么？
**答：** 三个阶段：(1) 朴素 RAG（2020-2021）——TF-IDF/BM25 关键词匹配，文档直接拼接到 Prompt；(2) 高级 RAG（2022-2023）——稠密嵌入语义检索，引入查询重写、分块优化、Reranker；(3) 模块化 RAG（2023-至今）——混合检索、MQE、HyDE，各模块可插拔组合，与 Agent 和记忆系统深度融合。当前主流处于第二到第三阶段之间。

### Q8: 多查询扩展（MQE）和查询改写有什么区别？
**答：** 查询改写侧重"角度变换"（专业术语、口语化、场景化），MQE 侧重"语义等价"（同一意思的多种说法）。MQE 有标准化的扩展-检索-合并流程：LLM 生成 N 个等价查询 → 分别检索 → 去重合并 → 按分数排序。MQE 对模糊查询和专业术语查询效果显著，召回率可提升 30%-50%。

### Q9: Markdown 结构感知分块相比 RecursiveCharacterTextSplitter 有什么优势？
**答：** RecursiveCharacterTextSplitter 按固定字符数切分，不理解文档结构，会导致标题与正文分离、代码块被截断。结构感知分块利用 Markdown 标题层次（#/##/###）进行语义分割，每个 chunk 携带 heading_path 元数据，保证标题和正文在同一个 chunk 中。按 Token 数控制大小，更贴合 Embedding 模型输入限制。

### Q10: RAG 和记忆系统如何协同工作？
**答：** RAG 负责从外部知识库检索文档，记忆系统负责存储交互历史和学习积累。协同流程：先查记忆是否有相关历史经验 → RAG 检索知识库 → LLM 融合两者生成回答 → 检索结果自动存入情景记忆，高价值知识点存入语义记忆（持久化）。通过记忆整合机制，工作记忆中的重要信息会被升级为长期记忆，形成"检索→生成→记忆"的知识积累闭环。

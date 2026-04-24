# 第五阶段：记忆持久化与 RAG 优化

> 日期：2026-04-24 | 代码：`~/lib/langchain-project/`

相关主题：[[../LangChain专题/6-LangGraph编排]] | [[../LangChain专题/5-RAG实战]]

---

## 一、SQLite 记忆持久化

> 🔑 用 `AsyncSqliteSaver` 替换 `MemorySaver`，对话历史写入 SQLite 文件，服务器重启后对话不丢失。

### 1.1 为什么需要持久化

第四阶段所有模式（ReAct / Plan-and-Execute / 人机协作）都用 `MemorySaver`，对话历史存在内存中，服务器一重启就丢失。对于一个实际可用的问答系统，这是不可接受的。

### 1.2 AsyncSqliteSaver 的使用

`langgraph-checkpoint-sqlite` 提供两个类：

| 类 | 适用场景 | 导入路径 |
|---|---|---|
| `SqliteSaver` | 同步环境（终端脚本） | `langgraph.checkpoint.sqlite` |
| `AsyncSqliteSaver` | 异步环境（FastAPI） | `langgraph.checkpoint.sqlite.aio` |

**FastAPI 中的初始化**（server.py）：

````python
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
import aiosqlite

_db_path = str(Path(__file__).parent / "checkpoints.db")
_checkpointer = None

@app.on_event("startup")
async def init_checkpointer():
    """启动时初始化异步 SQLite checkpointer"""
    global _checkpointer, graph_app, plan_app, human_app
    conn = await aiosqlite.connect(_db_path)
    _checkpointer = AsyncSqliteSaver(conn=conn)
    await _checkpointer.setup()
    # 延迟编译 — checkpointer 就绪后才能 compile
    graph_app = graph.compile(checkpointer=_checkpointer)
    plan_app = plan_graph.compile(checkpointer=_checkpointer)
    human_app = human_graph.compile(
        checkpointer=_checkpointer,
        interrupt_before=["tools"],
    )
````

### 1.3 关键坑：延迟编译

`AsyncSqliteSaver` 需要一个 `aiosqlite.Connection`，而 `aiosqlite.connect()` 是异步调用，必须在事件循环中执行。模块加载时没有事件循环，所以：

1. **不能**在模块级别创建 checkpointer
2. **不能**在模块级别调用 `graph.compile(checkpointer=...)`
3. **必须**在 `@app.on_event("startup")` 中初始化 checkpointer 并编译图

模块级别用 `None` 占位：

````python
graph_app = None
plan_app = None
human_app = None
````

### 1.4 session 隔离

用 `thread_id` 区分不同会话，4 个模式共用同一个 SQLite 文件：

````python
# graph_chat 用 thread_id
config = {"configurable": {"thread_id": req.session_id}}

# plan_chat 用 plan_ 前缀
config = {"configurable": {"thread_id": f"plan_{req.session_id}"}}

# human_chat 用 human_ 前缀
config = {"configurable": {"thread_id": f"human_{req.session_id}"}}
````

### 1.5 /clear 端点

删除会话时需要清理所有模式的 checkpoint：

````python
@app.post("/clear")
async def clear(req: ClearRequest):
    store.pop(req.session_id, None)
    await _checkpointer.delete_thread(req.session_id)
    await _checkpointer.delete_thread(f"plan_{req.session_id}")
    await _checkpointer.delete_thread(f"human_{req.session_id}")
    return {"status": "ok"}
````

> 🔑 `delete_thread` 是异步方法，必须 `await`。同步调用会抛 `InvalidStateError`。

---

## 二、混合检索（向量 + BM25 + RRF）

> 🔑 向量检索擅长语义匹配，BM25 擅长关键词匹配，RRF 融合两者的优势。

### 2.1 为什么需要混合检索

纯向量检索的问题：

| 场景 | 纯向量 | 混合检索 |
|------|--------|---------|
| 查询含专有名词（如 "FAISS"、"LCEL"） | 可能匹配语义相似但不含关键词的文档 | BM25 直接匹配关键词 |
| 查询是自然语言描述 | 向量检索效果好 | 向量检索主导 |
| 查询与文档用词不同但语义相同 | 向量检索能捕获 | 向量检索主导 |

### 2.2 BM25 算法

BM25（Best Matching 25）是经典的信息检索算法，基于词频（TF）和逆文档频率（IDF）：

````python
# BM25 核心公式（简化）
score(query, doc) = Σ IDF(qi) · (f(qi, doc) · (k1 + 1)) / (f(qi, doc) + k1 · (1 - b + b · |doc|/avgdl))

# IDF(qi) = log((N - n(qi) + 0.5) / (n(qi) + 0.5) + 1)
# f(qi, doc) = 词 qi 在 doc 中的词频
# k1 = 1.5, b = 0.75（常用参数）
````

在项目中使用 `rank_bm25` 库的 `BM25Okapi` 实现。

### 2.3 中文分词

BM25 需要先对文本分词。英文按空格分即可，中文需要用 `jieba`：

````python
import jieba

tokens = list(jieba.cut("LCEL 是什么？"))
# → ['LCEL', ' ', '是', '什么', '？']
````

### 2.4 构建 BM25 索引

在 `build_index.py` 中，构建 FAISS 索引的同时保存 BM25 数据：

````python
import json, jieba

bm25_corpus = []
for chunk in chunks:
    tokens = list(jieba.cut(chunk.page_content))
    bm25_corpus.append({
        "tokens": tokens,
        "content": chunk.page_content,
        "source": chunk.metadata.get("source", ""),
    })

bm25_path = INDEX_DIR / "bm25_corpus.json"
with open(bm25_path, "w", encoding="utf-8") as f:
    json.dump(bm25_corpus, f, ensure_ascii=False)
````

### 2.5 RRF 融合排序

**RRF（Reciprocal Rank Fusion）** 是一种简单有效的排序融合算法：

````python
def _rrf_merge(vector_docs, bm25_docs, k=60, top_n=3):
    """RRF 融合排序
    score = 1/(k + rank_vector) + 1/(k + rank_bm25)
    k=60 是常用常数，rank 从 0 开始
    """
    score_map = {}  # doc_id → score

    for rank, doc in enumerate(vector_docs):
        doc_id = id(doc)
        score_map[doc_id] = score_map.get(doc_id, 0) + 1 / (k + rank)

    for rank, doc in enumerate(bm25_docs):
        doc_id = id(doc)
        score_map[doc_id] = score_map.get(doc_id, 0) + 1 / (k + rank)

    # 按 score 降序，取 top_n
    all_docs = {id(d): d for d in vector_docs + bm25_docs}
    sorted_ids = sorted(score_map, key=lambda x: score_map[x], reverse=True)
    return [all_docs[doc_id] for doc_id in sorted_ids[:top_n]]
````

### 2.6 混合检索流程

````python
# tools.py 中的 knowledge_search 实现

# 1. 向量检索（语义）
vector_docs = retriever.invoke(query)  # k=5

# 2. BM25 检索（关键词）
tokens = list(jieba.cut(query))
scores = bm25.get_scores(tokens)
top_indices = sorted(range(len(scores)), key=lambda i: scores[i], reverse=True)[:5]
bm25_results = [bm25_docs[i] for i in top_indices if scores[i] > 0]

# 3. RRF 融合
merged = _rrf_merge(vector_docs, bm25_results, k=60, top_n=3)
````

---

## 三、RAG 评测集

> 🔑 没有评测就没有优化方向。用 25 个 QA 对量化检索质量，建立基线。

### 3.1 评测指标

| 指标 | 计算方式 | 说明 |
|------|---------|------|
| **关键词命中率** | 命中关键词数 / 总关键词数 | 检索结果是否包含预期内容 |
| **来源准确率** | source_file 匹配的 QA 数 / 总 QA 数 | 检索结果是否来自正确文档 |
| **平均检索耗时** | 总耗时 / QA 数 | 性能基线 |

### 3.2 评测数据格式

````json
{
  "question": "LCEL 是什么？",
  "expected_keywords": ["LCEL", "管道", "Runnable", "LangChain Expression Language"],
  "source_file": "3-LCEL表达式语言.md"
}
````

25 个 QA 对覆盖：LangChain 基础（LCEL、Tool、Agent）、RAG（FAISS、Embedding、分块、混合检索）、LangGraph（StateGraph、条件路由、自纠错、Plan-and-Execute）、Android（Activity、Handler、Jetpack、Gradle）、算法（双指针、动态规划）。

### 3.3 评测脚本

`eval_rag.py` 复用 `tools.py` 的检索逻辑：

````python
from tools import _get_retriever, _get_bm25, _rrf_merge

def eval_one(question, expected_keywords, source_file):
    retriever = _get_retriever()
    bm25, bm25_docs = _get_bm25()

    # 混合检索
    vector_docs = retriever.invoke(question)
    bm25_results = [...]  # BM25 检索
    merged = _rrf_merge(vector_docs, bm25_results, k=60, top_n=3)

    # 计算指标
    all_text = " ".join(merged)
    hits = [kw for kw in expected_keywords if kw in all_text]
    keyword_hit = len(hits) / len(expected_keywords)
    source_hit = any(source_file in str(doc.metadata.get("source", "")) for doc in vector_docs)

    return {"keyword_hit": keyword_hit, "source_hit": source_hit, ...}
````

### 3.4 基线结果

```
关键词平均命中率: 72.5%
来源准确率:       52.0%
平均检索耗时:     41ms
```

> 🔑 72.5% 的关键词命中率作为基线，后续可通过调优 Embedding 模型、调整分块策略、增加 top_k 等方式提升。

---

## 四、踩坑记录

### 4.1 AsyncSqliteSaver 的 InvalidStateError

**错误**：`Synchronous calls to AsyncSqliteSaver are only allowed from a different thread`

**原因**：在 FastAPI 的主事件循环中同步调用了 `delete_thread()`。

**解决**：改用 `await _checkpointer.delete_thread()`。

### 4.2 langchain.text_splitter 路径变更

**错误**：`ModuleNotFoundError: No module named 'langchain.text_splitter'`

**原因**：LangChain 1.0 后分拆了子包。

**解决**：`from langchain.text_splitter` → `from langchain_text_splitters`

### 4.3 HuggingFace 离线模式

**问题**：`build_index.py` 和 `server.py` 加载 Embedding 模型时连接 huggingface.co 超时。

**解决**：设置环境变量 `HF_HUB_OFFLINE=1`，使用本地缓存。

### 4.4 不完整的 tool_calls 导致 400 错误

**错误**：`openai.BadRequestError: An assistant message with 'tool_calls' must be followed by tool messages responding to each 'tool_call_id'`

**原因**：graph 中途崩溃或被中断时，checkpointer 保存的状态可能包含一个 `AIMessage` 带有 `tool_calls`，但后面没有对应的 `ToolMessage`。下次用户发消息时，LangGraph 从 checkpointer 加载这个不完整的状态，LLM API 拒绝请求。

**场景复现**：
1. 用户发消息 → LLM 返回 tool_calls → 工具执行中服务崩溃
2. checkpointer 保存了 `[HumanMessage, AIMessage(tool_calls)]`，没有 ToolMessage
3. 重启后用户发新消息 → LangGraph 加载旧状态 + 新 HumanMessage → agent 节点调 LLM → 400

**解决**：在 `graph_agent_node` 中调用 LLM 前，用 `_clean_tool_calls` 过滤不完整的消息：

````python
def _clean_tool_calls(messages):
    """过滤没有对应 ToolMessage 的 AIMessage tool_calls"""
    from langchain_core.messages import AIMessage, ToolMessage

    responded_ids = set()
    for msg in messages:
        if isinstance(msg, ToolMessage):
            responded_ids.add(msg.tool_call_id)

    cleaned = []
    for msg in messages:
        if isinstance(msg, AIMessage) and hasattr(msg, "tool_calls") and msg.tool_calls:
            missing = [tc for tc in msg.tool_calls if tc["id"] not in responded_ids]
            if missing:
                continue  # 跳过不完整的 tool_calls
        cleaned.append(msg)
    return cleaned
````

> 🔑 **通用原则**：使用 checkpointer 持久化状态时，必须考虑状态不完整的情况。任何可能中断的流程都应该在恢复时做状态校验。

---

## 五、面试要点

**Q: 你的 Agent 记忆是怎么做的？**

用 LangGraph 的 checkpointer 机制，底层是 `AsyncSqliteSaver`，对话历史持久化到 SQLite 文件。每个会话用 `thread_id` 隔离，支持多会话并存。服务器重启后历史不丢失。

**Q: RAG 检索用了什么策略？**

混合检索：向量检索（语义匹配）+ BM25（关键词匹配），用 RRF（Reciprocal Rank Fusion）融合排序。向量检索用 FAISS + bge-zh 模型，BM25 用 jieba 中文分词 + rank_bm25 库。融合后取 top 3。

**Q: 怎么评估 RAG 效果？**

构建了 25 个 QA 对的评测集，指标是关键词命中率（72.5%）和来源准确率（52%）。评测脚本复用线上检索逻辑，确保评测环境与生产一致。

**Q: RRF 是怎么工作的？**

每个检索方法对文档排名，RRF 按公式 `score = 1/(k + rank)` 融合多个排名。k=60 是常用常数。最终按融合分数排序取 top_n。优点是不需要归一化不同检索方法的分数。

---

## 六、涉及的文件改动

| 文件 | 改动 |
|------|------|
| `server.py` | MemorySaver → AsyncSqliteSaver，延迟编译，await delete_thread，_clean_tool_calls 修复 400 |
| `graph_agent.py` | MemorySaver → SqliteSaver（同步版） |
| `tools.py` | 新增 BM25 检索 + RRF 融合 |
| `build_index.py` | 新增 jieba 分词 + bm25_corpus.json 保存 |
| `eval_rag.py` | 新建，RAG 评测脚本 |
| `eval_set.json` | 新建，25 个 QA 对 |
| `requirements.txt` | 新增 langgraph-checkpoint-sqlite、rank_bm25、jieba |

# Agent 记忆系统设计

## 1. 为什么 Agent 需要记忆

LLM 本身是无状态的——每次调用都是独立的。Agent 需要记忆系统来：

- 维持多轮对话的连贯性
- 记住用户偏好和历史交互
- 积累经验，避免重复犯错
- 在长任务中保持上下文

## 2. 记忆类型

### 2.1 认知科学理论基础：Atkinson-Shiffrin 记忆模型

> 🔑 **核心理论**：Agent 记忆系统的设计深受认知心理学启发。1968 年，Atkinson 和 Shiffrin 提出了经典的**多存储记忆模型**（Multi-Store Model），将人类记忆划分为三个独立但相互关联的存储系统。这一理论是几乎所有 Agent 记忆架构的理论基石。

**人类记忆的三层结构：**

```
外部刺激
    ↓
┌──────────────────────────────────────────────────────────┐
│  感觉记忆（Sensory Memory）                               │
│  · 持续时间：0.5 - 3 秒                                    │
│  · 容量：巨大（所有感官输入）                                │
│  · 功能：暂存感官接收到的原始信息，由注意力筛选进入工作记忆      │
│  · 对应 Agent：消息队列、输入缓冲区、多模态预处理管道           │
└──────────────────┬───────────────────────────────────────┘
                   ↓（注意力筛选）
┌──────────────────────────────────────────────────────────┐
│  工作记忆（Working Memory）                                │
│  · 持续时间：15 - 30 秒（可复述延长）                        │
│  · 容量有限：7 ± 2 个项目（Miller, 1956）                   │
│  · 功能：当前任务的信息处理、推理和决策                        │
│  · 对应 Agent：Context Window、滑动窗口、推理中间状态          │
└──────────────────┬───────────────────────────────────────┘
                   ↓（编码 + 复述）
┌──────────────────────────────────────────────────────────┐
│  长期记忆（Long-term Memory）                              │
│  · 持续时间：可达终生                                       │
│  · 容量：几乎无限                                          │
│  · 三种子类型：                                             │
│    ├─ 程序性记忆（Procedural）：技能与习惯，如骑自行车         │
│    │   → Agent 对应：工具使用模板、函数调用模式               │
│    ├─ 语义记忆（Semantic）：一般知识与概念，如"巴黎是法国首都"  │
│    │   → Agent 对应：知识图谱、向量数据库、用户画像            │
│    └─ 情景记忆（Episodic）：个人经历与事件，如"昨天的会议"     │
│        → Agent 对应：对话历史、事件日志、操作轨迹              │
└──────────────────────────────────────────────────────────┘
```

> 🔑 **从认知到工程的映射**：Atkinson-Shiffrin 模型中的"编码 → 存储 → 检索"三个核心过程，直接对应了 Agent 记忆系统的写入、持久化和召回三个操作。模型中的"遗忘"机制则启发了 Agent 的记忆清理和压缩策略。

### 2.2 记忆类型总览

````
┌─────────────────────────────────────────────┐
│              Agent 记忆体系                   │
├─────────────┬───────────────────────────────┤
│ 短期记忆     │ 当前对话上下文                  │
│ (Working)   │ 生命周期: 单次会话              │
├─────────────┼───────────────────────────────┤
│ 长期记忆     │ 持久化的知识和经验              │
│ (Long-term) │ 生命周期: 跨会话持久            │
├─────────────┼───────────────────────────────┤
│ 情景记忆     │ 具体的历史事件和交互            │
│ (Episodic)  │ "上次用户问了什么"              │
├─────────────┼───────────────────────────────┤
│ 语义记忆     │ 通用知识和事实                  │
│ (Semantic)  │ "Python 是一种编程语言"         │
├─────────────┼───────────────────────────────┤
│ 程序记忆     │ 如何执行任务的经验              │
│ (Procedural)│ "处理 CSV 应该先检查编码"       │
└─────────────┴───────────────────────────────┘
````
### 2.3 记忆形成的认知过程与工程映射

> 🔑 **核心洞察**：人类记忆的形成不是简单的"写入-读取"，而是经历编码、存储、检索、整合、遗忘五个认知阶段。将这五个阶段映射到工程实现，能帮助我们设计出更符合人类认知规律的 Agent 记忆系统。

| 认知阶段 | 人类行为 | Agent 工程实现 | 关键技术 |
|---------|---------|---------------|---------|
| **编码** | 将感知信息转换为可存储的神经信号 | 将原始文本/多模态数据转换为结构化表示 | Embedding 模型、NLP 预处理、实体提取 |
| **存储** | 将编码后的信息保存在神经网络中 | 将结构化数据持久化到存储后端 | 向量数据库、图数据库、关系型数据库 |
| **检索** | 根据线索从记忆中提取相关信息 | 根据查询从存储中召回相关记忆 | 语义检索、图查询、混合排序算法 |
| **整合** | 将短期记忆固化为长期记忆（睡眠中完成） | 将工作记忆中的重要内容提升为长期记忆 | 记忆整合（consolidate）、摘要压缩 |
| **遗忘** | 删除不重要或过时的信息（选择性遗忘） | 清理低价值记忆，控制存储规模 | 遗忘策略（重要性/时间/容量） |

**工程实现中的典型流程：**

```
用户输入 "我最近在学 Rust"
    ↓ 编码阶段
Embedding 模型 → 向量 [0.12, -0.34, ...]
NLP 提取 → 实体: {用户, Rust}，关系: {正在学习}
    ↓ 存储阶段
Qdrant 向量库 ← 存入向量 + 元数据
Neo4j 图数据库 ← (用户)-[:正在学习]->(Rust)
SQLite ← 持久化原始文本和时间戳
    ↓ 检索阶段（后续用户问"我学什么语言来着"）
Query Embedding → 向量相似度检索 → 命中 "正在学 Rust"
图查询 → (用户)-[:正在学习]->(?) → 返回 Rust
    ↓ 整合阶段
工作记忆中反复出现 "学习 Rust" → 提升为语义记忆
    ↓ 遗忘阶段
30 天前的 "用户问了天气" → 重要性低 + 时间久 → 自动清理
```

> 🔑 **关键设计原则**：编码质量决定检索上限。如果 Embedding 模型无法捕捉"学 Rust"和"学习 Rust 语言"的语义等价性，后续所有检索都会失败。因此，选择合适的 Embedding 模型是记忆系统的第一优先级。

## 3. 短期记忆（对话上下文）

> **本节核心问题：** 对话越来越长，LLM 的上下文窗口（Token 上限）不够用了怎么办？下面 4 种策略是逐步进化的关系：完整历史 → 滑动窗口 → Token 感知截断 → 摘要记忆，每一种都是为了解决前一种的缺陷。

### 3.1 完整历史

> **思路：** 最朴素的方案——把所有对话消息原封不动地全部传给 LLM。优点是实现零成本、信息零丢失；缺点是 Token 消耗随对话线性增长，一旦超出模型的上下文窗口（如 GPT-4o 的 128K Token），就会报错或被截断。因此只适合非常短的对话场景。

````python
# 最简单: 保留所有对话历史
messages = [
    {"role": "system", "content": "你是一个助手"},
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好！有什么可以帮你的？"},
    {"role": "user", "content": "帮我写个排序算法"},
    # ... 随着对话增长，Token 消耗越来越大
]
````
问题：上下文窗口有限，对话太长会超出限制。

> **小结：** 完整历史方案在生产环境中几乎不可用，但它是理解后续策略的基础——后面的每种策略本质上都是在回答"哪些消息可以丢掉/压缩"这个问题。

### 3.2 滑动窗口

> **思路：** 只保留最近 N 条消息，超出的直接丢弃。核心技巧是始终保留第一条 system prompt（它定义了 Agent 的角色），然后只保留最近 `max_messages` 条对话。相比完整历史，Token 消耗有了上限，但缺点是早期的重要信息（比如用户一开始提出的需求）会被无差别丢弃。

````python
from langchain_core.chat_history import InMemoryChatMessageHistory

class SlidingWindowMemory:
    def __init__(self, max_messages: int = 20):
        self.messages = []
        self.max_messages = max_messages
    
    def add(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})
        # 保留系统消息 + 最近 N 条
        if len(self.messages) > self.max_messages + 1:
            system = self.messages[0]  # 保留 system prompt
            self.messages = [system] + self.messages[-(self.max_messages):]
    
    def get_messages(self):
        return self.messages
````

> **关键代码解读：** `self.messages = [system] + self.messages[-(self.max_messages):]` 这一行是核心——用 Python 的负数切片取最后 N 条消息，再把 system prompt 拼回开头。简单但有效。
>
> **局限性：** 按"条数"裁剪不够精确，因为每条消息长度不同。一条消息可能只有 5 个 Token，另一条可能有 500 个。所以引出了下一个策略。

### 3.3 Token 感知截断

> **思路：** 不按条数，而是按实际 Token 数来裁剪。使用 `tiktoken` 库精确计算每条消息占用的 Token 数，当总量超过预算时，从最早的消息开始逐条删除（但始终保留 system prompt 和最新一条消息）。这样能最大化利用上下文窗口。

````python
import tiktoken

class TokenAwareMemory:
    def __init__(self, max_tokens: int = 4000, model: str = "gpt-4o"):
        self.max_tokens = max_tokens
        self.encoder = tiktoken.encoding_for_model(model)
        self.messages = []
    
    def add(self, role: str, content: str):
        self.messages.append({"role": role, "content": content})
        self._trim()
    
    def _trim(self):
        """从最早的消息开始删除，直到总 Token 数在限制内"""
        while self._total_tokens() > self.max_tokens and len(self.messages) > 2:
            # 保留第一条（system）和最后一条
            self.messages.pop(1)
    
    def _total_tokens(self) -> int:
        return sum(len(self.encoder.encode(m["content"])) for m in self.messages)
````

> **关键代码解读：**
> - `tiktoken.encoding_for_model(model)` — 获取指定模型的 tokenizer，不同模型的 Token 编码方式不同
> - `self.messages.pop(1)` — 删除索引 1（即 system 之后最早的那条），而不是 pop(0)，因为索引 0 是 system prompt 要保留
> - `_total_tokens()` — 遍历所有消息计算总 Token 数，作为裁剪的判断依据
>
> **对比滑动窗口：** 滑动窗口可能保留了 20 条短消息只用了 200 Token，也可能 20 条长消息用了 20000 Token。Token 感知截断则确保总量始终在预算内，更可控。

### 3.4 摘要记忆

> **思路：** 前面的策略都是"丢弃"早期消息，信息不可避免地会丢失。摘要记忆换了个思路——不丢弃，而是让 LLM 把早期对话**压缩成一段摘要**。这样既控制了 Token 数量，又保留了早期对话的关键信息。这是生产环境中最推荐的方案。

对话太长时，将早期对话压缩为摘要：

````python
class SummaryMemory:
    def __init__(self, llm, max_messages: int = 10):
        self.llm = llm
        self.max_messages = max_messages
        self.summary = ""
        self.recent_messages = []
    
    def add(self, role: str, content: str):
        self.recent_messages.append({"role": role, "content": content})
        
        if len(self.recent_messages) > self.max_messages:
            # 将最早的一半消息压缩为摘要
            half = len(self.recent_messages) // 2
            to_summarize = self.recent_messages[:half]
            self.recent_messages = self.recent_messages[half:]
            
            summary_input = "\n".join(
                f"{m['role']}: {m['content']}" for m in to_summarize
            )
            self.summary = self.llm.invoke(
                f"请将以下对话压缩为简洁的摘要:\n"
                f"之前的摘要: {self.summary}\n"
                f"新对话:\n{summary_input}"
            ).content
    
    def get_messages(self):
        messages = []
        if self.summary:
            messages.append({
                "role": "system",
                "content": f"之前的对话摘要: {self.summary}"
            })
        messages.extend(self.recent_messages)
        return messages
````

> **关键代码解读：**
> - 当消息数超过 `max_messages` 时，取前一半消息交给 LLM 压缩为摘要
> - `self.summary` 是累积的——每次压缩时会把"之前的摘要"也传给 LLM，让它在旧摘要基础上整合新内容
> - `get_messages()` 返回时，把摘要作为一条 system 消息放在最前面，后面跟最近的原始消息
>
> **四种策略对比总结：**
> | 策略 | Token 控制 | 信息保留 | 额外开销 | 适用场景 |
> |------|-----------|---------|---------|----------|
> | 完整历史 | ❌ 无控制 | ✅ 完整 | 无 | 极短对话 |
> | 滑动窗口 | ⚠️ 按条数 | ❌ 丢弃早期 | 无 | 简单场景 |
> | Token 感知 | ✅ 精确 | ❌ 丢弃早期 | 无 | 需要精确控制 |
> | 摘要记忆 | ✅ 可控 | ⚠️ 压缩保留 | 额外 LLM 调用 | 生产环境推荐 |

## 4. 长期记忆

> **短期 vs 长期：** 短期记忆只在当前会话内有效，关掉就没了。长期记忆需要持久化存储，让 Agent 在下次会话时还能"记得"之前的交互。本质上就是对用户历史做 RAG。

### 4.1 基于向量数据库

> **思路：** 使用向量数据库（这里用 ChromaDB）存储历史记忆。存储时把文本通过 embedding 模型转成向量；检索时把当前问题也转成向量，通过向量相似度找出最相关的 K 条历史记忆。这和 RAG 的原理完全一样——只不过"知识库"换成了用户的历史交互。

````python
import chromadb
from datetime import datetime

class LongTermMemory:
    def __init__(self, embeddings):
        self.client = chromadb.PersistentClient(path="./memory_db")
        self.collection = self.client.get_or_create_collection("agent_memory")
        self.embeddings = embeddings
    
    def store(self, content: str, metadata: dict = None):
        """存储记忆"""
        vector = self.embeddings.embed_query(content)
        meta = metadata or {}
        meta["timestamp"] = datetime.now().isoformat()
        
        self.collection.add(
            embeddings=[vector],
            documents=[content],
            metadatas=[meta],
            ids=[f"mem_{datetime.now().timestamp()}"]
        )
    
    def recall(self, query: str, k: int = 5) -> list[str]:
        """检索相关记忆"""
        vector = self.embeddings.embed_query(query)
        results = self.collection.query(
            query_embeddings=[vector],
            n_results=k
        )
        return results["documents"][0]
    
    def recall_recent(self, k: int = 10) -> list[str]:
        """获取最近的记忆"""
        results = self.collection.get(
            limit=k,
            include=["documents", "metadatas"]
        )
        # 按时间排序
        pairs = zip(results["documents"], results["metadatas"])
        sorted_pairs = sorted(pairs, key=lambda x: x[1].get("timestamp", ""), reverse=True)
        return [p[0] for p in sorted_pairs]
````

> **关键代码解读：**
> - `store()` — 存储一条记忆：文本 → embedding 向量 → 存入 ChromaDB，同时记录时间戳等元数据
> - `recall(query, k)` — 语义检索：把 query 转成向量，在向量库中找最相似的 k 条记忆返回
> - `recall_recent(k)` — 按时间检索：取最近的 k 条记忆，按时间倒序排列
> - `PersistentClient` — 数据持久化到磁盘，重启后记忆不丢失
>
> **实际使用时：** Agent 每次回答前，先用当前用户问题调用 `recall()` 检索相关历史记忆，注入到 prompt 中，让 LLM 能参考历史经验来回答。

### 4.2 用户画像记忆

> **思路：** 除了存储具体的对话记忆，还可以从对话中提取结构化的用户信息——偏好、技术专长、交互风格等，形成一个"用户画像"。这样 Agent 不需要每次都检索大量历史，只需加载一个精简的画像就能"认识"用户。

````python
class UserProfileMemory:
    def __init__(self, llm):
        self.llm = llm
        self.profile = {
            "preferences": [],
            "expertise": [],
            "interaction_style": "",
            "common_tasks": []
        }
    
    def update_from_conversation(self, messages: list):
        """从对话中提取用户信息更新画像"""
        conversation = "\n".join(f"{m['role']}: {m['content']}" for m in messages)
        
        extraction = self.llm.invoke(f"""
        从以下对话中提取用户信息:
        {conversation}
        
        当前用户画像: {self.profile}
        
        请输出更新后的用户画像（JSON 格式），包含:
        - preferences: 用户偏好
        - expertise: 技术专长
        - interaction_style: 交互风格偏好
        - common_tasks: 常见任务类型
        """)
        
        self.profile = json.loads(extraction.content)
    
    def get_context(self) -> str:
        return f"用户画像: {json.dumps(self.profile, ensure_ascii=False)}"
````

> **关键代码解读：**
> - `update_from_conversation()` — 把对话内容和当前画像一起传给 LLM，让 LLM 提取新信息并更新画像 JSON
> - `get_context()` — 返回画像的字符串表示，可以直接拼入 system prompt
> - 画像是**增量更新**的：每次对话后更新，而不是从零开始重建
>
> **向量记忆 vs 用户画像：** 向量记忆存的是"具体事件"（情景记忆），用户画像存的是"抽象总结"（语义记忆）。两者互补——画像提供全局认知，向量记忆提供具体细节。

### 4.3 知识图谱语义记忆：Neo4j + Qdrant 混合架构

> 🔑 **为什么需要知识图谱？** 向量数据库擅长"模糊语义匹配"（"学 Rust" ≈ "学习 Rust 语言"），但不擅长"结构化关系推理"（"用户正在学什么语言？"、"Rust 和 Go 哪个用户更熟悉？"）。知识图谱用节点和边显式建模实体与关系，弥补了向量检索在精确关系查询上的不足。两者结合是语义记忆的最优方案。

**混合架构设计：**

```
┌─────────────────────────────────────────────────────────┐
│                    语义记忆层                              │
│                                                         │
│  ┌──────────────────┐      ┌──────────────────────┐    │
│  │  Qdrant 向量库    │      │  Neo4j 图数据库       │    │
│  │                  │      │                      │    │
│  │  · 文本向量存储   │      │  · 实体节点           │    │
│  │  · 语义相似度检索 │      │  · 关系边             │    │
│  │  · 模糊匹配      │      │  · 属性图存储         │    │
│  └────────┬─────────┘      └──────────┬───────────┘    │
│           │                           │                 │
│           └─────────┬─────────────────┘                 │
│                     ↓                                   │
│            ┌────────────────┐                           │
│            │  混合排序引擎   │                           │
│            │ 向量分×0.7     │                           │
│            │ + 图分×0.3     │                           │
│            │ × 重要性权重   │                           │
│            └────────────────┘                           │
└─────────────────────────────────────────────────────────┘
```

**实体提取与关系抽取流程：**

```python
def add_semantic_memory(content: str, embedder, qdrant_store, neo4j_store):
    """添加语义记忆：同时写入向量库和图数据库"""

    # 1. 生成文本嵌入 → 存入 Qdrant
    embedding = embedder.encode(content)
    qdrant_store.add_vectors(
        vectors=[embedding.tolist()],
        metadata=[{"content": content, "memory_type": "semantic"}]
    )

    # 2. 实体提取 → 存入 Neo4j
    entities = extract_entities(content)  # NLP 实体识别
    # 例如："用户正在学习 Rust" → 实体: [用户, Rust]

    # 3. 关系抽取 → 存入 Neo4j
    relations = extract_relations(content, entities)
    # 例如：关系: (用户)-[:正在学习]->(Rust)

    # 4. 写入图数据库
    for entity in entities:
        neo4j_store.create_node(label=entity.type, properties=entity.props)
    for relation in relations:
        neo4j_store.create_edge(
            from_node=relation.source,
            to_node=relation.target,
            rel_type=relation.type
        )
```

**混合检索策略：**

```python
def search_semantic_memory(query: str, qdrant_store, neo4j_store, limit=5):
    """混合检索：向量语义 + 图关系推理"""

    # 1. 向量检索：找语义相似的记忆
    query_vec = embedder.encode(query)
    vector_results = qdrant_store.search_similar(query_vec, limit=limit * 2)

    # 2. 图检索：找关系关联的记忆
    graph_results = neo4j_store.graph_search(query, limit=limit * 2)

    # 3. 混合排序
    combined = {}
    for r in vector_results:
        combined[r["id"]] = {**r, "vector_score": r["score"], "graph_score": 0.0}
    for r in graph_results:
        if r["id"] in combined:
            combined[r["id"]]["graph_score"] = r["similarity"]
        else:
            combined[r["id"]] = {**r, "vector_score": 0.0, "graph_score": r["similarity"]}

    # 评分公式：(向量×0.7 + 图×0.3) × (0.8 + 重要性×0.4)
    for item in combined.values():
        base = item["vector_score"] * 0.7 + item["graph_score"] * 0.3
        importance_weight = 0.8 + (item.get("importance", 0.5) * 0.4)
        item["final_score"] = base * importance_weight

    sorted_results = sorted(combined.values(), key=lambda x: x["final_score"], reverse=True)
    return sorted_results[:limit]
```

> 🔑 **权重设计的原因**：向量权重 0.7 高于图权重 0.3，因为大多数查询是语义模糊的（"帮我找之前聊过的编程话题"），向量检索更擅长处理这类查询。图检索权重 0.3 作为补充，在精确关系查询（"用户学过哪些语言？"）时发挥关键作用。重要性权重范围 [0.8, 1.2] 避免过度影响相似度排序。

**适用场景对比：**

| 查询类型 | 向量检索 | 图检索 | 推荐策略 |
|---------|---------|--------|---------|
| "之前聊过什么编程话题" | ✅ 擅长 | ❌ 需要精确实体 | 向量为主 |
| "用户学过哪些语言" | ❌ 列举型查询 | ✅ 图遍历 | 图为主 |
| "Rust 和 Go 的区别" | ✅ 语义匹配 | ✅ 属性对比 | 混合 |
| "上次讨论的项目进展" | ✅ 语义+时间 | ✅ 事件链 | 混合 |

## 5. 状态管理

> **记忆 vs 状态：** 前面讲的"记忆"关注的是对话内容的存储和检索；"状态管理"关注的是 Agent 执行任务过程中的中间状态（当前进度、已完成的步骤、待处理的子任务等）。LangGraph 通过 Checkpointer 机制把这两者统一起来。

### 5.1 LangGraph 状态管理

> **思路：** LangGraph 用 `TypedDict` 定义 Agent 的状态结构，用 `Checkpointer` 自动持久化状态。每个会话通过 `thread_id` 唯一标识，同一个 `thread_id` 的多次调用会自动恢复上次的状态，实现"有状态的 Agent 服务"。

````python
from langgraph.graph import StateGraph
from langgraph.checkpoint.memory import MemorySaver
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    context: str
    task_status: str
    iteration: int

# 使用 Checkpointer 持久化状态
memory = MemorySaver()

graph = StateGraph(AgentState)
# ... 添加节点和边 ...
app = graph.compile(checkpointer=memory)

# 使用 thread_id 区分不同会话
config = {"configurable": {"thread_id": "user_001_session_1"}}

# 第一次调用
result = app.invoke({"messages": ["你好"], "iteration": 0}, config)

# 第二次调用（自动恢复状态）
result = app.invoke({"messages": ["继续上次的任务"], "iteration": 0}, config)
````

> **关键代码解读：**
> - `AgentState(TypedDict)` — 定义状态结构，包含消息列表、上下文、任务状态、迭代次数等字段
> - `Annotated[list, operator.add]` — 表示 `messages` 字段的更新方式是"追加"而非"覆盖"，每次 invoke 传入的新消息会追加到已有列表
> - `MemorySaver()` — 内存级别的状态持久化，适合开发调试
> - `thread_id` — 会话唯一标识，同一个 thread_id 的调用共享状态。第二次调用时 LangGraph 自动从 checkpointer 恢复上次的完整状态
>
> **核心价值：** 开发者不需要手动管理状态的保存和恢复，LangGraph 框架自动处理。

### 5.2 持久化到数据库

> **思路：** `MemorySaver` 是内存存储，进程重启就丢了。生产环境需要持久化到数据库。只需把 checkpointer 换成 `PostgresSaver`，其他代码完全不变——这就是 Checkpointer 抽象层的好处。

````python
from langgraph.checkpoint.postgres import PostgresSaver

# 使用 PostgreSQL 持久化
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/agent_db"
)

app = graph.compile(checkpointer=checkpointer)
````

> **小结：** 状态管理的核心就是 Checkpointer 模式——定义状态结构 → 选择存储后端 → 用 thread_id 区分会话。框架自动处理序列化/反序列化和状态恢复。

## 6. 记忆系统设计模式

> **本节讲什么：** 前面分别介绍了短期记忆、长期记忆、状态管理的具体实现。本节讲的是如何把它们**组合**起来形成一个完整的记忆系统，以及两个重要的维护机制：记忆衰减和记忆整合。

### 6.1 分层记忆架构

> **核心思想：** Agent 每次调用 LLM 时，不是只传当前消息，而是把多层记忆按优先级组合成一个完整的上下文。从上到下依次是：系统指令 → 用户画像 → 相关长期记忆 → 对话摘要 → 最近消息 → 当前输入。每一层有不同的生命周期和更新频率。

````
┌─────────────────────────────────┐
│         Agent 调用时             │
│                                 │
│  1. 检查短期记忆（当前对话）      │
│  2. 检索长期记忆（相关经验）      │
│  3. 加载用户画像                 │
│  4. 组合为完整上下文             │
│                                 │
│  System Prompt                  │
│  + 用户画像                     │
│  + 相关长期记忆 (Top-K)         │
│  + 对话摘要                     │
│  + 最近 N 条消息                │
│  + 当前用户输入                 │
└─────────────────────────────────┘
````

> **为什么要分层：** 如果把所有记忆平铺塞进 prompt，Token 会爆炸。分层架构让每层只取最关键的信息——用户画像是高度压缩的（几百 Token），长期记忆只取 Top-K 条相关的，对话摘要也是压缩过的。这样在有限的上下文窗口里塞入了最大价值的信息。

### 6.2 记忆衰减

> **问题：** 长期记忆会不断积累，时间久了检索噪声越来越大（很多过时的、不相关的记忆）。记忆衰减机制模拟人类的遗忘曲线，让不重要的记忆逐渐"淡化"，让系统聚焦于最有价值的记忆。

````python
import math
from datetime import datetime, timedelta

def memory_score(relevance: float, created_at: datetime, 
                 access_count: int) -> float:
    """综合评分: 相关性 × 时间衰减 × 访问频率"""
    # 时间衰减（半衰期 7 天）
    age_days = (datetime.now() - created_at).days
    time_decay = math.exp(-0.1 * age_days)
    
    # 访问频率加成
    frequency_boost = math.log(1 + access_count)
    
    return relevance * time_decay * (1 + 0.1 * frequency_boost)
````

> **关键代码解读：**
> - 三个评分因子相乘：`relevance`（语义相关性，0~1）× `time_decay`（时间衰减）× `frequency_boost`（访问频率加成）
> - `math.exp(-0.1 * age_days)` — 指数衰减，7 天后衰减到约 50%（半衰期），30 天后只剩约 5%
> - `math.log(1 + access_count)` — 对数增长，被频繁访问的记忆得分更高，但增长速度递减
>
> **使用场景：** 在 `recall()` 检索记忆时，不只看语义相似度，还要乘以衰减分数，最终按综合分排序返回 Top-K。

### 6.3 记忆整合

> **问题：** 随着时间推移，记忆库中会出现很多内容相似的记忆条目（比如用户多次问过类似的问题）。记忆整合就是把这些重复/相似的记忆合并成一条精简的总结，减少冗余。

````python
def consolidate_memories(memories: list[str], llm) -> str:
    """将多条相关记忆整合为一条"""
    return llm.invoke(f"""
    请将以下多条记忆整合为一条简洁的总结:
    
    {chr(10).join(f'- {m}' for m in memories)}
    
    要求:
    1. 保留关键信息
    2. 去除重复内容
    3. 保持逻辑连贯
    """).content
````

> **关键代码解读：** 把多条记忆拼成列表传给 LLM，要求它保留关键信息、去除重复、保持连贯。整合后的一条记忆替代原来的多条，既节省存储空间，也提高检索质量。
>
> **衰减 + 整合配合使用：** 定期运行整合任务，把相似记忆合并；同时用衰减机制降低过时记忆的权重。两者配合让记忆库始终保持精简和高质量。

### 6.4 遗忘策略工程实现

> 🔑 **遗忘不是缺陷，而是能力。** 认知心理学研究表明，人类的遗忘机制是一种主动的信息过滤——大脑会自动淘汰不重要、过时的信息，为新信息腾出空间。Agent 记忆系统同样需要遗忘策略，否则记忆库会无限膨胀，检索噪声越来越大。

**三种工程遗忘策略：**

```python
def forget_memories(strategy: str, memories: list, **kwargs) -> list:
    """统一遗忘接口：根据策略决定哪些记忆应该被清除"""

    if strategy == "importance_based":
        # 策略1：基于重要性 — 删除重要性低于阈值的记忆
        threshold = kwargs.get("threshold", 0.2)
        return [m for m in memories if m.importance >= threshold]

    elif strategy == "time_based":
        # 策略2：基于时间 — 删除超过指定天数的记忆
        max_age_days = kwargs.get("max_age_days", 30)
        cutoff = datetime.now() - timedelta(days=max_age_days)
        return [m for m in memories if m.created_at >= cutoff]

    elif strategy == "capacity_based":
        # 策略3：基于容量 — 当记忆数量超限时，删除最不重要的
        max_capacity = kwargs.get("max_capacity", 1000)
        if len(memories) <= max_capacity:
            return memories
        # 按重要性排序，保留前 max_capacity 条
        sorted_memories = sorted(memories, key=lambda m: m.importance, reverse=True)
        return sorted_memories[:max_capacity]
```

**三种策略的适用场景：**

| 策略 | 触发条件 | 优点 | 缺点 | 适用场景 |
|------|---------|------|------|---------|
| 基于重要性 | 定期清理 | 精准淘汰低价值记忆 | 需要准确的重要性评估 | 日常维护 |
| 基于时间 | 定期清理 | 实现简单，语义清晰 | 可能误删重要旧记忆 | 对话历史清理 |
| 基于容量 | 存储接近上限 | 确保系统稳定运行 | 可能丢失有价值的低分记忆 | 资源受限环境 |

> 🔑 **组合使用**：生产环境中通常组合多种策略。例如：先用"基于容量"确保不超限，再用"基于重要性"精细筛选，最后用"基于时间"兜底清理过期数据。这与人类大脑的选择性遗忘机制一致——不重要的快速遗忘，重要的长期保留，过时的逐渐淡化。

### 6.5 记忆系统四层架构

> 🔑 **架构原则**：大型 Agent 记忆系统通常采用四层架构设计，将基础设施、记忆类型、存储后端和嵌入服务解耦，每层可独立替换和扩展。

```
┌─────────────────────────────────────────────────────────────┐
│  第一层：基础设施层（Infrastructure Layer）                     │
│  · MemoryManager — 记忆管理器（统一调度和协调）                 │
│  · MemoryItem — 记忆数据结构（标准化记忆项：内容+元数据+向量）    │
│  · MemoryConfig — 配置管理（容量上限、TTL、衰减参数等）         │
│  · BaseMemory — 记忆基类（定义 add/retrieve/forget 接口）      │
├─────────────────────────────────────────────────────────────┤
│  第二层：记忆类型层（Memory Types Layer）                       │
│  · WorkingMemory — 工作记忆（临时信息，TTL 管理，纯内存）       │
│  · EpisodicMemory — 情景记忆（具体事件，时间序列存储）          │
│  · SemanticMemory — 语义记忆（抽象知识，图谱+向量混合）         │
│  · ProceduralMemory — 程序记忆（技能模板，经验回放）            │
├─────────────────────────────────────────────────────────────┤
│  第三层：存储后端层（Storage Backend Layer）                    │
│  · Qdrant/Chroma/Milvus — 向量存储（语义检索）                │
│  · Neo4j/ArangoDB — 图存储（关系推理）                        │
│  · SQLite/PostgreSQL — 文档存储（结构化持久化）                │
│  · Redis — 缓存存储（高速热点数据）                            │
├─────────────────────────────────────────────────────────────┤
│  第四层：嵌入服务层（Embedding Service Layer）                  │
│  · OpenAI/Cohere — 云端 Embedding API                       │
│  · Sentence-Transformers — 本地预训练模型                     │
│  · TF-IDF/BM25 — 轻量级统计方法（兜底方案）                   │
└─────────────────────────────────────────────────────────────┘
```

**分层设计的核心价值：**

- **可替换性**：想把向量库从 ChromaDB 换成 Qdrant？只需改第三层，上层代码不用动
- **可扩展性**：新增一种记忆类型（如"情感记忆"）？只需在第二层添加一个新类
- **可测试性**：每层可以独立单元测试，用 mock 替换依赖层
- **渐进式采用**：初期只用第二层（工作记忆）+ 第四层（TF-IDF），按需逐步引入图数据库等重型组件

> 🔑 **实践建议**：不要一开始就搭建完整的四层架构。从最简方案开始（工作记忆 + 向量数据库），随着需求增长逐层引入新组件。过度设计是记忆系统最常见的反模式。

## 7. 实践建议

````
1. 从简单开始
   - 先用滑动窗口，够用就不要过度设计
   - 需要跨会话记忆时再引入长期记忆

2. 记忆质量 > 数量
   - 不是所有对话都值得记住
   - 定期清理无用记忆

3. 隐私考虑
   - 用户数据加密存储
   - 提供记忆清除功能
   - 遵守数据保护法规

4. 性能优化
   - 记忆检索要快（< 100ms）
   - 异步存储，不阻塞主流程
   - 缓存热点记忆
````
---

## 面试题精选

### Q1: Agent 的短期记忆有哪些管理策略？各自的优缺点？
**答：** 完整历史（简单但 Token 无限增长）、滑动窗口（保留最近 N 条，可能丢失早期重要信息）、Token 感知截断（按 Token 预算裁剪，更精确）、摘要记忆（将早期对话压缩为摘要，兼顾信息保留和 Token 控制）。生产环境推荐摘要记忆。

### Q2: 长期记忆通常怎么实现？和 RAG 有什么关系？
**答：** 长期记忆通常用向量数据库存储，通过语义检索召回与当前对话相关的历史记忆。本质上就是对用户历史交互做 RAG——将历史对话/经验作为知识库，当前问题作为 query 检索相关记忆注入上下文。

### Q3: 记忆衰减机制是什么？为什么需要？
**答：** 记忆衰减根据时间、访问频率和相关性综合评分，让不重要的记忆逐渐"遗忘"。需要它是因为记忆无限积累会导致检索噪声增大、存储成本上升，衰减机制让系统聚焦于最有价值的记忆。

### Q4: 如何设计一个分层记忆架构？
**答：** 调用时组合多层记忆：System Prompt + 用户画像（长期）+ 语义检索的相关长期记忆（Top-K）+ 对话摘要（中期）+ 最近 N 条消息（短期）+ 当前输入。每层有不同的生命周期和更新策略。

### Q5: LangGraph 的 Checkpointer 解决了什么问题？
**答：** Checkpointer 将 Agent 的完整状态（消息历史、中间结果、任务进度等）持久化到存储中，支持跨请求恢复状态。通过 thread_id 区分不同会话，实现了有状态的 Agent 服务，解决了 LLM 无状态的根本问题。

### Q6: Agent 记忆系统设计需要注意哪些隐私和安全问题？
**答：** 用户数据必须加密存储、提供记忆清除功能让用户可以删除自己的数据、遵守 GDPR 等数据保护法规、不同用户的记忆严格隔离、敏感信息（密码、密钥等）不应存入记忆。

### Q7: Atkinson-Shiffrin 记忆模型如何映射到 Agent 记忆系统？
**答：** 三层映射：感觉记忆 → 输入缓冲区/消息队列（暂存原始输入）；工作记忆 → Context Window/滑动窗口（当前任务推理）；长期记忆 → 向量数据库/知识图谱（持久化知识和经验）。核心过程映射：编码 → Embedding 向量化，存储 → 持久化到数据库，检索 → 语义检索/图查询召回。

### Q8: 为什么语义记忆要同时使用向量数据库和图数据库？
**答：** 两者互补。向量数据库擅长模糊语义匹配（"聊过的编程话题"），但不擅长结构化关系推理（"用户学过哪些语言"）。图数据库用节点-边显式建模实体关系，支持多跳遍历和精确查询。混合检索公式：`(向量分×0.7 + 图分×0.3) × 重要性权重`，向量权重更高是因为大多数查询是语义模糊的。

### Q9: Agent 记忆系统有哪些遗忘策略？如何选择？
**答：** 三种策略：基于重要性（删除低分记忆，适合日常维护）、基于时间（删除过期记忆，适合对话历史清理）、基于容量（超限时淘汰最不重要的，适合资源受限环境）。生产环境通常组合使用：容量兜底 → 重要性筛选 → 时间清理。关键原则是遗忘不是缺陷，而是主动的信息过滤能力。

### Q10: 记忆系统的四层架构是什么？为什么要分层？
**答：** 四层：基础设施层（MemoryManager、MemoryItem、配置管理）→ 记忆类型层（工作/情景/语义/程序记忆）→ 存储后端层（向量库、图库、关系库）→ 嵌入服务层（云端API、本地模型、TF-IDF 兜底）。分层的核心价值是可替换性（换存储后端不影响上层）、可扩展性（新增记忆类型不改其他层）、可测试性（每层独立 mock 测试）。实践建议从最简方案开始，按需逐层引入。

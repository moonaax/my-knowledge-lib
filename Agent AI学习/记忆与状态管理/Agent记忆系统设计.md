# Agent 记忆系统设计

## 1. 为什么 Agent 需要记忆

LLM 本身是无状态的——每次调用都是独立的。Agent 需要记忆系统来：

- 维持多轮对话的连贯性
- 记住用户偏好和历史交互
- 积累经验，避免重复犯错
- 在长任务中保持上下文

## 2. 记忆类型

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

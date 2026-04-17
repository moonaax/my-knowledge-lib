# Agent 记忆系统设计

## 1. 为什么 Agent 需要记忆

LLM 本身是无状态的——每次调用都是独立的。Agent 需要记忆系统来：

- 维持多轮对话的连贯性
- 记住用户偏好和历史交互
- 积累经验，避免重复犯错
- 在长任务中保持上下文

## 2. 记忆类型

```
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
```

## 3. 短期记忆（对话上下文）

### 3.1 完整历史

```python
# 最简单: 保留所有对话历史
messages = [
    {"role": "system", "content": "你是一个助手"},
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好！有什么可以帮你的？"},
    {"role": "user", "content": "帮我写个排序算法"},
    # ... 随着对话增长，Token 消耗越来越大
]
```

问题：上下文窗口有限，对话太长会超出限制。

### 3.2 滑动窗口

```python
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
```

### 3.3 Token 感知截断

```python
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
```

### 3.4 摘要记忆

对话太长时，将早期对话压缩为摘要：

```python
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
```

## 4. 长期记忆

### 4.1 基于向量数据库

```python
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
```

### 4.2 用户画像记忆

```python
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
```

## 5. 状态管理

### 5.1 LangGraph 状态管理

```python
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
```

### 5.2 持久化到数据库

```python
from langgraph.checkpoint.postgres import PostgresSaver

# 使用 PostgreSQL 持久化
checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/agent_db"
)

app = graph.compile(checkpointer=checkpointer)
```

## 6. 记忆系统设计模式

### 6.1 分层记忆架构

```
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
```

### 6.2 记忆衰减

```python
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
```

### 6.3 记忆整合

```python
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
```

## 7. 实践建议

```
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
```

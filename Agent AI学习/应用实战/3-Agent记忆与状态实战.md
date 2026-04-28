# Agent 记忆与状态实战

> 本文以"多 Agent 协作的虚拟小镇"项目为背景，提炼出一套通用的 Agent 记忆管理、状态驱动 Prompt 和成本优化设计模式。这些模式可直接迁移到客服系统、虚拟助手、教育机器人等需要"有记忆、有性格、有状态"的 Agent 应用中。

相关主题：[[../记忆与状态管理/Agent记忆系统设计]] | [[1-Agent应用实战案例]]

---

## 一、Agent 记忆系统实战架构

### 1.1 两层记忆模型

在实际 Agent 应用中，单一的对话历史远远不够。一个能"记住用户"的 Agent 需要两层记忆协同工作：

| 层级 | 名称 | 存储方式 | 生命周期 | 典型容量 |
|------|------|---------|---------|---------|
| 短期记忆 | WorkingMemory | 内存列表 | 会话级，结束清空 | 10-20 条消息 |
| 长期记忆 | EpisodicMemory | 向量数据库 + 关系数据库 | 持久化 | 无上限 |

> 🔑 **核心思想：** 短期记忆保证对话连贯（指代消解、话题延续），长期记忆保证跨会话的个性化（记住用户偏好、历史事件）。两者配合才能实现"越用越懂你"的 Agent。

### 1.2 WorkingMemory 实现

短期记忆的本质是一个带容量和过期策略的消息队列：

```python
from collections import deque
from datetime import datetime, timedelta
from typing import List, Optional

class WorkingMemory:
    """短期记忆：当前会话的上下文窗口"""

    def __init__(self, capacity: int = 10, ttl_minutes: int = 120):
        self.capacity = capacity          # 最大消息数
        self.ttl = timedelta(minutes=ttl_minutes)  # 过期时间
        self.messages: deque = deque(maxlen=capacity)

    def add(self, role: str, content: str):
        """添加一条消息"""
        self.messages.append({
            "role": role,
            "content": content,
            "timestamp": datetime.now()
        })

    def get_recent_messages(self, n: int = 5) -> List[dict]:
        """获取最近 n 条未过期的消息"""
        cutoff = datetime.now() - self.ttl
        recent = [m for m in self.messages if m["timestamp"] > cutoff]
        return recent[-n:]

    def clear(self):
        """清空记忆（会话结束时调用）"""
        self.messages.clear()
```

**设计要点：**
- `deque(maxlen=capacity)` 自动淘汰最旧的消息，无需手动清理
- TTL 机制防止长时间未活跃的会话占用过期上下文
- `get_recent_messages` 同时考虑数量和时间，返回的是"有效窗口"

### 1.3 EpisodicMemory 实现

长期记忆的核心是将对话存入向量数据库，支持语义检索：

```python
from typing import List, Dict

class EpisodicMemory:
    """长期记忆：基于向量检索的历史对话"""

    def __init__(self, db_path: str, collection_name: str):
        self.db_path = db_path
        self.collection_name = collection_name
        # 实际项目中使用 ChromaDB / Qdrant / FAISS 等
        self.vector_store = self._init_vector_store()

    def add_interaction(self, user_msg: str, agent_reply: str,
                        metadata: dict = None):
        """将一次交互存入长期记忆"""
        # 将用户消息和 Agent 回复合并为一个记忆条目
        memory_text = f"用户: {user_msg}\n助手: {agent_reply}"
        self.vector_store.add(
            text=memory_text,
            metadata=metadata or {}
        )

    def search(self, query: str, top_k: int = 3) -> List[Dict]:
        """语义检索最相关的历史记忆"""
        results = self.vector_store.search(query, top_k=top_k)
        return [{"text": r.text, "score": r.score} for r in results]
```

### 1.4 两层记忆的协作流程

Agent 处理一次用户消息时，两层记忆的协作流程如下：

```python
def process_with_memory(agent, memory_mgr, user_message: str) -> str:
    """完整的记忆驱动对话流程"""

    # 1. 短期记忆：获取最近对话（保持上下文连贯）
    recent = memory_mgr.working_memory.get_recent_messages(n=5)

    # 2. 长期记忆：语义检索相关历史（实现个性化）
    relevant = memory_mgr.episodic_memory.search(
        query=user_message, top_k=3
    )

    # 3. 构建上下文：短期 + 长期 + 当前消息
    context_messages = []
    for m in recent:
        context_messages.append({"role": m["role"], "content": m["content"]})
    if relevant:
        history_text = "\n".join([r["text"] for r in relevant])
        context_messages.append({
            "role": "system",
            "content": f"以下是你们之前的相关对话:\n{history_text}"
        })
    context_messages.append({"role": "user", "content": user_message})

    # 4. 调用 LLM 生成回复
    reply = agent.llm.invoke(context_messages)

    # 5. 写入两层记忆
    memory_mgr.working_memory.add("user", user_message)
    memory_mgr.working_memory.add("assistant", reply)
    memory_mgr.episodic_memory.add_interaction(user_message, reply)

    return reply
```

> 🔑 **为什么不能只用长期记忆？** 向量检索是"模糊匹配"，无法精确还原最近 3 轮的对话顺序。短期记忆提供的是精确的、有序的上下文窗口，两者互补。

---

## 二、好感度系统设计

### 2.1 系统概述

好感度系统是一个典型的"状态量化 + 行为反馈"模式。它的核心思想是：**将 Agent 与用户的关系抽象为一个数值，用 LLM 自动分析每次交互的情感倾向来更新这个数值，再用数值驱动 Agent 的行为变化。**

这种模式不限于游戏，任何需要"根据关系深浅调整交互风格"的场景都可以使用：
- 客服 Agent：新客户 vs 老客户的回复风格不同
- 教育 Agent：根据学生的学习进度和信任度调整引导方式
- 健康 Agent：根据用户的配合度调整提醒强度

### 2.2 好感度等级划分

将连续的数值映射为离散的等级，每个等级对应不同的行为描述：

```python
AFFINITY_LEVELS = {
    "陌生": {"range": (0, 20),  "desc": "礼貌但保持距离，回复简短专业"},
    "熟悉": {"range": (21, 40), "desc": "正常交流，偶尔分享工作信息"},
    "友好": {"range": (41, 60), "desc": "把用户当朋友，主动分享更多信息"},
    "亲密": {"range": (61, 80), "desc": "非常信任，愿意分享私人话题"},
    "挚友": {"range": (81, 100),"desc": "无话不谈，分享内心想法和感受"},
}

def get_affinity_level(score: int) -> str:
    """根据分数获取好感度等级"""
    for level, info in AFFINITY_LEVELS.items():
        low, high = info["range"]
        if low <= score <= high:
            return level
    return "陌生"
```

### 2.3 LLM 情感分析驱动评分

好感度变化不是固定加减分，而是由 LLM 分析对话内容后动态决定：

```python
class RelationshipManager:
    """好感度管理器"""

    def __init__(self):
        self.affinity_data: dict = {}  # key: "npc_id_player_id" -> {score, level, count}

    def analyze_sentiment(self, player_message: str, agent_reply: str) -> int:
        """用 LLM 分析对话情感，返回好感度变化值"""
        prompt = f"""分析以下对话中用户的态度:
用户: {player_message}
助手: {agent_reply}

请判断用户的态度是:
1. 友好(+5分): 礼貌、热情、表示感谢或赞同
2. 中立(+2分): 普通的询问或陈述
3. 不友好(-3分): 粗鲁、冷漠、批评或否定

只返回数字，不要其他内容。"""

        response = self.llm.invoke([{"role": "user", "content": prompt}])
        try:
            score_change = int(response.strip())
            return max(-3, min(5, score_change))  # 限制范围
        except ValueError:
            return 2  # 解析失败默认中立

    def update_affinity(self, agent_id: str, user_id: str,
                        player_message: str, agent_reply: str) -> dict:
        """更新好感度并返回最新状态"""
        key = f"{agent_id}_{user_id}"

        if key not in self.affinity_data:
            self.affinity_data[key] = {"score": 0, "level": "陌生", "count": 0}

        # LLM 情感分析
        score_change = self.analyze_sentiment(player_message, agent_reply)

        # 更新分数（钳位到 0-100）
        current = self.affinity_data[key]["score"]
        new_score = max(0, min(100, current + score_change))

        self.affinity_data[key].update({
            "score": new_score,
            "level": get_affinity_level(new_score),
            "count": self.affinity_data[key]["count"] + 1,
        })
        return self.affinity_data[key]
```

> 🔑 **关键设计：** 让 LLM 做情感分析而不是写规则（正则/关键词），是因为自然语言的情感表达方式太多样了。"你这个人还行吧"和"太棒了谢谢！"用规则很难覆盖，但 LLM 可以轻松判断。

---

## 三、状态驱动的动态 Prompt

### 3.1 核心思想

传统 Agent 使用静态 Prompt，即系统提示词在整个会话中不变。但在需要"有状态"的 Agent 应用中，**Prompt 应该由外部状态变量动态生成**。

```
状态变量 ──→ Prompt 模板 ──→ 最终 System Prompt ──→ LLM 行为
```

状态变量可以是：
- **好感度等级**：决定 Agent 的语气亲疏
- **情绪状态**：决定 Agent 的回应态度
- **用户角色**：决定 Agent 的知识深度
- **对话阶段**：决定 Agent 的话术策略

### 3.2 好感度驱动的 Prompt 示例

这是最直观的"状态 → Prompt"映射。同一个 Agent，好感度不同，Prompt 不同，行为也不同：

```python
# 好感度等级对应的 Prompt 片段
AFFINITY_PROMPTS = {
    "陌生": "你刚认识这位用户，保持礼貌但不要过于热情。回复简短专业。",
    "熟悉": "你已经认识这位用户，可以进行正常的交流。回复自然友好。",
    "友好": "你把这位用户当作朋友，愿意分享更多信息。回复详细热情。",
    "亲密": "你非常信任这位用户，可以分享私人话题。回复充满关心。",
    "挚友": "你把这位用户当作最好的朋友，无话不谈。回复亲切真诚。",
}

def build_dynamic_system_prompt(agent_name: str, role: str,
                                personality: str, affinity_level: str) -> str:
    """根据状态变量动态构建系统提示词"""
    affinity_desc = AFFINITY_PROMPTS.get(affinity_level, AFFINITY_PROMPTS["陌生"])

    return f"""你是{agent_name}，一位{role}。
你的性格特点：{personality}

当前与用户的关系：{affinity_level}
{affinity_desc}

请根据你的角色、性格和与用户的关系，自然地回复。"""
```

### 3.3 状态驱动 vs 静态 Prompt 对比

| 维度 | 静态 Prompt | 状态驱动 Prompt |
|------|-----------|---------------|
| 实现复杂度 | 低 | 中等（需要状态管理） |
| 行为一致性 | 始终如一 | 随状态变化 |
| 用户体验 | 机械、缺乏层次 | 自然、有成长感 |
| 适用场景 | 工具型 Agent（翻译、摘要） | 陪伴型 Agent（客服、教育、社交） |
| 可调试性 | 简单 | 需要记录状态变化日志 |

> 🔑 **判断标准：** 如果你的 Agent 需要对同一个用户在不同阶段表现出不同的行为，就应该用状态驱动 Prompt。如果 Agent 的行为不依赖历史交互，静态 Prompt 足够。

### 3.4 多状态变量组合

实际项目中往往有多个状态变量同时影响 Prompt。推荐用数据类统一管理：

```python
from dataclasses import dataclass, field

@dataclass
class AgentState:
    """Agent 的外部状态集合"""
    affinity_level: str = "陌生"       # 好感度等级
    emotion: str = "neutral"           # 当前情绪
    interaction_count: int = 0         # 交互次数
    user_role: str = "user"            # 用户角色
    context_tags: list = field(default_factory=list)  # 上下文标签

    def to_prompt_segment(self) -> str:
        """将所有状态转为 Prompt 片段"""
        segments = []
        segments.append(f"与用户的关系：{self.affinity_level}")
        segments.append(f"当前情绪：{self.emotion}")
        if self.interaction_count <= 1:
            segments.append("这是你们的第一次对话")
        elif self.interaction_count < 10:
            segments.append(f"你们已经交流了{self.interaction_count}次")
        else:
            segments.append(f"你们是老朋友了，已经交流了{self.interaction_count}次")
        return "\n".join(segments)
```

---

## 四、批量生成与成本优化

### 4.1 问题背景

在多 Agent 系统中，如果每个 Agent 的每次交互都独立调用 LLM，成本会随着 Agent 数量线性增长。以一个有 10 个 NPC 的虚拟场景为例，每个 NPC 每分钟更新一次背景状态，一天就是 14400 次 API 调用——其中大部分只是生成一句"正在看文档"之类的背景文案。

### 4.2 批量生成策略

核心思想：**将多个 Agent 的同类请求合并为一次 LLM 调用**。

```python
import json
from typing import Dict, Optional

class BatchDialogueGenerator:
    """批量生成多个 Agent 的背景对话"""

    def __init__(self, agent_configs: Dict[str, dict]):
        self.agent_configs = agent_configs

    def generate_batch(self, context: Optional[str] = None) -> Dict[str, str]:
        """一次 LLM 调用生成所有 Agent 的背景对话"""
        # 构建 Agent 描述
        agent_descs = []
        for name, cfg in self.agent_configs.items():
            desc = f"- {name}({cfg['role']}): 在{cfg['location']}{cfg['activity']}，性格{cfg['personality']}"
            agent_descs.append(desc)

        prompt = f"""请为以下 Agent 生成当前的对话或行为描述。

【场景】{context or '正常工作时间'}

【Agent 信息】
{chr(10).join(agent_descs)}

【生成要求】
1. 每个 Agent 生成 1 句话（20-40 字）
2. 内容要符合角色设定和当前活动
3. 自然真实，像真实的人在自言自语
4. 必须严格按 JSON 格式返回

【输出格式】{{"Agent名": "对话内容", ...}}

请生成（只返回 JSON）："""

        response = self.llm.invoke([
            {"role": "system", "content": "你是一个对话生成器，擅长创作自然真实的对话。"},
            {"role": "user", "content": prompt}
        ])
        return json.loads(response)
```

### 4.3 混合模式：批量背景 + 即时响应

最优方案是将批量生成和即时响应结合：

```
┌─────────────────────────────────────────────┐
│  后台定时任务（每 5 分钟）                      │
│  → 批量生成所有 Agent 的背景对话               │
│  → 缓存到内存/Redis                           │
└─────────────────────────────────────────────┘
         ↓ 缓存的背景对话
┌─────────────────────────────────────────────┐
│  用户未交互时：显示缓存的背景对话               │
│  用户发起交互时：切换为即时 Agent 响应          │
└─────────────────────────────────────────────┘
```

```python
import asyncio

class HybridDialogueManager:
    """混合模式对话管理器"""

    def __init__(self, batch_generator, agent_map):
        self.batch_gen = batch_generator
        self.agents = agent_map          # agent_id -> Agent 实例
        self.background_cache = {}       # agent_id -> 背景对话

    async def start_background_update(self, interval_seconds: int = 300):
        """后台定时批量更新背景对话"""
        while True:
            try:
                dialogues = self.batch_gen.generate_batch()
                self.background_cache.update(dialogues)
            except Exception as e:
                print(f"批量生成失败: {e}")
            await asyncio.sleep(interval_seconds)

    def get_background_dialogue(self, agent_id: str) -> str:
        """获取 Agent 的背景对话（无 LLM 调用）"""
        return self.background_cache.get(agent_id, "...")

    async def get_interactive_response(self, agent_id: str,
                                        user_message: str) -> str:
        """获取 Agent 的即时响应（调用 LLM）"""
        agent = self.agents.get(agent_id)
        if not agent:
            raise ValueError(f"Agent {agent_id} 不存在")
        return await agent.run(user_message)
```

> 🔑 **成本对比：** 假设 10 个 Agent，每 5 分钟更新一次背景。批量模式：288 次/天 LLM 调用；逐个调用：14400 次/天。成本降低约 50 倍，同时用户交互的响应质量不受影响。

---

## 五、并发与状态管理

### 5.1 Agent 忙碌状态

在多用户或多 Agent 系统中，需要防止同一个 Agent 同时处理多个请求导致状态混乱：

```python
from datetime import datetime
from typing import Dict, Optional

class StateManager:
    """Agent 状态管理器"""

    def __init__(self):
        self.agent_states: Dict[str, dict] = {}

    def initialize(self, agent_configs: list):
        """初始化所有 Agent 的状态"""
        for cfg in agent_configs:
            self.agent_states[cfg["id"]] = {
                "id": cfg["id"],
                "name": cfg["name"],
                "is_busy": False,
                "current_action": "idle",
                "last_interaction": None,
            }

    def is_busy(self, agent_id: str) -> bool:
        """检查 Agent 是否正在处理请求"""
        agent = self.agent_states.get(agent_id)
        return agent["is_busy"] if agent else False

    def set_busy(self, agent_id: str, busy: bool):
        """设置 Agent 忙碌状态"""
        if agent_id in self.agent_states:
            self.agent_states[agent_id]["is_busy"] = busy
            if busy:
                self.agent_states[agent_id]["last_interaction"] = datetime.now().isoformat()
```

### 5.2 防并发对话的请求处理

在 API 层使用"忙碌检查 + try/finally"模式确保状态一致性：

```python
from fastapi import HTTPException

async def handle_dialogue(agent_id: str, user_message: str):
    """带防并发保护的对话处理"""

    # 1. 忙碌检查
    if state_manager.is_busy(agent_id):
        raise HTTPException(
            status_code=409,
            detail=f"Agent {agent_id} 正在处理其他请求，请稍后重试"
        )

    # 2. 标记为忙碌
    state_manager.set_busy(agent_id, True)

    try:
        # 3. 获取当前状态（好感度等）
        state = relationship_manager.get_state(agent_id)

        # 4. 动态构建 Prompt
        system_prompt = build_dynamic_system_prompt(
            agent_name=state["name"],
            role=state["role"],
            personality=state["personality"],
            affinity_level=state["affinity_level"]
        )

        # 5. 调用 Agent
        reply = await agent.run(user_message, system_prompt=system_prompt)

        # 6. 更新状态
        new_affinity = relationship_manager.update_affinity(
            agent_id, user_message, reply
        )

        return {"reply": reply, "affinity": new_affinity}

    finally:
        # 7. 无论如何都释放状态
        state_manager.set_busy(agent_id, False)
```

### 5.3 状态持久化策略

| 状态类型 | 存储位置 | 更新频率 | 持久化方式 |
|---------|---------|---------|-----------|
| 忙碌状态 | 内存 | 每次请求 | 不持久化（重启重置） |
| 好感度 | 内存 + 数据库 | 每次交互 | 定期批量写入或即时写入 |
| 短期记忆 | 内存 | 每次交互 | 不持久化（会话级） |
| 长期记忆 | 向量数据库 | 每次交互 | 即时写入 |

> 🔑 **设计原则：** 越是临时的状态越应该存在内存中（快但易失），越是需要跨会话保留的状态越应该持久化（慢但可靠）。忙碌状态是纯临时的，好感度和长期记忆是需要持久化的。

---

## 六、完整架构总览

将上述所有模式组合在一起，形成一个完整的"有记忆、有状态"的 Agent 系统：

```
用户消息
    ↓
┌──────────────────────────────────────────┐
│  状态管理层                                │
│  - 忙碌检查（防并发）                       │
│  - 好感度查询（状态变量）                    │
│  - 动态 Prompt 构建                        │
└──────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────┐
│  记忆管理层                                │
│  - WorkingMemory（短期，取最近 N 条）       │
│  - EpisodicMemory（长期，语义检索 Top-K）   │
│  - 合并构建上下文                           │
└──────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────┐
│  LLM 调用层                               │
│  - 动态 System Prompt + 记忆上下文 + 用户消息│
│  - 生成个性化回复                           │
└──────────────────────────────────────────┘
    ↓
┌──────────────────────────────────────────┐
│  状态更新层                                │
│  - LLM 情感分析 → 好感度更新               │
│  - 写入 WorkingMemory + EpisodicMemory    │
│  - 释放忙碌状态                            │
│  - 记录日志                               │
└──────────────────────────────────────────┘
    ↓
回复 + 更新后的状态
```

---

## 面试题精选

**1. Agent 的短期记忆和长期记忆有什么区别？各自适用于什么场景？**

短期记忆（WorkingMemory）是会话级的上下文窗口，通常用内存中的消息队列实现，容量有限（10-20 条），会话结束即清空。它的作用是保持当前对话的连贯性，解决指代消解、话题延续等问题。长期记忆（EpisodicMemory）是持久化的，通常用向量数据库存储，支持语义检索，容量无上限。它的作用是跨会话记住用户偏好和历史事件。两者互补：短期记忆提供精确有序的上下文，长期记忆提供语义相关的历史信息。

**2. 什么是"状态驱动的动态 Prompt"？它和静态 Prompt 的区别是什么？**

状态驱动的动态 Prompt 是指系统提示词根据外部状态变量（如好感度、情绪、用户角色等）实时生成，而不是固定的。区别在于：静态 Prompt 对所有用户、所有会话都一样，适合工具型 Agent；动态 Prompt 会随状态变化而改变 Agent 的行为风格，适合需要个性化和成长感的 Agent（如陪伴、客服、教育）。核心流程是：状态变量变化 → Prompt 模板填充 → 新的 System Prompt → Agent 行为变化。

**3. 如何用 LLM 实现好感度系统？为什么不用规则（关键词匹配）？**

让 LLM 分析每次对话中用户的情感倾向（友好/中立/不友好），返回一个分值变化量，然后累加到好感度分数上。不用规则的原因是自然语言的情感表达方式太多样，"你这个人还行吧"和"太棒了谢谢"用关键词很难准确覆盖，而 LLM 可以理解语义和语境。实现要点包括：限制分值变化范围（如 -3 到 +5）、钳位总分到 0-100、将连续分数映射为离散等级、每个等级对应不同的 Prompt 片段。

**4. 批量生成策略的核心思想是什么？适用于什么场景？**

核心思想是将多个 Agent 的同类请求合并为一次 LLM 调用。比如 10 个 NPC 每 5 分钟更新背景对话，逐个调用需要 10 次 API，批量生成只需要 1 次。适用场景是：多个 Agent 需要生成"背景性"内容（自言自语、状态描述），而非与用户的直接交互。直接交互仍应使用即时响应以保证个性化。混合模式（批量背景 + 即时响应）是最佳实践。

**5. 在多 Agent 系统中如何防止并发冲突？**

使用"忙碌状态 + try/finally"模式：请求到达时先检查 Agent 是否忙碌，忙碌则返回 409；否则标记为忙碌，处理完毕后在 finally 块中释放。这保证了即使处理过程中出现异常，状态也能正确释放。忙碌状态通常只存内存（重启重置），不需要持久化。

**6. 如何设计一个"越用越懂你"的 Agent？**

三层设计：（1）两层记忆——短期记忆保持对话连贯，长期记忆通过向量检索实现跨会话个性化；（2）状态量化——用好感度/信任度等数值量化关系深浅，每次交互通过 LLM 情感分析自动更新；（3）动态 Prompt——将状态变量注入系统提示词，让 Agent 的行为风格随关系变化而调整。三者配合，Agent 就能表现出"记住你、了解你、适应你"的能力。

**7. WorkingMemory 为什么用 deque 而不是普通 list？**

deque(maxlen=N) 在容量满时自动淘汰最旧的元素，时间复杂度 O(1)；而普通 list 在头部删除元素是 O(N)（需要移动后续所有元素）。对于高频写入的消息队列，deque 的性能优势明显。此外，deque 的 maxlen 参数提供了天然的容量限制，不需要额外的清理逻辑。

**8. 好感度分数应该持久化到哪里？更新策略是什么？**

好感度应该持久化到关系数据库（如 SQLite/PostgreSQL），因为它需要跨会话保留且需要查询。更新策略有两种：即时写入（每次交互后立即保存，可靠性高但 IO 开销大）和批量写入（内存中积累，定期批量保存，性能好但可能丢失少量数据）。对于好感度这种低频更新的状态，即时写入是更好的选择。

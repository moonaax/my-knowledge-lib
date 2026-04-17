# Multi-Agent 协作模式

## 1. 为什么需要多 Agent

单个 Agent 的局限性：
- 单一 Prompt 难以覆盖所有能力
- 上下文窗口有限，无法处理超大任务
- 缺乏"第二意见"，容易产生偏差

多 Agent 的优势：
- 专业分工，每个 Agent 专注一个领域
- 互相检查，提高输出质量
- 并行处理，提升效率

## 2. 协作模式

### 2.1 顺序流水线（Sequential Pipeline）

```
Agent A → Agent B → Agent C → 最终输出

适用: 任务有明确的先后顺序
示例: 需求分析 → 代码编写 → 代码审查
```

```python
from langgraph.graph import StateGraph, END

class State(TypedDict):
    task: str
    analysis: str
    code: str
    review: str

def analyst(state):
    result = analyst_llm.invoke(f"分析需求: {state['task']}")
    return {"analysis": result.content}

def coder(state):
    result = coder_llm.invoke(f"基于分析编写代码:\n{state['analysis']}")
    return {"code": result.content}

def reviewer(state):
    result = reviewer_llm.invoke(f"Review 代码:\n{state['code']}")
    return {"review": result.content}

graph = StateGraph(State)
graph.add_node("analyst", analyst)
graph.add_node("coder", coder)
graph.add_node("reviewer", reviewer)
graph.set_entry_point("analyst")
graph.add_edge("analyst", "coder")
graph.add_edge("coder", "reviewer")
graph.add_edge("reviewer", END)

app = graph.compile()
```

### 2.2 主管模式（Supervisor）

```
              Supervisor（主管）
             /     |      \
        Worker1  Worker2  Worker3

主管负责: 任务分配、进度协调、结果汇总
Worker 负责: 执行具体子任务
```

```python
from langgraph.graph import StateGraph, END

class SupervisorState(TypedDict):
    task: str
    next_agent: str
    messages: list
    final_answer: str

def supervisor(state):
    """主管决定下一步由谁执行"""
    prompt = f"""
    你是团队主管。当前任务: {state['task']}
    已有信息: {state['messages']}
    
    可用团队成员:
    - researcher: 负责信息搜索和调研
    - coder: 负责编写代码
    - reviewer: 负责审查质量
    - FINISH: 任务已完成
    
    下一步应该由谁执行? 只回答名字。
    """
    result = llm.invoke(prompt)
    return {"next_agent": result.content.strip()}

def researcher(state):
    result = research_llm.invoke(f"调研: {state['task']}\n上下文: {state['messages']}")
    return {"messages": state["messages"] + [f"[Researcher] {result.content}"]}

def coder(state):
    result = code_llm.invoke(f"编码: {state['task']}\n上下文: {state['messages']}")
    return {"messages": state["messages"] + [f"[Coder] {result.content}"]}

def reviewer(state):
    result = review_llm.invoke(f"审查: {state['task']}\n上下文: {state['messages']}")
    return {"messages": state["messages"] + [f"[Reviewer] {result.content}"]}

def route(state):
    if state["next_agent"] == "FINISH":
        return "finish"
    return state["next_agent"]

graph = StateGraph(SupervisorState)
graph.add_node("supervisor", supervisor)
graph.add_node("researcher", researcher)
graph.add_node("coder", coder)
graph.add_node("reviewer", reviewer)

graph.set_entry_point("supervisor")
graph.add_conditional_edges("supervisor", route, {
    "researcher": "researcher",
    "coder": "coder",
    "reviewer": "reviewer",
    "finish": END
})
# 每个 worker 完成后回到 supervisor
for worker in ["researcher", "coder", "reviewer"]:
    graph.add_edge(worker, "supervisor")

app = graph.compile()
```

### 2.3 辩论模式（Debate）

```
Agent A (正方) ⟷ Agent B (反方) → Judge (裁判) → 结论

适用: 需要多角度分析的决策问题
```

```python
def debate(topic: str, rounds: int = 3) -> str:
    pro_messages = []
    con_messages = []
    
    for i in range(rounds):
        # 正方发言
        pro_response = llm.invoke(f"""
        你是正方，支持以下观点: {topic}
        反方之前的论点: {con_messages}
        请给出你的论据。第 {i+1}/{rounds} 轮。
        """)
        pro_messages.append(pro_response.content)
        
        # 反方发言
        con_response = llm.invoke(f"""
        你是反方，反对以下观点: {topic}
        正方之前的论点: {pro_messages}
        请给出你的反驳。第 {i+1}/{rounds} 轮。
        """)
        con_messages.append(con_response.content)
    
    # 裁判总结
    judge_response = llm.invoke(f"""
    作为中立裁判，请综合以下辩论给出客观结论:
    议题: {topic}
    正方论点: {pro_messages}
    反方论点: {con_messages}
    请给出平衡的分析和最终建议。
    """)
    
    return judge_response.content
```

### 2.4 层级模式（Hierarchical）

```
        CEO Agent
       /         \
  Tech Lead    PM Lead
  /    \          |
Dev1  Dev2    Designer

适用: 大型复杂项目，需要多层管理
```

### 2.5 投票/共识模式

```python
def consensus_agent(question: str, n_agents: int = 3) -> str:
    """多个 Agent 独立回答，取共识"""
    answers = []
    for i in range(n_agents):
        response = llm.invoke(
            f"你是专家 {i+1}，请独立回答: {question}",
            temperature=0.8  # 增加多样性
        )
        answers.append(response.content)
    
    # 汇总共识
    consensus = llm.invoke(f"""
    以下是 {n_agents} 位专家对同一问题的独立回答:
    {chr(10).join(f'专家{i+1}: {a}' for i, a in enumerate(answers))}
    
    请找出共识点和分歧点，给出最终综合答案。
    """)
    return consensus.content
```

## 3. 通信机制

### 3.1 共享状态

```python
# 所有 Agent 共享一个状态对象
class SharedState(TypedDict):
    messages: list        # 消息历史
    artifacts: dict       # 产出物（代码、文档等）
    current_phase: str    # 当前阶段
    decisions: list       # 决策记录
```

### 3.2 消息传递

```python
# Agent 之间通过消息通信
class Message:
    sender: str       # 发送者
    receiver: str     # 接收者（或 "all"）
    content: str      # 内容
    msg_type: str     # 类型: request/response/broadcast
```

### 3.3 黑板模式

```python
# 共享黑板，Agent 自由读写
blackboard = {
    "requirements": None,    # 需求文档
    "architecture": None,    # 架构设计
    "code": {},              # 代码文件
    "test_results": None,    # 测试结果
    "issues": [],            # 发现的问题
}

# 每个 Agent 检查黑板，决定是否需要行动
def agent_loop(agent, blackboard):
    while True:
        if agent.should_act(blackboard):
            result = agent.act(blackboard)
            blackboard.update(result)
        if is_complete(blackboard):
            break
```

## 4. 设计原则

### 4.1 Agent 数量

```
经验法则:
- 2-3 个 Agent: 大多数场景足够
- 4-6 个 Agent: 复杂项目
- 7+ 个 Agent: 谨慎使用，协调成本高

Agent 越多:
✅ 专业分工更细
❌ 协调成本越高
❌ Token 消耗越大
❌ 调试越困难
```

### 4.2 角色定义清单

```
每个 Agent 需要明确:
□ 角色名称和职责
□ 能力边界（能做什么，不能做什么）
□ 可用工具列表
□ 输入/输出格式
□ 与其他 Agent 的交互协议
□ 失败时的处理策略
```

### 4.3 防止常见问题

| 问题 | 解决方案 |
|------|---------|
| 无限循环对话 | 设置最大轮次 |
| Agent 角色混乱 | 强化 System Prompt |
| 信息丢失 | 使用共享状态 |
| 成本失控 | 监控 Token，设预算 |
| 结果不一致 | 加入仲裁/审查 Agent |

---

## 面试题精选

### Q1: 多 Agent 相比单 Agent 有什么优势和劣势？
**答：** 优势：专业分工更细、互相检查提高质量、可并行处理。劣势：协调成本高、Token 消耗大、调试困难、Agent 越多越难管理。经验上 2-3 个 Agent 覆盖大多数场景，超过 7 个需谨慎。

### Q2: Supervisor 模式和顺序流水线模式的区别是什么？
**答：** 流水线是固定顺序 A→B→C，适合有明确先后关系的任务。Supervisor 模式由主管 Agent 动态决定下一步由谁执行，更灵活，适合需要根据中间结果动态调整分工的复杂任务。

### Q3: 多 Agent 之间有哪些通信机制？各自适用什么场景？
**答：** 共享状态（所有 Agent 读写同一个 State 对象，简单直接）、消息传递（Agent 之间发送结构化消息，适合松耦合）、黑板模式（共享黑板自由读写，适合 Agent 自主决定何时行动）。LangGraph 主要用共享状态模式。

### Q4: 如何防止多 Agent 对话陷入无限循环？
**答：** 设置 max_round 限制最大对话轮次、在 System Prompt 中明确每个 Agent 的发言条件（"只在有代码需要审查时发言"）、加入终止条件判断（Supervisor 可以选择 FINISH）、设置超时机制。

### Q5: 辩论模式（Debate）适合什么场景？怎么实现？
**答：** 适合需要多角度分析的决策问题（如技术选型、方案评估）。实现方式是让正反两方 Agent 交替发言各自论证，经过多轮辩论后由裁判 Agent 综合双方观点给出客观结论。

### Q6: 设计多 Agent 系统时，每个 Agent 需要明确哪些要素？
**答：** 角色名称和职责、能力边界（能做什么不能做什么）、可用工具列表、输入输出格式、与其他 Agent 的交互协议、失败时的处理策略。职责越清晰，协作效果越好。

# ReAct 与推理策略

## 1. 推理策略全景

````
┌─────────────────────────────────────────────┐
│              Agent 推理策略                   │
├──────────────┬──────────────────────────────┤
│ 基础推理      │ CoT (思维链)                  │
│              │ Zero-Shot CoT                │
│              │ Few-Shot CoT                 │
├──────────────┼──────────────────────────────┤
│ 行动推理      │ ReAct (推理+行动)             │
│              │ MRKL (模块化推理)             │
├──────────────┼──────────────────────────────┤
│ 高级推理      │ ToT (思维树)                  │
│              │ GoT (思维图)                  │
│              │ Reflexion (反思)              │
└──────────────┴──────────────────────────────┘
````
## 2. Chain of Thought (CoT) — 思维链

### 2.1 核心思想

让 LLM 展示中间推理步骤，而不是直接给出答案。

````
❌ 直接回答:
Q: 一个商店有 23 个苹果，卖了 15 个，又进了 8 个，现在有多少?
A: 16

✅ CoT 推理:
Q: 一个商店有 23 个苹果，卖了 15 个，又进了 8 个，现在有多少?
A: 让我一步步计算:
   1. 初始: 23 个苹果
   2. 卖了 15 个: 23 - 15 = 8 个
   3. 又进了 8 个: 8 + 8 = 16 个
   所以现在有 16 个苹果。
````
### 2.2 Zero-Shot CoT

只需在 Prompt 末尾加一句话：

````python
prompt = f"""
{question}

Let's think step by step.
"""
# 或中文: "让我们一步步思考。"
````
### 2.3 Few-Shot CoT

提供带推理过程的示例：

````python
prompt = """
Q: 小明有 5 本书，借给小红 2 本，又买了 3 本，现在有几本?
A: 初始 5 本，借出 2 本剩 3 本，买了 3 本变成 6 本。答案: 6 本。

Q: 一个班有 30 人，转走 5 人，转来 8 人，现在多少人?
A: 初始 30 人，转走 5 人剩 25 人，转来 8 人变成 33 人。答案: 33 人。

Q: {user_question}
A:
"""
````
## 3. ReAct — 推理与行动

### 3.1 核心思想

ReAct = Reasoning（推理）+ Acting（行动），让 Agent 交替进行思考和工具调用。

````
循环:
  Thought  → 我需要做什么（推理）
  Action   → 调用什么工具（行动）
  Observation → 工具返回了什么（观察）
  ... 重复直到得出答案 ...
  Final Answer → 最终回答
````
### 3.2 ReAct 示例

````
问题: LangChain 的最新版本是什么?它有哪些新特性?

Thought: 我需要搜索 LangChain 的最新版本信息
Action: search("LangChain latest version 2024")
Observation: LangChain v0.3 发布于 2024 年...

Thought: 我找到了版本信息，现在需要了解新特性
Action: search("LangChain v0.3 new features changelog")
Observation: 主要更新包括: 1) 移除旧的依赖 2) LCEL 改进...

Thought: 我现在有足够的信息来回答了
Final Answer: LangChain 最新版本是 v0.3，主要新特性包括...
````
### 3.3 实现 ReAct Agent

````python
from langchain.agents import create_react_agent, AgentExecutor
from langchain_core.prompts import PromptTemplate
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """搜索互联网获取最新信息"""
    return f"搜索结果: {query}"

@tool
def calculator(expression: str) -> str:
    """计算数学表达式"""
    return str(eval(expression))

# ReAct Prompt 模板
react_prompt = PromptTemplate.from_template("""
Answer the following questions as best you can. You have access to the following tools:

{tools}

Use the following format:

Question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation can repeat N times)
Thought: I now know the final answer
Final Answer: the final answer to the original input question

Begin!

Question: {input}
Thought: {agent_scratchpad}
""")

agent = create_react_agent(llm, [search, calculator], react_prompt)
executor = AgentExecutor(agent=agent, tools=[search, calculator], verbose=True)

result = executor.invoke({"input": "计算 2024 年 Q1 的同比增长率，去年 Q1 收入 100 万，今年 Q1 收入 135 万"})
````
### 3.4 ReAct 的优缺点

| 优点 | 缺点 |
|------|------|
| 推理过程透明可解释 | 每步都需要 LLM 调用，延迟高 |
| 能动态调整策略 | 可能陷入循环 |
| 错误可追溯 | Token 消耗大 |
| 适合复杂多步任务 | 依赖 LLM 的推理能力 |

## 4. Tree of Thought (ToT) — 思维树

### 4.1 核心思想

不同于 CoT 的线性推理，ToT 探索多条推理路径，选择最优的：

````
                    问题
                   / | \
                思路1 思路2 思路3
               / \    |    / \
            1a  1b   2a  3a  3b
            ↓        ↓       ↓
          评估      评估    评估
            ↓
         最优路径 → 答案
````
### 4.2 简化实现

````python
def tree_of_thought(question: str, n_branches: int = 3) -> str:
    # 1. 生成多个思路
    branches_prompt = f"""
    针对以下问题，请提出 {n_branches} 个不同的解决思路:
    {question}
    """
    branches = llm.invoke(branches_prompt)
    
    # 2. 评估每个思路
    eval_prompt = f"""
    评估以下解决思路的可行性（1-10分）:
    问题: {question}
    思路: {{branch}}
    请给出评分和理由。
    """
    scores = []
    for branch in branches:
        score = llm.invoke(eval_prompt.format(branch=branch))
        scores.append(score)
    
    # 3. 选择最优思路深入
    best_branch = select_best(branches, scores)
    
    # 4. 基于最优思路生成答案
    return llm.invoke(f"基于以下思路回答问题:\n思路: {best_branch}\n问题: {question}")
````
## 5. Reflexion — 反思机制

### 5.1 核心思想

Agent 执行后进行自我反思，从失败中学习：

````
执行 → 评估结果 → 反思失败原因 → 改进策略 → 重新执行
````
### 5.2 实现

````python
def reflexion_agent(task: str, max_retries: int = 3) -> str:
    reflections = []
    
    for attempt in range(max_retries):
        # 1. 执行任务（带上之前的反思）
        context = "\n".join(reflections) if reflections else "无"
        result = execute_task(task, previous_reflections=context)
        
        # 2. 评估结果
        evaluation = evaluate_result(task, result)
        
        if evaluation["success"]:
            return result
        
        # 3. 反思
        reflection = llm.invoke(f"""
        任务: {task}
        我的输出: {result}
        评估结果: {evaluation['feedback']}
        
        请反思:
        1. 哪里做错了?
        2. 为什么会出错?
        3. 下次应该怎么改进?
        """)
        reflections.append(f"第{attempt+1}次反思: {reflection}")
    
    return result  # 返回最后一次结果
````
## 6. 策略选型指南

| 策略 | 适用场景 | 复杂度 | 成本 |
|------|---------|--------|------|
| Zero-Shot CoT | 简单推理 | 低 | 低 |
| Few-Shot CoT | 特定领域推理 | 低 | 低 |
| ReAct | 需要工具调用的任务 | 中 | 中 |
| ToT | 需要探索多种方案 | 高 | 高 |
| Reflexion | 需要迭代改进 | 高 | 高 |
| Plan-and-Execute | 复杂多步骤任务 | 中 | 中 |

````
简单问答          → CoT
需要调用工具      → ReAct
复杂规划任务      → Plan-and-Execute
需要创造性方案    → ToT
需要高质量输出    → Reflexion
生产环境          → ReAct + Reflexion
````
---

## 面试题精选

### Q1: ReAct 模式的核心思想是什么？和纯 CoT 有什么区别？
**答：** ReAct 交替进行推理（Thought）和行动（Action），能在推理过程中调用外部工具获取信息。纯 CoT 只有推理没有行动，无法与外部世界交互，所有知识只能来自模型自身。

### Q2: ReAct 模式有什么缺点？如何改进？
**答：** 主要缺点是每步都需要 LLM 调用导致延迟高、Token 消耗大，且可能陷入循环。改进方法包括：设置最大迭代次数、结合 Plan-and-Execute 做顶层规划减少冗余步骤、加入 Reflexion 反思机制从失败中学习。

### Q3: Tree of Thought（ToT）和 Chain of Thought（CoT）的区别是什么？
**答：** CoT 是线性推理（一条路走到底），ToT 是树形推理（同时探索多条路径并评估选择最优）。ToT 适合需要创造性方案或有多种可能解法的问题，但成本更高（需要多次 LLM 调用评估各分支）。

### Q4: Reflexion 反思机制是怎么工作的？
**答：** Agent 执行任务后进行自我评估，如果结果不满意则反思失败原因、总结经验教训，带着反思结果重新执行。这模拟了人类"从错误中学习"的过程，能逐步提高输出质量。

### Q5: 生产环境中推理策略怎么选型？
**答：** 简单问答用 CoT，需要工具调用用 ReAct，复杂多步骤任务用 Plan-and-Execute，需要高质量输出用 Reflexion。生产环境推荐 ReAct + Reflexion 组合，兼顾工具调用能力和输出质量。

### Q6: Zero-Shot CoT 只需要加一句"Let's think step by step"就能提升效果，原理是什么？
**答：** 这句话激活了模型在预训练阶段学到的逐步推理模式，迫使模型生成中间步骤而非直接跳到答案。中间步骤的生成为后续推理提供了更丰富的上下文，从而减少推理跳跃带来的错误。

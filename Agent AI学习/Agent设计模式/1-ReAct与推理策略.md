# ReAct 与推理策略

> **本文定位：** Agent 的"大脑"是怎么思考的？本文从最基础的 CoT（思维链）讲到 ReAct（推理+行动）、ToT（思维树）、Reflexion（反思），这些策略决定了 Agent 的推理质量和行为模式。


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

> **CoT 是所有推理策略的基础：** 核心思想很简单——让 LLM 展示中间推理步骤，而不是直接跳到答案。就像考试时要求"写出解题过程"，中间步骤为后续推理提供了更丰富的上下文，显著减少推理错误。
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

> **最简单的推理增强：** 只需在 prompt 末尾加一句 "Let's think step by step"，就能显著提升 LLM 的推理准确率。原理是这句话激活了模型预训练时学到的逐步推理模式。
### 2.2 Zero-Shot CoT

只需在 Prompt 末尾加一句话：

````python
prompt = f"""
{question}

Let's think step by step.
"""
# 或中文: "让我们一步步思考。"
````

> **Few-Shot vs Zero-Shot：** Zero-Shot 只加一句话，简单但效果有限。Few-Shot 提供完整的推理示例，LLM 能学会特定领域的推理模式，效果更好但需要手动准备示例。
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

> **ReAct 是 Agent 的核心推理模式：** CoT 只能思考不能行动，ReAct 让 Agent 能在推理过程中调用外部工具获取信息。这是从"纯推理"到"智能体"的关键跨越。
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


> **思路：** 用 LangChain 实现 ReAct Agent。关键是 ReAct Prompt 模板——它定义了 Thought/Action/Observation 的循环格式，LLM 按这个格式输出，框架解析后执行对应的工具调用。
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

> **关键代码解读：**
> - ReAct Prompt 模板定义了严格的格式：Thought → Action → Action Input → Observation 循环
> - `{tools}` 和 `{tool_names}` 会被自动替换为可用工具的描述和名称列表
> - `{agent_scratchpad}` 存放之前的思考和工具调用记录，让 Agent 能"记住"已经做过什么
> - `verbose=True` 打印完整的推理过程，方便调试
````
### 3.4 ReAct 的优缺点

| 优点 | 缺点 |
|------|------|
| 推理过程透明可解释 | 每步都需要 LLM 调用，延迟高 |
| 能动态调整策略 | 可能陷入循环 |
| 错误可追溯 | Token 消耗大 |
| 适合复杂多步任务 | 依赖 LLM 的推理能力 |


### 3.5 从零实现 ReAct 的关键差异

上面用 LangChain 的 `AgentExecutor` 封装了 ReAct，省去了大量工程细节。但从零实现时，有几个关键点值得注意：

#### 正则解析 LLM 输出

框架封装版自动解析 LLM 输出，从零实现则需要自己用正则表达式提取 `Thought` 和 `Action`：

````python
import re

def parse_react_output(text: str):
    """从 LLM 原始输出中提取 Thought 和 Action。"""
    thought_match = re.search(r"Thought:\s*(.*?)(?=\nAction:|$)", text, re.DOTALL)
    action_match = re.search(r"Action:\s*(.*?)$", text, re.DOTALL)
    thought = thought_match.group(1).strip() if thought_match else None
    action = action_match.group(1).strip() if action_match else None
    return thought, action

def parse_action(action_text: str):
    """从 Action 字符串中提取工具名和输入，如 Search[query]。"""
    match = re.match(r"(\w+)\[(.*)\]", action_text, re.DOTALL)
    if match:
        return match.group(1), match.group(2)
    return None, None
````

> 🔑 **正则解析的脆弱性：** 这是从零实现中最容易出问题的地方。LLM 可能输出多余换行、缺少冒号、格式偏差等，导致正则匹配失败。框架（如 LangChain）内部有大量容错逻辑和重试机制来处理这些情况，自己实现时需要格外注意边界情况。

#### 调试技巧

从零实现时，**打印每轮中间状态**是最重要的调试手段：

````python
# 在 while 循环的每一轮中打印
print(f"--- 第 {step} 步 ---")
print(f"完整提示词:\n{prompt}")          # 检查输入是否正确
print(f"LLM 原始输出:\n{response_text}")  # 检查输出格式
print(f"解析结果: thought={thought}, action={action}")  # 检查解析
print(f"工具返回: {observation}")         # 检查工具执行
````

当输出解析失败时，务必将 LLM 返回的**原始文本**打印出来——这能帮助判断是 LLM 没遵循格式，还是解析逻辑有误。

#### 与框架封装版的本质区别

| 维度 | 框架封装（LangChain） | 从零实现 |
|------|----------------------|---------|
| 输出解析 | 自动处理格式偏差、容错重试 | 需要自己写正则 + 处理异常 |
| 工具调用 | `@tool` 装饰器 + 自动参数注入 | 手动注册 + 手动分发 |
| 循环控制 | `AgentExecutor` 内置最大步数 + 错误处理 | 需要自己实现 `while` 循环 + 安全阀 |
| 记忆管理 | `agent_scratchpad` 自动拼接 | 需要自己维护 `history` 列表 |
| 学习价值 | 关注"怎么用" | 理解"怎么运转" |

> 🔑 **从零实现的核心价值：** 不是为了替代框架，而是理解框架在背后做了什么。当你知道正则解析的脆弱性、循环终止的边界条件、历史记录的拼接方式后，使用框架时才能在出问题时快速定位原因，而不是黑盒调用。

> **从线性到树形：** CoT 和 ReAct 都是线性推理，ToT 引入了"分支探索"的概念，更接近人类面对复杂问题时的思考方式——先想几个方向，评估哪个最靠谱，再深入。
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


> **思路：** 不同于 CoT 的线性推理，ToT 同时探索多条推理路径，评估每条路径的可行性，选择最优的深入展开。适合有多种可能解法的问题。
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

> **关键代码解读：**
> - 先生成 N 个不同思路（`n_branches`），增加 temperature 提高多样性
> - 对每个思路独立评估打分
> - 选择最优思路深入展开生成最终答案
>
> **成本注意：** ToT 需要 N+1 次 LLM 调用（N 次评估 + 1 次生成），成本是 CoT 的数倍。只在需要创造性方案时使用。
````

> **Reflexion 和 ReAct 的区别：** ReAct 是"边想边做"的即时推理，Reflexion 是"做完后回顾"的事后反思。两者可以结合——用 ReAct 执行任务，用 Reflexion 从失败中学习改进。
## 5. Reflexion — 反思机制

### 5.1 核心思想

Agent 执行后进行自我反思，从失败中学习：

````
执行 → 评估结果 → 反思失败原因 → 改进策略 → 重新执行
````
### 5.2 实现


> **思路：** Agent 执行后自我反思，从失败中学习。每次失败后总结经验教训，带着反思结果重新执行，逐步提高输出质量。
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

> **关键代码解读：**
> - `reflections` 列表累积每次的反思结果，下次执行时作为上下文传入
> - 最多重试 `max_retries` 次，每次都带着之前的反思经验
> - `evaluate_result()` 是关键——需要一个可靠的评估机制判断结果是否合格
>
> **实际应用：** 代码生成场景特别适合 Reflexion——生成代码 → 运行测试 → 测试失败 → 反思错误原因 → 重新生成。
````

### 5.3 Reflection 完整实现 — Memory 模块与迭代优化

上面的 `reflexion_agent` 只是一个简化版本。真正健壮的 Reflection 实现需要一个**记忆模块**来管理迭代历史，以及精心设计的**多角色提示词**来分别驱动执行、反思和优化。

#### 5.3.1 Memory 类设计

Reflection 的核心是迭代，而迭代的前提是记住每次尝试和反馈。Memory 类承担这个职责：

````python
from typing import List, Dict, Any, Optional

class Memory:
    """短期记忆模块，存储执行与反思的完整轨迹。"""

    def __init__(self):
        self.records: List[Dict[str, Any]] = []

    def add_record(self, record_type: str, content: str):
        """添加记录。record_type: 'execution' 或 'reflection'"""
        self.records.append({"type": record_type, "content": content})

    def get_trajectory(self) -> str:
        """将所有记录格式化为连贯文本，用于构建提示词上下文。"""
        parts = []
        for r in self.records:
            if r['type'] == 'execution':
                parts.append(f"--- 上一轮尝试 ---\n{r['content']}")
            elif r['type'] == 'reflection':
                parts.append(f"--- 评审反馈 ---\n{r['content']}")
        return "\n\n".join(parts)

    def get_last_execution(self) -> Optional[str]:
        """获取最近一次的执行结果。"""
        for r in reversed(self.records):
            if r['type'] == 'execution':
                return r['content']
        return None
````

> 🔑 **设计要点：** `get_trajectory()` 将完整历史序列化为文本，直接插入提示词；`get_last_execution()` 则只取最新版本供反思和优化使用。两者的分工避免了上下文膨胀——反思时不需要翻阅所有历史，只需要最新代码和最新反馈。

#### 5.3.2 迭代代码优化案例

一个经典的 Reflection 案例是**代码生成与迭代优化**。任务："编写一个 Python 函数，找出 1 到 n 之间所有的素数。"

典型的迭代过程：

````text
初始执行 → 生成试除法代码 O(n * sqrt(n))
第 1 轮反思 → "时间复杂度过高，建议使用埃拉托斯特尼筛法"
第 1 轮优化 → 生成筛法代码 O(n log log n)
第 2 轮反思 → "算法已足够高效，无需改进" → 终止
````

> 🔑 **关键洞察：** Reflection 的价值不仅在于修复错误，更在于**驱动方案在质量和效率上实现阶梯式提升**。从试除法到筛法的跨越，不是简单的 bug 修复，而是算法层面的质变。这正是"评审员"角色设定为"极其严格的算法工程师"的效果——它不会满足于"功能正确"，而是追求"算法最优"。

整个流程由三套提示词协同驱动：
- **执行提示词**：要求生成代码，角色为"资深 Python 程序员"
- **反思提示词**：要求批判性分析，角色为"严格的代码评审专家"，重点关注算法效率
- **优化提示词**：要求根据反馈修改，同时保留原始任务约束

> 🔑 **提示词角色的影响：** 反思提示词中"极其严格"和"专注于算法效率"的措辞直接决定了优化方向。如果改为"注重代码可读性的维护者"，优化方向就会变为命名规范、注释完善等，而非算法改进。角色设定是 Reflection 最容易被忽视但影响最大的设计决策。

### 5.4 Reflection 的成本收益分析

Reflection 是典型的**以成本换质量**的策略。是否使用，取决于任务的性质。

#### 成本

| 成本项 | 说明 |
|--------|------|
| API 调用倍增 | 每轮迭代至少 2 次 LLM 调用（反思 + 优化），多轮迭代成本成倍增长 |
| 延迟显著增加 | 串行执行，每轮优化必须等上轮反思完成，不适合实时场景 |
| 提示词工程复杂 | 执行/反思/优化三套提示词需分别设计和调试 |

#### 收益

| 收益项 | 说明 |
|--------|------|
| 方案质量跃迁 | 从"合格"到"优秀"的质变，而非线性改善 |
| 鲁棒性增强 | 内部纠错回路可发现逻辑漏洞、边界情况、事实错误 |
| 可组合性 | 可与 ReAct 结合——用 ReAct 执行，用 Reflection 事后优化 |

#### 何时值得反思

````text
值得反思：
  ✅ 代码生成（有明确的正确性/效率标准）
  ✅ 关键业务报告（质量要求极高）
  ✅ 复杂逻辑推演（容易出错且后果严重）

不值得反思：
  ❌ 简单问答（一次 CoT 就够了）
  ❌ 实时对话（延迟不可接受）
  ❌ 大致正确即可的场景（成本不划算）
````

> 🔑 **生产环境建议：** Reflexion 不是万能药。一个务实的做法是**设置 1-2 轮上限**，配合一个可靠的终止条件（如反思中出现"无需改进"或置信度超过阈值），在质量和成本之间取得平衡。

> **如何选择：** 简单问答用 CoT（零成本提升），需要调用工具用 ReAct（最常用），复杂规划用 Plan-and-Execute，需要创造性方案用 ToT，需要高质量输出用 Reflexion。生产环境推荐 ReAct + Reflexion 组合。
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

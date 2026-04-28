# 从零实现 Agent 范式

> 不依赖任何框架，从零手写 ReAct、Plan-and-Solve、Reflection 三大经典 Agent 范式，理解底层设计机制，从框架"使用者"转变为智能体"创造者"。

相关主题：[[1-ReAct与推理策略]] | [[2-Plan-and-Execute模式]]

---

市面上已有 LangChain、LlamaIndex 等众多优秀框架，为何还要"重复造轮子"？原因有三：第一，高度抽象的框架不利于理解背后的设计机制；第二，亲手处理模型输出解析、工具调用失败重试、防止死循环等工程挑战，是培养系统设计能力的最直接方式；第三，掌握设计原理后，你才能真正从框架的"使用者"转变为智能体应用的"创造者"。

本文聚焦三种经典范式，**全部不依赖框架，纯 Python 手写**：

| 范式 | 核心思想 | 一句话概括 |
|------|---------|-----------|
| ReAct | 推理 + 行动交替 | 边想边做，动态调整 |
| Plan-and-Solve | 先规划后执行 | 三思而后行 |
| Reflection | 执行-反思-优化 | 自我批判，迭代提升 |

## 一、环境准备与基础组件

### 1.1 LLM 调用封装

所有范式共用同一个 LLM 客户端。核心设计是将模型服务信息统一配置在环境变量中，通过 `.env` 文件管理，客户端类封装流式调用逻辑：

```python
import os
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

class HelloAgentsLLM:
    def __init__(self, model=None, apiKey=None, baseUrl=None):
        self.model = model or os.getenv("LLM_MODEL_ID")
        apiKey = apiKey or os.getenv("LLM_API_KEY")
        baseUrl = baseUrl or os.getenv("LLM_BASE_URL")
        self.client = OpenAI(api_key=apiKey, base_url=baseUrl)

    def think(self, messages, temperature=0):
        """调用 LLM 并返回完整响应（流式输出）"""
        response = self.client.chat.completions.create(
            model=self.model, messages=messages,
            temperature=temperature, stream=True,
        )
        collected = []
        for chunk in response:
            content = chunk.choices[0].delta.content or ""
            print(content, end="", flush=True)
            collected.append(content)
        return "".join(collected)
```

> 🔑 关键设计：`think()` 方法默认使用流式输出（`stream=True`），在调试时能实时观察 LLM 的生成过程；`temperature=0` 保证输出确定性，便于调试。

### 1.2 工具定义模式

一个良好定义的工具需要三个核心要素：

1. **名称（Name）**：简洁唯一的标识符，供 Agent 在 Action 中调用
2. **描述（Description）**：自然语言说明用途——这是最关键的要素，LLM 依赖描述判断何时使用哪个工具
3. **执行逻辑（Execution Logic）**：真正执行任务的函数

工具执行器（ToolExecutor）负责统一注册和调度所有工具：

```python
class ToolExecutor:
    def __init__(self):
        self.tools = {}

    def registerTool(self, name, description, func):
        self.tools[name] = {"description": description, "func": func}

    def getTool(self, name):
        return self.tools.get(name, {}).get("func")

    def getAvailableTools(self):
        return "\n".join([
            f"- {name}: {info['description']}"
            for name, info in self.tools.items()
        ])
```

> 🔑 当工具数量增长到 50+ 时，简单的字符串拼接描述会占用大量上下文。实际工程中需要引入工具检索机制（如基于语义相似度的工具选择），而非将所有工具描述一次性塞入提示词。

## 二、ReAct 范式从零实现

### 2.1 核心思想

ReAct（Reason + Act）由 Shunyu Yao 于 2022 年提出，核心思想是将**推理（Reasoning）**与**行动（Acting）**显式结合，形成"思考-行动-观察"循环。

在 ReAct 之前，主流方法分两类：纯思考型（如 CoT，能推理但无法与外部交互，容易产生幻觉）和纯行动型（直接输出动作，缺乏规划和纠错能力）。ReAct 的巧妙之处在于认识到**思考与行动是相辅相成的**——思考指导行动，行动的结果又反过来修正思考。

形式化表示：在每个时间步 t，Agent 策略（LLM）根据初始问题 q 和历史轨迹生成思考和行动：

```
(th_t, a_t) = pi(q, (a_1, o_1), ..., (a_{t-1}, o_{t-1}))
o_t = T(a_t)   # 工具执行行动，返回观察
```

### 2.2 提示词设计

提示词是 ReAct 的基石，它强制 LLM 的输出具有结构性：

```python
REACT_PROMPT = """
你是一个有能力调用外部工具的智能助手。

可用工具:
{tools}

请严格按照以下格式回应:

Thought: 你的思考过程，分析问题、拆解任务、规划下一步。
Action: 你决定采取的行动:
- `ToolName[input]`: 调用一个可用工具
- `Finish[最终答案]`: 当你已获得足够信息时

Question: {question}
History: {history}
"""
```

> 🔑 格式规约（Thought/Action）是最关键的部分。它强制 LLM 输出结构化内容，使代码能通过正则表达式精确解析意图。提示词中任何微小变动都可能影响 LLM 行为，这是 ReAct 的脆弱性所在。

### 2.3 输出解析器

LLM 返回的是纯文本，需要用正则表达式提取 Thought 和 Action：

```python
import re

def _parse_output(text):
    """从 LLM 响应中分离 Thought 和 Action"""
    thought_match = re.search(r"Thought:\s*(.*?)(?=\nAction:|$)", text, re.DOTALL)
    action_match = re.search(r"Action:\s*(.*?)$", text, re.DOTALL)
    thought = thought_match.group(1).strip() if thought_match else None
    action = action_match.group(1).strip() if action_match else None
    return thought, action

def _parse_action(action_text):
    """从 Action 字符串中提取工具名和输入，如 Search[query]"""
    match = re.match(r"(\w+)\[(.*)\]", action_text, re.DOTALL)
    if match:
        return match.group(1), match.group(2)
    return None, None
```

### 2.4 核心循环实现

ReActAgent 的核心是一个 while 循环——格式化提示词、调用 LLM、解析输出、执行动作、整合结果：

```python
class ReActAgent:
    def __init__(self, llm_client, tool_executor, max_steps=5):
        self.llm_client = llm_client
        self.tool_executor = tool_executor
        self.max_steps = max_steps
        self.history = []

    def run(self, question):
        self.history = []
        for step in range(1, self.max_steps + 1):
            # 1. 格式化提示词（注入工具描述、问题、历史）
            prompt = REACT_PROMPT.format(
                tools=self.tool_executor.getAvailableTools(),
                question=question,
                history="\n".join(self.history)
            )
            # 2. 调用 LLM
            response = self.llm_client.think([{"role": "user", "content": prompt}])
            # 3. 解析输出
            thought, action = _parse_output(response)
            # 4. 检查是否结束
            if action and action.startswith("Finish"):
                return re.match(r"Finish\[(.*)\]", action).group(1)
            # 5. 执行工具
            tool_name, tool_input = _parse_action(action)
            observation = self.tool_executor.getTool(tool_name)(tool_input)
            # 6. 将 Action 和 Observation 追加到历史
            self.history.append(f"Action: {action}")
            self.history.append(f"Observation: {observation}")
        return None  # 达到最大步数
```

> 🔑 `max_steps` 是重要的安全阀，防止 Agent 陷入无限循环。`self.history` 不断累积形成上下文，使 Agent 能"看到"之前所有行动的结果。

### 2.5 调试技巧

当 ReAct Agent 行为异常时，排查顺序：

1. **打印完整提示词**：在调用 LLM 前，输出格式化后的完整 prompt，追溯决策源头
2. **检查原始输出**：正则匹配失败时，打印 LLM 返回的原始文本，判断是格式问题还是解析问题
3. **验证工具输入输出**：检查 `tool_input` 格式是否正确，`observation` 是否可被 LLM 理解
4. **添加 Few-shot 示例**：在提示词中加入完整的 Thought-Action-Observation 成功案例
5. **调整 temperature**：设为 0 保证确定性，排除随机性干扰

### 2.6 ReAct 的特点与局限

**优势：**
- **高可解释性**：Thought 链清晰展示每一步推理过程
- **动态纠错**：根据 Observation 动态调整后续行动，搜索结果不理想可修正搜索词重试
- **工具协同**：LLM 负责推理规划，工具负责执行具体任务，突破单一 LLM 的知识和计算局限

**局限：**
- **强依赖 LLM 能力**：推理能力或格式遵循能力不足时，容易中断
- **串行效率低**：每步都需调用 LLM，多步任务耗时和成本较高
- **提示词脆弱**：模板微小变动可能影响行为
- **缺乏全局规划**：步进式决策可能陷入局部最优或原地打转

## 三、Plan-and-Solve 范式从零实现

### 3.1 与 Plan-and-Execute 的区别

Plan-and-Solve 由 Lei Wang 于 2023 年提出。如果说 ReAct 像侦探根据线索一步步推理，那么 Plan-and-Solve 更像建筑师——动工前必须先绘制完整蓝图，然后严格按图施工。

与 Plan-and-Execute 的关键区别：Plan-and-Solve 的规划和执行由**同一个 LLM** 完成，通过不同的提示词切换角色；Plan-and-Execute 通常使用**独立的 Planner 和 Executor 模块**，甚至可能引入 Replanner 进行动态重规划。

形式化表示：

```
P = pi_plan(q)                           # 规划：生成步骤列表
s_i = pi_solve(q, P, (s_1, ..., s_{i-1}))  # 逐步执行
```

### 3.2 规划阶段：结构化计划生成

规划器的目标是将复杂问题分解为结构化的步骤列表。关键设计是强制 LLM 输出 Python 列表格式，用 `ast.literal_eval` 安全解析：

```python
import ast

PLANNER_PROMPT = """
你是一个顶级的AI规划专家。将用户问题分解为多个简单步骤的行动计划。
每个步骤是独立的、可执行的子任务，按逻辑顺序排列。

问题: {question}

请严格按以下格式输出（```python 前后缀是必要的）:
```python
["步骤1", "步骤2", "步骤3", ...]
```
"""

class Planner:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def plan(self, question):
        prompt = PLANNER_PROMPT.format(question=question)
        response = self.llm_client.think([{"role": "user", "content": prompt}]) or ""
        try:
            plan_str = response.split("```python")[1].split("```")[0].strip()
            plan = ast.literal_eval(plan_str)
            return plan if isinstance(plan, list) else []
        except (ValueError, SyntaxError, IndexError):
            return []
```

> 🔑 使用 `ast.literal_eval` 而非 `eval` 来解析列表字符串，这是安全性最佳实践。`literal_eval` 只允许字面量表达式，不会执行任意代码。

### 3.3 执行阶段：状态管理与逐步执行

执行器的核心职责是**维护状态**——每一步的结果作为下一步的输入上下文：

```python
EXECUTOR_PROMPT = """
你是AI执行专家。严格按照计划逐步解决问题。

原始问题: {question}
完整计划: {plan}
历史步骤与结果: {history}

当前步骤: {current_step}

请仅输出当前步骤的答案:
"""

class Executor:
    def __init__(self, llm_client):
        self.llm_client = llm_client

    def execute(self, question, plan):
        history = ""
        for i, step in enumerate(plan):
            prompt = EXECUTOR_PROMPT.format(
                question=question, plan=plan,
                history=history or "无", current_step=step
            )
            result = self.llm_client.think([{"role": "user", "content": prompt}]) or ""
            history += f"步骤 {i+1}: {step}\n结果: {result}\n\n"
        return result  # 最后一步的结果即最终答案
```

### 3.4 组装 Agent

PlanAndSolveAgent 作为协调者，串联 Planner 和 Executor：

```python
class PlanAndSolveAgent:
    def __init__(self, llm_client):
        self.planner = Planner(llm_client)
        self.executor = Executor(llm_client)

    def run(self, question):
        plan = self.planner.plan(question)
        if not plan:
            return "无法生成有效计划"
        return self.executor.execute(question, plan)
```

> 🔑 Agent 本身不包含复杂逻辑，而是作为协调者（Orchestrator），体现"组合优于继承"的设计原则。

### 3.5 适用场景

Plan-and-Solve 特别适合结构性强、可清晰分解的任务：
- 多步数学应用题：先列计算步骤，再逐一求解
- 报告撰写：先规划结构（引言、数据、总结），再填充内容
- 代码生成：先构思函数和模块结构，再逐一实现

## 四、Reflection 范式从零实现

### 4.1 核心思想

ReAct 和 Plan-and-Solve 一旦完成任务，流程即告结束。但初始答案可能存在谬误或有待改进。Reflection 机制的核心是为 Agent 引入**事后的自我校正循环**——像人类一样审视自己的工作，发现不足，迭代优化。

这一思想源于 2023 年 Shinn 等人提出的 Reflexion 框架，核心流程为三步循环：**执行 -> 反思 -> 优化**。

形式化表示：

```
F_i = pi_reflect(Task, O_i)           # 反思：对当前输出生成反馈
O_{i+1} = pi_refine(Task, O_i, F_i)   # 优化：根据反馈修正输出
```

### 4.2 记忆模块设计

Reflection 的迭代前提是能记住之前的尝试和反馈，因此需要一个"短期记忆"模块：

```python
class Memory:
    def __init__(self):
        self.records = []

    def add_record(self, record_type, content):
        """record_type: 'execution' 或 'reflection'"""
        self.records.append({"type": record_type, "content": content})

    def get_trajectory(self):
        """将所有记忆格式化为文本，用于构建提示词"""
        parts = []
        for r in self.records:
            if r['type'] == 'execution':
                parts.append(f"--- 上一轮尝试 ---\n{r['content']}")
            elif r['type'] == 'reflection':
                parts.append(f"--- 评审员反馈 ---\n{r['content']}")
        return "\n\n".join(parts)

    def get_last_execution(self):
        """获取最近一次的执行结果"""
        for r in reversed(self.records):
            if r['type'] == 'execution':
                return r['content']
        return None
```

### 4.3 提示词设计：三个角色

Reflection 需要三个不同角色的提示词协同工作：

**初始执行提示词**——让 LLM 扮演程序员完成任务：

```python
INITIAL_PROMPT = """
你是一位资深Python程序员。请编写一个Python函数。
要求: {task}
请直接输出代码，不要包含额外解释。
"""
```

**反思提示词**——让 LLM 扮演严格的代码评审员：

```python
REFLECT_PROMPT = """
你是一位极其严格的代码评审专家，对性能有极致要求。

原始任务: {task}

待审查的代码:
```python
{code}
```

请分析时间复杂度，思考是否存在算法上更优的方案。
如果存在，指出不足并提出改进建议。如果已达到最优，回答"无需改进"。
"""
```

**优化提示词**——让 LLM 根据反馈修正代码：

```python
REFINE_PROMPT = """
你是一位资深Python程序员，正在根据评审反馈优化代码。

原始任务: {task}
上一轮代码: {last_code}
评审员反馈: {feedback}

请生成优化后的新版本代码。
"""
```

> 🔑 反思提示词中的角色设定至关重要。"极其严格"和"专注于算法效率"这两个约束，使评审员不会满足于功能正确的代码，而是追求算法层面的最优。

### 4.4 核心循环实现

```python
class ReflectionAgent:
    def __init__(self, llm_client, max_iterations=3):
        self.llm_client = llm_client
        self.memory = Memory()
        self.max_iterations = max_iterations

    def run(self, task):
        # 1. 初始执行
        initial_code = self._call_llm(INITIAL_PROMPT.format(task=task))
        self.memory.add_record("execution", initial_code)

        # 2. 迭代：反思 -> 优化
        for i in range(self.max_iterations):
            # 反思
            last_code = self.memory.get_last_execution()
            feedback = self._call_llm(REFLECT_PROMPT.format(task=task, code=last_code))
            self.memory.add_record("reflection", feedback)

            # 检查终止条件
            if "无需改进" in feedback:
                break

            # 优化
            refined = self._call_llm(REFINE_PROMPT.format(
                task=task, last_code=last_code, feedback=feedback
            ))
            self.memory.add_record("execution", refined)

        return self.memory.get_last_execution()

    def _call_llm(self, prompt):
        return self.llm_client.think([{"role": "user", "content": prompt}]) or ""
```

### 4.5 实例：代码生成与迭代优化

以"找出 1 到 n 之间所有素数"为例，典型迭代过程：

- **初始版本**：试除法，时间复杂度 O(n * sqrt(n))
- **第一轮反思**：指出性能瓶颈，建议使用埃拉托斯特尼筛法
- **第一轮优化**：实现筛法，复杂度降至 O(n log log n)
- **第二轮反思**：确认已达到最优，输出"无需改进"，循环终止

这个过程展示了 Reflection 的核心价值：不是修复错误，而是**驱动解决方案在质量和效率上实现阶梯式提升**。

### 4.6 成本收益分析

**成本：**
- 每轮迭代至少额外调用 2 次 LLM（反思 + 优化），API 成本成倍增加
- 串行过程，总耗时显著延长，不适合实时性要求高的场景
- 三个角色的提示词都需要精心设计和调试

**收益：**
- 将"合格"的初始方案迭代为"优秀"的最终方案
- 通过内部纠错发现逻辑漏洞、边界情况等问题，提高可靠性

> 🔑 Reflection 是典型的"以成本换质量"策略。适合对结果质量要求极高、对实时性要求宽松的场景（如关键业务代码、技术报告、科学推演）。如果"大致正确"就够用，ReAct 或 Plan-and-Solve 更具性价比。

## 五、三种范式对比与选型

### 5.1 核心差异

| 维度 | ReAct | Plan-and-Solve | Reflection |
|------|-------|----------------|------------|
| 决策方式 | 步进式，边想边做 | 先全局规划，再逐步执行 | 迭代优化，自我批判 |
| 规划能力 | 无全局规划，每步独立决策 | 一次性生成完整计划 | 依赖底层范式（可嵌套 ReAct/PS） |
| 纠错机制 | 通过 Observation 动态调整 | 无内置纠错（静态计划） | 通过反思反馈迭代修正 |
| 工具使用 | 核心能力，每步可调用工具 | 可选，主要用于推理任务 | 通常不直接调用外部工具 |
| LLM 调用次数 | 1 + 步数 | 1 + 计划步骤数 | (1 + 反思) * 迭代轮数 |
| 可解释性 | 高（Thought 链） | 中（计划可见） | 高（执行轨迹 + 反馈） |

### 5.2 选型决策树

```
任务需要外部工具吗？
├── 是 → ReAct（需要动态交互和实时信息）
└── 否 → 任务结构是否清晰可分解？
    ├── 是 → Plan-and-Solve（多步推理、数学题、报告生成）
    └── 否 → 对结果质量要求极高？
        ├── 是 → Reflection（代码生成、学术写作、决策支持）
        └── 否 → 直接使用 LLM 即可
```

### 5.3 混合使用

三种范式并非互斥，实际工程中常组合使用：

- **ReAct + Reflection**：用 ReAct 执行任务，用 Reflection 审查行动轨迹和最终结果
- **Plan-and-Solve + ReAct**：用 Plan-and-Solve 生成全局计划，每个步骤内部用 ReAct 执行
- **Plan-and-Solve + Reflection**：用 Plan-and-Solve 生成计划，用 Reflection 优化计划质量

## 面试题精选

**1. ReAct 的 Thought-Action-Observation 循环与 CoT（思维链）有什么本质区别？**

CoT 是纯推理，LLM 只展示中间思考步骤但无法与外部世界交互；ReAct 在推理的基础上引入了 Action（工具调用）和 Observation（工具返回），使 Agent 能获取实时信息、执行计算、操作外部系统。CoT 容易产生事实幻觉，ReAct 通过工具获取事实依据来缓解这一问题。

**2. 为什么 ReAct 的输出解析推荐用正则表达式而不是 JSON？有什么更好的方案？**

早期 ReAct 实现用正则解析是因为 LLM 的格式遵循能力有限，自然语言格式比 JSON 更容易生成。但正则解析脆弱，LLM 输出稍有偏差就会失败。更好的方案是使用 Function Calling（工具调用）机制，让 LLM 直接输出结构化的工具调用参数，完全避免文本解析问题。这也是 LangChain 等框架的主流做法。

**3. Plan-and-Solve 的静态计划有什么缺陷？如何改进？**

静态计划一旦生成就不再修改，如果执行中发现某步骤无法完成或结果不符合预期，整个计划就会偏离。改进方案是引入"动态重规划"：在每个步骤执行后检查结果，如果偏差超过阈值，则调用 Planner 重新生成后续计划。这就是 Plan-and-Execute 模式中 Replanner 的设计思想。

**4. Reflection 机制的终止条件有哪些设计方式？各有什么优缺点？**

三种常见方式：（1）关键词检测（如"无需改进"），简单但依赖 LLM 的措辞一致性；（2）固定迭代次数，可控但可能过早终止或浪费资源；（3）质量评分，让 LLM 对当前输出打分（如 1-10），分数超过阈值时终止，更灵活但增加了调用成本。实际工程中通常组合使用：关键词检测 + 最大迭代次数作为兜底。

**5. 在 ReAct 实现中，`max_steps` 设为多少合适？如何防止 Agent 陷入死循环？**

没有固定值，取决于任务复杂度。一般简单任务 3-5 步，复杂任务 10-15 步。防止死循环的策略：（1）设置 max_steps 硬上限；（2）检测重复的 Action，如果连续两次调用相同工具且输入相同，强制终止；（3）在提示词中明确告知 LLM 步数限制，促使其更高效地规划。

**6. ToolExecutor 的工具描述如何影响 Agent 的行为？当工具数量很多时怎么优化？**

LLM 依赖工具描述来决定何时使用哪个工具。描述不准确会导致工具选择错误。当工具数量增长到 50+ 时，所有工具描述一次性塞入提示词会占用大量上下文、增加成本、降低选择准确率。优化方案：（1）工具分类，先选择类别再选择具体工具；（2）语义检索，用向量相似度从工具库中检索最相关的 Top-K 个工具；（3）工具推荐，根据用户问题预筛选候选工具。

**7. 如何设计一个结合 ReAct 和 Reflection 的混合 Agent？**

设计思路：外层用 Reflection 的"执行-反思-优化"循环，内层用 ReAct 作为执行器。具体来说：（1）ReAct Agent 执行任务并产出结果和完整的 Thought-Action-Observation 轨迹；（2）Reflection 模块审查轨迹，判断是否存在推理错误、工具使用不当或遗漏信息；（3）如果发现问题，将反馈注入 ReAct 的历史记录，重新执行。这种组合既保留了 ReAct 的工具调用能力，又增加了事后质量保障。

**8. 从工程角度看，手写 Agent 和使用 LangChain 等框架各有什么优劣？**

手写的优势：完全可控、无黑盒、便于调试、深度理解底层机制、可针对特定场景极致优化。劣势：工程量大、需要自己处理边界情况（解析失败、重试、并发等）。框架的优势：开箱即用、生态丰富、社区支持、经过大量场景验证。劣势：抽象层较厚、调试困难、定制成本高、依赖版本更新。建议：学习阶段手写以理解原理，生产环境根据复杂度选择框架或自研。

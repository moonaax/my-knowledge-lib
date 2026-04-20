# Plan-and-Execute 模式


> **本文定位：** ReAct 是"边想边做"，Plan-and-Execute 是"先想后做"。对于目标明确的复杂任务（如代码重构、报告生成），先制定完整计划再逐步执行，比 ReAct 的逐步试探更高效、更可控。
## 1. 核心思想

Plan-and-Execute 将 Agent 的工作分为两个阶段：先制定完整计划，再逐步执行。

````
ReAct:            思考 → 行动 → 观察 → 思考 → 行动 → ...（边想边做）
Plan-and-Execute: 规划完整计划 → 逐步执行 → 必要时重新规划（先想后做）
````
### 优势

- 全局视角，避免局部最优
- 计划可审查和修改
- 减少不必要的工具调用
- 更适合复杂多步骤任务

## 2. 架构设计

````
┌─────────────────────────────────────┐
│           Plan-and-Execute          │
│                                     │
│  ┌─────────┐    ┌───────────────┐  │
│  │ Planner │───→│   Executor    │  │
│  │ (规划器) │    │   (执行器)    │  │
│  └────┬────┘    └───────┬───────┘  │
│       │                 │          │
│       │    ┌────────────▼───┐      │
│       │    │   Re-Planner   │      │
│       │◄───│   (重规划器)    │      │
│       │    └────────────────┘      │
│  ┌────▼────────────────────────┐   │
│  │         Task Queue          │   │
│  │  [Task1, Task2, Task3, ...] │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
````

> **三个核心组件：** Planner（规划器）制定计划，Executor（执行器）逐步执行，Re-Planner（重规划器）根据实际情况调整计划。Task Queue 存放待执行的步骤列表。
## 3. 基础实现

````python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# === 规划器 ===
plan_prompt = ChatPromptTemplate.from_template("""
你是一个任务规划专家。请为以下目标制定详细的执行计划。

目标: {goal}

请输出 JSON 格式的计划:
{{
  "steps": [
    {{"id": 1, "task": "任务描述", "tool": "需要的工具", "depends_on": []}},
    {{"id": 2, "task": "任务描述", "tool": "需要的工具", "depends_on": [1]}}
  ]
}}

注意:
1. 步骤要具体可执行
2. 明确标注依赖关系
3. 可并行的步骤不要设置依赖
""")

plan_chain = plan_prompt | llm | JsonOutputParser()

# === 执行器 ===
execute_prompt = ChatPromptTemplate.from_template("""
请执行以下任务:

任务: {task}
可用工具: {tool}
之前步骤的结果: {previous_results}

请执行并返回结果。
""")

execute_chain = execute_prompt | llm

# === 重规划器 ===
replan_prompt = ChatPromptTemplate.from_template("""
原始目标: {goal}
原始计划: {original_plan}
已完成步骤: {completed_steps}
当前步骤结果: {current_result}

请评估:
1. 当前进展是否符合预期?
2. 是否需要调整后续计划?

如果需要调整，输出新的计划（JSON 格式）。
如果不需要，输出 {{"no_change": true}}
""")

replan_chain = replan_prompt | llm | JsonOutputParser()

# === 主循环 ===
def plan_and_execute(goal: str):
    # 1. 制定计划
    plan = plan_chain.invoke({"goal": goal})
    results = {}
    
    for step in plan["steps"]:
        # 2. 检查依赖
        prev_results = {k: results[k] for k in step.get("depends_on", []) if k in results}
        
        # 3. 执行步骤
        result = execute_chain.invoke({
            "task": step["task"],
            "tool": step["tool"],
            "previous_results": str(prev_results)
        })
        results[step["id"]] = result.content
        
        # 4. 重规划检查
        replan = replan_chain.invoke({
            "goal": goal,
            "original_plan": plan,
            "completed_steps": results,
            "current_result": result.content
        })
        
        if "steps" in replan:
            plan["steps"] = replan["steps"]  # 更新计划
    
    return results

> **关键代码解读（基础版）：**
> - `plan_chain` — 规划器，输入目标，输出带依赖关系的步骤列表（JSON）
> - `execute_chain` — 执行器，输入单个任务 + 之前的结果，输出执行结果
> - `replan_chain` — 重规划器，每步执行后评估是否需要调整后续计划
> - 主循环中先检查依赖（`depends_on`），确保前置步骤已完成
````
## 4. 使用 LangGraph 实现


> **思路：** 用 LangGraph 的图结构实现，三个节点：plan（规划）→ execute（循环执行）→ summarize（汇总）。条件边控制执行循环，比 for 循环更清晰可控。
````python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class PlanExecuteState(TypedDict):
    goal: str
    plan: list[str]
    current_step: int
    results: Annotated[list, operator.add]
    response: str

def planner(state: PlanExecuteState) -> PlanExecuteState:
    """制定计划"""
    plan = plan_chain.invoke({"goal": state["goal"]})
    return {"plan": plan["steps"], "current_step": 0}

def executor(state: PlanExecuteState) -> PlanExecuteState:
    """执行当前步骤"""
    step = state["plan"][state["current_step"]]
    result = execute_chain.invoke({
        "task": step["task"],
        "tool": step.get("tool", "none"),
        "previous_results": str(state.get("results", []))
    })
    return {
        "results": [result.content],
        "current_step": state["current_step"] + 1
    }

def should_continue(state: PlanExecuteState) -> str:
    """判断是否继续执行"""
    if state["current_step"] >= len(state["plan"]):
        return "summarize"
    return "execute"

def summarizer(state: PlanExecuteState) -> PlanExecuteState:
    """汇总结果"""
    summary = llm.invoke(
        f"目标: {state['goal']}\n结果: {state['results']}\n请汇总最终回答。"
    )
    return {"response": summary.content}

# 构建图
graph = StateGraph(PlanExecuteState)
graph.add_node("plan", planner)
graph.add_node("execute", executor)
graph.add_node("summarize", summarizer)

graph.set_entry_point("plan")
graph.add_edge("plan", "execute")
graph.add_conditional_edges("execute", should_continue, {
    "execute": "execute",
    "summarize": "summarize"
})
graph.add_edge("summarize", END)

app = graph.compile()
result = app.invoke({"goal": "分析项目代码质量并生成报告", "results": []})

> **关键代码解读（LangGraph 版）：**
> - `PlanExecuteState` — 状态包含目标、计划列表、当前步骤索引、结果列表
> - `should_continue` 条件边——还有未执行的步骤就继续，否则跳到汇总
> - `Annotated[list, operator.add]` — results 字段用追加模式，每步结果自动累积
> - 相比基础实现，LangGraph 版本状态管理更清晰，支持持久化和可视化
````

> **对比总结：** ReAct 适合探索性任务（边想边做），Plan-and-Execute 适合目标明确的任务（先想后做）。实际项目中最佳实践是混合使用——顶层用 Plan-and-Execute 规划，每个子任务内部用 ReAct 执行。
## 5. 与 ReAct 的对比

| 维度 | ReAct | Plan-and-Execute |
|------|-------|-----------------|
| 规划方式 | 逐步规划 | 预先全局规划 |
| 适合任务 | 探索性任务 | 目标明确的任务 |
| 工具调用 | 可能冗余 | 更精准 |
| 错误恢复 | 即时调整 | 需要重规划 |
| Token 消耗 | 每步都推理 | 规划一次 + 执行 |
| 可控性 | 较低 | 较高（计划可审查） |

### 混合使用

````
复杂任务的最佳实践:
1. 用 Plan-and-Execute 做顶层规划
2. 每个子任务内部用 ReAct 执行
3. 执行完每个子任务后检查是否需要重规划
````

> **Plan-and-Execute 最适合这类目标明确、步骤可预见的任务：** 代码重构、报告生成、数据迁移等。如果任务目标模糊或需要大量探索，ReAct 更合适。
## 6. 实际应用场景

### 代码重构

````
目标: 将一个单体应用拆分为微服务

计划:
1. 分析现有代码结构和依赖关系
2. 识别可独立的业务模块
3. 设计微服务边界和接口
4. 逐个模块拆分（按依赖顺序）
5. 编写集成测试
6. 验证功能完整性
````
### 数据分析报告

````
目标: 生成月度销售分析报告

计划:
1. 从数据库获取本月销售数据
2. 计算关键指标（总额、增长率、TOP 产品）
3. 与上月/去年同期对比
4. 生成可视化图表
5. 撰写分析结论和建议
6. 输出 PDF 报告
````
---

## 面试题精选

### Q1: Plan-and-Execute 和 ReAct 的核心区别是什么？
**答：** ReAct 是"边想边做"，每步都重新推理决策；Plan-and-Execute 是"先想后做"，先制定完整计划再逐步执行。Plan-and-Execute 有全局视角、计划可审查、减少冗余工具调用，更适合目标明确的复杂任务。

### Q2: Plan-and-Execute 中的"重规划"（Re-planning）什么时候触发？
**答：** 每个子任务执行完后评估结果是否符合预期，如果出现意外情况（工具调用失败、中间结果与预期不符、发现新信息改变了任务方向），就触发重规划调整后续步骤。

### Q3: 如何用 LangGraph 实现 Plan-and-Execute？
**答：** 定义 State 包含 goal、plan、current_step、results 等字段，创建 planner（规划）、executor（执行）、summarizer（汇总）三个节点，用条件边控制循环执行直到所有步骤完成，最后汇总结果。

### Q4: Plan-and-Execute 模式的主要缺点是什么？
**答：** 初始规划可能不准确（信息不足时难以制定完美计划）、重规划有额外开销、对于探索性任务不如 ReAct 灵活。如果任务目标模糊或需要大量试错，ReAct 更合适。

### Q5: 实际项目中 Plan-and-Execute 和 ReAct 怎么结合使用？
**答：** 最佳实践是用 Plan-and-Execute 做顶层规划（拆分大任务为子任务），每个子任务内部用 ReAct 执行（灵活调用工具），执行完每个子任务后检查是否需要重规划。这样兼顾全局规划和局部灵活性。

# AutoGen 与多 Agent 框架

## 1. 概述

多 Agent 框架让多个 AI Agent 协作完成复杂任务。每个 Agent 有不同的角色和能力，通过对话和协调共同工作。

### 主流框架对比

| 框架 | 厂商 | 核心特点 | 适用场景 |
|------|------|---------|---------|
| AutoGen | 微软 | 对话驱动的多 Agent | 复杂推理、代码生成 |
| CrewAI | 开源社区 | 角色扮演式协作 | 业务流程自动化 |
| LangGraph | LangChain | 有状态图编排 | 精细控制的 Agent 流程 |
| MetaGPT | 开源社区 | 软件公司模拟 | 软件开发全流程 |

## 2. AutoGen

### 2.1 核心概念

````
AutoGen 的核心 = Agent 之间的对话

UserProxy Agent  → 代表用户，可执行代码
Assistant Agent  → AI 助手，负责推理和生成
Group Chat       → 多个 Agent 的群聊协作
````
### 2.2 安装

````bash
pip install autogen-agentchat autogen-ext[openai]
````
### 2.3 双 Agent 对话

> **这是 AutoGen 最基础的模式：** 两个 Agent 对话——AssistantAgent（AI 助手）负责思考和生成代码，UserProxyAgent（用户代理）负责自动执行代码并把结果反馈给 Assistant。这形成了一个"生成→执行→反馈→改进"的自动闭环，不需要人工介入。

````python
from autogen import AssistantAgent, UserProxyAgent

# AI 助手
assistant = AssistantAgent(
    name="coder",
    llm_config={"model": "gpt-4o", "api_key": "your-key"},
    system_message="你是一个 Python 专家，擅长编写高质量代码。"
)

# 用户代理（可自动执行代码）
user_proxy = UserProxyAgent(
    name="user",
    human_input_mode="NEVER",       # 不需要人工输入
    code_execution_config={"work_dir": "coding", "use_docker": False},
    max_consecutive_auto_reply=5
)

# 发起对话
user_proxy.initiate_chat(
    assistant,
    message="写一个 Python 脚本，分析 CSV 文件中的销售数据并生成图表"
)
````

> **关键代码解读：**
> - `human_input_mode="NEVER"` — 完全自动化，不暂停等待人工输入。也可以设为 `"ALWAYS"`（每轮都问人）或 `"TERMINATE"`（结束时问人）
> - `code_execution_config` — 允许 UserProxy 自动执行 Assistant 生成的代码，`work_dir` 指定工作目录
> - `max_consecutive_auto_reply=5` — 最多自动回复 5 轮，防止无限对话
> - `initiate_chat()` — 发起对话，UserProxy 把任务描述发给 Assistant，然后两者自动来回对话直到任务完成

### 2.4 Group Chat

> **从双人对话到群聊：** 双 Agent 只能处理简单任务。复杂任务需要多个角色协作——项目经理拆解任务、开发写代码、审查员 Review。GroupChat 让多个 Agent 在一个"群聊"中协作，由 `GroupChatManager` 自动决定每轮谁来发言。 — 多 Agent 协作

````python
from autogen import GroupChat, GroupChatManager

# 定义多个 Agent
planner = AssistantAgent(
    name="planner",
    system_message="你是项目经理，负责拆解任务和制定计划。",
    llm_config=llm_config
)

coder = AssistantAgent(
    name="coder",
    system_message="你是开发工程师，负责编写代码。只在需要写代码时发言。",
    llm_config=llm_config
)

reviewer = AssistantAgent(
    name="reviewer",
    system_message="你是代码审查员，负责 Review 代码质量。只在有代码需要审查时发言。",
    llm_config=llm_config
)

# 创建群聊
group_chat = GroupChat(
    agents=[user_proxy, planner, coder, reviewer],
    messages=[],
    max_round=15,
    speaker_selection_method="auto"  # LLM 自动选择下一个发言者
)

manager = GroupChatManager(groupchat=group_chat, llm_config=llm_config)

# 发起群聊
user_proxy.initiate_chat(
    manager,
    message="开发一个 REST API，包含用户注册、登录和个人信息管理功能"
)
````

> **关键代码解读：**
> - 每个 Agent 的 `system_message` 要明确职责和发言条件（如"只在有代码需要审查时发言"），否则会出现抢话、角色混乱
> - `max_round=15` — 群聊最多 15 轮，防止无限对话导致成本失控
> - `speaker_selection_method="auto"` — 让 LLM 根据对话上下文自动选择下一个发言者，也可以设为 `"round_robin"`（轮流）或自定义函数
> - `GroupChatManager` — 群聊的"主持人"，负责协调发言顺序和终止条件

## 3. CrewAI

> **CrewAI vs AutoGen：** AutoGen 是"对话驱动"——Agent 之间通过自由对话协作，灵活但不太可控。CrewAI 是"任务驱动"——先定义好角色（Agent）、任务（Task）、流程（Process），然后按流程执行，更结构化、更可预测。

### 3.1 核心概念

````
CrewAI 的核心 = 角色 + 任务 + 流程

Agent  → 有角色、目标、背景的智能体
Task   → 具体的任务描述和期望输出
Crew   → Agent 团队，按流程协作
Tool   → Agent 可使用的工具
````
### 3.2 安装

````bash
pip install crewai crewai-tools
````
### 3.3 基础示例

> **CrewAI 的三要素：** Agent（谁来做）、Task（做什么）、Crew（怎么协作）。每个 Agent 有明确的角色和目标，每个 Task 有明确的描述和期望输出，Crew 把它们组合起来按指定流程执行。

````python
from crewai import Agent, Task, Crew, Process

# 定义 Agent
researcher = Agent(
    role="技术调研员",
    goal="深入调研技术方案并输出分析报告",
    backstory="你是一个资深技术专家，擅长技术选型和方案对比分析。",
    verbose=True,
    llm="gpt-4o"
)

writer = Agent(
    role="技术文档工程师",
    goal="将技术调研结果整理为清晰易懂的文档",
    backstory="你擅长将复杂技术概念转化为易于理解的文档。",
    verbose=True,
    llm="gpt-4o"
)

# 定义任务
research_task = Task(
    description="调研 2024 年主流的 AI Agent 框架，对比它们的优缺点",
    expected_output="一份包含至少 5 个框架对比的调研报告",
    agent=researcher
)

writing_task = Task(
    description="基于调研报告，编写一篇技术选型指南",
    expected_output="一篇结构清晰的技术博客文章",
    agent=writer,
    context=[research_task]  # 依赖调研任务的输出
)

# 组建团队
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    process=Process.sequential,  # 顺序执行
    verbose=True
)

# 执行
result = crew.kickoff()
print(result)
````

> **关键代码解读：**
> - `Agent` 的 `role`、`goal`、`backstory` 三个字段共同定义了 Agent 的"人设"，LLM 会据此来扮演角色
> - `Task` 的 `context=[research_task]` — 任务依赖关系，writing_task 能获取 research_task 的输出作为输入
> - `Process.sequential` — 顺序执行，先调研再写作。也支持 `Process.hierarchical`（层级模式，由 Manager Agent 分配任务）
> - `crew.kickoff()` — 启动执行，按流程依次完成所有任务
>
> **对比 AutoGen：** AutoGen 的 Agent 通过自由对话协作，过程不太可控。CrewAI 的任务和流程是预定义的，执行过程更可预测，适合业务流程自动化。

### 3.4 自定义工具

> **给 Agent 配备工具：** 和 LangChain 类似，CrewAI 的 Agent 也可以使用工具。`@tool` 装饰器把函数变成工具，通过 `tools=[...]` 分配给 Agent。Agent 在执行任务时会自主决定是否需要调用工具。

````python
from crewai.tools import tool

@tool("搜索技术文档")
def search_docs(query: str) -> str:
    """搜索技术文档库获取相关信息"""
    # 实现搜索逻辑
    return f"搜索结果: {query}"

# 给 Agent 配备工具
researcher = Agent(
    role="调研员",
    goal="调研技术方案",
    tools=[search_docs],
    llm="gpt-4o"
)
````
## 4. MetaGPT — 软件公司模拟

> **MetaGPT 的独特之处：** 其他框架是通用的多 Agent 协作工具，MetaGPT 专门模拟软件公司的工作流程。每个角色有标准化的输入输出——产品经理输出 PRD 文档，架构师输出设计文档，工程师根据设计文档写代码，QA 写测试用例。这种"标准化交付物"的设计让协作更高效。

MetaGPT 模拟一个软件公司的完整团队：

````
产品经理 → 需求分析、PRD 文档
架构师   → 系统设计、技术选型
工程师   → 代码实现
QA       → 测试用例、质量保证
````
````python
from metagpt.software_company import SoftwareCompany
from metagpt.roles import ProjectManager, Architect, Engineer, QaEngineer

async def main():
    company = SoftwareCompany()
    company.hire([
        ProjectManager(),
        Architect(),
        Engineer(n_borg=3),  # 3个工程师
        QaEngineer()
    ])
    company.invest(investment=10.0)  # 预算（API 费用）
    company.start_project("开发一个待办事项管理的 Web 应用")
    await company.run(n_round=5)
````

> **关键代码解读：**
> - `company.hire([...])` — 组建团队，每个角色是一个预定义的 Agent
> - `Engineer(n_borg=3)` — 可以配置多个工程师并行开发
> - `company.invest(investment=10.0)` — 设置 API 调用预算（美元），防止成本失控
> - `company.run(n_round=5)` — 运行 5 轮迭代，每轮各角色按顺序交付自己的产出

## 5. 多 Agent 设计原则

> **本节是方法论：** 前面介绍了具体框架，这里讲的是不管用哪个框架都适用的设计原则。多 Agent 系统最常见的问题不是技术实现，而是角色设计不合理、协作流程不清晰。

### 5.1 角色设计

````
好的角色设计:
✅ 职责单一明确
✅ 有清晰的输入/输出定义
✅ 角色之间有互补性

差的角色设计:
❌ 一个 Agent 什么都做
❌ 角色职责重叠
❌ 没有明确的协作流程
````
### 5.2 协作模式

> **选择协作模式的关键：** 根据任务特点选择——有先后依赖用顺序模式，需要统一协调用层级模式，需要多角度分析用辩论模式，需要高可靠性用投票模式。实际项目中也可以混合使用。

````
1. 顺序模式 (Sequential)
   A → B → C → 输出
   适合: 流水线式任务

2. 层级模式 (Hierarchical)
   Manager → [Worker1, Worker2, Worker3]
   适合: 需要协调的复杂任务

3. 辩论模式 (Debate)
   Agent1 ⟷ Agent2 → 共识
   适合: 需要多角度分析的决策

4. 投票模式 (Voting)
   [Agent1, Agent2, Agent3] → 多数决
   适合: 需要可靠性的场景
````
### 5.3 常见陷阱

> **多 Agent 系统的"坑"比单 Agent 多得多：** 多个 Agent 交互会产生组合爆炸的复杂性——无限对话、角色混乱、成本失控都是高频问题。下面这张表是实践中总结的经验。

| 陷阱 | 解决方案 |
|------|---------|
| Agent 之间无限对话 | 设置 max_round 限制 |
| 角色混乱，抢话 | 明确 system_message 中的发言条件 |
| 成本失控 | 监控 Token 使用，设置预算上限 |
| 结果不一致 | 加入 Review Agent 做质量把关 |

## 6. 框架选型建议

> **选型核心原则：** 不要为了用多 Agent 而用多 Agent。先评估任务是否真的需要多个角色协作——如果一个 Agent + 几个工具就能搞定，就不要引入多 Agent 的复杂性。只有当任务确实需要不同视角、不同专长的角色协作时，才考虑多 Agent 框架。

````
简单的双 Agent 对话     → AutoGen
角色明确的团队协作       → CrewAI
需要精细流程控制         → LangGraph
模拟软件开发流程         → MetaGPT
生产环境部署            → LangGraph（最成熟）
快速原型验证            → CrewAI（最简单）
````
---

## 面试题精选

### Q1: AutoGen、CrewAI、LangGraph 三个多 Agent 框架怎么选？
**答：** AutoGen 适合对话驱动的复杂推理和代码生成；CrewAI 最简单，适合角色明确的团队协作和快速原型；LangGraph 最成熟，适合需要精细流程控制的生产环境。快速验证选 CrewAI，生产部署选 LangGraph。

### Q2: AutoGen 的 UserProxyAgent 和 AssistantAgent 分别是什么角色？
**答：** AssistantAgent 是 AI 助手，负责推理和生成（如写代码）。UserProxyAgent 代表用户，可以自动执行 AssistantAgent 生成的代码并返回结果，形成"生成-执行-反馈"的闭环。

### Q3: CrewAI 中 Agent、Task、Crew 三个概念的关系是什么？
**答：** Agent 是有角色和目标的智能体，Task 是具体的任务描述和期望输出，Crew 是 Agent 团队按指定流程（顺序/层级）协作执行 Task。Task 可以指定依赖关系，后续 Task 能获取前置 Task 的输出。

### Q4: 多 Agent 系统中如何控制成本？
**答：** 监控每个 Agent 的 Token 使用量、设置预算上限、限制最大对话轮次（max_round）、用更便宜的模型处理简单 Agent 的任务、缓存重复查询结果、减少不必要的 Agent 间对话。

### Q5: 多 Agent 协作有哪些常见模式？各自适合什么场景？
**答：** 顺序流水线（A→B→C，适合有先后关系的任务）、层级模式（Manager 分配任务给 Worker，适合复杂项目）、辩论模式（正反方辩论+裁判，适合决策分析）、投票模式（多 Agent 独立回答取共识，适合需要可靠性的场景）。

### Q6: MetaGPT 的设计理念是什么？和其他框架有什么不同？
**答：** MetaGPT 模拟真实软件公司的组织结构（产品经理→架构师→工程师→QA），每个角色有标准化的输入输出（PRD、设计文档、代码、测试用例）。它更关注软件开发全流程的自动化，而非通用的多 Agent 协作。

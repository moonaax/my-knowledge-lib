# Harness Engineering（驾驭工程）

> AI Agent 领域 2025-2026 年兴起的核心工程概念。Agent = Model + Harness，Harness 是套在模型外面的约束、工具、反馈循环等基础设施，决定了 Agent 的实际表现。综合整理自 Martin Fowler、On-Site Reliability、Infralovers、Louis Bouchard 等文章。

相关主题：[[1-什么是AI Agent]] | [[3-Prompt Engineering]]

---

## 一、什么是 Harness

### 1.1 核心公式

**Agent = Model + Harness**

Harness 一词源自马具——套在马身上的缰绳、笼头、鞍具。马有力量，但需要 harness 来控制和引导它干活。

在 AI Agent 语境下：
- **模型（Model）** = 马（原始智能）
- **Harness** = 套在模型外面的整个运行环境（system prompt、工具定义、上下文管理、反馈循环、约束规则等）
- **人类** = 骑手/车夫，负责转向（steering）

> 🔑 "模型提供智能，Harness 提供控制。" —— Harness 不只是 prompt，而是模型周围的整个运行基础设施。

### 1.2 Harness 的层次

在编码 Agent（如 Claude Code、Cursor）中，harness 分为两层：
- **内层 harness（Builder Harness）**：Agent 产品内建的，如 system prompt、代码检索机制、编排系统
- **外层 harness（User Harness）**：用户为自己的项目构建的，如 AGENTS.md、自定义 linter、Skill 文件

Harness Engineering 主要关注的是外层——用户如何为自己的场景构建约束和反馈系统。

## 二、为什么需要 Harness Engineering

### 2.1 痛点

Agent 足够强大到有用，但还不够可靠到可以信任。当模型能力足够时，瓶颈从"它能不能生成代码"变成了"它能不能在真实系统中可靠地工作"。

### 2.2 数据佐证

| 研究 | 发现 |
|------|------|
| METR 研究 | 16 名资深开发者，246 个真实任务，AI 辅助反而慢了 19%。但开发者自己以为快了 20% |
| DORA 2024 | AI 采用率 +25%，但吞吐量 -1.5%，稳定性 -7.2% |
| Faros AI | 10000+ 开发者：完成任务 +21%，合并 PR +98%，但 review 时间 +91% |

> 🔑 **关键发现**：LangChain 仅通过改进 harness（不换模型），在 Terminal Bench 2.0 上从 52.8% 提升到 66.5%，提升 13.7 个百分点。模型决定能力上限，Harness 决定实际表现。

### 2.3 概念起源与演进

```
2025.12  Karpathy 描述工作流反转，暴露 Agent 脆弱性
2025 末  Anthropic 发布长时运行 Agent 蓝图（外化记忆、角色拆分）
2026.02  Mitchell Hashimoto 正式命名 "Harness Engineering"
2026.02  OpenAI 发布百万行零手写代码实验报告
2026.02  Martin Fowler 发布初始备忘录
2026.03  Louis Bouchard、Infralovers、On-Site Reliability 等跟进解读
2026.04  Martin Fowler 发布完整文章（Guides/Sensors 框架）
```

Mitchell Hashimoto 的核心主张：

> 🔑 **每次 Agent 犯错，不要期望它下次做得更好。改造环境，让它不可能再以同样的方式犯错。**

## 三、核心框架：Guides 与 Sensors（Martin Fowler）

### 3.1 前馈（Guides）与反馈（Sensors）

| 类型 | 作用 | 时机 |
|------|------|------|
| **Guides（前馈控制）** | 预判 Agent 行为，在它行动前引导 | 行动前 |
| **Sensors（反馈控制）** | 观察 Agent 行动结果，帮助自我纠正 | 行动后 |

只有反馈 → Agent 反复犯同样的错。只有前馈 → 编码了规则但不知道是否有效。两者缺一不可。

### 3.2 计算型 vs 推理型

| | 计算型（Computational） | 推理型（Inferential） |
|---|---|---|
| 执行方式 | CPU，确定性 | GPU/NPU，非确定性 |
| 速度 | 毫秒到秒级 | 较慢，较贵 |
| 例子 | 测试、linter、类型检查、静态分析 | AI 代码审查、LLM-as-judge |
| 可靠性 | 高 | 概率性 |

### 3.3 四象限实例

| 控制手段 | 方向 | 类型 | 实现 |
|---------|------|------|------|
| 编码规范 | 前馈 | 推理型 | AGENTS.md、Skills 文件 |
| 项目脚手架 | 前馈 | 两者 | Skill + 引导脚本 |
| 代码迁移工具 | 前馈 | 计算型 | OpenRewrite recipes |
| 结构测试 | 反馈 | 计算型 | ArchUnit 检查模块边界 |
| 代码审查指令 | 反馈 | 推理型 | Review Skills |

> 🔑 计算型传感器足够便宜，可以每次提交都跑；推理型传感器更适合在关键节点使用。特别强大的做法是：自定义 linter 的错误信息中直接包含修复指令——一种正向的 prompt injection。

## 四、转向循环与质量左移

### 4.1 转向循环（Steering Loop）

人类的工作是**转向**——迭代改进 harness：

```
Agent 犯错 → 分析原因 → 改进 guides/sensors → 降低未来出错概率 → 循环
```

核心心态转变：
- ❌ "这个模型太蠢了"
- ✅ "我的系统允许了这种失败模式"

### 4.2 质量左移（Keep Quality Left）

反馈传感器应尽早介入，越早发现问题修复成本越低：

1. **提交前**：LSP、linter、快速测试、基础代码审查 Agent
2. **集成后（CI 流水线）**：变异测试、更全面的架构审查
3. **持续漂移检测**：死代码检测、测试覆盖质量、依赖扫描
4. **运行时反馈**：SLO 监控、日志异常检测、响应质量采样

## 五、三个调控维度

### 5.1 可维护性 Harness

调控内部代码质量。**最成熟**，已有大量现成工具。

计算型传感器能可靠捕获：重复代码、圈复杂度、缺失测试、架构漂移、风格违规。

推理型传感器能部分处理：语义重复代码、冗余测试、暴力修复、过度工程。但更贵且概率性。

> 🔑 两者都无法可靠捕获的高影响问题：误诊 issue、过度工程、误解指令。这些仍需人类监督。

### 5.2 架构适应度 Harness

定义和检查应用的架构特征（即 Fitness Functions）。例如：
- 性能要求的 Skill + 性能测试反馈
- 可观测性编码规范 + 调试时反思日志质量

### 5.3 行为 Harness

功能正确性——**目前最难的部分**。当前主流做法：
- 前馈：功能规格说明（从简短 prompt 到多文件描述）
- 反馈：AI 生成的测试套件 + 覆盖率 + 变异测试 + 人工测试

> 🔑 这种做法对 AI 生成的测试寄予了过多信任，目前还不够好。行为 Harness 仍有大量未解决的问题。

## 六、可驾驭性与 Harness 模板

### 6.1 可驾驭性（Harnessability）

不是每个代码库都同样容易被驾驭：
- 强类型语言 → 天然有类型检查传感器
- 清晰的模块边界 → 可以定义架构约束规则
- 成熟框架（如 Spring）→ 隐式提高 Agent 成功率
- "无聊"的技术 → 可组合性好、API 稳定、训练数据中表示充分

**新项目**可以从第一天就内建可驾驭性。**遗留项目**面临更大挑战：最需要 harness 的地方，恰恰最难构建。

### 6.2 Harness 模板

企业常见的服务拓扑（API 服务、事件处理、数据看板）可以演变为 **Harness 模板**：一组预定义的 guides + sensors，绑定到特定技术栈和架构。

> 🔑 Ashby 必要多样性定律：调控器必须至少拥有与被调控系统同等的多样性。承诺一个拓扑就是缩减多样性，让全面的 harness 变得更可实现。团队未来可能会基于"已有哪些 harness 模板"来选择技术栈。

## 七、Prompt / Context / Harness 的关系

### 7.1 三者对比

| | Prompt Engineering | Context Engineering | Harness Engineering |
|---|---|---|---|
| 关注点 | 问什么 | 给模型看什么 | 整个系统如何运作 |
| 范围 | 单次输入 | 上下文窗口内容 | 工具、权限、状态、测试、日志、重试、检查点 |
| 类比 | 给马下指令 | 给马看地图 | 整套马具 + 道路 + 围栏 |

> 🔑 Context Engineering 是 Harness Engineering 的子集。Harness 决定何时加载上下文、哪些工具可用、如何处理失败、什么条件下才算"完成"。

### 7.2 一个具体例子

假设你让 Agent 修一个 API bug：
- **Context Engineering**：确保模型看到相关的 API 代码、错误日志、数据库 schema
- **Harness Engineering**：确保修完后自动跑 linter → 跑测试 → 检查是否引入新依赖 → 通过才能提交 → 失败则自动重试并附上错误信息

前者是"给它看什么"，后者是"整个修复流程怎么运转"。

## 八、行业案例深入对比

### 8.1 OpenAI 内部产品（Codex）

OpenAI 团队用 Codex 构建了一个内部产品，**零行手写代码**，约 100 万行代码，1500 个 PR，3-7 人团队。

关键实践：

| 维度 | 做法 |
|------|------|
| 知识管理 | AGENTS.md 约 100 行当目录，指向 `docs/` 深层文档。渐进式披露 |
| 架构执行 | 分层模型 + 自定义 linter + 结构测试。linter 错误信息中包含修复指令 |
| Review | Agent-to-Agent review，人类可选参与 |
| 技术债 | "垃圾回收"Agent 定期扫描偏差，开修复 PR |
| 可观测性 | 日志/指标/追踪暴露给 Agent，可用 LogQL/PromQL 查询 |
| 运行时长 | 单次运行经常连续工作 6 小时以上（人类睡觉时） |

> 🔑 仓库即知识系统——Slack 讨论如果没进 repo，对 Agent 来说就不存在。和新员工入职三个月后不知道之前的讨论是一个道理。

### 8.2 Cursor 的多 Agent 架构

- **架构**：Root Planner → 递归 Sub-Planner → Worker（隔离 repo 副本）
- **Planner 不直接执行代码**，只做规划和分配
- **成果**：从零构建 Web 浏览器，100 万行代码，1000 个文件，一周

### 8.3 Anthropic Agent Teams

- **架构**：Lead Agent 协调 + Teammate 各自独立上下文窗口
- **通信**：点对点消息 + 基于文件的任务板（含依赖逻辑）
- **早期实践**：把角色拆分为 initializer agent 和 coding agent，外化记忆为 artifact（功能清单、进度日志、Git 提交、初始化脚本）
- **成果**：Nicolas Carlini 用 16 个 Agent 写了 C 编译器（Rust），约 10 万行，GCC Torture Test 通过率 99%

### 8.4 DeepMind Aletheia

- **架构**：不是多 Agent，而是单模型的迭代 Generate-Verify-Revise 循环
- **核心**：验证反馈循环本身就能带来巨大提升，不需要多 Agent 协调
- **成果**：解决了 FirstProof 10 个问题中的 6 个

### 8.5 共同模式

> 🔑 这些系统的共同点不是"多 Agent"，而是：**规划 → 执行 → 验证 → 迭代**。研究表明多 Agent 系统在很多情况下可以压缩为等效的单 Agent 系统（节省 53.7% token，准确率不变）。多 Agent 的优势不在于数量，而在于背后的组织原则。

## 九、可验证性层级

Harness 模式在不同验证难度下效果不同：

| 层级 | 类型 | 验证方式 | Harness 效果 |
|------|------|---------|-------------|
| L1 | 形式化（数学证明） | 精确、可自动化 | 极好 |
| L2 | 可测试（代码、CI/CD） | 自动化测试，秒级 | 很好 |
| L3 | 基于规则（合规、法律） | 规则验证，较慢 | 可用，需人工审查 |
| L4 | 启发式（客服、研究） | 评估模型/人工反馈 | 效果有限 |
| L5 | 不可验证（创意、伦理） | 无客观评判 | 模式失效 |

> 🔑 目前最亮眼的成果都在 L1-L2 层级。向 L4-L5 外推是假设而非已证明的结论。反馈循环越短，harness 效果越好。

## 十、人类的角色——隐式 Harness

### 10.1 人类开发者自带"隐式 harness"

作为人类开发者，我们把自己的技能和经验作为一种**隐式 harness** 带到每个代码库中：
- 内化了编码规范和最佳实践
- 感受过复杂性带来的认知痛苦
- 知道自己的名字会出现在 commit 上（社会责任感）
- 了解组织目标——哪些技术债是出于业务原因被容忍的
- 以人类的节奏小步前进，创造了让经验被触发和应用的思考空间

### 10.2 Agent 没有这些

Agent 没有社会责任感、没有对 300 行函数的审美厌恶、没有"我们这里不这么做"的直觉、不知道哪个规范是承重墙哪个只是习惯、无法判断技术上正确的方案是否符合团队目标。

### 10.3 Harness 的本质

> 🔑 Harness Engineering 是试图把人类开发者的隐式经验外化和显式化的过程。但它只能做到一定程度。好的 harness 不应该以完全消除人类输入为目标，而是把人类的注意力引导到最重要的地方。

## 十一、实操指南

### 11.1 AGENTS.md 怎么写

**❌ 错误做法：一个巨大的指令手册**
- 上下文是稀缺资源，巨大的指令文件会挤掉任务和代码
- 当所有东西都"重要"时，什么都不重要
- 会迅速腐烂，Agent 无法判断哪些还有效

**✅ 正确做法：当目录用，约 100 行**

```markdown
# AGENTS.md

## 项目概述
简短描述项目做什么，核心技术栈。

## 架构
见 docs/architecture.md

## 编码规范
见 docs/coding-standards.md

## 测试要求
- 所有新代码必须有测试
- 运行 `npm test` 验证
- 覆盖率不低于 80%

## 常见陷阱
- 不要直接修改 generated/ 目录下的文件
- API 变更必须同步更新 docs/api-spec.md
```

### 11.2 渐进式构建路径

1. **基础前馈**：写简洁的 AGENTS.md，定义编码规范和项目结构
2. **基础反馈**：确保 linter 和测试能在 Agent 工作时运行
3. **迭代改进**：每次 Agent 犯错，问自己"能不能加一条规则/检查来防止？"
4. **高级控制**：添加架构约束测试、AI 代码审查、自定义 linter（错误信息含修复指令）

### 11.3 Skill 文件的自我进化

> 🔑 在每个 Skill 文件的最后一步，加上"回顾整个交互，总结哪些做得好、哪些不好，然后修改 Skill 本身以便下次更好"。这让 harness 能自我进化——Agent 不仅完成任务，还改进自己的约束系统。

## 十二、开放问题

1. **Harness 一致性** —— 随着 harness 增长，如何保持 guides 和 sensors 同步、不互相矛盾？
2. **冲突信号** —— 当指令和反馈信号指向不同方向时，Agent 能做出合理权衡吗？
3. **传感器有效性** —— 如果传感器从不触发，是质量高还是检测不够？
4. **Harness 覆盖率** —— 需要类似代码覆盖率的机制来评估 harness 的质量
5. **行为 Harness** —— 功能正确性的验证仍然高度依赖人工
6. **Harness 债务** —— Harness 本身也会有 bug 和漂移，成为需要维护的产品

> 🔑 "没有结构的 Agent，不是坏 Agent，而是一个礼貌的随机数生成器。" —— Infralovers

## 面试要点

- Harness 的定义与核心公式（Agent = Model + Harness）
- Guides（前馈）与 Sensors（反馈）的区别和协作
- 计算型控制与推理型控制的适用场景
- Prompt / Context / Harness 三者的边界
- 转向循环的心态转变
- 可驾驭性（Harnessability）的影响因素
- 可验证性层级与 Harness 的适用边界
- 人类"隐式 harness"的不可替代性
- AGENTS.md 的正确写法（目录 vs 百科全书）

## 面试题精选

### Q1: 什么是 Harness Engineering？它和 Prompt Engineering 有什么区别？
**答：** Harness Engineering 是构建 AI Agent 周围的约束、工具、反馈循环等基础设施的工程实践。公式：Agent = Model + Harness。Prompt Engineering 关注"问什么"，Context Engineering 关注"给模型看什么"，Harness Engineering 关注"整个系统如何运作"——包括工具权限、状态管理、验证机制、重试策略、检查点等。Context Engineering 是 Harness Engineering 的子集。

### Q2: Harness 的两大控制方向是什么？各自的作用？
**答：** Guides（前馈控制）和 Sensors（反馈控制）。Guides 在 Agent 行动前引导它，提高首次成功率，如 AGENTS.md、编码规范。Sensors 在 Agent 行动后观察结果并帮助自我纠正，如 linter、测试、AI 代码审查。只有反馈则 Agent 反复犯同样的错，只有前馈则不知道规则是否有效，两者缺一不可。

### Q3: 计算型控制和推理型控制有什么区别？各举两个例子。
**答：** 计算型控制是确定性的、CPU 执行的，速度快且结果可靠，如 linter、类型检查、单元测试、ArchUnit 架构测试。推理型控制是非确定性的、GPU 执行的，更慢更贵但能处理语义问题，如 AI 代码审查（LLM-as-judge）、基于 Skill 文件的编码规范指导。计算型适合每次提交都跑，推理型适合关键节点使用。

### Q4: 为什么说"模型决定能力上限，Harness 决定实际表现"？
**答：** LangChain 的 OPENDEV 实验证明：同一个模型，仅通过改进 harness（更好的任务结构、验证机制、编排方式），Terminal Bench 2.0 得分从 52.8% 提升到 66.5%。没换模型，没改 prompt，只改了框架。同时 METR 研究显示，没有好的 harness 时，AI 辅助开发反而慢了 19%。

### Q5: AGENTS.md 应该怎么写？为什么不能写成大而全的指令手册？
**答：** 应该当目录用，约 100 行的地图，指向 docs/ 目录中的深层文档。原因：上下文是稀缺资源，巨大的指令文件会挤掉任务和代码；当所有东西都"重要"时什么都不重要；单体文件会迅速腐烂且难以验证。正确做法是渐进式披露——Agent 从小入口开始，按需深入。

### Q6: 什么是"转向循环"（Steering Loop）？它体现了怎样的心态转变？
**答：** 转向循环是指：Agent 犯错 → 分析原因 → 改进 guides/sensors → 降低未来出错概率 → 循环。心态转变是从"这个模型太蠢了"变为"我的系统允许了这种失败模式"。不再等模型升级，而是主动改造环境。这个概念由 Mitchell Hashimoto 在 2026 年 2 月提出。

### Q7: 什么是"可驾驭性"（Harnessability）？哪些因素影响它？
**答：** 可驾驭性指代码库对 harness 的友好程度。影响因素：强类型语言天然有类型检查传感器；清晰的模块边界可以定义架构约束规则；成熟框架隐式提高 Agent 成功率；"无聊"的技术可组合性好、训练数据充分。新项目可以从第一天内建可驾驭性，遗留项目最需要 harness 的地方恰恰最难构建。

### Q8: Harness 模式在什么场景下效果最好？什么场景下会失效？
**答：** 效果取决于任务的可验证性层级。L1 形式化验证（数学证明）和 L2 可测试（代码/CI）效果极好，反馈快速、客观、可自动化。L3 基于规则（合规）可用但需人工审查。L4 启发式（客服、研究）效果有限。L5 不可验证（创意、伦理决策）模式基本失效。目前最亮眼的成果都在 L1-L2 层级。

### Q9: 人类开发者的"隐式 harness"是什么？为什么 Agent 无法替代？
**答：** 人类自带隐式 harness：内化的编码规范、对复杂性的认知痛感、commit 署名带来的社会责任感、组织目标的理解、人类节奏创造的思考空间。Agent 没有审美厌恶、没有"我们这里不这么做"的直觉、不知道哪个规范是承重墙。Harness Engineering 是试图外化这些隐式经验，但永远只能做到一定程度。

### Q10: 当前 Harness Engineering 领域有哪些未解决的核心问题？
**答：** ①Harness 一致性——guides 和 sensors 如何保持同步不矛盾；②冲突信号处理——指令和反馈指向不同方向时 Agent 能否合理权衡；③传感器有效性——从不触发是质量高还是检测不够；④缺少 harness 覆盖率评估机制；⑤行为 Harness——功能正确性验证仍高度依赖人工；⑥Harness 债务——harness 本身也会有 bug 和漂移。

## 总结与参考

Harness Engineering 的核心洞察：当模型能力足够时，瓶颈不在模型本身，而在模型周围的系统。构建好的 harness 不是一次性配置，而是一项持续的工程实践——通过转向循环不断迭代改进，让 Agent 在约束中高效工作。

### 参考来源

1. [Martin Fowler - Harness Engineering for Coding Agent Users](https://martinfowler.com/articles/harness-engineering.html)（2026.04，最系统全面）
2. [On-Site Reliability - Harness Engineering](https://onsitereliability.com/Harness-Engineering/)（2026.03，OpenAI 实践案例）
3. [Infralovers - Why the Frame Matters More Than the Model](https://www.infralovers.com/blog/2026-03-13-harness-engineering-rahmen-wichtiger-als-modell/)（2026.03，数据驱动分析）
4. [Louis Bouchard - The Missing Layer Behind AI Agents](https://www.louisbouchard.ai/harness-engineering/)（2026.03，通俗入门）
5. [Anthropic - Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
6. [LangChain - The Anatomy of an Agent Harness](https://blog.langchain.com/the-anatomy-of-an-agent-harness/)

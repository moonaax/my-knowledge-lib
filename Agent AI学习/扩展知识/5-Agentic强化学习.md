# Agentic 强化学习

> 将 LLM 作为可学习策略，嵌入智能体的感知-决策-执行循环，通过强化学习优化多步任务表现——从"对话助手"进化为"自主智能体"。

相关主题：[[../基础概念/2-LLM大语言模型基础]] | [[../Agent设计模式/1-ReAct与推理策略]]

---

## 一、LLM 训练全景图

一个强大的 LLM（如 GPT、Claude、Qwen）的诞生，通常经历**预训练**和**后训练**两个主要阶段。

### 1.1 预训练阶段

预训练的目标是让模型学习语言规律和世界知识。使用海量文本数据（TB 级别），通过**因果语言建模**（Causal Language Modeling）进行自监督学习：

$$
\mathcal{L}_{\text{pretrain}} = -\sum_{t=1}^{T} \log P(x_t | x_1, x_2, ..., x_{t-1}; \theta)
$$

给定文本序列，模型预测下一个词，最小化负对数似然。通过海量文本训练，模型学会语法规则、语义知识、世界知识和基础推理能力。

### 1.2 后训练阶段

预训练模型只是"预测下一个词"的模型，不知道如何遵循指令、拒绝不当请求、以对话方式交互。后训练包含三个步骤：

**第一步：监督微调（SFT）**。用 (prompt, completion) 对训练模型遵循指令：

$$
\mathcal{L}_{\text{SFT}} = -\sum_{i=1}^{N} \log P(y_i | x_i; \theta)
$$

**第二步：奖励建模（RM）**。用偏好对比数据（chosen vs rejected）训练奖励模型：

$$
\mathcal{L}_{\text{RM}} = -\mathbb{E}_{(x, y_w, y_l)} [\log \sigma(r_\phi(x, y_w) - r_\phi(x, y_l))]
$$

**第三步：强化学习微调（RLHF）**。用 PPO 算法优化语言模型：

$$
J_{\text{PPO}} = \mathbb{E}_{x, y \sim \pi_\theta} [r_\phi(x, y)] - \beta \cdot D_{KL}(\pi_\theta || \pi_{\text{ref}})
$$

> 🔑 **RLHF vs RLAIF**：传统 RLHF 需要大量人工标注偏好数据，成本高昂。RLAIF 用强大的 AI 模型（如 GPT-4）替代人类标注员，效果接近甚至超过 RLHF，同时成本大幅降低。

## 二、从 RLHF 到 Agentic RL

### 2.1 RLHF 的局限

传统后训练（PBRFT，Preference-Based Reinforcement Fine-Tuning）主要关注**单轮对话**的质量优化：给定一个问题，模型生成一个回答，根据质量直接给分。这种方式适合优化对话助手，但对需要多步推理、工具使用、长期规划的智能体任务力不从心。

### 2.2 Agentic RL 的形式化

**Agentic RL** 将 LLM 视为可学习策略，嵌入顺序决策循环。强化学习基于**马尔可夫决策过程（MDP）**框架形式化，由五元组 $(S, A, P, R, \gamma)$ 定义。

PBRFT 与 Agentic RL 的核心差异：

| 维度 | PBRFT | Agentic RL |
|------|-------|------------|
| 状态 | $s_0 = \text{prompt}$（静态） | $s_t = (\text{prompt}, o_1, ..., o_t)$（动态演化） |
| 行动 | 纯文本生成 | $a_t \in \{a_t^{\text{text}}, a_t^{\text{tool}}\}$（文本+工具） |
| 转移 | 无状态转移 | $s_{t+1} \sim P(s_{t+1}|s_t, a_t)$ |
| 奖励 | 单步奖励 $r(s_0, y)$ | 多步累积 $R = \sum_{t=0}^{T} \gamma^t r(s_t, a_t)$ |
| 目标 | 最大化单步期望奖励 | 最大化累积折扣奖励 |

这种转变使得 LLM 从"对话助手"进化为"自主智能体"。

### 2.3 六大核心能力

Agentic RL 旨在赋予 LLM 智能体六大核心能力：

1. **推理（Reasoning）**：通过试错学习有效推理策略，发现训练数据中没有的推理路径
2. **工具使用（Tool Use）**：学会何时需要使用工具、选择哪个工具、如何组合多个工具
3. **记忆（Memory）**：学习记忆管理策略——哪些信息值得记住、何时更新、何时删除
4. **规划（Planning）**：通过试错发现有效行动序列，权衡短期和长期收益
5. **自我改进（Self-Improvement）**：识别错误、分析失败原因、调整策略
6. **感知（Perception）**：理解多模态信息，提升视觉推理能力

## 三、GRPO 算法

### 3.1 从 PPO 到 GRPO

PPO 是最经典的强化学习算法，但在 LLM 训练中存在问题：需要训练 Value Model，增加复杂度和显存占用；需同时维护四个模型（Policy、Reference、Value、Reward）；训练不稳定，容易奖励崩塌。

**GRPO（Group Relative Policy Optimization）** 是专为 LLM 设计的简化 PPO 变体，核心思想是**不需要 Value Model**，用组内相对奖励代替绝对奖励。

### 3.2 GRPO 的数学原理

PPO 的优势函数需要 Value Model 估计：

$$
A(s,a) = r(s,a) + \gamma V(s') - V(s)
$$

GRPO 用**组内相对奖励**替代优势函数：

$$
J_{\text{GRPO}}(\theta) = \mathbb{E}_{s,a \sim \pi_\theta} \left[ \frac{\pi_\theta(a|s)}{\pi_{\text{ref}}(a|s)} \cdot (r(s,a) - \bar{r}_{\text{group}}) \right] - \beta \cdot D_{KL}(\pi_\theta || \pi_{\text{ref}})
$$

其中 $\bar{r}_{\text{group}}$ 是组内平均奖励，$\beta$ 是 KL 散度惩罚系数。

### 3.3 GRPO 训练循环

GRPO 的每轮训练包含以下步骤：

1. **采样**：对每个问题，用当前策略生成多个答案（如 4-8 个），构成一个"组"
2. **奖励计算**：对每个答案计算奖励 $r_i$
3. **相对奖励**：计算 $\hat{r}_i = r_i - \bar{r}_{\text{group}}$，减少奖励方差
4. **策略更新**：用相对奖励更新策略，同时添加 KL 散度惩罚

> 🔑 **关键对比**：PPO 需要 4 个模型（Policy + Reference + Value + Reward），GRPO 只需 2 个（Policy + Reference）。GRPO 省去 Value Model 训练，更简单、更稳定、显存占用更低。

### 3.4 KL 散度惩罚

KL 散度惩罚防止策略偏离参考模型太远：

$$
D_{KL}(\pi_\theta || \pi_{\text{ref}}) = \mathbb{E}_{s,a \sim \pi_\theta} \left[ \log \frac{\pi_\theta(a|s)}{\pi_{\text{ref}}(a|s)} \right]
$$

`kl_coef`（$\beta$）的选择：太小（0.01）策略可能偏离太远导致格式混乱；太大（0.5）策略更新受限难以超越 SFT。建议 0.05-0.1。

## 四、LoRA 低秩微调

### 4.1 核心思想

全量微调需要大量显存（0.6B 参数约需 12GB FP16），对更大模型几乎不可能在消费级 GPU 上进行。

**LoRA（Low-Rank Adaptation）** 的核心假设：模型微调时的参数变化可以用**低秩矩阵**表示。将权重变化分解为两个低秩矩阵的乘积：

$$
\Delta W = BA, \quad B \in \mathbb{R}^{d \times r}, \quad A \in \mathbb{R}^{r \times k}, \quad r \ll \min(d, k)
$$

前向传播时：$h = Wx + BAx$，原模型参数 $W$ 冻结，只训练 $B$ 和 $A$。

### 4.2 参数量对比

以 $d=4096, k=4096, r=8$ 为例：

- 原模型参数量：$4096 \times 4096 = 16,777,216$
- LoRA 参数量：$8 \times (4096 + 4096) = 65,536$
- **参数量减少 256 倍**

### 4.3 关键超参数

| 参数 | 说明 | 典型值 |
|------|------|--------|
| rank（r） | LoRA 矩阵的秩，越大表达能力越强 | 4-64，默认 8 |
| alpha（$\alpha$） | 缩放因子，实际更新为 $\Delta W = \frac{\alpha}{r} BA$ | 通常等于 rank 的 2 倍 |
| target_modules | 应用 LoRA 的层 | q_proj, k_proj, v_proj, o_proj |

> 🔑 **适用场景**：LoRA 显存占用大幅降低、训练速度更快、易于部署、防止过拟合。但效果通常比全量微调略差。建议始终开启 LoRA，除非有充足显存。

## 五、端到端训练 Pipeline

### 5.1 数据准备：GSM8K 三种格式

GSM8K 是小学数学应用题数据集（7473 训练 + 1319 测试），需要 2-8 步推理。数据需转换为三种格式：

| 格式 | 用途 | 特点 |
|------|------|------|
| 原始格式 | 人类阅读 | question + answer（含解题步骤） |
| SFT 格式 | 监督微调 | prompt（对话模板）+ completion（完整解答） |
| RL 格式 | 强化学习 | prompt + ground_truth（仅最终答案） |

RL 格式不提供解题过程，迫使模型学会自主推理，而不是简单记忆答案。

### 5.2 SFT 阶段

SFT 是从预训练模型到强化学习的**桥梁**。预训练模型输出冗长、缺乏结构、没有明确答案；SFT 后模型输出结构清晰（Step 1/Step 2/Final Answer）、推理正确、格式统一。

SFT 的作用：
- 学习输出格式（如何组织答案）
- 学习推理模式（如何分解问题）
- 建立基线能力（为 RL 提供合理起点）
- 减少探索空间（RL 不需从零开始）

### 5.3 三种奖励函数

**准确率奖励**：答案正确得 1，错误得 0。简单直接但奖励稀疏。

$$
r_{\text{acc}}(a, a^*) = \begin{cases} 1 & \text{if } a = a^* \\ 0 & \text{otherwise} \end{cases}
$$

**长度惩罚**：鼓励简洁回答，在正确答案基础上扣除过长惩罚。

$$
r_{\text{length}} = r_{\text{acc}} - \alpha \cdot \max(0, l - l_{\text{target}})
$$

**步骤奖励**：鼓励清晰推理步骤，提高可解释性。

$$
r_{\text{step}} = r_{\text{acc}} + \beta \cdot s
$$

实际应用中通常组合使用：$r = r_{\text{acc}} - \alpha \cdot \max(0, l - l_{\text{target}}) + \beta \cdot s$

### 5.4 GRPO 训练阶段

GRPO 训练的关键参数：
- **num_generations**（4-8）：每个问题生成的答案数，用于计算组内相对奖励
- **learning_rate**（1e-5 ~ 5e-5）：比 SFT 更小，避免偏离太远
- **kl_coef**（0.05-0.1）：KL 散度惩罚系数
- **temperature**（0.7-1.0）：保持一定探索性

训练监控关键指标：平均奖励（应逐渐上升）、KL 散度（保持 0.01-0.1）、准确率（应逐渐提升）。

### 5.5 评估指标体系

| 类别 | 指标 | 说明 |
|------|------|------|
| 准确性 | Accuracy | 答案完全正确的比例 |
| 准确性 | Accuracy@K | K 次采样中至少一次正确的比例 |
| 效率 | 平均长度 | 更短 = 更低推理成本 |
| 效率 | 推理步骤数 | 2-5 步为合理范围 |
| 质量 | 格式正确率 | 是否包含 Step/Final Answer 标记 |
| 质量 | 推理连贯性 | 步骤间逻辑是否连贯 |

典型效果：Qwen3-0.6B 预训练模型准确率较低，SFT 后达 40-50%，GRPO 后可提升至 60-70%。

## 六、分布式训练

### 6.1 方案选择

| 场景 | 方案 | 说明 |
|------|------|------|
| 单机多卡（2-8 卡） | DDP | 每个 GPU 持有完整模型副本，数据分割 |
| 大模型（>7B） | DeepSpeed ZeRO-2 | 分片优化器状态和梯度 |
| 多节点集群 | DeepSpeed ZeRO-3 | 分片优化器状态、梯度和模型参数 |

### 6.2 DDP（数据并行）

最简单的分布式方案，训练代码无需修改，通过 `accelerate launch` 启动即可。

### 6.3 DeepSpeed ZeRO

- **ZeRO-2**：分片优化器状态和梯度，不卸载到 CPU
- **ZeRO-3**：额外分片模型参数，可卸载到 CPU，支持更大模型

分布式训练时总 batch size = `per_device_batch_size x num_gpus x gradient_accumulation_steps`，学习率按线性缩放规则调整：$lr_{new} = lr_{base} \times \sqrt{B_{new} / B_{base}}$。

## 面试题精选

**1. 什么是 Agentic RL？它与传统 RLHF 有什么区别？**

Agentic RL 将 LLM 视为可学习策略，嵌入多步序贯决策循环。与传统 RLHF 的核心区别：状态空间从静态 prompt 扩展为动态演化的环境状态；行动空间从纯文本扩展为文本+工具；奖励从单步评估扩展为长期累积回报；优化目标从响应质量扩展为任务完成度。

**2. GRPO 相比 PPO 的优势是什么？为什么不需要 Value Model？**

GRPO 用组内相对奖励（$r_i - \bar{r}_{\text{group}}$）替代 PPO 的优势函数 $A(s,a) = r + \gamma V(s') - V(s)$。组内相对奖励天然消除了基线偏差，无需 Value Model 估计状态价值。优势：训练流程简化（只需 Policy + Reference 两个模型）、显存占用降低、训练更稳定。

**3. LoRA 的核心原理是什么？为什么参数量能减少 256 倍？**

LoRA 假设微调时权重变化 $\Delta W$ 是低秩的，将其分解为 $\Delta W = BA$（$B \in \mathbb{R}^{d \times r}, A \in \mathbb{R}^{r \times k}$）。当 $r=8, d=k=4096$ 时，原参数 16M 降为 65K，减少 256 倍。原模型参数冻结，只训练低秩矩阵。

**4. 为什么 SFT 是强化学习的必要基础？直接用 RL 训练预训练模型会怎样？**

预训练模型不知道任务格式、没有结构化输出能力、无法提取答案计算奖励。SFT 教会模型输出格式（Step 1/Final Answer）、建立基线推理能力、缩小 RL 的探索空间。没有 SFT，RL 无法获得有效的奖励信号，训练会失败。

**5. 在 GRPO 训练中，KL 散度惩罚的作用是什么？系数过大或过小会怎样？**

KL 散度惩罚 $-\beta \cdot D_{KL}(\pi_\theta || \pi_{\text{ref}})$ 防止策略偏离参考模型（SFT 模型）太远，避免"遗忘"SFT 阶段学到的知识。系数太小（0.01）策略可能偏离太远导致格式混乱；太大（0.5）策略更新受限，难以超越 SFT。建议 0.05-0.1。

**6. 如何设计一个好的奖励函数？有哪些常见陷阱？**

好的奖励函数应：清楚定义成功、提供梯度信号、方差不过大、易于组合。常见陷阱：奖励稀疏（只有最终答案有反馈）、奖励欺骗（智能体找到"作弊"方式获高奖励）、目标矛盾（多个目标相互冲突）。解决方案包括组合多种奖励、添加长度/格式惩罚、使用相对奖励。

**7. 解释 Agentic RL 中"六大核心能力"及其与强化学习训练的关系。**

六大能力：推理（RL 试错学习推理策略）、工具使用（RL 学会选择和组合工具）、记忆（RL 学习信息管理策略）、规划（RL 发现有效行动序列）、自我改进（RL 从错误中学习调整策略）、感知（RL 提升多模态理解能力）。RL 的核心优势是通过探索发现训练数据中没有的策略，实现超越模仿的自主学习。

**8. 分布式训练中 ZeRO-2 和 ZeRO-3 有什么区别？各自适用什么场景？**

ZeRO-2 分片优化器状态和梯度，不卸载到 CPU，适合中等规模模型。ZeRO-3 额外分片模型参数并可卸载到 CPU，支持更大模型但通信开销更大。选择建议：单机多卡用 ZeRO-2，大模型（>7B）或多节点用 ZeRO-3。

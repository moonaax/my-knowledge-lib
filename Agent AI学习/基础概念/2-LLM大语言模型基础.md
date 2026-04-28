# LLM 大语言模型基础

## 1. 什么是 LLM

大语言模型（Large Language Model）是基于 Transformer 架构、在海量文本数据上训练的深度学习模型。它是 AI Agent 的"大脑"，提供理解、推理和生成能力。

## 2. Transformer 架构核心

### 2.1 整体结构

````
输入文本 → Tokenizer → Embedding → [Transformer Blocks × N] → Output
                                          │
                                    ┌─────┴─────┐
                                    │ Self-Attention │
                                    │ Feed-Forward   │
                                    │ Layer Norm     │
                                    └───────────────┘
````
### 2.2 Self-Attention 机制

Self-Attention 是 Transformer 的核心，让模型理解词与词之间的关系：

````
Q (Query)  = Input × W_q    # 我在找什么
K (Key)    = Input × W_k    # 我能提供什么
V (Value)  = Input × W_v    # 我的实际内容

Attention(Q, K, V) = softmax(QK^T / √d_k) × V
````
直觉理解：
- 处理"他把苹果放在桌子上，然后吃了**它**"时
- "它"的 Query 会与"苹果"的 Key 产生高注意力分数
- 模型因此理解"它"指代"苹果"

### 2.3 关键参数

| 参数             | 含义     | 影响              |
| -------------- | ------ | --------------- |
| Temperature    | 输出随机性  | 0=确定性，1=创造性     |
| Top-p          | 核采样阈值  | 控制候选词范围         |
| Max Tokens     | 最大输出长度 | 控制回复长度          |
| Context Window | 上下文窗口  | 单次能处理的总 Token 数 |
| Stop Sequences | 停止标记   | 控制生成何时结束        |

#### 采样参数数学原理

LLM 生成下一个 Token 时，模型输出的是一个概率分布。采样参数通过调整这个分布来控制输出特性。

**Temperature**：引入温度系数 T > 0，改写 Softmax 公式：

```
p_i(T) = exp(z_i / T) / Σ exp(z_j / T)
```

- T → 0：分布极度陡峭，趋向贪心解码（确定性最高）
- T = 1：原始分布不变
- T > 1：分布平坦，低概率项权重提升，输出更随机

**Top-k**：只保留概率最高的 k 个 Token，其余概率置零后重新归一化。k=1 时退化为贪心采样。

**Top-p（核采样）**：按概率从高到低累加，直到累积概率 ≥ p，取这个最小集合作为候选。相比 Top-k，Top-p 能自适应分布的"长尾"特性。

> 🔑 **三参数协同工作**：优先级为 Temperature → Top-k → Top-p。Temperature 调整整体分布，Top-k 先截断候选集，Top-p 再从其中精选。通常 Top-k 和 Top-p 二选一即可。若 Temperature=0，则 Top-k/Top-p 无效（最高概率 Token 必被选中）。

**Agent 场景推荐**：
- 工具调用 / 推理：Temperature=0（确定性输出，避免参数生成错误）
- 日常对话：Temperature=0.3~0.7（平衡自然度和准确性）
- 创意生成：Temperature=0.7~1.2（发散思维）

## 3. 主流模型对比

### 3.1 闭源模型（2026 年 4 月）

| 模型                | 厂商        | 上下文窗口    | API 价格（输入/百万 Token） | 特点                              |
| ----------------- | --------- | -------- | ------------------- | ------------------------------- |
| GPT-5.4 Pro       | OpenAI    | 128K     | $2.50               | 综合能力顶级，与 Gemini 3.1 Pro 并列榜首    |
| Claude Opus 4.6   | Anthropic | 200K     | $5.00               | 代码能力最强（SWE-bench 领先），深度推理       |
| Claude Sonnet 4.6 | Anthropic | 1M（beta） | $3.00               | 实际工作效率最高，GitHub Copilot 默认模型    |
| Gemini 3.1 Pro    | Google    | 1M       | $2.00               | 13/16 基准测试领先，性价比最高的旗舰模型         |
| Grok 4.20         | xAI       | 128K     | 未公开                 | 四 Agent 并行架构，实时 X 数据，金融交易唯一盈利模型 |
| 通义千问 Max          | 阿里        | 128K     | ¥0.02               | 中文生态完善                          |
| 文心一言 4.0          | 百度        | 128K     | ¥0.02               | 中文理解优秀                          |

> 💡 GPT-5.5（代号 Spud）已完成预训练，预计 Q2 发布。Claude Mythos 因网络安全风险不会公开发布。

### 3.2 开源模型（2026 年 4 月）

| 模型 | 参数量 | 上下文窗口 | 许可证 | 特点 |
|------|--------|-----------|--------|------|
| Gemma 4 | 多尺寸 | 256K | Apache 2.0 | Google 出品，前沿级性能，可跑在手机上，完全开源 |
| Llama 4 Maverick | 400B (128 experts) | 10M | Meta 许可 | 最长上下文窗口，需要大量 GPU |
| Llama 4 Scout | 109B (16 experts) | 1M | Meta 许可 | Maverick 的轻量版，更易部署 |
| DeepSeek V3.2 | MoE | 128K | 开源 | $0.27/百万 Token，性价比炸裂 |
| Qwen 3.6 Plus | MoE | 1M | 开源 | 阿里最新，强 Agent 能力，中文最优 |
| MiniMax M2.7 | - | - | 开源 | "自进化"训练，编码接近 Claude Opus 水平 |

> 💡 2026 年开源模型已不再是"妥协之选"——Gemma 4 和 Llama 4 在多项基准上与闭源模型持平。

### 3.3 选型建议（2026 年 4 月）

````
综合能力最强   → Gemini 3.1 Pro（性价比最高）/ GPT-5.4 Pro
代码/编程     → Claude Sonnet 4.6（GitHub Copilot 默认）/ Claude Opus 4.6
超长上下文    → Llama 4 Maverick（10M）/ Gemini 3.1 Pro（1M）
本地部署      → Gemma 4（Apache 2.0，多尺寸）/ Qwen 3.6 Plus
中文优化      → Qwen 3.6 Plus / 通义千问 Max
预算有限      → DeepSeek V3.2（$0.27/M tokens，比 Claude 便宜 18 倍）
实时数据      → Grok 4.20（接入 X 实时数据）
隐私优先      → Gemma 4（Apache 2.0，无限制）
````
## 4. Token 与计费

### 4.1 Token 概念

````python
# Token 不等于字符，也不等于单词
"Hello, world!"     → ["Hello", ",", " world", "!"]     = 4 tokens
"你好世界"           → ["你", "好", "世", "界"]           = 4 tokens
"ChatGPT is great"  → ["Chat", "G", "PT", " is", " great"] = 5 tokens

# 经验法则
# 英文: 1 token ≈ 4 个字符 ≈ 0.75 个单词
# 中文: 1 token ≈ 1-2 个汉字
````
### 4.2 计费模型

````
总费用 = 输入 Token 数 × 输入单价 + 输出 Token 数 × 输出单价

示例 (GPT-4o):
  输入: $2.50 / 1M tokens
  输出: $10.00 / 1M tokens

  一次 Agent 调用（输入 2000 tokens，输出 500 tokens）:
  费用 = 2000/1M × $2.50 + 500/1M × $10.00 = $0.01
````

### 4.3 分词（Tokenization）

在将文本喂给 LLM 之前，必须先将其切分成 Token。这个过程叫**分词（Tokenization）**，分词器（Tokenizer）定义了切分规则。

#### 为什么不能按词或按字符分词？

- **按词分词**：词表会爆炸（英语数十万词），且无法处理未登录词（OOV）。"look"、"looks"、"looking" 被视为三个无关的词。
- **按字符分词**：词表很小，但单个字符缺乏语义，模型学习效率低下。

现代 LLM 采用**子词分词（Subword Tokenization）**——常见词保留为完整 Token，罕见词拆分成有意义的子词片段（如 "Tokenization" → "Token" + "ization"）。

#### BPE 算法原理

**字节对编码（Byte-Pair Encoding, BPE）** 是最主流的子词分词算法，GPT 系列模型均采用此算法。核心思想是一个"贪心合并"过程：

1. **初始化**：词表为所有基本字符
2. **迭代合并**：统计所有相邻词元对的频率，将最高频的一对合并为新词元加入词表
3. **重复**：直到词表大小达到预设阈值

```python
# BPE 算法简化演示
# 假设语料库: {"hug": 1, "pug": 1, "pun": 1, "bun": 1}
# 初始词表: {h, u, g, p, n, b}
#
# 第1次合并: 统计发现 u,g 相邻频率最高 → 合并为 "ug"
# 第2次合并: ug,</w> 频率最高 → 合并为 "ug</w>"
# 第3次合并: u,n 频率最高 → 合并为 "un"
# ...直到词表达到目标大小
#
# 推理时，未见过的词 "bug" 会被拆分为 ['b', 'ug']
```

其他主流分词算法：
- **WordPiece**：BERT 采用，合并标准是"最大化语料库语言模型概率"而非最高频率
- **SentencePiece**：Llama 系列采用，将空格也视为普通字符，分词完全可逆，不依赖特定语言

> 🔑 **分词对开发者的影响**
> - **上下文窗口管理**：窗口大小以 Token 数计算。同样内容，中文通常比英文消耗更多 Token
> - **API 成本**：按 Token 计费，了解分词规则有助于预估成本
> - **模型表现异常**：`2+2`（无空格）可能被分为一个不常见的 Token，导致模型计算出错；首字母大小写不同可能导致完全不同的分词结果
## 5. API 调用实践

### 5.1 OpenAI 风格 API

````python
from openai import OpenAI

client = OpenAI(api_key="your-api-key")

# 基础对话
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "你是一个有帮助的助手"},
        {"role": "user", "content": "什么是 AI Agent?"}
    ],
    temperature=0.7,
    max_tokens=1000
)

print(response.choices[0].message.content)
````
### 5.2 流式输出

````python
# 流式输出 - 适合实时展示
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "解释 Transformer"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
````
### 5.3 多轮对话

````python
# 维护对话历史
conversation = [
    {"role": "system", "content": "你是一个 Python 专家"}
]

def chat(user_input: str) -> str:
    conversation.append({"role": "user", "content": user_input})
    
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=conversation
    )
    
    reply = response.choices[0].message.content
    conversation.append({"role": "assistant", "content": reply})
    return reply

# 使用
chat("如何读取 JSON 文件?")
chat("如果文件很大怎么办?")  # 模型记得上一轮的上下文
````
### 5.4 使用国产模型 API

````python
# 通义千问 - 兼容 OpenAI 格式
client = OpenAI(
    api_key="your-dashscope-key",
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
)

response = client.chat.completions.create(
    model="qwen-max",
    messages=[{"role": "user", "content": "你好"}]
)

# DeepSeek - 同样兼容 OpenAI 格式
client = OpenAI(
    api_key="your-deepseek-key",
    base_url="https://api.deepseek.com"
)
````
## 6. Embedding 模型

Embedding 将文本转为向量，是 RAG 和语义搜索的基础：

````python
# 生成文本向量
response = client.embeddings.create(
    model="text-embedding-3-small",
    input="AI Agent 是什么"
)

vector = response.data[0].embedding  # 1536 维向量
print(f"向量维度: {len(vector)}")

# 计算相似度
import numpy as np

def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

v1 = get_embedding("AI Agent 是什么")
v2 = get_embedding("什么是智能体")
v3 = get_embedding("今天天气怎么样")

print(cosine_similarity(v1, v2))  # ~0.92 高相似度
print(cosine_similarity(v1, v3))  # ~0.45 低相似度
````
### 主流 Embedding 模型

| 模型 | 维度 | 特点 |
|------|------|------|
| text-embedding-3-small | 1536 | OpenAI，性价比高 |
| text-embedding-3-large | 3072 | OpenAI，精度最高 |
| bge-large-zh | 1024 | 智源，中文最佳之一 |
| m3e-base | 768 | 开源，中文友好 |
| jina-embeddings-v2 | 768 | 支持 8K 长文本 |

## 7. 模型能力边界

### 模型擅长的
- ✅ 文本理解和生成
- ✅ 代码编写和解释
- ✅ 逻辑推理（简单到中等）
- ✅ 翻译和摘要
- ✅ 格式转换和数据提取

### 模型不擅长的
- ❌ 精确数学计算（需要工具辅助）
- ❌ 实时信息获取（需要搜索工具）
- ❌ 100% 事实准确性（存在幻觉）
- ❌ 长期记忆（受上下文窗口限制）
- ❌ 多模态感知（需要专门模型）

> **这正是 Agent 存在的意义**：通过工具调用弥补 LLM 的不足，让 LLM 专注于推理和决策。

## 8. 缩放法则与涌现能力

理解 LLM 为何如此强大，需要了解两个关键概念：缩放法则（Scaling Laws）和能力涌现（Emergence）。

### 8.1 缩放法则

研究发现，在对数-对数坐标系下，模型性能（以 Loss 衡量）与**参数量、数据量、计算量**三个因素都呈现平滑的幂律关系。只要按比例增加这三个要素，性能就会可预测地提升。

**Chinchilla 定律（DeepMind, 2022）**：在给定计算预算下，模型参数量和训练数据量存在最优配比——最优模型应该比通常认为的更小，但需要用更多数据训练。一个 700 亿参数的 Chinchilla 模型用 4 倍于 GPT-3 的数据训练，性能反而超越了后者。

> 🔑 Chinchilla 定律纠正了"越大越好"的认知，强调了数据效率的重要性。Llama 系列等高效模型的设计都受此启发。

### 8.2 能力涌现

**涌现（Emergence）** 是指当模型规模达到一定阈值后，突然展现出小规模模型中完全不存在的新能力。例如：

- **思维链推理（Chain-of-Thought）**：参数达到数百亿后才显著出现
- **指令遵循（Instruction Following）**
- **多步推理、代码生成**

这种现象表明，大模型不只是记忆和复述，而是在学习过程中形成了更深层次的抽象和推理能力。对 Agent 开发者而言，**选择足够大规模的模型是实现复杂自主决策和规划能力的前提**。

### 8.3 模型幻觉分类学

**幻觉（Hallucination）** 是 LLM 生成看似合理但不真实内容的现象。笼统地谈"幻觉"不够精确，理解其分类有助于针对性地缓解。

| 幻觉类型 | 定义 | 示例 |
|---------|------|------|
| **事实性幻觉** | 生成与现实世界事实不符的内容 | 声称"爱因斯坦在 1921 年获得诺贝尔化学奖"（实际是物理学奖） |
| **忠实性幻觉** | 生成内容未能忠实反映源文本 | 摘要中添加了原文没有的信息 |
| **内在幻觉** | 生成内容与提供的输入直接矛盾 | 用户说"公司收入 100 万"，模型回答"公司收入 500 万" |

**幻觉成因**：
- 训练数据中包含错误或矛盾信息
- 自回归生成机制——模型只预测"最可能的下一个词"，没有内置事实核查模块
- 复杂推理时逻辑链出错，"编造"出错误结论

**缓解方法体系**：

| 层级 | 方法 | 说明 |
|------|------|------|
| 数据层 | 高质量数据清洗、RLHF | 从源头减少幻觉 |
| 模型层 | 不确定性表达、新架构探索 | 让模型知道自己"不确定" |
| 推理层 | **RAG** | 从外部知识库检索事实后生成，最实用 |
| 推理层 | 多步推理与自我验证 | 引导模型逐步检查 |
| 推理层 | 工具调用 | 用搜索引擎、计算器等外部工具获取实时信息或精确计算 |

> 🔑 **RAG 是当前缓解幻觉最有效且最易落地的方案**——先检索相关事实，再基于事实生成回答。这也是本项目 `knowledge_search` 工具的核心思想。

### 8.4 思维链（Chain-of-Thought）提示

**思维链（CoT）** 是一种通过引导模型"逐步思考"来提升推理能力的提示技巧。

**核心思想**：不直接要答案，而是要求模型展示推理过程。只需在提示中加入一句引导语，如"请逐步思考"或"Let's think step by step"。

````
# 普通提示
Q: 一个篮球队 80 场赢了 60%，接下来 15 场赢了 12 场，总胜率？
A: 63.16%  （模型可能直接猜一个数字）

# CoT 提示
Q: 一个篮球队 80 场赢了 60%，接下来 15 场赢了 12 场，总胜率？请一步一步思考。
A: 第一步：80 × 60% = 48 场
   第二步：总比赛 80+15=95 场，总胜利 48+12=60 场
   第三步：60/95 ≈ 63.16%
   所以总胜率约为 63.16%。
````

**CoT 的价值**：
- 显著提升数学、逻辑推理等复杂任务的准确率
- 推理过程可检查、可纠错，提升可信度
- 是能力涌现的典型案例——小模型使用 CoT 效果不明显，大模型（数百亿参数以上）才能有效利用

**CoT 变体**：
- **Zero-shot CoT**：仅加一句"Let's think step by step"
- **Few-shot CoT**：提供带推理过程的示例
- **Auto-CoT**：让模型自动生成推理示例

> 🔑 Agent 场景下，CoT 尤其重要——当 Agent 面对复杂任务需要规划和推理时，使用 CoT 提示能显著提升决策质量。

## 9. 本地部署方案

````bash
# 使用 Ollama 本地运行模型
# 安装
curl -fsSL https://ollama.com/install.sh | sh

# 下载并运行模型
ollama run llama3:8b
ollama run qwen2.5:7b

# API 调用（兼容 OpenAI 格式）
curl http://localhost:11434/v1/chat/completions \
  -d '{
    "model": "llama3:8b",
    "messages": [{"role": "user", "content": "Hello"}]
  }'
````
````python
# Python 中使用本地模型
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # 任意值即可
)

response = client.chat.completions.create(
    model="llama3:8b",
    messages=[{"role": "user", "content": "你好"}]
)
````
---

## 面试题精选

### Q1: Transformer 中 Self-Attention 的计算公式是什么？为什么要除以 √d_k？
**答：** Attention(Q,K,V) = softmax(QK^T / √d_k) × V。除以 √d_k 是为了防止点积值过大导致 softmax 梯度消失，起到缩放稳定训练的作用。

### Q2: Temperature 参数对 LLM 输出有什么影响？Agent 场景下应该怎么设置？
**答：** Temperature 控制输出随机性，0 表示确定性输出，1 表示高创造性。Agent 场景下推荐设为 0 或接近 0，因为工具调用和推理需要确定性和一致性，避免随机性导致参数生成错误。

### Q3: 开源模型和闭源模型各有什么优劣？什么场景选哪种？
**答：** 闭源模型（GPT-5.4、Claude Opus 4.6、Gemini 3.1 Pro）能力最强但有数据隐私风险和 API 依赖；开源模型（Gemma 4、Llama 4、Qwen 3.6、DeepSeek V3.2）可本地部署、数据可控，且 2026 年已在多项基准上与闭源模型持平。涉及敏感数据或需要离线运行选开源（推荐 Gemma 4，Apache 2.0 无限制），追求最强能力和快速上线选闭源，预算有限选 DeepSeek V3.2（$0.27/M tokens）。

### Q4: Token 是什么？中英文的 Token 计算有什么区别？
**答：** Token 是 LLM 处理文本的基本单位，不等于字符或单词。英文约 1 token ≈ 4 字符 ≈ 0.75 个单词；中文约 1 token ≈ 1-2 个汉字。理解 Token 对成本估算和上下文窗口管理至关重要。

### Q5: Embedding 模型的作用是什么？如何评估两段文本的语义相似度？
**答：** Embedding 模型将文本转为高维向量，语义相近的文本在向量空间中距离更近。通常用余弦相似度计算两个向量的相似程度，值越接近 1 表示语义越相似。这是 RAG 语义检索的基础。

### Q6: 如何在本地部署一个开源 LLM？有哪些方案？
**答：** 最简单的方案是使用 Ollama，一行命令即可运行（ollama run llama4-scout）。它提供兼容 OpenAI 格式的 API，Python 代码只需改 base_url 即可切换。其他方案还有 vLLM（高性能推理）、llama.cpp（CPU 推理）等。2026 年推荐 Gemma 4（Apache 2.0，多尺寸可选，手机都能跑）或 Qwen 3.6 Plus（中文最优）。

### Q7: 什么是模型幻觉（Hallucination）？在 Agent 系统中如何缓解？
**答：** 幻觉是模型生成看似合理但不真实的内容。在 Agent 中可通过 RAG（基于检索的事实回答）、工具调用（用搜索引擎验证）、自我反思机制（让模型检查自己的输出）来缓解。

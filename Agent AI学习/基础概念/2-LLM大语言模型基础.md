# LLM 大语言模型基础

## 1. 什么是 LLM

大语言模型（Large Language Model）是基于 Transformer 架构、在海量文本数据上训练的深度学习模型。它是 AI Agent 的"大脑"，提供理解、推理和生成能力。

## 2. Transformer 架构核心

### 2.1 整体结构

```
输入文本 → Tokenizer → Embedding → [Transformer Blocks × N] → Output
                                          │
                                    ┌─────┴─────┐
                                    │ Self-Attention │
                                    │ Feed-Forward   │
                                    │ Layer Norm     │
                                    └───────────────┘
```

### 2.2 Self-Attention 机制

Self-Attention 是 Transformer 的核心，让模型理解词与词之间的关系：

```
Q (Query)  = Input × W_q    # 我在找什么
K (Key)    = Input × W_k    # 我能提供什么
V (Value)  = Input × W_v    # 我的实际内容

Attention(Q, K, V) = softmax(QK^T / √d_k) × V
```

直觉理解：
- 处理"他把苹果放在桌子上，然后吃了**它**"时
- "它"的 Query 会与"苹果"的 Key 产生高注意力分数
- 模型因此理解"它"指代"苹果"

### 2.3 关键参数

| 参数 | 含义 | 影响 |
|------|------|------|
| Temperature | 输出随机性 | 0=确定性，1=创造性 |
| Top-p | 核采样阈值 | 控制候选词范围 |
| Max Tokens | 最大输出长度 | 控制回复长度 |
| Context Window | 上下文窗口 | 单次能处理的总 Token 数 |
| Stop Sequences | 停止标记 | 控制生成何时结束 |

## 3. 主流模型对比

### 3.1 闭源模型

| 模型 | 厂商 | 上下文窗口 | 特点 |
|------|------|-----------|------|
| GPT-4o | OpenAI | 128K | 综合能力强，多模态 |
| Claude 3.5 Sonnet | Anthropic | 200K | 长文本理解优秀，代码能力强 |
| Gemini 1.5 Pro | Google | 1M | 超长上下文，多模态 |
| 文心一言 4.0 | 百度 | 128K | 中文理解优秀 |
| 通义千问 Max | 阿里 | 128K | 中文生态完善 |

### 3.2 开源模型

| 模型 | 参数量 | 特点 | 适用场景 |
|------|--------|------|---------|
| Llama 3 | 8B/70B | Meta 出品，社区活跃 | 通用场景 |
| Qwen 2.5 | 7B/72B | 阿里出品，中文优秀 | 中文场景 |
| Mistral | 7B/8x7B | 高效 MoE 架构 | 资源受限场景 |
| DeepSeek V2 | 236B(MoE) | 性价比极高 | 代码/推理 |
| Yi | 6B/34B | 零一万物，中英双语 | 双语场景 |

### 3.3 选型建议

```
需要最强能力 → GPT-4o / Claude 3.5 Sonnet
需要长上下文 → Gemini 1.5 Pro / Claude
需要本地部署 → Llama 3 / Qwen 2.5
需要中文优化 → Qwen / 文心一言
需要代码能力 → DeepSeek Coder / Claude
预算有限     → DeepSeek / Mistral
```

## 4. Token 与计费

### 4.1 Token 概念

```python
# Token 不等于字符，也不等于单词
"Hello, world!"     → ["Hello", ",", " world", "!"]     = 4 tokens
"你好世界"           → ["你", "好", "世", "界"]           = 4 tokens
"ChatGPT is great"  → ["Chat", "G", "PT", " is", " great"] = 5 tokens

# 经验法则
# 英文: 1 token ≈ 4 个字符 ≈ 0.75 个单词
# 中文: 1 token ≈ 1-2 个汉字
```

### 4.2 计费模型

```
总费用 = 输入 Token 数 × 输入单价 + 输出 Token 数 × 输出单价

示例 (GPT-4o):
  输入: $2.50 / 1M tokens
  输出: $10.00 / 1M tokens

  一次 Agent 调用（输入 2000 tokens，输出 500 tokens）:
  费用 = 2000/1M × $2.50 + 500/1M × $10.00 = $0.01
```

## 5. API 调用实践

### 5.1 OpenAI 风格 API

```python
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
```

### 5.2 流式输出

```python
# 流式输出 - 适合实时展示
stream = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "解释 Transformer"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### 5.3 多轮对话

```python
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
```

### 5.4 使用国产模型 API

```python
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
```

## 6. Embedding 模型

Embedding 将文本转为向量，是 RAG 和语义搜索的基础：

```python
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
```

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

## 8. 本地部署方案

```bash
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
```

```python
# Python 中使用本地模型
client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # 任意值即可
)

response = client.chat.completions.create(
    model="llama3:8b",
    messages=[{"role": "user", "content": "你好"}]
)
```

---

## 面试题精选

### Q1: Transformer 中 Self-Attention 的计算公式是什么？为什么要除以 √d_k？
**答：** Attention(Q,K,V) = softmax(QK^T / √d_k) × V。除以 √d_k 是为了防止点积值过大导致 softmax 梯度消失，起到缩放稳定训练的作用。

### Q2: Temperature 参数对 LLM 输出有什么影响？Agent 场景下应该怎么设置？
**答：** Temperature 控制输出随机性，0 表示确定性输出，1 表示高创造性。Agent 场景下推荐设为 0 或接近 0，因为工具调用和推理需要确定性和一致性，避免随机性导致参数生成错误。

### Q3: 开源模型和闭源模型各有什么优劣？什么场景选哪种？
**答：** 闭源模型（GPT-4o、Claude）能力强但有数据隐私风险和 API 依赖；开源模型（Llama、Qwen）可本地部署、数据可控但能力稍弱。涉及敏感数据或需要离线运行选开源，追求最强能力和快速上线选闭源。

### Q4: Token 是什么？中英文的 Token 计算有什么区别？
**答：** Token 是 LLM 处理文本的基本单位，不等于字符或单词。英文约 1 token ≈ 4 字符 ≈ 0.75 个单词；中文约 1 token ≈ 1-2 个汉字。理解 Token 对成本估算和上下文窗口管理至关重要。

### Q5: Embedding 模型的作用是什么？如何评估两段文本的语义相似度？
**答：** Embedding 模型将文本转为高维向量，语义相近的文本在向量空间中距离更近。通常用余弦相似度计算两个向量的相似程度，值越接近 1 表示语义越相似。这是 RAG 语义检索的基础。

### Q6: 如何在本地部署一个开源 LLM？有哪些方案？
**答：** 最简单的方案是使用 Ollama，一行命令即可运行（ollama run llama3:8b）。它提供兼容 OpenAI 格式的 API，Python 代码只需改 base_url 即可切换。其他方案还有 vLLM（高性能推理）、llama.cpp（CPU 推理）等。

### Q7: 什么是模型幻觉（Hallucination）？在 Agent 系统中如何缓解？
**答：** 幻觉是模型生成看似合理但不真实的内容。在 Agent 中可通过 RAG（基于检索的事实回答）、工具调用（用搜索引擎验证）、自我反思机制（让模型检查自己的输出）来缓解。

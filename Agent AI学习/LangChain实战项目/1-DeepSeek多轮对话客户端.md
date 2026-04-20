# DeepSeek 多轮对话客户端

> 状态：✅ 已完成 | 日期：2026-04-20 | 代码：`~/lib/langchain-deepseek-chat/`

## 项目目标

用 LangChain 接入 DeepSeek API，实现终端多轮对话，作为后续 Agent/RAG 项目的基座。

## 技术栈

| 组件 | 用途 |
|------|------|
| `ChatOpenAI` | 对接 DeepSeek（兼容 OpenAI 接口） |
| `ChatPromptTemplate` + `MessagesPlaceholder` | 系统提示 + 动态历史消息 |
| LCEL `prompt \| llm` | 管道符编排 Chain |
| `RunnableWithMessageHistory` | 自动管理多轮对话记忆 |
| `stream()` | 流式输出 |

## 核心代码解析

### 1. 模型初始化

DeepSeek 兼容 OpenAI 接口，只需改 `base_url`：

```python
llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
)
```

### 2. Prompt 模板

`MessagesPlaceholder` 是关键，它会在运行时被替换为历史消息列表：

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有帮助的 AI 助手"),
    MessagesPlaceholder(variable_name="history"),  # 历史消息插入点
    ("human", "{input}"),
])
```

### 3. LCEL 管道

用 `|` 把 prompt 和 llm 串起来，这就是一个最简单的 Chain：

```python
chain = prompt | llm
```

### 4. 记忆管理

`RunnableWithMessageHistory` 包装 Chain，自动存取对话历史：

```python
chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,        # 返回 ChatMessageHistory 的函数
    input_messages_key="input",
    history_messages_key="history",
)
```

## 学到了什么

- DeepSeek 完全兼容 OpenAI SDK，切换模型只需改 `base_url` 和 `model`
- LCEL 的 `|` 管道符让 Chain 组合非常直观
- `MessagesPlaceholder` 是实现多轮对话的关键组件
- `RunnableWithMessageHistory` 封装了记忆管理的样板代码
- `stream()` 流式输出体验比 `invoke()` 好很多

## 下一步

- [ ] 添加 `@tool` 工具，让 AI 能搜索、计算
- [ ] 用 `AgentExecutor` 构建真正的 Agent
- [ ] 接入 LangSmith 追踪调用链路

相关文档：[[../LangChain专题/2-Model IO详解]] · [[../LangChain专题/3-LCEL表达式语言]]

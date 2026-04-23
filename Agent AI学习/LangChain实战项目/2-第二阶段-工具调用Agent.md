# 第二阶段：工具调用 Agent

> 日期：2026-04-21 | 代码：`~/lib/langchain-project/`

相关主题：[[../LangChain专题/4-Tool与Agent]] | [[../LangChain专题/3-LCEL表达式语言]]

---

## 一、工具定义 (`chat.py` / `server.py`)

| 代码              | 知识点                                               | 对应文档                            |
| --------------- | ------------------------------------------------- | ------------------------------- |
| `@tool` 装饰器     | Tool — 最简工具定义方式，自动提取 name/description/args_schema | [[../LangChain专题/4-Tool与Agent]] |
| `calculator` 工具 | 安全沙箱 — `eval` + `{"__builtins__": {}}` 限制执行环境     | [[../LangChain专题/4-Tool与Agent]] |
| 工具 docstring    | 工具描述 — LLM 完全依赖描述决定何时使用，质量直接影响 Agent 表现           | [[../LangChain专题/4-Tool与Agent]] |
| `tools = [...]` | 工具列表 — 控制在 3-8 个，太多会降低选择准确率                       | [[../LangChain专题/4-Tool与Agent]] |

## 二、Agent 构建 (`chat.py` / `server.py`)

| 代码                                              | 知识点                                              | 对应文档                            |
| ----------------------------------------------- | ------------------------------------------------ | ------------------------------- |
| `MessagesPlaceholder("agent_scratchpad")`       | Agent Prompt — 与普通 Chain 的关键区别，存放中间推理过程          | [[../LangChain专题/4-Tool与Agent]] |
| `create_tool_calling_agent(llm, tools, prompt)` | Function Calling Agent — 基于 LLM 原生能力，比 ReAct 更准确 | [[../LangChain专题/4-Tool与Agent]] |
| `AgentExecutor(agent, tools, ...)`              | 推理循环引擎 — 执行 Thought→Action→Observation 循环        | [[../LangChain专题/4-Tool与Agent]] |
| `max_iterations=10`                             | 安全边界 — 防止 Agent 无限循环                             | [[../LangChain专题/4-Tool与Agent]] |
| `handle_parsing_errors=True`                    | 错误恢复 — 自动处理 LLM 输出格式错误                           | [[../LangChain专题/4-Tool与Agent]] |
| `RunnableWithMessageHistory` 包装 AgentExecutor   | Memory + Agent — 记忆机制同样适用于 Agent                 | [[../LangChain专题/2-Model IO详解]] |

## 三、Agent API 流式推送 (`server.py`)

| 代码 | 知识点 | 对应文档 |
|------|--------|---------|
| `astream_events(version="v2")` | 细粒度事件流 — 比 astream 更详细，能捕获工具调用过程 | [[../LangChain专题/3-LCEL表达式语言]] |
| `on_tool_start` / `on_tool_end` | 工具事件 — 获取工具名、输入参数、返回结果 | [[../LangChain专题/4-Tool与Agent]] |
| `on_chat_model_stream` | LLM token 事件 — 最终回答的逐字输出 | [[../LangChain专题/3-LCEL表达式语言]] |
| `event: tool_start\ndata: {...}\n\n` | SSE 自定义事件类型 — 标准 SSE 支持 event 字段区分消息类型 | [[../LangChain专题/7-生产化部署]] |

## 四、前端工具调用展示 (`index.html`)

| 功能 | 知识点 |
|------|--------|
| SSE 多事件类型解析 | 按 `\n\n` 分割消息，解析 `event:` 和 `data:` 字段 |
| `.tool-call` 卡片 | 工具调用可视化 — pending/done 状态切换 |
| 文本节点管理 | DOM 操作 — 工具卡片后追加文本需要新建 TextNode |
| `escHtml()` | XSS 防护 — 转义工具返回的 HTML 特殊字符 |

## 五、架构变化

````
第一阶段: prompt | llm → RunnableWithMessageHistory → stream/astream
第二阶段: create_tool_calling_agent → AgentExecutor → RunnableWithMessageHistory → astream_events
第三阶段: 同上，新增 knowledge_search 工具（内部: FAISS Retriever → 格式化文档 → 返回给 Agent）
````

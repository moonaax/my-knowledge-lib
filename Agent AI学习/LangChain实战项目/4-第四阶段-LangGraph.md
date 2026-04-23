# 第四阶段：LangGraph 多步骤 Agent

> 日期：2026-04-23 | 代码：`~/lib/langchain-project/`

相关主题：[[../LangChain专题/6-LangGraph编排]] | [[../LangChain专题/4-Tool与Agent]]

---

## 一、LangGraph 图构建 (`graph_agent.py`)

| 代码 | 知识点 | 对应文档 |
|------|--------|---------|
| `StateGraph(MessagesState)` | 图初始化 — 使用内置 MessagesState，自动管理消息列表追加 | [[../LangChain专题/6-LangGraph编排]] |
| `graph.add_node("agent", agent_node)` | 节点注册 — LLM 推理节点，接收 State 返回更新 | [[../LangChain专题/6-LangGraph编排]] |
| `graph.add_node("tools", tool_node)` | ToolNode — LangGraph 内置工具执行节点，自动提取 tool_calls | [[../LangChain专题/6-LangGraph编排]] |
| `graph.add_edge(START, "agent")` | 普通边 — 固定路由，图从 agent 节点开始 | [[../LangChain专题/6-LangGraph编排]] |
| `graph.add_conditional_edges("agent", should_continue)` | 条件路由 — 根据 LLM 输出动态决定走向 | [[../LangChain专题/6-LangGraph编排]] |
| `graph.add_edge("tools", "agent")` | 循环边 — 工具执行完回到 agent，形成 ReAct 循环 | [[../LangChain专题/6-LangGraph编排]] |
| `llm.bind_tools(all_tools)` | 工具绑定 — 让 LLM 知道有哪些工具可调用 | [[../LangChain专题/4-Tool与Agent]] |
| `MemorySaver()` + `graph.compile(checkpointer=memory)` | 状态持久化 — 内存检查点，支持断点恢复 | [[../LangChain专题/6-LangGraph编排]] |

图结构：

````
START → agent(LLM 推理) ──┬── 有 tool_calls → tools(ToolNode) → agent（循环）
                          └── 无 tool_calls → END
````

## 二、对比 AgentExecutor vs LangGraph

| 维度 | AgentExecutor（旧） | LangGraph（新） |
|------|---------------------|-----------------|
| 执行模型 | 黑盒 while 循环 | 可视化有向图 |
| 状态管理 | 无持久化 | MemorySaver 每步存档 |
| 控制粒度 | 只能配 max_iterations | 精确控制每个节点和边 |
| 扩展性 | 难以插入自定义步骤 | 新增节点/边即可扩展 |
| 多轮对话 | RunnableWithMessageHistory | checkpointer + thread_id |

## 三、服务端新增端点 (`server.py`)

| 代码 | 知识点 | 对应文档 |
|------|--------|---------|
| `@app.post("/graph_chat")` | 双端点共存 — 旧 `/chat` 保留，新 `/graph_chat` 并行 | [[../LangChain专题/7-生产化部署]] |
| `graph_app.astream_events(..., version="v2")` | LangGraph 流式 — 与 AgentExecutor 使用相同的事件协议 | [[../LangChain专题/3-LCEL表达式语言]] |
| `configurable: {"thread_id": session_id}` | 会话隔离 — LangGraph 用 thread_id 区分会话 | [[../LangChain专题/6-LangGraph编排]] |
| `graph_memory.delete_thread(...)` | 清除历史 — 删除指定会话的所有检查点 | [[../LangChain专题/6-LangGraph编排]] |

## 四、终端版模式选择 (`chat.py`)

| 改动 | 知识点 |
|------|--------|
| `run_agent_executor()` / `run_langgraph_agent()` | 模式分离 — 两种实现独立封装，互不干扰 |
| 运行时选择模式 | 对比体验 — 同一工具集，不同执行引擎 |
| `llm.bind_tools(tools)` + `ToolNode(tools)` | LangGraph 工具集成 — 比 AgentExecutor 更显式 |

## 五、当前架构

````
第四阶段架构：
  AgentExecutor 模式: create_tool_calling_agent → AgentExecutor → astream_events
  LangGraph 模式:     StateGraph(agent → tools → agent) → astream_events
  两种模式共存，共享 tools.py 工具定义
````

## 六、踩坑记录

1. **ToolMessage is not JSON serializable**：LangGraph 的 `astream_events` 在 `on_tool_end` 事件中，`output` 字段是 `ToolMessage` 对象而非字符串，`json.dumps` 无法序列化。解决：`str(output)` 转换后再序列化
2. **HuggingFace 连接超时**：LangGraph 模式下调用 `knowledge_search` 工具时，`HuggingFaceEmbeddings` 尝试联网检查更新导致超时。解决：启动时加 `HF_ENDPOINT=https://hf-mirror.com HF_HUB_OFFLINE=1` 环境变量

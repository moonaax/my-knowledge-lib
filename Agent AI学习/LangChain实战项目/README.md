# LangChain 实战项目

记录基于 LangChain + DeepSeek 的实战项目开发进度。

> GitHub：https://github.com/moonaax/langchain-project

## 项目列表

| # | 项目 | 状态 | 说明 |
|---|------|------|------|
| 1 | [[1-DeepSeek多轮对话客户端]] | ✅ 已完成 | 终端版 + Electron 桌面客户端（深浅主题） |
| 2 | [[2-项目知识点分析\|工具调用 Agent]] | ✅ 已完成 | @tool、create_tool_calling_agent、AgentExecutor、astream_events |
| 3 | [[2-项目知识点分析\|知识库 RAG 问答]] | ✅ 已完成 | DirectoryLoader、TextSplitter、FAISS、HuggingFace Embedding、RAG Tool |
| 4 | [[2-项目知识点分析\|LangGraph 多步骤 Agent]] | 🟡 进行中 | StateGraph、ToolNode、条件路由（自纠错待实现） |
| 5 | 生产化部署 | 🔲 待开始 | LangServe API、LangSmith |

## 对应学习路线

````
学习阶段          实战项目                    涉及知识点
─────────────────────────────────────────────────────────
第一阶段 基础      → 1. 多轮对话客户端         Model I/O、LCEL、Memory
第二阶段 Agent    → 2. 添加工具调用            Tool、Agent、Callback
第三阶段 RAG      → 3. 知识库 RAG 问答        Loader、Splitter、Retriever
第四阶段 LangGraph → 4. 多步骤 Agent          StateGraph、Multi-Agent
第五阶段 生产化    → 5. 部署上线               LangServe、LangSmith
````
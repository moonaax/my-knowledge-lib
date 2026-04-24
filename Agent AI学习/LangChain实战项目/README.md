# LangChain 实战项目

记录基于 LangChain + DeepSeek 的实战项目开发进度。

> GitHub：https://github.com/moonaax/langchain-project

## 项目列表

| # | 项目 | 状态 | 知识点文档 |
|---|------|------|-----------|
| 1 | [[1-DeepSeek多轮对话客户端]] | ✅ 已完成 | [[1-第一阶段-多轮对话客户端]] |
| 2 | 工具调用 Agent | ✅ 已完成 | [[2-第二阶段-工具调用Agent]] |
| 3 | 知识库 RAG 问答 | ✅ 已完成 | [[3-第三阶段-知识库RAG问答]] |
| 4 | LangGraph 多步骤 Agent | ✅ 已完成 | [[4-第四阶段-LangGraph]] |
| 5 | 记忆持久化 + RAG 优化 | ✅ 已完成 | [[5-第五阶段-记忆持久化与RAG优化]] |
| 6 | 生产化部署 | 🔲 待开始 | [[5-第五阶段-生产化部署]] |

> 进度总览：[[0-进度总览]]

## 对应学习路线

````
学习阶段              实战项目                    涉及知识点
─────────────────────────────────────────────────────────────
第一阶段 基础          → 1. 多轮对话客户端         Model I/O、LCEL、Memory
第二阶段 Agent        → 2. 添加工具调用            Tool、Agent、Callback
第三阶段 RAG          → 3. 知识库 RAG 问答        Loader、Splitter、Retriever
第四阶段 LangGraph     → 4. 多步骤 Agent          StateGraph、Multi-Agent
第五阶段 记忆+RAG优化  → 5. 持久化+混合检索        SQLite、BM25、RRF、评测
第六阶段 生产化        → 6. 部署上线               Docker、LangSmith
````

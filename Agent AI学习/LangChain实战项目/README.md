# LangChain 实战项目

记录基于 LangChain + DeepSeek 的实战项目开发进度。

## 项目列表

| # | 项目 | 状态 | 说明 |
|---|------|------|------|
| 1 | [[1-DeepSeek多轮对话客户端]] | ✅ 已完成 | 基础对话，LangChain 入门 |
| 2 | 添加工具调用（Agent） | 🔲 待开始 | @tool、AgentExecutor |
| 3 | 知识库 RAG 问答 | 🔲 待开始 | 基于本知识库的 RAG 系统 |
| 4 | LangGraph 多步骤 Agent | 🔲 待开始 | StateGraph、循环、自纠错 |
| 5 | 生产化部署 | 🔲 待开始 | LangServe API、LangSmith |

## 对应学习路线

```
学习阶段          实战项目                    涉及知识点
─────────────────────────────────────────────────────────
第一阶段 基础      → 1. 多轮对话客户端         Model I/O、LCEL、Memory
第二阶段 Agent    → 2. 添加工具调用            Tool、Agent、Callback
第三阶段 RAG      → 3. 知识库 RAG 问答        Loader、Splitter、Retriever
第四阶段 LangGraph → 4. 多步骤 Agent          StateGraph、Multi-Agent
第五阶段 生产化    → 5. 部署上线               LangServe、LangSmith
```

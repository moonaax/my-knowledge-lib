# LangChain 专题

系统学习 LangChain 框架的完整知识体系，从基础到生产化。

## 学习路线

```
第一阶段: 核心基础（1-2 周）
├── Model I/O — 模型调用、Prompt 模板、输出解析
├── LCEL 表达式语言 — 管道编排、并行、分支、重试
└── Memory 记忆管理 — 多轮对话、会话存储

第二阶段: 工具与 Agent（1-2 周）
├── Tool 工具体系 — @tool、StructuredTool、工具描述
├── Agent 构建与调试 — AgentExecutor、LangSmith 追踪
└── Callback 与流式输出 — 事件回调、streaming

第三阶段: RAG 实战（1-2 周）
├── Document Loader 与 Text Splitter — 数据加载与分块
├── VectorStore 与 Retriever — 向量存储与检索策略
└── RAG Chain 构建 — 端到端检索增强生成

第四阶段: LangGraph 进阶（1-2 周）
├── StateGraph 基础 — 状态、节点、边、条件路由
├── 循环与自纠错 Agent — 反思、重试、人机协作
└── Multi-Agent 编排 — 多智能体协作模式

第五阶段: 生产化（1 周）
├── LangServe 部署 — API 服务化
├── LangSmith 深度使用 — 评估、数据集、监控
└── 性能优化与降级 — 缓存、并发、fallback
```

## 文档列表

- [[学习路线与资源导航]] — 完整学习路线、官方资源、推荐项目
- [[Model IO 详解]] — ChatModel、Prompt Template、Output Parser
- [[LCEL 表达式语言]] — 管道编排、RunnableParallel、RunnableBranch
- [[Tool 与 Agent]] — 工具定义、Agent 构建、调试技巧
- [[RAG 实战]] — 基于 LangChain 的完整 RAG 实现
- [[LangGraph 编排]] — 有状态 Agent、循环、多 Agent 协作
- [[生产化部署]] — LangServe、LangSmith、性能优化

## 前置知识

- [[../基础概念/LLM大语言模型基础|LLM 大语言模型基础]]
- [[../基础概念/Prompt Engineering|Prompt Engineering]]
- [[../基础概念/什么是AI Agent|什么是 AI Agent]]
- [[../框架与工具/LangChain框架详解|LangChain 框架详解]]（概览）

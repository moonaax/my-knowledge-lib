# Agent 应用实战案例

> **本文定位：** 前面的模块讲了各种概念和框架，本文把它们组合起来做完整的项目。每个案例都是可以直接跑的代码，覆盖了知识库问答、代码助手、数据分析、API 服务四个最常见的 Agent 应用场景。

## 案例一：知识库问答 Agent

> **这是最经典的 Agent 应用：** 基于企业内部文档构建智能问答系统。核心是 RAG（检索增强生成）——先从文档中检索相关内容，再让 LLM 基于检索结果回答问题，避免幻觉。

### 需求

构建一个基于企业内部文档的智能问答系统，支持：
- 多格式文档导入（PDF、Markdown、Word）
- 精准的语义检索
- 带来源引用的回答
- 多轮对话

### 完整实现

> **整体架构：** 文档加载 → 分块 → Embedding 向量化 → 存入 ChromaDB → 用户提问时语义检索 Top-K → 检索结果 + 问题一起传给 LLM → 生成带来源引用的回答。同时维护对话历史支持多轮对话。

````python
"""企业知识库问答 Agent"""
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import (
    DirectoryLoader, PyPDFLoader, UnstructuredMarkdownLoader
)
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser
from langchain_core.messages import HumanMessage, AIMessage

class KnowledgeBaseAgent:
    def __init__(self, docs_dir: str, db_dir: str = "./chroma_db"):
        self.llm = ChatOpenAI(model="gpt-4o", temperature=0)
        self.embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
        self.db_dir = db_dir
        self.chat_history = []
        
        # 加载或创建向量库
        self.vectorstore = self._init_vectorstore(docs_dir)
        self.retriever = self.vectorstore.as_retriever(
            search_type="mmr",  # 最大边际相关性，增加多样性
            search_kwargs={"k": 5, "fetch_k": 20}
        )
        
        # 构建链
        self.chain = self._build_chain()
    
    def _init_vectorstore(self, docs_dir: str):
        """初始化向量数据库"""
        try:
            return Chroma(
                persist_directory=self.db_dir,
                embedding_function=self.embeddings
            )
        except Exception:
            pass
        
        # 加载文档
        loader = DirectoryLoader(docs_dir, glob="**/*.*", show_progress=True)
        documents = loader.load()
        
        # 分块
        splitter = RecursiveCharacterTextSplitter(
            chunk_size=800,
            chunk_overlap=150,
            separators=["\n## ", "\n### ", "\n\n", "\n", " "]
        )
        chunks = splitter.split_documents(documents)
        
        # 创建向量库
        return Chroma.from_documents(
            chunks, self.embeddings, persist_directory=self.db_dir
        )
    
    def _build_chain(self):
        prompt = ChatPromptTemplate.from_messages([
            ("system", """你是企业知识库助手。基于检索到的文档回答问题。

规则:
1. 只基于提供的文档回答，不要编造信息
2. 如果文档中没有相关信息，明确告知用户
3. 回答末尾标注信息来源
4. 保持回答简洁专业"""),
            MessagesPlaceholder(variable_name="chat_history"),
            ("human", "参考文档:\n{context}\n\n问题: {question}")
        ])
        
        return (
            {
                "context": self.retriever | self._format_docs,
                "question": RunnablePassthrough(),
                "chat_history": lambda _: self.chat_history
            }
            | prompt
            | self.llm
            | StrOutputParser()
        )
    
    @staticmethod
    def _format_docs(docs):
        return "\n\n---\n\n".join(
            f"[来源: {d.metadata.get('source', '未知')}]\n{d.page_content}"
            for d in docs
        )
    
    def chat(self, question: str) -> str:
        answer = self.chain.invoke(question)
        self.chat_history.extend([
            HumanMessage(content=question),
            AIMessage(content=answer)
        ])
        # 保持历史在合理范围
        if len(self.chat_history) > 20:
            self.chat_history = self.chat_history[-20:]
        return answer

# 使用
agent = KnowledgeBaseAgent("./company_docs")
print(agent.chat("我们的 API 认证方式是什么?"))
print(agent.chat("具体怎么获取 Token?"))  # 能关联上下文
````

> **关键代码解读：**
> - `search_type="mmr"` — 最大边际相关性检索，在相关性和多样性之间取平衡，避免检索到的文档内容重复
> - `RecursiveCharacterTextSplitter` — 按层级分隔符（标题 → 段落 → 句子）递归分块，比简单按字符数切割更智能
> - `_build_chain()` 用 LCEL 管道组装：检索器获取文档 → 格式化 → 拼入 prompt → LLM 生成 → 解析输出
> - `chat_history` 限制在 20 条以内，防止 Token 溢出
> - `_format_docs()` 在每段文档前标注来源，让 LLM 能在回答中引用出处

## 案例二：代码助手 Agent

> **和案例一的区别：** 案例一是 RAG 模式（检索+生成），案例二是 Agent 模式（自主决策+工具调用）。Agent 会根据用户问题自主决定：先看项目结构 → 读取相关文件 → 搜索代码 → 运行命令 → 给出分析结果。

### 需求

一个能读取项目代码、分析问题、生成修复方案的 Agent。

### 实现

> **工具设计是核心：** 这个 Agent 有 4 个工具——读文件、列目录、搜索代码、执行命令。每个工具的 docstring 描述了用途，LLM 靠这些描述决定何时调用哪个工具。注意 `run_command` 有安全检查，禁止危险命令。

````python
"""代码助手 Agent"""
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.tools import tool
import subprocess
import os

@tool
def read_file(file_path: str) -> str:
    """读取指定文件的内容。用于查看源代码或配置文件。"""
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
        if len(content) > 5000:
            return content[:5000] + "\n... (文件过长，已截断)"
        return content
    except FileNotFoundError:
        return f"文件不存在: {file_path}"

@tool
def list_directory(dir_path: str = ".") -> str:
    """列出目录下的文件和文件夹。用于了解项目结构。"""
    try:
        items = os.listdir(dir_path)
        result = []
        for item in sorted(items):
            full_path = os.path.join(dir_path, item)
            prefix = "📁" if os.path.isdir(full_path) else "📄"
            result.append(f"{prefix} {item}")
        return "\n".join(result)
    except FileNotFoundError:
        return f"目录不存在: {dir_path}"

@tool
def search_in_files(pattern: str, directory: str = ".") -> str:
    """在项目文件中搜索指定模式。用于查找特定代码或配置。"""
    try:
        result = subprocess.run(
            ["grep", "-rn", "--include=*.py", "--include=*.js",
             "--include=*.ts", "--include=*.java", pattern, directory],
            capture_output=True, text=True, timeout=10
        )
        output = result.stdout[:3000] if result.stdout else "未找到匹配"
        return output
    except subprocess.TimeoutExpired:
        return "搜索超时"

@tool
def run_command(command: str) -> str:
    """执行 shell 命令。用于运行测试、检查依赖等。仅限安全命令。"""
    # 安全检查
    dangerous = ["rm ", "sudo", "chmod", "chown", "mkfs", "> /"]
    if any(d in command for d in dangerous):
        return "拒绝执行: 命令包含危险操作"
    
    try:
        result = subprocess.run(
            command, shell=True, capture_output=True,
            text=True, timeout=30
        )
        output = result.stdout + result.stderr
        return output[:3000] if output else "命令执行完成，无输出"
    except subprocess.TimeoutExpired:
        return "命令执行超时"

# 创建 Agent
llm = ChatOpenAI(model="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个资深代码助手。你可以:
1. 读取和分析项目代码
2. 搜索代码中的特定模式
3. 运行命令检查项目状态
4. 提供代码改进建议

工作流程:
1. 先了解项目结构
2. 读取相关代码文件
3. 分析问题
4. 给出具体的修复方案（包含代码）"""),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

tools = [read_file, list_directory, search_in_files, run_command]
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True, max_iterations=10)

# 使用
result = executor.invoke({
    "input": "分析当前项目的代码结构，找出可能的性能问题"
})
print(result["output"])
````

> **关键代码解读：**
> - `@tool` 装饰器 — 把普通函数变成 Agent 可调用的工具，docstring 作为工具描述
> - `dangerous` 列表 — 安全白名单机制，禁止 rm、sudo 等危险命令
> - `max_iterations=10` — 限制 Agent 最多执行 10 步，防止无限循环
> - `agent_scratchpad` — Agent 的"草稿纸"，存放中间思考过程和工具调用结果
> - 文件内容超过 5000 字符会截断，防止单次工具调用消耗过多 Token

## 案例三：数据分析 Agent

> **自然语言驱动的数据分析：** 用户用中文描述需求（如"分析各产品销售表现"），Agent 自动完成数据查询 → 编写 pandas 分析代码 → 执行 → 生成图表 → 用通俗语言解释结论。核心是 `analyze_data` 工具，它接收 Agent 生成的 Python 代码并执行。

### 需求

自然语言驱动的数据分析，用户用中文描述需求，Agent 自动完成数据查询、分析和可视化。

### 实现

````python
"""数据分析 Agent"""
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.tools import tool
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
import pandas as pd
import json

# 模拟数据
SAMPLE_DATA = pd.DataFrame({
    "date": pd.date_range("2024-01-01", periods=90, freq="D"),
    "product": ["A", "B", "C"] * 30,
    "sales": [100 + i * 2 + (i % 7) * 10 for i in range(90)],
    "region": ["华东", "华北", "华南"] * 30
})

@tool
def query_data(description: str) -> str:
    """根据描述查询数据。描述你需要什么数据，如'最近30天的销售数据'。"""
    df = SAMPLE_DATA
    # 简化: 实际应用中这里会解析自然语言为 SQL 或 pandas 操作
    return df.to_string(max_rows=20)

@tool
def analyze_data(code: str) -> str:
    """执行 pandas 分析代码。变量 df 已预加载为销售数据。"""
    try:
        local_vars = {"df": SAMPLE_DATA, "pd": pd}
        exec(code, {}, local_vars)
        result = local_vars.get("result", "代码执行完成，请将结果赋值给 result 变量")
        return str(result)
    except Exception as e:
        return f"执行错误: {e}"

@tool
def create_chart(chart_type: str, title: str, data_description: str) -> str:
    """生成图表描述（实际应用中会生成真实图表）。
    chart_type: bar/line/pie
    title: 图表标题
    data_description: 数据描述"""
    return f"✅ 已生成{chart_type}图: {title}\n数据: {data_description}"

llm = ChatOpenAI(model="gpt-4o", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", """你是数据分析专家。用户会用自然语言描述分析需求。

工作流程:
1. 理解用户需求
2. 查询所需数据
3. 编写分析代码
4. 生成可视化图表
5. 用通俗语言解释分析结论

注意: 使用 analyze_data 工具时，将分析结果赋值给 result 变量。"""),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

tools = [query_data, analyze_data, create_chart]
agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 使用
result = executor.invoke({
    "input": "分析各产品在不同区域的销售表现，找出表现最好和最差的组合"
})
print(result["output"])
````

> **关键代码解读：**
> - `analyze_data` 工具用 `exec()` 执行 Agent 生成的 pandas 代码，结果通过 `result` 变量返回
> - `local_vars = {"df": SAMPLE_DATA, "pd": pd}` — 预注入数据和库，Agent 生成的代码可以直接用 `df` 和 `pd`
> - 生产环境中 `exec()` 有安全风险，应该用沙箱（如 Docker 容器）隔离执行
> - `create_chart` 这里是模拟的，实际项目中可以用 matplotlib/plotly 生成真实图表

## 案例四：Web API Agent 服务

> **把 Agent 变成 API 服务：** 前面的案例都是本地运行的，实际项目中需要部署为 HTTP API 供前端/其他服务调用。用 FastAPI 包装 Agent，通过 session_id 实现多用户会话隔离。

### 使用 FastAPI 部署 Agent

````python
"""Agent API 服务"""
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from langchain_openai import ChatOpenAI
import uvicorn

app = FastAPI(title="Agent API")

# 初始化 Agent（复用上面的知识库 Agent）
agent = KnowledgeBaseAgent("./docs")

class ChatRequest(BaseModel):
    message: str
    session_id: str = "default"

class ChatResponse(BaseModel):
    answer: str
    session_id: str

# 会话管理
sessions = {}

@app.post("/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    if request.session_id not in sessions:
        sessions[request.session_id] = KnowledgeBaseAgent("./docs")
    
    agent = sessions[request.session_id]
    answer = agent.chat(request.message)
    
    return ChatResponse(answer=answer, session_id=request.session_id)

@app.delete("/session/{session_id}")
async def clear_session(session_id: str):
    sessions.pop(session_id, None)
    return {"message": "会话已清除"}

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
````

> **关键代码解读：**
> - `sessions` 字典 — 按 session_id 隔离不同用户的 Agent 实例，每个用户有独立的对话历史
> - `ChatRequest/ChatResponse` — Pydantic 模型定义请求和响应格式，FastAPI 自动做参数校验
> - `DELETE /session/{session_id}` — 清除会话，释放内存
> - 生产环境中 sessions 应该用 Redis 存储（而非内存字典），并设置过期时间自动清理

## 5. 项目结构模板

````
my-agent-project/
├── agents/              # Agent 定义
│   ├── __init__.py
│   ├── base_agent.py    # 基础 Agent 类
│   ├── qa_agent.py      # 问答 Agent
│   └── code_agent.py    # 代码 Agent
├── tools/               # 工具定义
│   ├── __init__.py
│   ├── search.py
│   ├── code_exec.py
│   └── data_query.py
├── memory/              # 记忆管理
│   ├── __init__.py
│   ├── short_term.py
│   └── long_term.py
├── prompts/             # Prompt 模板
│   └── templates.py
├── api/                 # API 层
│   ├── __init__.py
│   └── routes.py
├── config.py            # 配置
├── requirements.txt
├── Dockerfile
└── main.py              # 入口
````
## 6. 开发检查清单

````
□ 明确 Agent 的能力边界
□ 工具描述是否足够精确
□ 是否有错误处理和降级方案
□ 是否设置了最大迭代次数
□ 是否有 Token 使用监控
□ 是否考虑了并发和会话隔离
□ 敏感操作是否有安全检查
□ 是否有日志和可观测性
□ 是否做了 Prompt 注入防护
□ 是否有用户反馈收集机制
````
---

## 面试题精选

### Q1: 构建一个知识库问答 Agent，核心模块有哪些？
**答：** 文档加载与分块、Embedding 向量化与索引存储、检索器（支持混合检索）、Prompt 模板（含来源引用指令）、LLM 生成、对话历史管理。关键是检索质量和 Prompt 设计。

### Q2: 代码助手 Agent 的工具设计需要注意什么安全问题？
**答：** 代码执行工具必须有沙箱隔离、禁止危险命令（rm、sudo 等）、设置执行超时、限制文件系统访问范围、输出长度截断防止内存溢出。所有工具调用都应有审计日志。

### Q3: Agent 应用如何做会话隔离？
**答：** 通过 session_id 区分不同用户/会话，每个 session 独立维护对话历史和状态。可以用 Redis 存储会话状态，设置过期时间自动清理。API 层面要确保用户只能访问自己的会话。

### Q4: 将 Agent 部署为 API 服务时，流式输出怎么实现？
**答：** 使用 Server-Sent Events（SSE），后端用 FastAPI 的 StreamingResponse，Agent 每生成一个 chunk 就通过 SSE 推送给前端。前端用 EventSource API 接收。这样用户不用等完整回答生成就能看到内容。

### Q5: Agent 项目的标准目录结构应该怎么组织？
**答：** agents/（Agent 定义）、tools/（工具定义）、memory/（记忆管理）、prompts/（Prompt 模板）、api/（API 路由）、config.py（配置）。关键是职责分离，工具和 Agent 解耦，便于复用和测试。

### Q6: Agent 上线前的检查清单应该包含哪些项？
**答：** 能力边界是否明确、工具描述是否精确、错误处理和降级方案、最大迭代次数限制、Token 使用监控、并发和会话隔离、敏感操作安全检查、日志和可观测性、Prompt 注入防护、用户反馈收集机制。

# 第三阶段：知识库 RAG 问答

> 日期：2026-04-21 | 代码：`~/lib/langchain-project/`

相关主题：[[../LangChain专题/5-RAG实战]] | [[../LangChain专题/4-Tool与Agent]]

---

## 一、离线索引构建 (`build_index.py`)

| 代码 | 知识点 | 对应文档 |
|------|--------|---------|
| `DirectoryLoader(glob="**/*.md")` | Document Loader — 递归扫描目录，按 glob 匹配文件批量加载 | [[../LangChain专题/5-RAG实战]] |
| `TextLoader(encoding="utf-8")` | 文本加载器 — 以纯文本方式加载 Markdown，保留原始格式 | [[../LangChain专题/5-RAG实战]] |
| `RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)` | 文本分块 — 按分隔符优先级递归分割，中文优化 separators | [[../LangChain专题/5-RAG实战]] |
| `separators=["\n\n", "\n", "。", "！", "？", ...]` | 中文分隔符 — 优先段落→行→句子→字符，保持语义完整 | [[../LangChain专题/5-RAG实战]] |
| `HuggingFaceEmbeddings(model_name="BAAI/bge-base-zh-v1.5")` | Embedding 模型 — 本地中文向量化，768 维，无需 API Key | [[../LangChain专题/5-RAG实战]] |
| `normalize_embeddings=True` | 向量归一化 — 使余弦相似度等价于内积，提高检索效率 | [[../LangChain专题/5-RAG实战]] |
| `FAISS.from_documents(chunks, embeddings)` | 向量存储 — 一步完成向量化 + FAISS 索引构建 | [[../LangChain专题/5-RAG实战]] |
| `vectorstore.save_local("faiss_index")` | 索引持久化 — 序列化到磁盘（index.faiss + index.pkl） | [[../LangChain专题/5-RAG实战]] |

使用方式：

````bash
# 构建索引（首次需下载 Embedding 模型约 400MB）
python3 build_index.py                      # 默认加载 ~/lib/my-knowledge-lib
python3 build_index.py /path/to/docs        # 指定文档目录

# 国内网络用镜像加速下载模型
HF_ENDPOINT=https://hf-mirror.com python3 build_index.py
````

## 二、RAG 工具封装 (`tools.py`)

| 代码 | 知识点 | 对应文档 |
|------|--------|---------|
| `FAISS.load_local(path, embeddings, allow_dangerous_deserialization=True)` | 加载索引 — 从磁盘反序列化，需传入相同的 Embedding 模型 | [[../LangChain专题/5-RAG实战]] |
| `vectorstore.as_retriever(search_kwargs={"k": 3})` | Retriever — 从 VectorStore 创建检索器，返回 top-k 相似文档 | [[../LangChain专题/5-RAG实战]] |
| `retriever.invoke(query)` | 检索调用 — 输入查询文本，返回 Document 列表 | [[../LangChain专题/5-RAG实战]] |
| 懒加载模式 `_get_retriever()` | 性能优化 — 首次调用时才加载索引，避免启动时阻塞 | 工程实践 |
| `@tool knowledge_search` | RAG + Agent 结合 — 将检索封装为工具，Agent 自主决定是否使用 | [[../LangChain专题/4-Tool与Agent]] [[../LangChain专题/5-RAG实战]] |

使用方式：

````python
# tools.py 导出 all_tools 列表，chat.py 和 server.py 统一导入
from tools import all_tools

# Agent 会根据用户问题自动判断是否调用 knowledge_search：
#   "LCEL 是什么？"        → 调用 knowledge_search("LCEL")
#   "今天几号？"            → 调用 get_current_time()
#   "你好"                 → 直接回答，不调用工具
````

## 三、工具模块化重构

| 改动 | 知识点 |
|------|--------|
| 新建 `tools.py` 集中管理工具 | 模块化 — 消除 chat.py 和 server.py 中的重复工具定义 |
| `from tools import all_tools` | 单一数据源 — 两端共享同一份工具列表，新增工具只改一处 |
| Prompt 新增 knowledge_search 说明 | 工具引导 — 告诉 Agent "技术问题优先从知识库检索" |

## 四、RAG 架构说明

````
离线阶段（build_index.py 一次性执行）：
  Markdown 文件 → DirectoryLoader → TextSplitter → HuggingFace Embedding → FAISS 持久化

在线阶段（Agent 运行时按需调用）：
  用户提问 → Agent 判断需要检索 → knowledge_search 工具
    → FAISS Retriever 检索 top-3 → 格式化文档+来源 → 返回给 Agent
    → Agent 基于检索结果生成回答
````

## 五、踩坑记录

1. **HuggingFace 下载超时**：国内网络无法直连 huggingface.co，用 `HF_ENDPOINT=https://hf-mirror.com` 环境变量走镜像
2. **HuggingFaceEmbeddings 弃用警告**：`langchain_community` 中的类将迁移到 `langchain-huggingface` 包，暂不影响功能
3. **运行时仍尝试联网**：模型下载完后，`sentence-transformers` 每次加载仍会尝试连 HuggingFace 检查更新，导致卡住。解决：启动时加 `HF_HUB_OFFLINE=1` 强制离线模式
4. **ClearRequest 接口报错**：第二阶段给 `ChatRequest` 加了 `message` 必填字段后，`/clear` 接口复用 `ChatRequest` 导致 422 错误。解决：单独定义 `ClearRequest(session_id)` 模型
5. **astream_events 推送中间推理**：Agent 多次调用 LLM（决策+最终回答），`on_chat_model_stream` 会捕获所有 token 包括中间推理（"我来帮您搜索..."）。解决：用 `in_tool` 标志位，工具调用期间暂停推送 token
6. **Markdown 渲染未完成**：DeepSeek 返回的 token 流中 `###` 标题前没有换行符，`marked.js` 无法识别。尝试了 CDN/npm/本地 UMD/流式渲染等方案，均有兼容性问题。暂时保持纯文本输出，后续优化

## 六、待优化

- [ ] 前端 Markdown 渲染（需处理 DeepSeek 非标准格式）
- [ ] 工具调用中间推理过程的展示优化（目前直接过滤掉了）

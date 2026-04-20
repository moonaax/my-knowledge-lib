# Agent 部署与性能优化

## 1. 部署架构

### 1.1 基础架构

```
┌──────────┐     ┌──────────────┐     ┌─────────────┐
│  Client  │────→│  API Gateway │────→│ Agent Service│
│ (Web/App)│     │  (Nginx/ALB) │     │  (FastAPI)   │
└──────────┘     └──────────────┘     └──────┬──────┘
                                             │
                    ┌────────────────────────┼────────────────┐
                    │                        │                │
              ┌─────▼─────┐          ┌──────▼──────┐  ┌─────▼─────┐
              │ LLM API   │          │ Vector DB   │  │  Redis    │
              │(OpenAI/   │          │(Milvus/     │  │(会话缓存) │
              │ 本地模型)  │          │ Qdrant)     │  │           │
              └───────────┘          └─────────────┘  └───────────┘
```

### 1.2 Docker 部署

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  agent:
    build: .
    ports:
      - "8000:8000"
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
      - CHROMA_HOST=chroma
    depends_on:
      - redis
      - chroma

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  chroma:
    image: chromadb/chroma:latest
    ports:
      - "8001:8000"
    volumes:
      - chroma_data:/chroma/chroma

volumes:
  chroma_data:
```

## 2. 性能优化

### 2.1 延迟优化

```python
# 1. 流式输出 — 减少用户感知延迟
from fastapi.responses import StreamingResponse

@app.post("/chat/stream")
async def chat_stream(request: ChatRequest):
    async def generate():
        async for chunk in agent.astream(request.message):
            yield f"data: {json.dumps({'content': chunk})}\n\n"
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate(), media_type="text/event-stream")

# 2. 并行工具调用
import asyncio

async def parallel_tool_calls(tool_calls):
    tasks = [execute_tool(tc) for tc in tool_calls]
    return await asyncio.gather(*tasks)

# 3. 缓存常见查询
from functools import lru_cache
import hashlib

query_cache = {}

def cached_query(question: str) -> str | None:
    key = hashlib.md5(question.encode()).hexdigest()
    return query_cache.get(key)

def cache_result(question: str, answer: str):
    key = hashlib.md5(question.encode()).hexdigest()
    query_cache[key] = answer
```

### 2.2 Token 优化

```python
# 1. Prompt 压缩
def compress_context(docs: list[str], max_tokens: int = 2000) -> str:
    """压缩检索到的文档，减少 Token 消耗"""
    compressed = []
    total_tokens = 0
    for doc in docs:
        doc_tokens = len(doc) // 4  # 粗略估算
        if total_tokens + doc_tokens > max_tokens:
            break
        compressed.append(doc)
        total_tokens += doc_tokens
    return "\n---\n".join(compressed)

# 2. 使用更小的模型处理简单任务
def route_to_model(question: str) -> str:
    """根据问题复杂度选择模型"""
    complexity = estimate_complexity(question)
    if complexity == "simple":
        return "gpt-4o-mini"  # 便宜 10x
    return "gpt-4o"

# 3. 减少不必要的对话历史
def trim_history(messages: list, max_tokens: int = 2000) -> list:
    """只保留最相关的历史消息"""
    if not messages:
        return messages
    # 保留 system + 最近的消息
    system = [m for m in messages if m["role"] == "system"]
    others = [m for m in messages if m["role"] != "system"]
    trimmed = others[-6:]  # 最近 3 轮对话
    return system + trimmed
```

### 2.3 检索优化

```python
# 1. 预计算热门查询的检索结果
hot_queries_cache = {}

async def warm_cache(common_queries: list[str]):
    for query in common_queries:
        results = await retriever.ainvoke(query)
        hot_queries_cache[query] = results

# 2. 向量数据库索引优化
# Milvus 示例
collection.create_index(
    field_name="embedding",
    index_params={
        "index_type": "HNSW",
        "metric_type": "COSINE",
        "params": {"M": 16, "efConstruction": 256}
    }
)

# 3. 分级检索
async def tiered_retrieval(query: str):
    # 先查缓存
    cached = hot_queries_cache.get(query)
    if cached:
        return cached
    
    # 再查向量库
    results = await retriever.ainvoke(query)
    return results
```

## 3. 成本控制

### 3.1 监控 Token 使用

```python
from langchain_core.callbacks import BaseCallbackHandler

class CostTracker(BaseCallbackHandler):
    def __init__(self):
        self.total_tokens = 0
        self.total_cost = 0.0
    
    def on_llm_end(self, response, **kwargs):
        usage = response.llm_output.get("token_usage", {})
        input_tokens = usage.get("prompt_tokens", 0)
        output_tokens = usage.get("completion_tokens", 0)
        
        self.total_tokens += input_tokens + output_tokens
        # GPT-5.4 价格
        self.total_cost += (input_tokens * 2.5 + output_tokens * 10) / 1_000_000
        
        print(f"累计: {self.total_tokens} tokens, ${self.total_cost:.4f}")

tracker = CostTracker()
llm = ChatOpenAI(model="gpt-4o", callbacks=[tracker])
```

### 3.2 成本优化策略

```
1. 模型分级
   简单任务 → gpt-4o-mini ($0.15/1M input)
   复杂任务 → gpt-4o ($2.50/1M input)
   节省: 约 90%

2. 缓存策略
   相同问题直接返回缓存 → 节省 100%
   相似问题复用检索结果 → 节省 LLM 调用

3. Prompt 精简
   去除冗余指令
   压缩检索文档
   限制对话历史长度

4. 批量处理
   合并多个小请求为一个批量请求
   使用 Batch API（OpenAI 提供 50% 折扣）

5. 预算告警
   设置每日/每月预算上限
   超出预算自动降级到更便宜的模型
```

## 4. 可观测性

### 4.1 日志

```python
import logging
import structlog

logger = structlog.get_logger()

class AgentLogger(BaseCallbackHandler):
    def on_agent_action(self, action, **kwargs):
        logger.info("agent_action",
            tool=action.tool,
            input=action.tool_input[:200]
        )
    
    def on_tool_end(self, output, **kwargs):
        logger.info("tool_result", output=output[:200])
    
    def on_agent_finish(self, finish, **kwargs):
        logger.info("agent_finish", output=finish.return_values)
    
    def on_llm_error(self, error, **kwargs):
        logger.error("llm_error", error=str(error))
```

### 4.2 指标监控

```python
from prometheus_client import Counter, Histogram, Gauge

# 定义指标
request_count = Counter("agent_requests_total", "总请求数", ["status"])
response_time = Histogram("agent_response_seconds", "响应时间")
token_usage = Counter("agent_tokens_total", "Token 使用量", ["type"])
active_sessions = Gauge("agent_active_sessions", "活跃会话数")

# 使用
@app.post("/chat")
async def chat(request: ChatRequest):
    active_sessions.inc()
    with response_time.time():
        try:
            result = await agent.achat(request.message)
            request_count.labels(status="success").inc()
            return result
        except Exception as e:
            request_count.labels(status="error").inc()
            raise
        finally:
            active_sessions.dec()
```

### 4.3 LangSmith 追踪

```python
import os
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "your-key"
os.environ["LANGCHAIN_PROJECT"] = "production-agent"

# 所有 LangChain 调用自动追踪
# 可在 LangSmith 控制台查看:
# - 完整调用链路
# - 每步的输入/输出
# - Token 使用和延迟
# - 错误详情
```

## 5. 安全防护

### 5.1 Prompt 注入防护

```python
def sanitize_input(user_input: str) -> str:
    """清理用户输入，防止 Prompt 注入"""
    # 检测常见注入模式
    injection_patterns = [
        "ignore previous instructions",
        "忽略之前的指令",
        "你现在是",
        "system:",
        "assistant:",
    ]
    
    input_lower = user_input.lower()
    for pattern in injection_patterns:
        if pattern in input_lower:
            return "检测到异常输入，请重新描述您的问题。"
    
    return user_input
```

### 5.2 输出过滤

```python
def filter_output(response: str) -> str:
    """过滤 Agent 输出中的敏感信息"""
    import re
    # 过滤可能泄露的 API Key
    response = re.sub(r'(sk-|api_key[=:]\s*)[a-zA-Z0-9]{20,}', '[REDACTED]', response)
    # 过滤内部 URL
    response = re.sub(r'https?://internal\.[^\s]+', '[INTERNAL_URL]', response)
    return response
```

## 6. 生产环境检查清单

```
部署前:
□ 所有 API Key 使用环境变量，不硬编码
□ 设置了请求速率限制
□ 配置了健康检查端点
□ 日志级别和格式正确
□ 错误处理覆盖所有路径

运行时:
□ Token 使用量监控和告警
□ 响应时间监控
□ 错误率监控
□ 会话数监控
□ 成本监控和预算告警

安全:
□ 输入验证和清理
□ 输出过滤
□ 工具调用权限控制
□ 审计日志
□ 定期安全审查
```

---

## 面试题精选

### Q1: Agent 服务的延迟优化有哪些关键手段？
**答：** 流式输出减少用户感知延迟、并行执行多个工具调用、缓存常见查询结果、用更小的模型处理简单任务、预热热门查询的检索结果、减少不必要的对话历史传入。

### Q2: 如何控制 Agent 的 API 调用成本？
**答：** 模型分级（简单任务用 mini 模型省 90%）、缓存相同/相似查询、Prompt 精简（压缩检索文档、限制历史长度）、使用 Batch API（OpenAI 提供 50% 折扣）、设置每日预算告警和自动降级。

### Q3: Agent 服务的可观测性应该监控哪些指标？
**答：** 请求量和错误率、响应时间分布、Token 使用量和成本、活跃会话数、工具调用成功率。推荐用 Prometheus + Grafana 做指标监控，LangSmith 做调用链路追踪。

### Q4: 如何防护 Prompt 注入攻击？
**答：** 检测常见注入模式（"忽略之前的指令"等）、用分隔符隔离用户输入和系统指令、对输出做敏感信息过滤（API Key、内部 URL）、工具调用做权限控制和输入验证、敏感操作需人工确认。

### Q5: Agent 服务用 Docker 部署时需要注意什么？
**答：** API Key 用环境变量不硬编码、向量数据库数据要挂载 volume 持久化、设置健康检查端点、多 worker 提高并发（uvicorn --workers）、Redis 做会话缓存、配置合理的资源限制。

### Q6: 生产环境 Agent 上线前的检查清单有哪些关键项？
**答：** 部署前：API Key 环境变量化、速率限制、健康检查、错误处理全覆盖。运行时：Token 和成本监控告警、响应时间监控、错误率监控。安全：输入验证、输出过滤、工具权限控制、审计日志。

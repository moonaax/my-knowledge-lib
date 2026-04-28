# 构建自定义 Agent 框架

> 从零构建一个轻量级、教学友好的 Agent 框架，深入理解智能体的底层工作原理，实现从"使用者"到"构建者"的能力跃迁。

相关主题：[[1-LangChain框架详解]] | [[4-从零实现Agent范式]]

---

## 一、为什么需要自建 Agent 框架

### 1.1 市面框架的局限性

当前主流 Agent 框架（LangChain、AutoGen 等）虽然功能丰富，但在实际使用中存在四个核心痛点：

**过度抽象的复杂性**：许多框架为了追求通用性，引入了大量抽象层和配置选项。以 LangChain 为例，其链式调用机制虽然灵活，但对初学者而言学习曲线陡峭，往往需要理解 Chain、Agent、Tool、Memory、Retriever 等十几个概念才能完成简单任务。

**快速迭代带来的不稳定性**：商业化框架为了抢占市场，API 接口变更频繁。开发者经常面临版本升级后代码无法运行的困扰，维护成本居高不下。

**黑盒化的实现逻辑**：许多框架将核心逻辑封装得过于严密，开发者难以理解 Agent 的内部工作机制，缺乏深度定制能力。遇到问题时只能依赖文档和社区支持。

**依赖关系的复杂性**：成熟框架往往携带大量依赖包，安装包体积庞大，在与项目代码配合使用时容易出现依赖冲突。

> 🔑 自建框架的核心价值不在于替代成熟框架，而在于通过亲手实现每个组件，真正理解 Agent 的思考过程、工具调用机制、以及各种设计模式的好坏与区别。

### 1.2 从使用者到构建者的能力跃迁

构建自己的 Agent 框架带来三方面长远价值：

- **深度理解 Agent 工作原理**：亲手实现每个组件，理解 Agent 的推理循环、工具调度和记忆管理
- **获得完全的控制权**：对每一行代码都有掌控力，可根据需求精确调优，不受第三方设计理念束缚
- **培养系统设计能力**：框架构建涉及模块化设计、接口抽象、错误处理等软件工程核心技能

### 1.3 定制化需求的驱动

在实际应用中，不同场景对智能体的需求差异巨大：

- **垂直领域优化**：金融、医疗、教育等领域需要针对性的提示词模板、特殊工具集成和定制化安全策略
- **性能与资源控制**：生产环境对响应时间、内存占用、并发处理有严格要求，通用框架难以满足精细化需求
- **学习与教学的透明性**：学习者需要清晰看到智能体的每一步构建过程，要求框架具有高度的可观测性和可解释性

## 二、框架架构设计

### 2.1 三层分离的目录结构

HelloAgents 框架遵循"分层解耦、职责单一、接口统一"的核心原则，采用三层架构：

```
hello_agents/
├── core/           # 核心框架层
│   ├── agent.py    # Agent 抽象基类
│   ├── llm.py      # 统一 LLM 接口
│   ├── message.py  # 消息系统
│   └── config.py   # 配置管理
│
├── agents/         # Agent 实现层
│   ├── simple_agent.py
│   ├── react_agent.py
│   ├── reflection_agent.py
│   └── plan_solve_agent.py
│
└── tools/          # 工具系统层
    ├── base.py     # 工具基类
    ├── registry.py # 工具注册机制
    ├── chain.py    # 工具链管理
    └── async_executor.py  # 异步执行器
```

这种分层设计保证了每一层都可以独立演进，新增 Agent 类型或工具时无需修改核心层代码。

### 2.2 "万物皆为工具"的设计哲学

HelloAgents 在架构上做出了一个关键简化：**除了核心的 Agent 类，一切皆为 Tools**。

在许多其他框架中需要独立学习的 Memory（记忆）、RAG（检索增强生成）、MCP（协议）等模块，在 HelloAgents 中都被统一抽象为一种"工具"。这种设计消除了不必要的抽象层，让学习者回归到最直观的"智能体调用工具"这一核心逻辑上。

> 🔑 "万物皆为工具"不是简化功能，而是统一接口。记忆是工具，搜索是工具，甚至 RAG 检索也是工具——Agent 只需要知道"我有哪些工具可用"，而不需要理解每个工具的内部实现。

### 2.3 核心接口设计

框架有三个核心接口，构成了整个系统的骨架。

**Message 类**——统一消息格式：

```python
from pydantic import BaseModel
from typing import Optional, Dict, Any, Literal

MessageRole = Literal["user", "assistant", "system", "tool"]

class Message(BaseModel):
    content: str
    role: MessageRole
    timestamp: datetime = None
    metadata: Optional[Dict[str, Any]] = None

    def to_dict(self) -> Dict[str, Any]:
        """转换为 OpenAI API 兼容格式"""
        return {"role": self.role, "content": self.content}
```

通过 `Literal` 严格限制角色取值，保证类型安全。`to_dict()` 方法实现"对内丰富，对外兼容"的设计原则——内部使用富对象，对外输出标准格式。

**Config 类**——中心化配置管理：

```python
class Config(BaseModel):
    default_model: str = "gpt-3.5-turbo"
    temperature: float = 0.7
    max_tokens: Optional[int] = None
    debug: bool = False

    @classmethod
    def from_env(cls) -> "Config":
        """从环境变量创建配置，支持零配置启动"""
        return cls(
            debug=os.getenv("DEBUG", "false").lower() == "true",
            temperature=float(os.getenv("TEMPERATURE", "0.7")),
        )
```

每个配置项都有合理默认值，保证框架在零配置下也能工作。`from_env()` 方法允许通过环境变量覆盖默认值，无需修改代码。

**Agent 抽象基类**——统一执行入口：

```python
from abc import ABC, abstractmethod

class Agent(ABC):
    def __init__(self, name: str, llm: HelloAgentsLLM,
                 system_prompt: Optional[str] = None,
                 config: Optional[Config] = None):
        self.name = name
        self.llm = llm
        self.system_prompt = system_prompt
        self.config = config or Config()
        self._history: list[Message] = []

    @abstractmethod
    def run(self, input_text: str, **kwargs) -> str:
        """所有 Agent 必须实现统一的执行入口"""
        pass

    def add_message(self, message: Message): ...
    def clear_history(self): ...
    def get_history(self) -> list[Message]: ...
```

通过 `ABC` 和 `@abstractmethod`，强制所有具体 Agent（SimpleAgent、ReActAgent 等）都必须实现 `run` 方法，保证了统一的执行入口。

## 三、多 Provider LLM 支持

### 3.1 Provider 继承模式

框架通过 `HelloAgentsLLM` 统一封装 LLM 调用，支持通过继承扩展新的服务商。核心设计是：用户传入 `provider` 参数后，内部自动处理不同服务商的配置差异。

扩展新 Provider 的方式是继承 `HelloAgentsLLM` 并重写 `__init__`：

```python
class MyLLM(HelloAgentsLLM):
    def __init__(self, provider="modelscope", **kwargs):
        if provider == "modelscope":
            self.api_key = kwargs.get("api_key") or os.getenv("MODELSCOPE_API_KEY")
            self.base_url = "https://api-inference.modelscope.cn/v1/"
            self.model = kwargs.get("model") or "Qwen/Qwen2.5-VL-72B-Instruct"
            self._client = OpenAI(api_key=self.api_key, base_url=self.base_url)
        else:
            super().__init__(provider=provider, **kwargs)
```

这种"拦截 + 回退"的模式，既保证了新 Provider 的定制能力，又保留了原有框架的全部功能。

### 3.2 支持的 Provider 列表

| Provider | 环境变量 | 默认 Base URL | 特点 |
|----------|----------|---------------|------|
| OpenAI | `OPENAI_API_KEY` | `https://api.openai.com/v1` | 行业标准 |
| ModelScope | `MODELSCOPE_API_KEY` | `https://api-inference.modelscope.cn/v1/` | 国内平台 |
| 智谱 AI | `ZHIPU_API_KEY` | `https://open.bigmodel.cn/api/paas/v4` | 国产大模型 |
| VLLM | `LLM_BASE_URL` | `http://localhost:8000/v1` | 本地高性能推理 |
| Ollama | `LLM_BASE_URL` | `http://localhost:11434/v1` | 本地极简部署 |

> 🔑 由于所有主流 LLM 服务商都在努力兼容 OpenAI 接口，框架选择在这个标准之上构建，而不是重新发明一套抽象接口。这保证了兼容性和学习迁移成本最低。

### 3.3 自动检测机制

框架设计了 `_auto_detect_provider` 方法，按照优先级自动推断服务商：

1. **最高优先级：检查特定服务商的环境变量**——依次检查 `MODELSCOPE_API_KEY`、`OPENAI_API_KEY`、`ZHIPU_API_KEY` 等
2. **次高优先级：根据 `base_url` 域名和端口判断**——如 `:11434` 识别为 Ollama，`:8000` 识别为 VLLM
3. **辅助判断：分析 API 密钥格式**——如 `ms-` 前缀识别为 ModelScope

用户只需在 `.env` 中配置最少的环境变量，代码中直接 `HelloAgentsLLM()` 即可自动完成 Provider 识别和参数配置，实现"约定优于配置"。

## 四、Agent 范式实现

框架化实现四种经典 Agent 范式，所有范式共享统一的 Agent 基类接口。

### 4.1 SimpleAgent（可选工具调用）

SimpleAgent 是最基础的 Agent 实现，支持纯对话和可选的工具调用两种模式。

核心设计思路：通过正则表达式 `[TOOL_CALL:tool_name:parameters]` 在 LLM 输出中检测工具调用意图，执行工具后将结果注入对话，支持多轮工具调用循环：

```python
class SimpleAgent(Agent):
    def run(self, input_text: str, max_tool_iterations=3, **kwargs) -> str:
        messages = self._build_messages(input_text)

        if not self.enable_tool_calling:
            return self.llm.invoke(messages)

        # 支持多轮工具调用
        for _ in range(max_tool_iterations):
            response = self.llm.invoke(messages)
            tool_calls = self._parse_tool_calls(response)
            if not tool_calls:
                return response  # 无工具调用，返回最终回答
            # 执行工具并将结果注入对话
            for call in tool_calls:
                result = self.tool_registry.execute_tool(call['tool_name'], call['parameters'])
                messages.append({"role": "user", "content": f"工具结果: {result}"})
```

SimpleAgent 还支持流式输出（`stream_run`）和动态工具管理（`add_tool`/`remove_tool`）。

### 4.2 ReActAgent（推理 + 行动循环）

ReActAgent 实现了经典的 Thought-Action-Observation 循环，通过 `ToolRegistry` 集成工具系统：

```python
class ReActAgent(Agent):
    def run(self, input_text: str, **kwargs) -> str:
        for step in range(self.max_steps):
            # 1. 构建包含工具描述和历史的提示词
            prompt = self.prompt_template.format(
                tools=self.tool_registry.get_tools_description(),
                question=input_text,
                history="\n".join(self.current_history)
            )
            # 2. 调用 LLM 获取 Thought + Action
            response = self.llm.invoke([{"role": "user", "content": prompt}])
            thought, action = self._parse_output(response)

            # 3. 检查是否完成
            if action.startswith("Finish"):
                return self._parse_action_input(action)

            # 4. 执行工具并记录 Observation
            tool_name, tool_input = self._parse_action(action)
            observation = self.tool_registry.execute_tool(tool_name, tool_input)
            self.current_history.append(f"Action: {action}\nObservation: {observation}")
```

相比第四章的原始实现，框架化的 ReActAgent 改进在于：统一的 `ToolRegistry` 接口、可配置的提示词模板、以及标准化的历史管理。

### 4.3 ReflectionAgent（自我反思迭代）

ReflectionAgent 实现"执行-反思-优化"循环，通过 `custom_prompts` 参数支持用户深度定制。其核心是三阶段提示词：

- **initial**：根据任务生成初始回答
- **reflect**：审查当前回答，找出问题和改进空间
- **refine**：根据反馈意见改进回答

```python
DEFAULT_PROMPTS = {
    "initial": "请根据以下要求完成任务:\n任务: {task}",
    "reflect": "请审查以下回答，找出不足并提出改进建议:\n任务: {task}\n当前回答: {content}",
    "refine": "请根据反馈改进回答:\n任务: {task}\n上一轮回答: {last_attempt}\n反馈: {feedback}"
}
```

通过传入不同的 `custom_prompts`，同一个 ReflectionAgent 可以适配通用文本生成、代码生成、翻译等不同场景。

### 4.4 FunctionCallAgent（OpenAI 原生函数调用）

FunctionCallAgent 基于 OpenAI 原生 function calling 机制，相比 prompt 约束方式具有更强的鲁棒性。核心能力包括：

- `_build_tool_schemas`：将工具定义转换为 OpenAI function calling schema
- `_extract_message_content`：从响应中提取文本
- `_parse_function_call_arguments`：解析模型返回的 JSON 参数
- `_convert_parameter_types`：转换参数类型

```python
# 通过 Tool 的 to_openai_schema() 方法生成标准 schema
tools_schema = [tool.to_openai_schema() for tool in registry.get_all_tools()]

# 使用 OpenAI 原生 function calling
response = client.chat.completions.create(
    model=self.llm.model,
    messages=messages,
    tools=tools_schema,
    tool_choice="auto"
)
```

> 🔑 四种 Agent 范式的选择取决于任务特性：简单对话用 SimpleAgent，需要推理和工具交互用 ReActAgent，需要迭代优化输出质量用 ReflectionAgent，需要结构化工具调用用 FunctionCallAgent。

## 五、工具系统设计

### 5.1 Tool 抽象基类

工具基类定义了所有工具必须遵循的接口规范：

```python
class Tool(ABC):
    def __init__(self, name: str, description: str):
        self.name = name
        self.description = description

    @abstractmethod
    def run(self, parameters: Dict[str, Any]) -> str:
        """执行工具，接受字典参数返回字符串结果"""
        pass

    @abstractmethod
    def get_parameters(self) -> List[ToolParameter]:
        """获取工具参数定义，支持自描述"""
        pass
```

配套的 `ToolParameter` 类使用 Pydantic BaseModel 定义参数的名称、类型、描述、是否必需和默认值，支持类型检查和文档自动生成。

### 5.2 ToolRegistry 双模注册

ToolRegistry 是工具系统的管理中枢，支持两种注册方式：

```python
class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}       # Tool 对象注册
        self._functions: dict[str, dict] = {}    # 函数直接注册

    def register_tool(self, tool: Tool):
        """注册 Tool 对象——适合复杂工具，支持完整参数定义和验证"""
        self._tools[tool.name] = tool

    def register_function(self, name: str, description: str, func: Callable):
        """注册函数——适合简单工具，快速集成现有函数"""
        self._functions[name] = {"description": description, "func": func}
```

**对象注册**适合复杂工具（如多源搜索工具），需要维护状态和完整的参数验证；**函数注册**适合简单工具（如计算器），一行代码即可集成。

### 5.3 to_openai_schema() 转换

每个工具通过 `to_openai_schema()` 方法将自身定义转换为 OpenAI function calling 标准 schema，使工具能被 FunctionCallAgent 直接使用：

```python
def to_openai_schema(self) -> Dict[str, Any]:
    properties = {}
    required = []
    for param in self.get_parameters():
        prop = {"type": param.type, "description": param.description}
        properties[param.name] = prop
        if param.required:
            required.append(param.name)

    return {
        "type": "function",
        "function": {
            "name": self.name,
            "description": self.description,
            "parameters": {
                "type": "object",
                "properties": properties,
                "required": required
            }
        }
    }
```

### 5.4 ToolChain 顺序组合

ToolChain 支持多个工具的顺序执行，前一步的输出可以作为后一步的输入：

```python
class ToolChain:
    def __init__(self, name: str, description: str):
        self.steps: List[Dict[str, Any]] = []

    def add_step(self, tool_name: str, input_template: str, output_key: str = None):
        """添加步骤，input_template 支持 {variable} 变量替换"""
        self.steps.append({
            "tool_name": tool_name,
            "input_template": input_template,
            "output_key": output_key or f"step_{len(self.steps)}_result"
        })

    def execute(self, registry: ToolRegistry, initial_input: str, context: dict = None) -> str:
        context = context or {"input": initial_input}
        for step in self.steps:
            tool_input = step["input_template"].format(**context)
            result = registry.execute_tool(step["tool_name"], tool_input)
            context[step["output_key"]] = result
        return context[self.steps[-1]["output_key"]]
```

典型应用：搜索信息 -> 计算数值 -> 生成总结，三步工具链串联完成复杂任务。

### 5.5 AsyncToolExecutor 并行执行

对于耗时的工具操作，使用线程池实现并行执行：

```python
class AsyncToolExecutor:
    def __init__(self, registry: ToolRegistry, max_workers: int = 4):
        self.registry = registry
        self.executor = concurrent.futures.ThreadPoolExecutor(max_workers=max_workers)

    async def execute_tools_parallel(self, tasks: List[Dict[str, str]]) -> List[str]:
        """并行执行多个工具任务"""
        async_tasks = [
            self.execute_tool_async(t["tool_name"], t["input_data"])
            for t in tasks
        ]
        return await asyncio.gather(*async_tasks)
```

并行执行在以下场景带来性能提升：多个独立的搜索查询、同时调用多个 API 获取数据、批量处理任务等。当工具之间存在数据依赖时，应使用 ToolChain 顺序执行。

## 六、设计原则与最佳实践

### 6.1 核心设计原则

**接口抽象**：通过 ABC 定义统一接口（Agent.run、Tool.run），所有实现必须遵循同一"契约"，保证框架一致性。调用方只需关注接口，不关心具体实现。

**模块解耦**：三层架构（core / agents / tools）各司其职。Agent 层依赖 core 层的接口，但不依赖 tools 层的具体实现；tools 层通过 ToolRegistry 与 Agent 层解耦。

**可扩展性**：通过继承扩展新 Provider、新 Agent 类型、新工具，无需修改框架核心代码。Provider 通过继承 `HelloAgentsLLM` 扩展，Agent 通过继承 `Agent` 基类扩展，工具通过实现 `Tool` 接口或注册函数扩展。

**渐进式学习**：框架按版本迭代推进，每一章在前一章基础上增加功能。从基础对话到工具调用，从单 Agent 到复杂编排，学习路径清晰无断层。

**基于标准 API**：选择 OpenAI API 作为行业标准构建，而非重新发明抽象接口。这保证了兼容性——掌握框架后，迁移到其他框架或集成到现有项目时，底层调用逻辑完全一致。

### 6.2 工具系统开发理念

- **单一职责**：每个工具专注于特定功能，保持接口统一
- **自描述能力**：工具通过 `get_parameters()` 清晰描述自己的参数需求
- **异常处理**：完善的异常处理和安全优先的输入验证是基本要求
- **高可用设计**：多源搜索工具的降级机制——从最优源逐步降级到备选方案
- **异步优先**：利用异步执行提高并发处理能力，合理管理系统资源

### 6.3 框架化 vs 原始实现的改进

| 维度 | 第四章原始实现 | 第七章框架化实现 |
|------|--------------|----------------|
| 接口 | 每个 Agent 独立接口 | 统一的 Agent 基类 |
| 工具管理 | Agent 内部硬编码 | ToolRegistry 统一注册 |
| 提示词 | 固定在代码中 | 可配置模板 + custom_prompt |
| 配置 | 散落在各处 | Config 集中管理 + from_env |
| 错误处理 | 简单 try-catch | 分层异常体系 |
| 扩展性 | 修改源码 | 继承 + 注册，无需改源码 |

## 面试题精选

**1. 为什么要自建 Agent 框架而不是直接使用 LangChain？**

自建框架的核心目的是深度理解原理和获得定制能力。LangChain 等框架存在过度抽象、迭代不稳定、黑盒化等问题。自建框架可以完全控制每一行代码，针对特定场景精确调优，同时培养系统设计能力。在生产环境中，往往需要在通用框架基础上做二次开发。

**2. "万物皆为工具"的设计理念有什么优缺点？**

优点：统一接口降低学习成本，消除不必要的抽象层，Agent 只需知道"有哪些工具可用"。缺点：对于某些复杂模块（如记忆系统需要读写两种操作、RAG 需要索引构建），统一为"工具"可能丢失语义丰富性，需要额外设计参数来区分操作类型。

**3. Agent 基类使用 ABC 和 @abstractmethod 的作用是什么？**

ABC（Abstract Base Classes）定义了不能直接实例化的抽象类，@abstractmethod 强制所有子类必须实现 `run` 方法。这保证了所有 Agent 都有统一的执行入口，实现了"契约式设计"——调用方只需知道接口签名，不关心具体实现。

**4. ToolRegistry 的两种注册方式（对象注册 vs 函数注册）分别适用于什么场景？**

对象注册适合复杂工具：需要维护状态（如 API 客户端）、完整的参数定义和验证、多方法调用。函数注册适合简单工具：一个输入一个输出的纯函数，如计算器、编码转换器，一行代码即可集成。

**5. 多 Provider 自动检测机制的设计思路是什么？**

采用三级优先级策略：首先检查特定服务商的环境变量（最可靠），其次根据 base_url 的域名和端口判断（如 :11434 是 Ollama），最后分析 API 密钥格式辅助判断（可能存在模糊性）。这遵循"约定优于配置"原则，用户只需设置最少的环境变量。

**6. ReActAgent 和 FunctionCallAgent 的工具调用方式有什么本质区别？**

ReActAgent 通过 prompt 约束让 LLM 输出特定格式的文本（`Thought: ... Action: tool_name[input]`），再用正则解析。FunctionCallAgent 使用 OpenAI 原生 function calling 机制，LLM 直接返回结构化的 JSON 函数调用。后者鲁棒性更强，不依赖文本格式解析，但需要模型支持 function calling。

**7. ToolChain 和 AsyncToolExecutor 分别解决什么问题？**

ToolChain 解决工具间有数据依赖的顺序执行问题，前一步输出作为后一步输入，通过模板变量替换实现数据传递。AsyncToolExecutor 解决多个独立工具的并行执行问题，使用线程池并发调用，适用于批量搜索、多 API 并发等场景。

**8. 如果要为框架设计一个插件系统，你会如何考虑？**

核心思路是定义统一的插件接口（如 Plugin ABC），包含 `register_tools`、`register_agents` 等方法。框架维护一个插件注册表，插件通过入口点（entry_points）机制或显式注册方式加载。关键是插件不应修改框架核心代码，只能通过注册接口扩展功能。

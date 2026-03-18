# 架构概述（中文文档）

本文档提供了 DeerFlow 后端架构的全面概述。

> **AI 知识点 - 微服务架构**：
> DeerFlow 采用微服务架构，将应用程序分解为松耦合服务。关键优势包括：
> 1. **可扩展性**：可以独立扩展每个服务
> 2. **模块化**：服务可以使用不同的技术栈
> 3. **故障隔离**：一个服务中的故障不会影响整个系统
> 4. **独立部署**：可以单独更新服务
>
> 参考论文：[Microservices Architecture](https://arxiv.org/abs/1702.06309)

## 系统架构

```
┌──────────────────────────────────────────────────────────────────────────┐
│                              客户端（浏览器）                             │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                          Nginx（端口 2026）                               │
│                    统一反向代理入口点                      │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  /api/langgraph/*  →  LangGraph 服务器（2024）                      │  │
│  │  /api/*            →  Gateway API（8001）                           │  │
│  │  /*                →  前端（3000）                               │  │
│  └────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────┬────────────────────────────────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          │                       │                       │
          ▼                       ▼                       ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│   LangGraph 服务器  │ │    Gateway API      │ │     前端        │
│     （端口 2024）     │ │    （端口 8001）      │ │    （端口 3000）      │
│                     │ │                     │ │                     │
│  - 代理运行时    │ │  - 模型 API       │ │  - Next.js 应用      │
│  - 线程管理      │ │  - MCP 配置       │ │  - React UI         │
│  - SSE 流式传输    │ │  - 技能管理      │ │  - 聊天界面       │
│  - 检查点保存    │ │  - 文件上传     │ │                     │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
          │                       │
          │     ┌─────────────────┘
          │     │
          ▼     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                         共享配置                              │
│  ┌─────────────────────────┐  ┌────────────────────────────────────────┐ │
│  │      config.yaml        │  │      extensions_config.json            │ │
│  │  - 模型               │  │  - MCP 服务器                         │ │
│  │  - 工具                │  │  - 技能状态                        │ │
│  │  - 沙箱              │  │                                        │ │
│  │  - 摘要        │  │                                        │ │
│  └─────────────────────────┘  └────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

> **AI 知识点 - 系统架构模式**：
>
> DeerFlow 采用多种架构模式：
>
> 1. **反向代理模式**：Nginx 统一入口点
> 2. **关注点分离**：路由分离代理和非代理请求
> 3. **配置集中化**：共享配置文件
> 4. **关注点分离**：状态、配置和持久化分离
>
> 参考文档：[Architecture Patterns](https://www.enterprise-architecture-patterns.dev/)

## 组件详情

### LangGraph 服务器

LangGraph 服务器是核心代理运行时，基于 LangGraph 构建，用于强大的多代理工作流编排。

**入口点**：`src/agents/lead_agent/agent.py:make_lead_agent`

**关键职责**：
- 代理创建和配置
- 线程状态管理
- 中间件链执行
- 工具执行编排
- 实时响应的 SSE 流式传输

**配置**：`langgraph.json`

```json
{
  "agent": {
    "type": "agent",
    "path": "src.agents:make_lead_agent"
  }
}
```

> **AI 知识点 - LangGraph 工作流**：
> LangGraph 是一个有状态的工作流框架，用于构建有状态的多代理应用程序。关键概念：
> 1. **节点**：可调用的单元（LLM、工具、代理）
> 2. **边**：连接节点的转换
> 3. **状态**：在工作流中传递的数据
> 4. **图**：节点和边的有向图
> 5. **编译**：将图转换为可执行的工作流
>
> 参考文档：[LangGraph](https://langchain-ai.github.io/langgraph/)

### Gateway API

为非代理操作提供 REST 端点的 FastAPI 应用程序。

**入口点**：`src/gateway/app.py`

**路由器**：
- `models.py` - `/api/models` - 模型列表和详情
- `mcp.py` - `/api/mcp` - MCP 服务器配置
- `skills.py` - `/api/skills` - 技能管理
- `uploads.py` - `/api/threads/{id}/uploads` - 文件上传
- `artifacts.py` - `/api/threads/{id}/artifacts` - 工件提供

### 代理架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           make_lead_agent(config)                        │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                            中间件链                              │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 1. ThreadDataMiddleware  - 初始化 workspace/uploads/outputs  │   │
│  │ 2. UploadsMiddleware     - 处理上传的文件               │   │
│  │ 3. SandboxMiddleware     - 获取沙箱环境          │   │
│  │ 4. SummarizationMiddleware - 上下文减少（如果启用）     │   │
│  │ 5. TitleMiddleware       - 自动生成标题                 │   │
│  │ 6. TodoListMiddleware    - 任务跟踪（如果 plan_mode）         │   │
│  │ 7. ViewImageMiddleware   - 视觉模型支持                 │   │
│  │ 8. ClarificationMiddleware - 处理澄清              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              代理核心                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │
│  │      模型       │  │      工具       │  │    系统提示     │   │
│  │  （来自工厂）  │  │  （配置 +   │  │  （带技能）       │   │
│  │                  │  │   MCP + builtin） │  │                      │   │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

> **AI 知识点 - 中间件模式**：
> 中间件模式允许在请求/响应处理链中插入可重用的逻辑。在 AI 代理中，中间件用于：
> 1. **上下文管理**：修改代理状态和消息
> 2. **安全检查**：验证和清理输入
> 3. **日志记录**：记录和监控代理行为
> 4. **性能优化**：缓存、批处理和请求优化
>
> 参考文档：[LangChain Middleware](https://python.langchain.com/v0.1/docs/modules/agents/middleware/)

### 线程状态

`ThreadState` 扩展 LangGraph 的 `AgentState` 并带有额外字段：

```python
class类 ThreadState(AgentState):
    # 来自 AgentState 的核心状态
    messages: list[BaseMessage]

    # DeerFlow 扩展
    sandbox: dict             # 沙箱环境信息
    artifacts: list[str]      # 生成的文件路径
    thread_data: dict         # {workspace, uploads, outputs} 路径
    title: str | None         # 自动生成的对话标题
    todos: list[dict]         # 任务跟踪（规划模式）
    viewed_images: dict       # 视觉模型图像数据
```

### 沙箱系统

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           沙箱架构                           │
└─────────────────────────────────────────────────────────────────────────┘

                      ┌─────────────────────────┐
                      │    SandboxProvider      │ （抽象）
                      │  - acquire()            │
                      │  - get()                │
                      │  - release()            │
                      └────────────┬────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                                         │
              ▼                                         ▼
┌─────────────────────────┐              ┌─────────────────────────┐
│  LocalSandboxProvider   │              │  AioSandboxProvider     │
│  (src/sandbox/local.py) │              │  (src/community/)       │
│                         │              │                         │
│  - 单例实例   │              │  - 基于 Docker         │
│  - 直接执行     │              │  - 隔离容器  │
│  - 开发使用      │              │  - 生产使用       │
└─────────────────────────┘              └─────────────────────────┘

                      ┌─────────────────────────┐
                      │        沙箱          │ （抽象）
                      │  - execute_command()    │
                      │  - read_file()          │
                      │  - write_file()         │
                      │  - list_dir()           │
                      └─────────────────────────┘
```

**虚拟路径映射**：

| 虚拟路径 | 物理路径 |
|-------------|---------------|
| `/mnt/user-data/workspace` | `backend/.deer-flow/threads/{thread_id}/user-data/workspace` |
| `/mnt/user-data/uploads` | `backend/.deer-flow/threads/{thread_id}/user-data/uploads` |
| `/mnt/user-data/outputs` | `backend/.deer-flow/threads/{thread_id}/user-data/outputs` |
| `/mnt/skills` | `deer-flow/skills/` |

> **AI 知识点 - 沙箱化**：
> 沙箱化是 AI 代理安全执行代码的关键技术。通过将代理限制在隔离环境中：
> 1. **防止代码注入**：隔离环境阻止恶意代码影响主机
> 2. **资源配额**：限制 CPU、内存和磁盘使用
> 3. **网络隔离**：控制容器的网络访问
> 4. **文件系统命名空间**：限制可访问的目录
>
> 参考论文：[Sandboxing for AI Agents](https://www.anthropic.com/research)

### 工具系统

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            工具来源                                  │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   内置工具    │  │  配置工具   │  │     MCP 工具       │
│  (src/tools/)       │  │  (config.yaml)      │  │  (extensions.json)  │
├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
│ - present_file      │  │ - web_search        │  │ - github            │
│ - ask_clarification │  │ - web_fetch         │  │ - filesystem        │
│ - view_image        │  │ - bash              │  │ - postgres          │
│                     │  │ - read_file         │  │ - brave-search      │
│                     │  │ - write_file        │  │ - puppeteer         │
│                     │  │ - str_replace       │  │ - ...               │
│                     │  │ - ls                │  │                     │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
           │                       │                       │
           └───────────────────────┴───────────────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   get_available_tools() │
                      │   (src/tools/__init__)  │
                      └─────────────────────────┘
```

> **AI 知识点 - 工具增强的 LLM**：
> 工具扩展使 LLM 能够与外部世界交互。关键原则包括：
> 1. **工具描述**：清晰的工具名称、描述和参数模式
> 2. **工具选择**：模型根据任务选择合适的工具
> 3. **工具执行**：安全地执行工具调用
> 4. **结果整合**：将工具输出整合到对话上下文中
>
> 参考论文：[Tool-Augmented Large Language Models](https://arxiv.org/abs/2304.05395)
> 参考论文：[ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629)

### 模型工厂

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          模型工厂                                   │
│                     (src/models/factory.py)                              │
└─────────────────────────────────────────────────────────────────────────┘

config.yaml:
┌─────────────────────────────────────────────────────────────────────────┐
│ models:                                                                  │
│   - name: gpt-4                                                         │
│     display_name: GPT-4                                                 │
│     use: langchain_openai:ChatOpenAI                                    │
│     model: gpt-4                                                        │
│     api_key: $OPENAI_API_KEY                                            │
│     max_tokens: 4096                                                    │
│     supports_thinking: false                                            │
│     supports_vision: true                                               │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   create_chat_model()   │
                      │  - name: str            │
                      │  - thinking_enabled     │
                      └────────────┬────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   resolve_class()       │
                      │  （反射系统）    │
                      └────────────┬────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │   BaseChatModel         │
                      │  （LangChain 实例）   │
                      └─────────────────────────┘
```

**支持的提供者**：
- OpenAI (`langchain_openai:ChatOpenAI`)
- Anthropic (`langchain_anthropic:ChatAnthropic`)
- DeepSeek (`langchain_deepseek:ChatDeepSeek`)
- 通过 LangChain 集成自定义

### MCP 集成

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          MCP 集成                                 │
│                        (src/mcp/manager.py)                              │
└─────────────────────────────────────────────────────────────────────────┘

extensions_config.json:
┌─────────────────────────────────────────────────────────────────────────┐
│ {                                                                        │
│   "mcpServers": {                                                       │
│     "github": {                                                         │
│       "enabled": true,                                                  │
│       "type": "stdio",                                                  │
│       "command": "npx",                                                 │
│       "args": ["-y", "@modelcontextprotocol/server-github"],           │
│       "env": {"GITHUB_TOKEN": "$GITHUB_TOKEN"}                          │
│     }                                                                   │
│   }                                                                     │
│ }                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                      ┌─────────────────────────┐
                      │  MultiServerMCPClient   │
                      │  （langchain-mcp-adapters）│
                      └────────────┬────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
       ┌───────────┐        ┌───────────┐        ┌───────────┐
       │  stdio    │        │   SSE     │        │   HTTP    │
       │ transport │        │ transport │        │ transport │
       └───────────┘        └───────────┘        └───────────┘
```

> **AI 知识点 - Model Context Protocol (MCP)**：
> MCP 是一个开放协议，允许 AI 模型与外部工具进行标准化交互。它定义了：
> 1. **工具发现**：代理可以动态发现可用工具
> 2. **工具调用**：标准化工具执行协议
> 3. **结果返回**：统一结果格式
> 4. **传输层**：支持 stdio、SSE 和 HTTP
>
> 参考文档：[Model Context Protocol](https://modelcontextprotocol.io/)

### 技能系统

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          技能系统                                   │
│                       (src/skills/loader.py)                             │
└─────────────────────────────────────────────────────────────────────────┘

目录结构:
┌─────────────────────────────────────────────────────────────────────────┐
│ skills/                                                                  │
│ ├── public/                        # 公共技能（已提交）           │
│ │   ├── pdf-processing/                                                 │
│ │ │   └── SKILL.md                                                    │
│ │   ├── frontend-design/`                                                │
│ │ │   └── SKILL.md                                                    │
│ │   └── ...                                                             │
│ └── custom/                        # 自定义技能（已忽略）          │
│     └── user-installed/                                                 │
│         └── SKILL.md                                                    │
└─────────────────────────────────────────────────────────────────────────┘

SKILL.md 格式:
┌─────────────────────────────────────────────────────────────────────────┐
│ ---                                                                      │
│ name: PDF 处理                                                     │
│ description: 高效处理 PDF 文档                            │
│ license: MIT                                                            │
│ allowed-tools:                                                          │
│   - read_file                                                           │
│   - write_file                                                          │
│   - bash                                                                │
│ ---                                                                      │
│                                                                          │
│ # 技能说明                                                     │
│ 注入到系统提示中的内容...                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

> **AI 知识点 - 技能系统设计**：
> 技能系统设计原则：
> 1. **可发现性**：技能应易于发现和加载
> 2. **元数据**：技能描述功能和要求
> 3. **工具约束**：技能声明其所需的工具
> 4. **上下文注入**：技能通过系统提示启用
> 5. **可扩展性**：用户可以添加自定义技能
>
> 参考论文：[Prompt Engineering for Skills](https://arxiv.org/abs/2311.12994)

### 请求流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         请求流程示例                             │
│                    用户向代理发送消息                           │
└─────────────────────────────────────────────────────────────────────────┘

1. 客户端 → Nginx
   POST /api/langgraph/threads/{thread_id}/runs
   {"input": {"messages": [{"role": "user", "content": "你好"}]}}

2. Nginx → LangGraph 服务器（2024）
   代理到 LangGraph 服务器

3. LangGraph 服务器
   a. 加载/创建线程状态
   b. 执行中间件链：
      - ThreadDataMiddleware: 设置路径
      - UploadsMiddleware: 注入文件列表
      - SandboxMiddleware: 获取沙箱
      - SummarizationMiddleware: 检查令牌限制
      - TitleMiddleware: 如果需要生成标题
      - TodoListMiddleware: 加载 todos（如果规划模式）
      - ViewImageMiddleware: 处理图像
      - ClarificationMiddleware: 检查澄清

   c. 执行代理：
      - 模型处理消息
      - 可能调用工具（bash、web_search 等）
      - 工具通过沙箱执行
      - 结果添加到消息

   d. 通过 SSE 流式传输响应

4. 客户端接收流式响应
```

### 数据流程

#### 文件上传流程

```
1. 客户端上传文件
   POST /api/threads/{thread_id}/uploads
   Content-Type: multipart/form-data

2. Gateway 接收文件
   - 验证文件
   - 存储在 .deer-flow/threads/{thread_id}/user-data/uploads/
   - 如果是文档：通过 markitdown 转换为 Markdown

3. 返回响应
   {
     "files": [{
       "filename": "doc.pdf",
       "path": ".deer-flow/.../uploads/doc.pdf",
       "virtual_path": "/mnt/user-data/uploads/doc.pdf",
       "artifact_url": "/api/threads/.../artifacts/mnt/.../doc.pdf"
     }]
   }

4. 下一次代理运行
   - UploadsMiddleware 列出文件
   - 将文件列表注入到消息中
   - 代理可以通过 virtual_path 访问
```

#### 配置重新加载

```
1. 客户端更新 MCP 配置
   PUT /api/mcp/config

2. Gateway 写入 extensions_config.json
   - 更新 mcpServers 部分
   - 文件 mtime 更改

3. MCP 管理器检测更改
   - get_cached_mcp_tools() 检查 mtime
   - 如果更改：重新初始化 MCP 客户端
   - 加载更新的服务器配置

4. 下一次代理运行使用新工具
```

## 安全考虑事项

### 沙箱隔离

- 代理代码在沙箱边界内执行
- 本地沙箱：直接执行（仅开发）
- Docker 沙箱：容器隔离（生产推荐）
- 文件操作中的路径遍历预防

> **AI 知识点 - 沙箱安全**：
> 沙箱化对于 AI 代理安全至关重要，因为：
> 1. **防止代码注入攻击**：隔离环境阻止恶意代码影响主机
> 2. **资源配额**：防止代理耗尽系统资源
> 3. **审计跟踪**：所有操作都记录并可审查
> 4. **临时性**：沙箱可以轻松销毁和重建
>
> 参考论文：[Sandboxing AI Agents](https://www.anthropic.com/research)

### API 安全

- 线程隔离：每个线程有单独的数据目录
- 文件验证：上传检安路径安全性
- 环境变量解析：密钥不存储在配置中

### MCP 安全

- 每个 MCP 服务器在自己的进程中运行
- 环境变量在运行时解析
- 服务器可以独立启用/禁用

## 性能考虑事项

### 缓存

- MCP 工具通过文件 mtime 失效缓存
- 配置加载一次，在文件更改时重新加载
- 技能在启动时解析一次，在内存中缓存

### 流式传输

- 使用 SSE 进行实时响应流式传输
- 减少到第一个令牌的时间
- 为长时间操作启用进度可见性

> **AI 知识点 - 服务器发送事件 (SSE)**：
> SSE 是一种服务器推送技术，允许服务器通过 HTTP 连接向客户端发送事件。对于 AI 应用程序：
> 1. **实时更新**：客户端接收流式响应
> 2. **低延迟**：无需轮询即可获取更新
> 3. **自动重连**：浏览器原生支持重连
> 4. **进度指示**：长时间操作可以显示进度
>
> 参考文档：[Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)

### 上下文管理

- 摘要中间件在接近限制时减少上下文
- 可配置的触发器：令牌、消息或分数
- 保留最近消息同时摘要旧消息

> **AI 知识点 - 上下文管理**：
> 随着对话的增长，上下文窗口最终会被填满。有效的上下文管理策略包括：
> 1. **摘要总结**：将早期对话压缩为简洁摘要
> 2. **滑动窗口**：保留最近 N 条消息
> 3. **优先级过滤**：根据任务相关性保留信息
> 4. **文件系统缓存**：将中间结果存储在文件中以避免重复处理
>
> 参考论文：[Context Window Management](https://arxiv.org/abs/2309.09597)

---

## 参考资源

### 重要研究论文

1. **Constitutional AI** (Anthropic, 2022)
   - 描述如何通过自我改进和宪法约束使 AI 系统更加安全、有用和诚实
   - [论文链接](https://www.anthropic.com/research/constitutional-ai-harmlessness-ai-feedback)
   - 关联：负责任的 AI 对齐、安全约束

2. **Chain-of-Thought Prompting Elicits Reasoning** (Wei et al., 2022)
   - 通过思维链提示显著提高大型语言模型的推理能力
   - [论文链接](https://arxiv.org/abs/2201.11903)
   - 关联：代理规划、任务分解

3. **Tool-Augmented Large Language Models** (Schick et al., 2023)
   - 通过工具增强大型语言模型，使其能够与外部系统交互
   - [论文链接](https://arxiv.org/abs/2304.05395)
   - 关联：工具系统、MCP 集成

4. **Memory-Augmented Large Language Models** (Lewis et al., 2020)
   - 增强大型语言模型以保留长期记忆
   - [论文链接](https://arxiv.org/abs/2104.07863)
   - 关联：持久记忆系统

### 官方文档资源

- [Anthropic Research Hub](https://www.anthropic.com/research) - Anthropic 的最新研究论文和发现
- [OpenAI Research](https://openai.com/research) - OpenAI 的研究出版物和技术报告
- [LangChain Documentation](https://python.langchain.com/) - LangChain 框架完整文档
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/) - LangGraph 工作流框架文档

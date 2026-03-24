# DeerFlow 后端（中文文档）

DeerFlow 是一个基于 LangGraph 的 AI 超级代理，具有沙箱执行、持久记忆和可扩展工具集成。后端使 AI 代理能够执行代码、浏览 Web、管理文件、将任务委派给子代理，并在对话中保留上下文——所有这些都在操作在隔离的、每线程的环境中。

> **AI 知识点 - 超级代理架构**：
> 超级代理是指超越简单聊天机器人能力，具备复杂任务分解、工具使用和自我改进能力的 AI 代理。DeerFlow 的超级代理架构包含：
> 1. **沙箱执行**：安全地运行代码和命令
> 2. **持久记忆**：跨会话保持上下文
> 3. **子代理委派**：并行处理复杂任务
> 4. **可扩展工具系统**：动态添加新能力
>
> 参考论文：[AutoGPT: Autonomous GPT-4 Agent](https://arxiv.org/abs/2304.03142)
> 参考论文：[BabyAGI: Scripted AI Agent](https://github.com/joebrock/agent)

---

## 架构

```
                        ┌──────────────────────────────┐
                        │          Nginx (端口 2026)           │
                        │      统一反向代理           │
                        └───────┬──────────────────┬───────────┘
                                │                  │
              /api/langgraph/*  │                  │  /api/* (其他)
                                ▼                  ▼
               ┌────────────────────┐  ┌────────────────────────┐
               │ LangGraph 服务器   │  │   Gateway API (8001)   │
               │    (端口 2024)     │  │   FastAPI REST         │
               │                    │  │                        │
               │ ┌────────────────┐ │  │ Models, MCP, Skills,   │
               │ │  主导代理    │ │  │ Memory, Uploads,       │
               │ │  ┌──────────┐  │ │  │ Artifacts              │
               │ │  │中间件链│  │ │  │  └────────────────────────┘
               │ │  │  Chain   │  │ │  │
               │ │  └──────────┘  │ │  │
               │ │  ┌──────────┐  │ │  │
               │ │  │  工具   │  │ │  │
               │ │  └──────────┘  │ │  │
               │ │  ┌──────────┐  │ │  │
               │ │  │子代理   │  │ │  │
               │ │  └──────────┘  │ │  │
               │ └────────────────┘ │
               └────────────────────┘
```

**请求路由**（通过 Nginx）：
- `/api/langgraph/*` → LangGraph 服务器 - 代理交互、线程、流式传输
- `/api/*`（其他）→ Gateway API - 模型、MCP、技能、记忆、工件、上传
- `/`（非 API）→ 前端 - Next.js Web 界面

> **AI 知识点 - 反向代理模式**：
> 反向代理模式是分布式系统中的常见模式，它代表客户端访问后端服务。对于 AI 应用程序，反向代理提供：
> 1. **统一入口点**：客户端无需知道多个服务地址
> 2. **负载均衡**：在多个实例之间分配请求
> 3. **SSL 终止**：集中处理加密
> 4. **请求路由**：基于路径将请求定向到适当的服务
>
> 参考文档：[Nginx Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

---

## 核心组件

### 主导代理

单个 LangGraph 代理（`lead_agent`）是运行时入口点，通过 `make_lead_agent(config)` 创建。它结合了：

- **动态模型选择**，支持思维和视觉能力
- **中间件链**，用于横切关注点（9 个中间件）
- **工具系统**，包括沙箱、MCP、社区和内置工具
- **子代理委派**，用于并行任务执行
- **系统提示**，注入技能、记忆上下文和工作目录指导

> **AI 知识点 - 动态模型选择**：
> 根据任务需求动态选择最合适的模型是一种优化策略。这可以：
> 1. **成本优化**：对简单任务使用更便宜的模型
> 2. **性能优化**：对复杂推理使用更强大的模型
> 3. **功能匹配**：根据任务需要选择支持视觉/思维的模型
>
> 参考论文：[Model Routing for LLMs](https://arxiv.org/abs/2311.08362)

### 中间件链

中间件按严格顺序执行，每个处理特定关注点：

| # | 中间件 | 目的 |
|---|-----------|---------|
| 1 | **ThreadDataMiddleware** | 创建每线程隔离目录（workspace、uploads、outputs） |
| 2 | **UploadsMiddleware** | 将新上传的文件注入到对话上下文 |
| 3 | **SandboxMiddleware** | 获取代码执行的沙箱环境 |
| 4 | **SummarizationMiddleware** | 在接近令牌限制时减少上下文（可选） |
| 5 | **TodoListMiddleware** | 在规划模式下跟踪多步骤任务（可选） |
| 6 | **TitleMiddleware** | 在第一次交换后自动生成对话标题 |
| 7 | **MemoryMiddleware** | 将对话排队用于异步记忆提取 |
| 8 | **ViewImageMiddleware** | 为支持视觉的模型注入图像数据（条件） |
| 9 | **ClarificationMiddleware** | 拦截澄清请求并中断执行（必须最后） |

> **AI 知识点 - 中间件模式**：
> 中间件模式允许在请求/响应处理链中插入可重用的逻辑。在 AI 代理中，中间件用于：
> 1. **上下文管理**：修改代理状态和消息
> 2. **安全检查**：验证和清理输入
> 3. **日志记录**：记录和监控代理行为
> 4. **性能优化**：缓存、批处理和请求优化
>
> 参考文档：[LangChain Middleware](https://python.langchain.com/v0.1/docs/modules/agents/middleware/)

### 沙箱系统

每线程隔离执行，具有虚拟路径转换：

- **抽象接口**：`execute_command`、`read_file`、`write_file`、`list_dir`
- **提供者**：`LocalSandboxProvider`（文件系统）和 `AioSandboxProvider`（Docker，在 community/ 中）
- **虚拟路径**：`/mnt/user-data/{workspace,uploads,outputs}` → 线程特定的物理目录
- **技能路径**：`/mnt/skills` → `deer-flow/skills/` 目录
- **技能加载**：递归发现 `skills/{public,custom}` 下嵌套的 `SKILL.md` 文件并保留嵌套的容器路径
- **工具**：`bash`、`ls`、`read_file`、`write_file`、`str_replace`

> **AI 知识点 - 沙箱化**：
> 沙箱化是 AI 代理安全执行代码的关键技术。通过将代理限制在隔离环境中：
> 1. **防止代码注入**：恶意代码无法影响主机系统
> 2. **资源限制**：限制 CPU、内存和磁盘使用
> 3. **网络隔离**：控制网络访问权限
> 4. **文件系统隔离**：限制可访问的目录
>
> 参考论文：[Sandboxing for AI Agents](https://www.anthropic.com/research)

### 子代理系统

具有并发执行的异步任务委派：

- **内置代理**：`general-purpose`（完整工具集）和 `bash`（命令专家）
- **并发**：每轮最多 3 个子代理，15 分钟超时
- **执行**：带有状态跟踪和 SSE 事件的后台线程池
- **流程**：代理调用 `task()` 工具 → 执行器在后台运行子代理 → 轮询完成 → 返回结果

> **AI 知识点 - 多代理系统**：
> 多代理系统（MAS）是 AI 研究中的活跃领域，专注于代理之间的协调和通信。关键优势包括：
> 1. **并行处理**：多个代理同时处理任务的不同部分
> 2. **专业化**：不同代理专注于不同领域
> 3. **鲁棒性**：单个代理的故障不会导致系统失败
> 4. **可扩展性**：可以通过添加更多代理来扩展系统
>
> 参考论文：[Multi-Agent Systems](https://arxiv.org/abs/2305.16753)
> 参考文档：[LangGraph](https://langchain-ai.github.io/langgraph/)

### 记忆系统

基于 LLM 的跨对话持久上下文保留：

- **自动提取**：分析对话以获取用户上下文、事实和偏好
- **结构化存储**：用户上下文（工作、个人、首要关注点）、历史记录和置信度评分的事实
- **防抖更新**：批量更新以最小化 LLM 调用（可配置等待时间）
- **系统提示注入**：顶部事实 + 上下文注入到代理提示中
- **存储**：JSON 文件，基于 mtime 的缓存失效

> **AI 知识点 - 记忆增强的 LLM**：
> 记忆系统使 AI 代理能够跨会话保持个性化和连续性。关键组件包括：
> 1. **短期记忆**：当前对话上下文
> 2. **长期记忆**：持久存储的用户偏好和事实
> 3. **向量数据库**：用于语义相似性搜索
> 4. **检索策略**：基于相关性选择要注入的记忆
>
> 参考论文：[Memory-Augmented Large Language Models](https://arxiv.org/abs/2104.07863)
> 参考论文：[Retrieval-Augmented Generation](https://arxiv.org/abs/2005.11401)

### 工具生态系统

| 类别 | 工具 |
|----------|-------|
| **沙箱** | `bash`、`ls`、`read_file`、`write_file`、`str_replace` |
| **内置** | `present_files`、`ask_clarification`、`view_image`、`task`（子代理） |
| **社区** | Tavily（Web 搜索）、Jina AI（Web 获取）、Firecrawl（爬取）、DuckDuckGo（图像搜索） |
| **MCP** | 任何 Model Context Protocol 服务器（stdio、SSE、HTTP 传输） |
| **技能** | 通过系统提示注入的领域特定工作流程 |

> **AI 知识点 - 工具增强的 LLM**：
> 工具扩展使 LLM 能够与外部世界交互。关键原则包括：
> 1. **工具描述**：清晰的工具名称、描述和参数模式
> 2. **工具选择**：模型根据任务选择合适的工具
> 3. **工具执行**：安全地执行工具调用
> 4. **结果整合**：将工具输出整合到对话上下文中
>
> 参考论文：[Tool-Augmented Large Language Models](https://arxiv.org/abs/2304.05395)
> 参考论文：[ReAct: Synergizing Reasoning and Acting](https://arxiv.org/abs/2210.03629)

### Gateway API

为前端集成提供 REST 端点的 FastAPI 应用程序：

| 路由 | 目的 |
|-------|---------|
| `GET /api/models` | 列出可用的 LLM 模型 |
| `GET/PUT /api/mcp/config` | 管理 MCP 服务器配置 |
| `GET/PUT /api/skills` | 列出和管理技能 |
| `POST /api/skills/install` | 从 `.skill` 存档安装技能 |
| `GET /api/memory` | 检索记忆数据 |
| `POST /api/memory/reload` | 强制记忆重新加载 |
| `GET /api/memory/config` | 记忆配置 |
| `GET /api/memory/status` | 组合配置 + 数据 |
| `POST /api/threads/{id}/uploads` | 上传文件（自动转换 PDF/PPT/Excel/Word 为 Markdown） |
| `GET /api/threads/{id}/uploads/list` | 列出已上传的文件 |
| `GET /api/threads/{id}/artifacts/{path}` | 提供生成的工件 |

> **AI 知识点 - API 网关模式**：
> API 网关充当面向前后端服务的单一入口点。对于 AI 应用程序，网关处理：
> 1. **身份验证和授权**：验证用户身份
> 2. **速率限制**：防止 API 滥用
> 3. **请求/响应转换**：修改数据格式
> 4. **服务发现**：动态路由到可用服务
>
> 参考文档：[API Gateway Pattern](https://aws.amazon.com/cn/api-gateway/)

---

## 快速开始

### 前提条件

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) 包管理器
- 所选 LLM 提供商的 API 密钥

### 安装

```bash
cd deer-flow

# 复制配置文件
cp config.example.yaml config.yaml

# 安装后端依赖项
cd backend
make install
```

### 配置

在项目根目录中编辑 `config.yaml`：

```yaml
models:
  - name: gpt-4o
    display_name: GPT-4o
    use: langchain_openai:ChatOpenAI
    model: gpt-4o
    api_key: $OPENAI_API_KEY
    supports_thinking: false
    supports_vision: true
```

设置您的 API 密钥：

```bash
export OPENAI_API_KEY="your-api-key-here"
```

### 运行

**完整应用程序**（从项目根目录）：

```bash
make dev  # 启动 LangGraph + Gateway + 前端 + Nginx
```

访问地址：http://localhost:2026

**仅后端**（从后端目录）：

```bash
# 终端 1：LangGraph 服务器
make dev

# 终端 2：Gateway API
make gateway
```

直接访问：LangGraph 在 http://localhost:2024，Gateway 在 http://localhost:8001

---

## 项目结构

```
backend/
├── src/
│   ├── agents/                  # 代理系统
│   │   ├── lead_agent/         # 主代理（工厂、提示）
│   │   ├── middlewares/        # 9 个中间件组件
│   │   ├── memory/             # 记忆提取和存储
│   │   └── thread_state.py    # ThreadState 架构
│   ├── gateway/                # FastAPI Gateway API
│   │   ├── app.py             # 应用程序设置
│   │   └── routers/           # 6 个路由模块
│   ├── sandbox/                # 沙箱执行系统
│   │   ├── local/             # 本地文件系统提供者
│   │   ├── sandbox.py         # 抽象接口
│   │   ├── tools.py           # bash、ls、read/write/str_replace
│   │   └── middleware.py      # 沙箱生命周期
│   ├── subagents/              # 子代理委派系统
│   │   ├── builtins/          # general-purpose、bash 代理
│   │   ├── executor.py        # 后台执行引擎
│   │   └── registry.py        # 代理注册表
│   ├── tools/builtins/         # 内置工具
│   ├── mcp/                    # MCP 协议集成
│   ├── models/                 # 模型工厂
│   ├── skills/                 # 技能发现和加载
│   ├── config/                 # 配置系统
│   ├── community/              # 社区工具和提供者
│   ├── reflection/             # 动态模块加载
│   └── utils/                  # 工具
├── docs/                       # 文档
├── tests/                      # 测试套件
├── langgraph.json              # LangGraph 服务器配置
├── pyproject.toml              # Python 依赖项
├── Makefile                    # 开发命令
└── Dockerfile                  # 容器构建
```

---

## 配置

### 主配置（`config.yaml`）

放置在项目根目录中。以 `$` 开头的配置值解析为环境变量。

关键部分：
- `models` - 具有类路径、API 密钥、思维/视觉标志的 LLM 配置
- `tools` - 具有模块路径和组的工具定义
- `tool_groups` - 逻辑工具分组
- `sandbox` - 执行环境提供者
- `skills` - 技能目录路径
- `title` - 自动标题生成设置
- `summarization` - 上下文摘要设置
- `subagents` - 子代理系统（启用/禁用）
- `memory` - 记忆系统设置（启用、存储、防抖、事实限制）

提供者注意：
- `models[*].use` 通过模块路径引用提供者类（例如 `langchain_openai:ChatOpenAI`）。
- 如果提供者模块丢失，DeerFlow 现在返回可操作的错误并提供安装指导（例如 `uv add langchain-google-genai`）。

### 扩展配置（`extensions_config.json`）

单个文件中的 MCP 服务器和技能状态：

```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {"GITHUB_TOKEN": "$GITHUB_TOKEN"}
    },
    "secure-http": {
      "enabled": true,
      "type": "http",
      "url": "https://api.example.com/mcp",
      "oauth": {
        "enabled": true,
        "token_url": "https://auth.example.com/oauth/token",
        "grant_type": "client_credentials",
        "client_id": "$MCP_OAUTH_CLIENT_ID",
        "client_secret": "$MCP_OAUTH_CLIENT_SECRET"
      }
    }
  },
  "skills": {
    "pdf-processing": {"enabled": true}
  }
}
```

### 环境变量

- `DEER_FLOW_CONFIG_PATH` - 覆盖 config.yaml 位置
- `DEER_FLOW_EXTENSIONS_CONFIG_PATH` - 覆盖 extensions_config.json 位置
- 模型 API 密钥：`OPENAI_API_KEY`、`ANTHROPIC_API_KEY`、`DEEPSEEK_API_KEY` 等
- 工具 API 密钥：`TAVILY_API_KEY`、`GITHUB_TOKEN` 等

---

## 开发

### 命令

```bash
make install    # 安装依赖项
make dev        # 运行 LangGraph 服务器（端口 2024）
make gateway    # 运行 Gateway API（端口 8001）
make lint       # 运行 linter（ruff）
make format     # 格式化代码（ruff）
```

### 代码风格

- **Linter/Formatter**：`ruff`
- **行长**：240 个字符
- **Python**：3.12+ 带类型提示
- **引号**：双引号
- **缩进**：4 个空格

### 测试

```bash
uv run pytest
```

> **AI 知识点 - 测试策略**：
> 对于 AI 应用程序，全面的测试至关重要：
> 1. **单元测试**：测试单个函数和类
> 2. **集成测试**：测试组件之间的交互
> 3. **端到端测试**：测试完整用户工作流
> 4. **幻觉测试**：验证模型输出准确性
> 5. **安全测试**：测试沙箱隔离和输入验证
>
> 参考论文：[Testing AI Systems](https://arxiv.org/abs/2311.07563)

---

## 技术栈

- **LangGraph** (1.0.6+) - 代理框架和多代理编排
- **LangChain** (1.2.3+) - LLM 抽象和工具系统
- **FastAPI** (0.115.0+) - Gateway REST API
- **langchain-mcp-adapters** - Model Context Protocol 支持
- **agent-sandbox** - 沙箱化代码执行
- **markitdown** - 多格式文档转换
- **tavily-python** / **firecrawl-py** - Web 搜索和爬取

> **AI 知识点 - LangGraph**：
> LangGraph 是由 LangChain 构建的有状态的多代理工作流框架。关键优势包括：
> 1. **有状态管理**：工作流具有明确定义的状态和转换
> 2. **可视化**：可以生成工作流图以进行调试
> 3. **持久化**：状态可以保存和恢复
> 4. **人机协同**：支持人工输入和干预
>
> 参考文档：[LangGraph](https://langchain-ai.github.io/langgraph/)

---

## 文档

- [配置指南](docs/CONFIGURATION.md)
- [架构详情](docs/ARCHITECTURE.md)
- [API 参考](docs/API.md)
- [文件上传](docs/FILE_UPLOAD.md)
- [路径示例](docs/PATH_EXAMPLES.md)
- [上下文摘要](docs/summarization.md)
- [规划模式](docs/plan_mode_usage.md)
- [设置指南](docs/SETUP.md)

---

## 许可证

请参阅项目根目录中的 [LICENSE](../LICENSE) 文件。

## 贡献

请参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解贡献指南。

---

## 附加 AI 参考资源

### 重要研究论文

1. **Chain-of-Thought Prompting Elicits Reasoning** (Wei et al., 2022)
   - 通过思维链提示显著提高大型语言模型的推理能力
   - [论文链接](https://arxiv.org/abs/2201.11903)
   - 关联：代理规划、任务分解

2. **ReAct: Synergizing Reasoning and Acting** (Yao et al., 2022)
   - 结合推理和行动的框架，实现代理进行迭代决策
   - [论文链接](https://arxiv.org/abs/2210.03629)
   - 关联：工具使用、代理行为

3. **Tool-Augmented Large Language Models** (Schick et al., 2023)
   - 通过工具增强大型语言模型，使其能够与外部系统交互
   - [论文链接](https://arxiv.org/abs/2304.05395)
   - 关联：工具系统、MCP 集成

4. **Constitutional AI** (Anthropic, 2022)
   - 通过自我改进训练和宪法约束使 AI 系统更加安全、有用和诚实
   - [论文链接](https://www.anthropic.com/research/constitutional-ai-harmlessness-ai-feedback)
   - 关联：负责任的 AI 对齐、安全约束

### 官方文档资源

- [Anthropic Research Hub](https://www.anthropic.com/research) - Anthropic 的最新研究论文和发现
- [OpenAI Research](https://openai.com/research) - OpenAI 的研究出版物和技术报告
- [LangChain Documentation](https://python.langchain.com/) - LangChain 框架完整文档
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/) - LangGraph 工作流框架文档

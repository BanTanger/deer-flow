# 🦌 DeerFlow - 2.0 中文文档

<a href="https://trendshift.io/repositories/14699" target="_blank"><img src="https://trendshift.io/api/badge/repositories/14699" alt="bytedance%2Fdeer-flow | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>

> 2026 年 2 月 28 日，DeerFlow 发布 2.0 版本后荣登 GitHub Trending 榜首 🏆。感谢我们了不起的社区——是你们让这一切成为可能！💪🔥

DeerFlow (**D**eep **E**xploration and **E**fficient **R**esearch **Flow**) 是一个开源的**超级代理框架**（Super Agent Harness），它编排**子代理**（sub-agents）、**记忆**（memory）和**沙箱**（sandboxes）来完成几乎任何事情——由**可扩展技能**（extensible skills）驱动。

https://github.com/user-attachments/assets/a8bcadc4-e040-4cf2-8fda-dd768b999c18

> [!NOTE]
> **DeerFlow 2.0 是从头重写的版本。** 它与 v1 版本没有任何共享代码。如果您在寻找原始的 Deep Research 框架，它仍在 [`1.x` 分支](https://github.com/bytedance/deer-flow/tree/main-1.x) 上维护——欢迎继续贡献。活跃开发已转移到 2.0。

## 官方网站

在我们的官方网站了解更多信息并查看**真实演示**。

**[deerflow.tech](https://deerflow.tech/)**

## InfoQuest

DeerFlow 新集成了由 BytePlus 独立开发的智能搜索和爬虫工具套件——[InfoQuest（支持免费在线体验）](https://docs.byteplus.com/en/docs/InfoQuest/What_is_Info_Quest)

<a href="https://docs.byteplus.com/en/docs/InfoQuest/What_is_Info_Quest" target="_blank">
  <img
    src="https://sf16-sg.tiktokcdn.com/obj/eden-sg/hubseh7bsbps/20251208-160108.png"   alt="InfoQuest_banner"
  />
</a>

---

## 目录

- [🦌 DeerFlow - 2.0](#-deerflow---20)
  - [官方网站](#官方网站)
  - [InfoQuest](#infoquest)
  - [目录](#目录)
  - [快速开始](#快速开始)
    - [配置](#配置)
    - [运行应用](#运行应用)
      - [选项 1：Docker（推荐）](#选项-1docker推荐)
      - [选项 2：本地开发](#选项-2本地开发)
    - [高级功能](#高级功能)
      - [沙箱模式](#沙箱模式)
      - [MCP 服务器](#mcp-服务器)
    - [即时通讯渠道](#即时通讯渠道)
  - [从深度研究到超级代理框架](#从深度研究到超级代理框架)
  - [核心功能](#核心功能)
    - [技能与工具](#技能与工具)
      - [Claude Code 集成](#claude-code-集成)
    - [子代理](#子代理)
    - [沙箱与文件系统](#沙箱与文件系统)
    - [上下文工程](#上下文工程)
    - [长期记忆](#长期记忆)
  - [推荐模型](#推荐模型)
  - [嵌入式 Python 客户端](#嵌入式-python-客户端)
  - [文档](#文档)
  - [贡献](#贡献)
  - [许可证](#许可证)
  - [致谢](#致谢)
    - [主要贡献者](#主要贡献者)
  - [Star 历史](#star-历史)

## 快速开始

### 配置

1. **克隆 DeerFlow 仓库**

   ```bash
   git clone https://github.com/bytedance/deer-flow.git
   cd deer-flow
   ```

2. **生成本地配置文件**

   从项目根目录（`deer-flow/`）运行：

   ```bash
   make config
   ```

   此命令基于提供的示例模板创建本地配置文件。

3. **配置您偏好的模型**

   编辑 `config.yaml` 并定义至少一个模型：

   ```yaml
   models:
     - name: gpt-4                       # 内部标识符
       display_name: GPT-4               # 人类可读名称
       use: langchain_openai:ChatOpenAI  # LangChain 类路径
       model: gpt-4                      # API 的模型标识符
       api_key: $OPENAI_API_KEY          # API 密钥（推荐：使用环境变量）
       max_tokens: 4096                  # 每个请求的最大令牌数
       temperature: 0.7                  # 采样温度
   ```

   > **AI 知识点 - Temperature 参数**：
   > Temperature 控制模型输出的随机性。值越低（接近 0），输出越确定和集中；值越高（接近 1），输出越随机和多样化。对于需要精确性的任务（如代码生成），使用较低温度；对于创造性任务（如故事写作），使用较高温度。
   >
   > 参考论文：[Temperature in Neural Network Sampling](https://arxiv.org/abs/1906.04059)

4. **为您配置的模型设置 API 密钥**

   选择以下方法之一：

- 选项 A：编辑项目根目录中的 `.env` 文件（推荐）


   ```bash
   TAVILY_API_KEY=your-tavily-api-key
   OPENAI_API_KEY=your-openai-api-key
   # 根据需要添加其他提供商提供密钥
   INFOQUEST_API_KEY=your-infoquest-api-key
   ```

- 选项 B：在您的 shell 中导出环境变量

   ```bash
   export OPENAI_API_KEY=your-openai-api-key
   ```

- 选项 C：直接编辑 `config.yaml`（不推荐用于生产环境）

   ```yaml
   models:
     - name: gpt-4
       api_key: your-actual-api-key-here  # 替换占位符
   ```

### 运行应用

#### 选项 1：Docker（推荐）

以一致环境快速启动的最快方式：

1. **初始化并启动**：
   ```bash
   make docker-init    # 拉取沙箱镜像（仅首次或镜像更新时）
   make docker-start   # 启动服务（从 config.yaml 自动检测沙箱模式）
   ```

   `make docker-start` 现在仅在 `config.yaml` 使用 provisioner 模式时才启动 `provisioner`（`sandbox.use: src.community.aio_sandbox:AioSandboxProvider` 和 `provisioner_url`）。

2. **访问**：http://localhost:2026

有关详细的 Docker 开发指南，请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)。

#### 选项 2：本地开发

如果您更喜欢在本地运行服务：

前提条件：首先完成"配置"步骤（`make config` 和模型 API 密钥）。`make dev` 需要有效的配置文件（默认为项目根目录中的 `config.yaml`；可通过 `DEER_FLOW_CONFIG_PATH` 覆盖）。

1. **检查前提条件**：
   ```bash
   make check  # 验证 Node.js 22+、pnpm、uv、nginx
   ```

2. **安装依赖项**：
   ```bash
   make install  # 安装后端 + 前端依赖项
   ```

3. **（可选）预拉取沙箱镜像**：
   ```bash
   # 如果使用 Docker/基于容器的沙箱，推荐
   make setup-sandbox
   ```

4. **启动服务**：
   ```bash
   make dev
   ```

5. **访问**：http://localhost:2026

### 高级功能

#### 沙箱模式

DeerFlow 支持多种沙箱执行模式：
- **本地执行**（在主机上直接运行沙箱代码）
- **Docker 执行**（在隔离的 Docker 容器中运行沙箱代码）
- **带 Kubernetes 的 Docker 执行**（通过 provisioner 服务在 Kubernetes pod 中运行沙箱代码）

对于 Docker 开发，服务启动遵循 `config.yaml` 沙箱模式。在本地/Docker 模式下，不启动 `provisioner`。

请参阅[沙箱配置指南](backend/docs/CONFIGURATION.md#sandbox)来配置您偏好的模式。

> **AI 知识点 - 沙箱化执行**：
> 沙箱化是 AI 代理安全执行代码的关键技术。通过将代理限制在隔离环境中，可以防止恶意或意外代码影响主机系统。Docker 容器提供进程隔离、资源限制和文件系统命名空间，是沙箱的理想实现。
>
> 参考论文：[Sandboxing for AI Agents](https://www.anthropic.com/research)

#### MCP 服务器

DeerFlow 支持可配置的 MCP 服务器和技能来扩展其功能。
对于 HTTP/SSE MCP 服务器，支持 OAuth 令牌流（`client_credentials`、`refresh_token`）。
有关详细说明，请参阅 [MCP 服务器指南](backend/docs/MCP_SERVER.md)。

> **AI 知识点 - Model Context Protocol (MCP)**：
> MCP 是一个开放协议，允许 AI 模型与外部工具和数据源进行标准化交互。它定义了工具发现、调用和结果返回的统一接口，使 AI 代理能够无缝集成各种服务。
>
> 参考文档：[Model Context Protocol](https://modelcontextprotocol.io/)

#### 即时通讯渠道

DeerFlow 支持从消息应用接收任务。渠道在配置后自动启动——任何渠道都不需要公共 IP。

| 渠道 | 传输方式 | 难度 |
|---------|-----------|------------|
| Telegram | Bot API（长轮询） | 简单 |
| Slack | Socket Mode | 中等 |
| 飞书 / Lark | WebSocket | 中等 |

**在 `config.yaml` 中配置：**

```yaml
channels:
  # LangGraph 服务器 URL（默认：http://localhost:2024）
  langgraph_url: http://localhost:2024
  # Gateway API URL（默认：http://localhost:8001）
  gateway_url: http://localhost:8001

  # 可选：所有移动渠道的全局会话默认设置
  session:
    assistant_id: lead_agent
    config:
      recursion_limit: 100
    context:
      thinking_enabled: true
      is_plan_mode: false
      subagent_enabled: false

  feishu:
    enabled: true
    app_id: $FEISHU_APP_ID
    app_secret: $FEISHU_APP_SECRET

  slack:
    enabled: true
    bot_token: $SLACK_BOT_TOKEN     # xoxb-...
    app_token: $SLACK_APP_TOKEN     # xapp-...（Socket Mode）
    allowed_users: []               # 空 = 允许所有

  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: []               # 空 = 允许所有

    # 可选：每个渠道 / 用户的会话设置
    session:
      assistant_id: mobile_agent
      context:
        thinking_enabled: false
      users:
        "123456789":
          assistant_id: vip_agent
          config:
            recursion_limit: 150
          context:
            thinking_enabled: true
            subagent_enabled: true
```

在您的 `.env` 文件中设置相应的 API 密钥：

```bash
# Telegram
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ

# Slack
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...

# 飞书 / Lark
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=your_app_secret
```

**Telegram 设置**

1. 与 [@BotFather](https://t.me/BotFather) 聊天，发送 `/newbot`，并复制 HTTP API 令牌。
2. 在 `.env` 中设置 `TELEGRAM_BOT_TOKEN` 并在 `config.yaml` 中启用渠道。

**Slack 设置**

1. 在 [api.slack.com/apps](https://api.slack.com/apps) 创建一个 Slack App → Create New App → From scratch。
2. 在 **OAuth & Permissions** 下，添加 Bot Token Scopes：`app_mentions:read`、`chat:write`、`im:history`、`im:read`、`im:write`、`files:write`。
3. 启用 **Socket Mode** → 生成具有 `connections:write` 范围的应用级别令牌（`xapp-…`）。
4. 在 **Event Subscriptions** 下，订阅机器人事件：`app_mention`、`message.im`。
5. 在 `.env` 中设置 `SLACK_BOT_TOKEN` 和 `SLACK_APP_TOKEN` 并在 `config.yaml` 中启用渠道。

**飞书 / Lark 设置**

1. 在[飞书开放平台](https://open.feishu.cn/) 创建一个应用 → 启用 **Bot** 能力。
2. 添加权限：`im:message`、`im:message.p2p_msg:readonly`、`im:resource`。
3. 在 **事件** 下，订阅 `im.message.receive_v1` 并选择 **长连接** 模式。
4. 复制应用 ID 和应用密钥。在 `.env` 中设置 `FEISHU_APP_ID` 和 `FEISHU_APP_SECRET` 并在 `config.yaml` 中启用渠道。

**命令**

一旦连接了渠道，您可以直接从聊天中与 DeerFlow 交互：

| 命令 | 描述 |
|---------|-------------|
| `/new` | 开始新对话 |
| `/status` | 显示当前线程信息 |
| `/models` | 列出可用模型 |
| `/memory` | 查看记忆 |
| `/help` | 显示帮助 |

> 没有命令前缀的消息被视为普通聊天——DeerFlow 创建一个线程并以对话方式响应。

## 从深度研究到超级代理框架

DeerFlow 起初是一个深度研究框架——社区将其发扬光大。自发布以来，开发者们将其推向了研究之外：构建数据管道、生成幻灯片、启动仪表板、自动化内容工作流。这些是我们未曾预料的事情。

这告诉我们一个重要的信息：DeerFlow 不仅仅是一个研究工具。它是一个**框架**（harness）——一个为代理提供基础设施以实际完成工作的运行时。

所以我们从头重建了它。

DeerFlow 2.0 不再是您需要连接在一起的框架。它是一个超级代理框架——开箱即用，完全可扩展。基于 LangGraph 和 LangChain 构建，它内置了代理所需的一切：文件系统、记忆、技能、沙箱化执行，以及规划和为复杂多步骤任务生成子代理的能力。

按原样使用它。或者拆解它并使其成为您自己的。

## 核心功能

### 技能与工具

技能是 DeerFlow 能够做*几乎任何事情*的原因。

标准的代理技能是一个结构化的功能模块——一个定义工作流程、最佳实践和支持资源引用的 Markdown 文件。DeerFlow 内置了研究、报告生成、幻灯片创建、网页、图像和视频生成等技能。但真正的力量在于可扩展性：添加您自己的技能、替换内置技能或将它们组合成复合工作流程。

技能是渐进式加载的——仅在任务需要时加载，而不是一次性全部加载。这保持上下文窗口精简，使 DeerFlow 即使对令牌敏感的模型也能良好工作。

工具遵循相同理念。DeerFlow 带有核心工具集——Web 搜索、Web 获取、文件操作、bash 执行——并支持通过 MCP 服务器和 Python 函数的自定义工具。交换任何东西。添加任何东西。

```
# 沙箱容器内的路径
/mnt/skills/public
├── research/SKILL.md
├── report-generation/SKILL.md
├── slide-creation/SKILL.md
├── web-page/SKILL.md
└── image-generation/SKILL.md

/mnt/skills/custom
└── your-custom-skill/SKILL.md      ← 您的技能
```

> **AI 知识点 - 技能系统设计**：
> 技能系统是代理编程中的模块化方法，类似于软件工程中的插件架构。通过将功能分解为独立的技能，系统可以动态加载和卸载能力，提高可维护性和可扩展性。这与人类使用"工具"解决问题的认知模型相一致。
>
> 参考论文：[Tool-Augmented Large Language Models](https://arxiv.org/abs/2304.05395)

#### Claude Code 集成

`claude-to-deerflow` 技能让您可以直接从 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 与正在运行的 DeerFlow 实例交互。发送研究任务、检查状态、管理线程——所有操作都不需要离开终端。

**安装技能**：

```bash
npx skills add https://github.com/bytedance/deer-flow --skill claude-to-deerflow
```

然后确保 DeerFlow 正在运行（默认在 `http://localhost:2026`）并在 Claude Code 中使用 `/claude-to-deerflow` 命令。

**您可以做什么**：
- 向 DeerFlow 发送消息并获取流式响应
- 选择执行模式：flash（快速）、standard、pro（规划）、ultra（子代理）
- 检查 DeerFlow 健康状况、列出模型/技能/代理
- 管理线程和对话历史
- 上传文件进行分析

**环境变量**（可选，用于自定义端点）：

```bash
DEERFLOW_URL=http://localhost:2026            # 统一代理基础 URL
DEERFLOW_GATEWAY_URL=http://localhost:2026    # Gateway API
DEERFLOW_LANGGRAPH_URL=http://localhost:2026/api/langgraph  # LangGraph API
```

有关完整 API 参考，请参阅 [`skills/public/claude-to-deerflow/SKILL.md`](skills/public/claude-to-deerflow/SKILL.md)。

### 子代理

复杂的任务很少能一次性完成。DeerFlow 将它们分解。

主导代理可以即时生成子代理——每个都有自己的作用域上下文、工具和终止条件。子代理在可能的情况下并行运行，报告结构化结果，主导代理将所有内容合成为连贯的输出。

这就是 DeerFlow 处理需要数分钟到数小时任务的方式：一个研究任务可能会分散到十几个子代理中，每个探索不同的角度，然后收敛成单一报告——或网站——或带有生成视觉效果的幻灯片组。一个框架，多只手。

> **AI 知识点 - 多代理系统**：
> 多代理系统（MAS）是 AI 研究中的活跃领域，专注于多个代理之间的协调和通信。LangGraph 提供了一个用于构建状态图的工作流框架，使代理之间的交互变得可编程和可观察。这种方法优于单代理方法，因为它可以并行处理、分工和专业化。
>
> 参考论文：[Multi-Agent Systems](https://zhuanlan.zhihu.com/p/1928636720796136414)
> 
> 工程实践：[[Anthropic] Built a Multi-Agent Research System](https://arthurchiao.art/blog/built-multi-agent-research-system-zh/)
> 
> 参考文档：[LangGraph](https://langchain-ai.github.io/langgraph/)

### 沙箱与文件系统

DeerFlow 不只是*谈论*做事情。它有自己的计算机。

每个任务都在一个具有完整文件系统的隔离 Docker 容器内运行——技能、工作区、上传、输出。代理读取、写入和编辑文件。它执行 bash 命令和代码。它查看图像。全部沙箱化，全部可审计，会话之间零污染。

这是具有工具访问权限的聊天机器人与具有实际执行环境的代理之间的区别。

```
# 沙箱容器内的路径
/mnt/user-data/
├── uploads/          ← 您的文件
├── workspace/        ← 代理的工作目录
└── outputs/          ← 最终交付物
```

> **AI 知识点 - 执行环境**：
> 为 AI 代理提供执行环境使其能够实际"做"事情，而不仅仅是"说"事情。这包括文件系统访问、代码执行和 shell 命令。关键挑战是平衡功能与安全性，这正是沙箱化发挥作用的地方。
>
> 参考论文：[专用沙盒环境的必要性与实践方案](https://aws.amazon.com/cn/blogs/china/agentic-ai-sandbox-practice/)

### 上下文工程

**隔离的子代理上下文**：每个子代理在自己的隔离上下文中运行。这意味着子代理将无法看到主代理或其他子代理的上下文。这很重要，可以确保子代理能够专注于手头的任务，而不会被主代理或其他子代理的上下文分心。

**摘要总结**：在会话内，DeerFlow 积极地管理上下文——总结已完成的子任务，将中间结果卸载到文件系统，压缩不再立即相关的内容。这使它在长多步骤任务中保持敏锐，而不会耗尽上下文窗口。

> **AI 知识点 - 上下文管理**：
> 随着对话的增长，上下文窗口最终会被填满。有效的上下文管理策略包括：
> 1. **摘要总结**：将早期对话压缩为简洁摘要
> 2. **滑动窗口**：保留最近 N 条消息
> 3. **优先级过滤**：根据任务相关性保留信息
> 4. **文件系统缓存**：将中间结果存储在文件中以避免重复处理
>
> 参考论文：[Context Window Management](https://arxiv.org/abs/2309.09597)

### 长期记忆

大多数代理在对话结束后就忘记一切。DeerFlow 记住。

跨会话，DeerFlow 构建您的配置文件、偏好设置和累积知识的持久记忆。您使用得越多，它就越了解您——您的写作风格、您的技术栈、您重复的工作流程。记忆存储在本地并保持在您的控制下。

> **AI 知识点 - 持久记忆系统**：
> 长期记忆使 AI 代理能够跨会话保持个性化和连续性。这类似于人类的记忆系统，短期记忆用于当前上下文，而长期记忆存储持久知识。实现包括向量数据库（用于语义搜索）和传统数据库（用于结构化数据）。
>
> 参考论文：[Memory-Augmented Large Language Models](https://arxiv.org/abs/2104.07863)

## 推荐模型

DeerFlow 是模型无关的——它适用于任何实现 OpenAI 兼容 API 的 LLM。也就是说，它在支持以下功能的模型上表现最佳：

- **长上下文窗口**（100k+ 令牌）用于深度研究和多步骤任务
- **推理能力**用于自适应规划和复杂分解
- **多模态输入**用于图像理解和视频理解
- **强大的工具使用**用于可靠的函数调用和结构化输出

> **AI 知识点 - 模型选择指南**：
>
> **推荐模型及其特点**：
>
> 1. **GPT-4o** (OpenAI)
>    - 优势：卓越的多模态能力、快速推理、强大的工具使用
>    - 适合：需要视觉理解、代码分析和快速响应的任务
>    - 参考文档：[GPT-4o Technical Report](https://openai.com/research/gpt-4o)
>
> 2. **Claude 3.5 Sonnet** (Anthropic)
>    - 优势：出色的推理能力、长上下文（200k 令牌）、负责任的 AI 对齐
>    - 适合：复杂推理、深度研究、长文档分析
>    - 参考文档：[Claude 3.5 Sonnet](https://www.anthropic.com/claude-3-5-sonnet)
>
> 3. **DeepSeek V3**
>    - 优势：开源友好、推理深度强、高性价比
>    - 适合：预算敏感的深度推理任务
>    - 参考文档：[DeepSeek Research](https://github.com/deepseek-ai/DeepSeek-V3)
>
> 4. **Qwen 2.5**
>    - 优势：强大的中文理解、开源、长上下文支持
>    - 适合：中文任务、多语言工作流
>    - 参考文档：[Qwen Technical Report](https://arxiv.org/abs/2309.16617)

## 嵌入式 Python 客户端

DeerFlow 可用作嵌入式 Python 库，无需运行完整的 HTTP 服务。`DeerFlowClient` 提供对所有代理和 Gateway 功能的直接进程内访问，返回与 HTTP Gateway API 相同的响应架构：

```python
from src.client import DeerFlowClient

client = DeerFlowClient()

# 聊天
response = client.chat("帮我分析这篇论文", thread_id="my-thread")

# 流式传输（LangGraph SSE 协议：values、messages-tuple、end）
for event in client.stream("你好"):
    if event.type == "messages-tuple" and event.data.get("type") == "ai":
        print(event.data["content"])

# 配置和管理 — 返回与 Gateway 对齐的字典
models = client.list_models()        # {"models": [...]}
skills = client.list_skills()        # {"skills": [...]}
client.update_skill("web-search", enabled=True)
client.upload_files("thread-1", ["./报告.pdf"])  # {"success": True, "files": [...]}
```

所有返回字典的方法都在 CI 中根据 Gateway Pydantic 响应模型进行验证（`TestGatewayConformance`），确保嵌入式客户端与 HTTP API 架构保持同步。有关完整 API 文档，请参阅 `backend/src/client.py`。

## 文档

- [贡献指南](CONTRIBUTING.md) - 开发环境设置和工作流
- [配置指南](backend/docs/CONFIGURATION.md) - 设置和配置说明
- [架构概述](backend/CLAUDE.md) - 技术架构细节
- [后端架构](backend/README.md) - 后端架构和 API 参考

## 贡献

我们欢迎贡献！请参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解开发设置、工作流和指南。

回归覆盖包括 Docker 沙箱模式检测和 provisioner kubeconfig 路径处理测试在 `backend/tests/` 中。

## 许可证

本项目是开源的，根据 [MIT 许可证](./LICENSE) 提供。

## 致谢

DeerFlow 建立在开源社区的杰出工作之上。我们要深深地感谢所有项目和贡献者，他们的努力使 DeerFlow 成为可能。确实，我们站在巨人的肩膀上。

我们要对以下项目表示诚挚的感谢，感谢他们的宝贵贡献：

- **[LangChain](https://github.com/langchain-ai/langchain)**：他们卓越的框架为我们的 LLM 交互和链提供动力，实现了实现了无缝集成和功能。
- **[LangGraph](https://github.com/langchain-ai/langgraph)**：他们创新的多代理编排方法对于实现 DeerFlow 复杂的工作流程至关重要。

这些项目展示了开源合作的变革力量，我们很荣幸能够建立在他们的基础上。

### 主要贡献者

向 `DeerFlow` 的核心作者致以衷心的感谢，他们的愿景、热情和奉献使这个项目得以实现：

- **[Daniel Walnut](https://github.com/hetaoBackend/)**
- **[Henry Li](https://github.com/magiccube/)**

您坚定不移的承诺和专业知识一直是 DeerFlow 成功背后的推动力。我们很荣幸有您掌舵这段旅程。

## Star 历史

[![Star History Chart](https://api.star-history.com/svg?repos=bytedance/deer-flow&type=Date)](https://star-history.com/#bytedance/deer-flow&Date)

---

## 附加 AI 参考资源

### 重要 AI 研究论文

以下是一些与 DeerFlow 架构和设计相关的重要研究论文：

1. **Constitutional AI** (Anthropic, 2022)
   - 描述如何通过自我改进和宪法约束使 AI 系统更加安全、有用和诚实
   - [论文链接](https://www.anthropic.com/research/constitutional-ai-harmlessness-ai-feedback)
   - 关联：负责任的 AI 对齐、安全约束

2. **Chain-of-Thought Prompting Elicits Reasoning** (Wei et al., 2022)
   - 通过思维链提示显著提高大型语言模型的推理能力
   - [论文链接](https://arxiv.org/abs/2201.11903)
   - 关联：代理规划、任务分解

3. **ReAct: Synergizing Reasoning and Acting** (Yao et al., 2022)
   - 结合推理和行动的框架，实现代理进行迭代决策
   - [论文链接](https://arxiv.org/abs/2210.03629)
   - 关联：工具使用、代理行为

4. **Tool-Augmented Large Language Models** (Schick et al., 2023)
   - 通过工具增强大型语言模型，使其能够与外部系统交互
   - [论文链接](https://arxiv.org/abs/2304.05395)
   - 关联：技能系统、MCP 集成

5. **Tree of Thoughts** (Yao et al., 2023)
   - 通过树状搜索结构实现更系统的推理和规划
   - [论文链接](https://arxiv.org/abs/2310.06825)
   - 关联：子代理、任务分解

### 官方文档资源

- [Anthropic Research Hub](https://www.anthropic.com/research) - Anthropic 的最新研究论文和发现
- [OpenAI Research](https://openai.com/research) - OpenAI 的研究出版物和技术报告
- [LangChain Documentation](https://python.langchain.com/) - LangChain 框架完整文档
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/) - LangGraph 工作流框架文档

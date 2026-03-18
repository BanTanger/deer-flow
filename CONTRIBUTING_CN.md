# 为 DeerFlow 做出贡献

感谢您对 DeerFlow 感兴趣！本指南将帮助您设置开发环境并了解我们的开发工作流程。

## 开发环境设置

我们提供两种开发环境。**推荐使用 Docker**，以获得最一致、无麻烦的体验。

### 选项 1：Docker 开发（推荐）

Docker 提供一个一致的、隔离的环境，预先配置了所有依赖项。无需在本地机器上安装 Node.js、Python 或 nginx。

#### 前提条件

- Docker Desktop 或 Docker Engine
- pnpm（用于缓存优化）

#### 设置步骤

1. **配置应用程序**：
   ```bash
   # 复制示例配置
   cp config.example.yaml config.yaml

   # 设置您的 API 密钥
   export OPENAI_API_KEY="your-key-here"
   # 或直接编辑 config.yaml
   ```

2. **初始化 Docker 环境**（仅首次）：
   ```bash
   make docker-init
   ```
   这将：
   - 构建 Docker 镜像
   - 安装前端依赖项（pnpm）
   - 安装后端依赖项（uv）
   - 与主机共享 pnpm 缓存以加快构建速度

3. **启动开发服务**：
   ```bash
   make docker-start
   ```
   `make docker-start` 读取 `config.yaml` 并仅在 provisioner/Kubernetes 沙箱模式下启动 `provisioner`。

   所有服务将启用热重载启动：
   - 前端更改会自动重新加载
   - 后端更改会触发自动重启
   - LangGraph 服务器支持热重载

4. **访问应用程序**：
   - Web 界面：http://localhost:2026
   - API 网关：http://localhost:2026/api/*
   - LangGraph：http://localhost:2026/api/langgraph/*

#### Docker 命令

```bash
# 构建（包含预缓存沙箱镜像的）自定义 k3s 镜像
make docker-init
# 启动 Docker 服务（模式感知，localhost:2026）
make docker-start
# 停止 Docker 开发服务
make docker-stop
# 查看 Docker 开发日志
make docker-logs
# 查看 Docker 前端日志
make docker-logs-frontend
# 查看 Docker 网关日志
make docker-logs-gateway
```

#### Docker 架构

```
主机
  ↓
Docker Compose (deer-flow-dev)
  ├→ nginx (端口 2026) ← 反向代理
  ├→ web (端口 3000) ← 带有热重载的前端
  ├→ api (端口 8001) ← 带有热重载的 Gateway API
   ├→ langgraph (端口 2024) ← 带有热重载的 LangGraph 服务器
   └→ provisioner (可选，端口 8002) ← 仅在 provisioner/K8s 沙箱模式下启动
```

**Docker 开发的优势**：
- ✅ 跨不同机器的一致环境
- ✅ 无需在本地安装 Node.js、Python 或 nginx
- ✅ 隔离的依赖项和服务
- ✅ 易于清理和重置
- ✅ 所有服务的热重载
- ✅ 类生产环境

### 选项 2：本地开发

如果您更喜欢直接在机器上运行服务：

#### 前提条件

检查是否已安装所有必需的工具：

```bash
make check
```

必需工具：
- Node.js 22+
- pnpm
- uv（Python 包管理器）
- nginx

#### 设置步骤

1. **配置应用程序**（与上述 Docker 设置相同）

2. **安装依赖项**`：
   ```bash
   make install
   ```

3. **运行开发服务器**（使用 nginx 启动所有服务）：
   ```bash
   make dev
   ```

4. **访问应用程序**：
   - Web 界面：http://localhost:2026
   - 所有 API 请求都通过 nginx 自动代理

#### 手动服务控制

如果您需要单独启动服务：

1. **启动后端服务**`：
   ```bash
   # 终端 1：启动 LangGraph 服务器（端口 2024）
   cd backend
   make dev

   # 终端 2：启动 Gateway API（端口 8001）
   cd backend
   make gateway

   # 终端 3：启动前端（端口 3000）
   cd frontend
   pnpm dev
   ```

2. **启动 nginx**`：
   ```bash
   make nginx
   # 或直接：nginx -c $(pwd)/docker/nginx/nginx.local.conf -g 'daemon off;'
   ```

3. **访问应用程序**：
   - Web 界面：http://localhost:2026

#### Nginx 配置

nginx 配置提供：
- 端口 2026 上的统一入口点
- 将 `/api/langgraph/*` 路由到 LangGraph 服务器（2024）
- 将其他 `/api/*` 端点路由到 Gateway API（8001）
- 将非 API 请求路由到前端（3000）
- 集中式 CORS 处理
- 用于实时代理响应的 SSE/流式传输支持
- 针对长时间运行操作优化的超时

> **AI 知识点 - Nginx 反向代理**：
> 反向代理是服务器架构中的关键组件，它接收客户端请求并将其转发到一个或多个后端服务器。对于 AI 应用程序，反向代理提供：
> 1. **负载均衡**：在多个服务器实例之间分配请求
> 2. **SSL 终止**：集中处理 HTTPS 加密
> 3. **请求路由**：基于路径规则将请求定向到不同的服务
> 4. **CORS 处理**：管理跨域资源共享策略
> 5. **SSE 支持**：启用服务器发送事件用于流式响应
>
> 参考文档：[Nginx Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)

## 项目结构

```
deer-flow/
├── config.example.yaml      # 配置模板
├── extensions_config.example.json  # MCP 和 Skills 配置模板
├── Makefile                 # 构建和开发命令
├── scripts/
│   └── docker.sh           # Docker 管理脚本
├── docker/
│   ├── docker-compose-dev.yaml  # Docker Compose 配置
│   └── nginx/
│       ├── nginx.conf      # Docker 的 Nginx 配置
│       └── nginx.local.conf # 本地开发的 Nginx 配置
├── backend/                 # 后端应用程序
│   ├── src/
│   │   ├── gateway/        # Gateway API（端口 8001）
│   │   ├── agents/         # LangGraph 代理（端口 2024）
│   │   ├── mcp/            # Model Context Protocol 集成
│   │   ├── skills/         # Skills 系统
│   │   └── sandbox/        # 沙箱执行
│   ├── docs/               # 后端文档
│   └── Makefile            # 后端命令
├── frontend/               # 前端应用程序
│   └── Makefile            # 前端命令
└── skills/                 # 代理技能
    ├── public/             # 公共技能
    └── custom/             # 自定义技能
```

> **AI 知识点 - 微服务架构**：
> DeerFlow 采用微服务架构，将应用程序分解为松耦合服务（前端、网关、LangGraph 服务器）。这种方法提供：
> 1. **可扩展性**：可以独立扩展每个服务
> 2. **模块化**：服务可以使用不同的技术栈
> 3. **故障隔离**：一个服务中的故障不会影响整个系统
> 4. **独立部署**：可以单独更新服务
>
> 参考论文：[Microservices Architecture](https://arxiv.org/abs/1702.06309)

## 架构

```
浏览器
  ↓
Nginx (端口 2026) ← 统一入口点
  ├→ 前端 (端口 3000) ← /（非 API 请求）
  ├→ Gateway API (端口 8001) ← /api/models、/api/mcp、/api/skills、/api/threads/*/artifacts
  └→ LangGraph 服务器 (端口 2024) ← /api/langgraph/*（代理交互）
```

> **AI 知识点 - 网关模式**：
> 网关模式在 API 架构中充当单一入口点。它处理横切关注点，如：
> 1. **身份验证和授权**：在请求到达服务之前验证用户身份
> 2. **请求路由**：将请求定向到适当的服务
> 3. **负载均衡**：在后端实例之间分配流量
> 4. **速率限制**：防止 API 滥用
> 5. **响应聚合**：合并来自多个服务的响应
>
> 参考文档：[API Gateway Pattern](https://aws.amazon.com/cn/api-gateway/)

## 开发工作流程

1. **创建功能分支**：
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **进行更改**，启用热重载

3. **彻底测试更改**

4. **提交更改**：
   ```bash
   git add .
   git commit -m "feat: 您的更改描述"
   ```

5. **推送并创建 Pull Request**：
   ```bash
   git push origin feature/your-feature-name
   ```

## 测试

```bash
# 后端测试
cd backend
uv run pytest

# 前端测试
cd frontend
pnpm test
```

### PR 回归检查

每个 Pull Request 都在 [.github/workflows/backend-unit-tests.yml](.github/workflows/backend-unit-tests.yml) 运行后端回归工作流，包括：

- `tests/test_provisioner_kubeconfig.py`
- `tests/test_docker_sandbox_mode_detection.py`

> **AI 知识点 - 持续集成/持续部署（CI/CD）**：
> CI/CD 是一种通过自动化方式频繁集成代码更改的实践。关键组件包括：
> 1. **自动化测试**：每个提交运行单元、集成和端到端测试
> 2. **代码质量检查**：自动运行 linter 和静态分析
> 3. **自动部署**：通过测试的更改自动部署到测试/生产环境
> 4. **回归测试**：确保新代码不会破坏现有功能
>
> 参考论文：[Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html)

## 代码风格

- **后端（Python）**：我们使用 `ruff` 进行 linting 和格式化
- **前端（TypeScript）**：我们使用 ESLint 和 Prettier

> **AI 知识点 - 代码质量工具**：
>
> **Python 工具**：
> - [Ruff](https://docs.astral.sh/ruff/)：极快的 Python linter 和 formatter，替代 Flake8、isort、pycodestyle 等
> - [Pytest](https://docs.pytest.org/)：成熟、功能丰富的 Python 测试框架
>
> **TypeScript/JavaScript 工具**：
> - [ESLint](https://eslint.org/)：可插拔的 linter 工具，用于查找和修复代码中的问题
> - [Prettier](https://prettier.io/)：代码 formatter，强制执行一致的代码风格
> - [TypeScript](https://www.typescriptlang.org/)：提供静态类型检查以提高代码质量和可维护性

## 文档

- [配置指南](backend/docs/CONFIGURATION.md) - 设置和配置
- [架构概述](backend/CLAUDE.md) - 技术架构
- [MCP 设置指南](MCP_SETUP.md) - Model Context Protocol 配置

## 需要帮助？

- 查看现有的 [Issues](https://github.com/bytedance/deer-flow/issues)
- 阅读 [文档](backend/docs/)
- 在 [Discussions](https://github.com/bytedance/deer-flow/discussions) 中提问

## 许可证

通过为 DeerFlow 做出贡献，您同意您的贡献将根据 [MIT 许可证](./LICENSE) 获得许可。

---

## 附加开发知识

### Git 工作流最佳实践

1. **提交消息规范**：
   - `feat:`：新功能
   - `fix:`：错误修复
   - `docs:`：文档更改
   - `style:`：不影响代码含义的格式更改
   - `refactor:`：代码重构
   - `test:`：添加或更新测试
   - `chore:`：构建过程或辅助工具的更改

2. **分支命名**：
   - `feature/描述性名称`：新功能
   - `bugfix/描述性名称`：错误修复
   - `hotfix/描述性名称`：紧急生产修复
   - `docs/描述性名称`：仅文档更改

3. **Pull Request 指南**：
   - 保持 PR 专注且小
   - 在 PR 描述中包含相关的 Issue 链接
   - 确保所有 CI 检查通过
   - 至少获得一个代码审查批准

### 调试技巧

**后端调试**：
```bash
# 使用 Python debugger
uv run pytest --pdb

# 启用详细日志
export LOG_LEVEL=DEBUG
make dev
```

**前端调试**：
```bash
# Next.js 调试模式
pnpm dev --debug
```

### 性能优化

- **后端**：使用异步 I/O 优化数据库查询，缓存频繁访问的数据
- **前端**：使用 React.memo、useCallback 和 useMemo 防止不必要的重新渲染
- **网络**：实施请求压缩、CDN 和懒加载

---

## 参考资源

- [Python 开发最佳实践](https://docs.python-guide.org/)
- [React 开发指南](https://react.dev/learn)
- [Docker 最佳实践](https://docs.docker.com/develop/dev-best-practices/)
- [Git 工作流](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)

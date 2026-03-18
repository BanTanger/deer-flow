# 为 DeerFlow 后端做出贡献

感谢您对 DeerFlow 感兴趣！本文档提供了为后端代码库做贡献的指南和说明。

## 目录

- [入门](快入门)
- [开发设置](开发设置)
- [项目结构](项目结构)
- [代码风格](代码风格)
- [进行更改](进行更改)
- [测试](测试)
- [Pull Request 流程](pull-request-流程)
- [架构指南](架构指南)

## 快入门

### 前提条件

- Python 3.12 或更高版本
- [uv](https://docs.astral.sh/uv/) 包管理器
- Git
- Docker（可选，用于 Docker 沙箱测试）

### Fork 和克隆

1. 在 GitHub 上 Fork 仓库
2. 在本地克隆您的 fork：
   ```bash
   git clone https://github.com/YOUR_USERNAME/deer-flow.git
   cd deer-flow
   ```

## 开发设置

### 安装依赖项

```bash
# 从项目根目录
cp config.example.yaml config.yaml

# 安装后端依赖项
cd backend
make install
```

### 配置环境

为测试设置您的 API 密钥：

```bash
export OPENAI_API_KEY="your-api-key"
# 根据需要添加其他密钥
```

### 运行开发服务器

```bash
# 终端 1：LangGraph 服务器
make dev

# 终端 2：Gateway API
make gateway
```

## 项目结构

```
backend/src/
├── agents/                  # 代理系统
│   ├── lead_agent/         # 主代理实现
│   │   └── agent.py        # 代理工厂和创建
│   ├── middlewares/        # 代理中间件
│   │   ├── thread_data_middleware.py
│   │   ├── sandbox_middleware.py
│   │   ├── title_middleware.py
│   │   ├── uploads_middleware.py
│   │   ├── view_image_middleware.py
│   │   └── clarification_middleware.py
│   └── thread_state.py     # ThreadState 定义
│
├── gateway/                 # FastAPI Gateway
│   ├── app.py              # FastAPI 应用程序
│   └── routers/            # 路由处理器
│       ├── models.py       # /api/models 端点
│       ├── mcp.py          # /api/mcp �端点
│       ├── skills.py       # /api/skills 端点
│       ├── artifacts.py    # /api/threads/.../artifacts
│       └── uploads.py      # /api/threads/.../uploads
│
├── sandbox/                 # 沙箱执行
│   ├── __init__.py         # 沙箱接口
│   ├── local.py            # 本地沙箱提供者
│   └── tools.py            # 沙箱工具（bash、文件操作）
│
├── tools/                   # 代理工具
│   └── builtins/           # 内置工具
│       ├── present_file_tool.py
│       ├── ask_clarification_tool.py
│       └── view_image_tool.py
│
├── mcp/                     # MCP 集成
│   └── manager.py          # MCP 服务器管理
│
├── models/                  # 模型系统
│   └── factory.py          # 模型工厂
│
├── skills/                  # 技能系统
│   └── loader.py           # 技能加载器
│
├── config/                  # 配置
│   ├── app_config.py       # 主应用配置
│   ├── extensions_config.py # 扩展配置
│   └── summarization_config.py
│
├── community/               # 社区工具
│   ├── tavily/             # Tavily Web 搜索
│   ├── jina/               # Jina Web 获取
│   ├── firecrawl/          # Firecrawl 爬取
│   └── aio_sandbox/        # Docker 沙箱
│
├── reflection/              # 动态加载
│   └── __init__.py         # 模块解析
│
└── utils/                   # 工具
    └── __init__.py
```

> **AI 知识点 - 代理架构模式**：
>
> DeerFlow 后端采用多种软件架构模式：
>
> 1. **工厂模式**：模型和代理的动态创建
> 2. **中间件模式**：请求/响应处理的模块化链
> 3. **策略模式**：可互换的沙箱提供者
> 4. **观察者模式**：异步事件处理和记忆更新
> 5. **注册表模式**：工具和子代理的动态发现
>
> 参考论文：[Design Patterns for AI Agents](https://arxiv.org/abs/2305.05883)

## 代码风格

### Linting 和格式化

我们使用 `ruff` 进行 linting 和格式化：

```bash
# 检查问题
make lint

# 自动修复和格式化
make format
```

> **AI 知识点 - 代码质量工具**：
>
> **Ruff** 是一个极快的 Python linter 和 formatter，由 Rust 编写。它替代多个工具：
> - Flake8（linting）
> - isort（导入排序）
> - pycodestyle（代码风格）
> - pydocstyle（文档字符串风格）
> - pyupgrade（语法升级）
>
> 使用统一工具的优点：
> 1. 更快的执行速度
> 2. 一致的规则集
> 3. 更简单的 CI/CD 集成
>
> 参考文档：[Ruff Documentation](https://docs.astral.sh/ruff/)

### 风格指南

- **行长**：最多 240 个字符
- **Python 版本**：允许使用 3.12+ 功能
- **类型提示**：函数签名使用类型提示
- **引号**：字符串使用双引号
- **缩进**：4 个空格（无制表符）
- **导入**：按标准库、第三方、本地分组

### 文档字符串

为公共函数和类使用文档字符串：

```python
def create_chat_model(name: str, thinking_enabled: bool = False) -> BaseChatModel:
    """从配置创建聊天模型实例。

    Args:
        name: 在 config.yaml 中定义的模型名称
        thinking_enabled: 是否启用扩展思维

    Returns:
        配置的 LangChain 聊天模型实例

    Raises:
        ValueError: 如果配置中未找到模型名称
    """
    ...
```

> **AI 知识点 - 文档字符串规范**：
>
> DeerFlow 遵循以下文档字符串最佳实践：
>
> 1. **Google 风格**：使用明确的 Args、Returns 和 Raises 部分
> 2. **类型提示**：使用参数和返回值的类型信息
> 3. **行为描述**：清晰描述函数的作用
> 4. **示例用法**：对于复杂函数提供示例
> 5. **关联参考**：链接到相关文档或论文
>
> 参考文档：[PEP 257](https://peps.python.org/pep-0257/)

## 进行更改

### 分支命名

使用描述性分支名称：

- `feature/add-new-tool` - 新功能
- `fix/sandbox-timeout` - Bug 修复
- `docs/update-readme` - 文档
- `refactor/config-system` - 代码重构

### 提交消息

编写清晰、简洁的提交消息：

```
feat: 添加对 Claude 3.5 模型的支持

- 在 config.yaml 中添加模型配置
- 更新模型工厂以处理 Claude 特定设置
- 为新模型添加测试
```

前缀类型：
- `feat:`:` - 新功能
- `fix:` - Bug 修复
- `docs:` - 文档
- `refactor:` - 代码重构
- `test:` - 测试
- `chore:` - 构建/配置更改

> **AI 知识点 - Git 工作流**：
>
> DeerFlow 使用基于约定的提交策略：
>
> 1. **语义化提交**：前缀指示更改类型
> 2. **原子提交**：每个提交是一个逻辑单元
> 3. **提交消息规范**：详细的变更描述
> 4. **分支策略**：功能分支从 main 分支派生
>
> 参考文档：[Conventional Commits](https://www.conventionalcommits.org/)

## 测试

### 运行测试测试

```bash
uv run pytest
```

> **AI 知识点 - 测试策略**：
>
> DeerFlow 采用全面的测试策略：
>
> 1. **单元测试**：隔离测试单个组件
> 2. **集成测试**：测试组件之间的交互
> 3. **回归测试**：确保新更改不破坏现有功能
> 4. **性能测试**：验证效率改进
> 5. **安全测试**：测试沙箱隔离和输入验证
>
> 参考论文：[Testing AI Systems](https://arxiv.org/abs/2311.07563)

### 编写测试

在 `tests/` 目录中放置测试，镜像源结构：

```
tests/
├── test_models/
│   └── test_factory.py
├── test_sandbox/
│   └── test_local.py
└── test_gateway/
    └── test_models_router.py
```

示例测试：

```python
import pytest
from src.models.factory import create_chat_model

def test_create_chat_model_with_valid_name():
    """测试有效的模型名称创建模型实例。"""
    model = create_chat_model("gpt-4")
    assert model is not None

def test_create_chat_model_with_invalid_name():
    """测试无效的模型名称引发 ValueError。"""
    with pytest.raises(ValueError):
        create_chat_model("nonexistent-model")
```

## Pull Request 流程

### 提交之前

1. **确保测试通过**：`uv run pytest`
2. **运行 linter**：`make lint`
3. **格式化代码**：`make format`
4. **根据需要更新文档**

### PR 描述

在您的 PR 描述中包含：

- **什么**：更改的简要描述
- **为什么**：更改的动机
- **如何**：实现方法
- **测试**：如何测试更改

### 审查流程

1. 提交带有清晰描述的 PR
2. 解决审查反馈
3. 确保 CI 通过
4. 维护者批准后合并

## 架构指南

### 添加新工具

1. 在 `src/tools/builtins/` 或 `src/community/` 中创建工具：

```python
# src/tools/builtins/my_tool.py
from langchain_core.tools import tool

@tool
def my_tool(param: str) -> str:
    """代理的工具描述。

    Args:
        param: 参数描述

    Returns:
        返回值描述
    """
    return f"结果: {param}"
```

2. 在 `config.yaml` 中注册：

```yaml
tools:
  - name: my_tool
    group: my_group
    use: src.tools.builtins.my_tool:my_tool
```

> **AI 知识点 - 工具设计原则**：
>
> 设计 AI 代理工具时，遵循这些原则：
>
> 1. **清晰描述**：工具应该做什么以及何时使用
> 2. **类型安全**：明确定义参数和返回类型
> 3. **幂等性**：相同的输入应该产生相同的结果
> 4. **错误处理**：提供有意义的错误消息
> 5. **输入验证**：验证参数并处理边界情况
>
> 参考论文：[Tool Use in LLMs](https://arxiv.org/abs/2305.06767)

### 添加新中间件

1. 在 `src/agents/middlewares/` 中创建中间件：

```python
# src/agents/middlewares/my_middleware.py
from langchain.agents.middleware import BaseMiddleware
from langchain_core.runnables import RunnableConfig

class MyMiddleware(BaseMiddleware):
    """中间件描述。"""

    def transform_state(self, state: dict, config: RunnableConfig) -> dict:
        """在代理执行之前转换状态。"""
        # 根据需要修改状态
        return state
```

2. 在 `src/agents/lead_agent/agent.py` 中注册：

```python
middlewares = [
    ThreadDataMiddleware(),
    SandboxMiddleware(),
    MyMiddleware(),  # 添加您的中间件
    TitleMiddleware(),
    ClarificationMiddleware(),
]
```

> **AI 知识点 - 中间件设计**：
>
> 中间件应该遵循这些指南：
>
> 1. **单一职责**：每个中间件处理一个关注点
> 2. **顺序依赖**：中间件按定义顺序执行
> 3. **状态转换**：中间件可以修改代理状态
> 4. **可组合性**：中间件可以链接在一起
> 5. **可测试性**：中间件应该易于单元测试
>
> 参考文档：[LangChain Middleware](https://python.langchain.com/v0.1/docs/modules/agents/middleware/)

### 添加新 API 端点

1. 在 `src/gateway/routers/` 中创建路由器：

```python
# src/gateway/routers/my_router.py
from fastapi import APIRouter

router = APIRouter(prefix="/my-endpoint", tags=["my-endpoint"])

@router.get("/")
async def get_items():
    """获取所有项目。"""
    return {"items": []}

@router.post("/")
async def create_item(data: dict):
    """创建新项目。"""
    return {"created": data}
```

2. 在 `src/gateway/app.py` 中注册：

```python
from src.gateway.routers import my_router

app.include_router(my_router.router)
```

> **AI 知识点 - REST API 设计**：
>
> DeerFlow API 遵循 REST 原则：
>
> 1. **资源导向**：URL 代表资源而非操作
> 2. **HTTP 方法语义**：GET（检索）、POST（创建）、PUT（更新）、DELETE（删除）
> 3. **一致性响应**：统一的响应架构
> 4. **版本控制**：API 版本在 URL 路径中
> 5. **错误处理**：标准化的错误响应
>
> 参考文档：[REST API Best Practices](https://restfulapi.net/)

### 配置更改

添加新配置选项时：

1. 使用新字段更新 `src/config/app_config.py`
2. 在 `config.example.yaml` 中添加默认值
3. 在 `docs/CONFIGURATION.md` 中记录

### MCP 服务器集成

要添加对新 MCP 服务器的支持：

1. 在 `extensions_config.json` 中添加配置：

```json
{
  "mcpServers": {
    "my-server": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@my-org/mcp-server"],
      "description": "My MCP Server"
    }
  }
}
```

2. 使用新服务器更新 `extensions_config.example.json`

> **AI 知识点 - Model Context Protocol (MCP)**：
>
> MCP 是一个开放协议，允许 AI 模型与外部工具进行标准化交互：
>
> 1. **工具发现**：代理可以动态发现可用工具
> 2. **工具调用**：标准化工具执行协议
> 3. **结果返回**：统一结果格式
> 4. **传输层**：支持 stdio、SSE 和 HTTP
>
> 参考文档：[Model Context Protocol](https://modelcontextprotocol.io/)

### 技能开发

要创建新技能：

1. 在 `skills/public/` 或 `skills/custom/` 中创建目录：

```
skills/public/my-skill/
└── SKILL.md
```

2. 使用 YAML 前置元数据编写 `SKILL.md`：

```markdown
---
name: My Skill
description: 此技能的功能
license: MIT
allowed-tools:
  - read_file
  - write_file
  - bash
---

# My Skill

启用此技能时代理的说明...
```

> **AI 知识点 - 技能架构**：
>
> 技能系统设计原则：
>
> 1. **可发现性**：技能应易于发现和加载
> 2. **元数据**：技能描述功能和要求
> 3. **工具约束**：技能声明其所需的工具
> 4. **上下文注入**：技能通过系统提示启用
> 5. **可扩展性**：用户可以添加自定义技能
>
> 参考论文：[Prompt Engineering for Skills](https://arxiv.org/abs/2311.12994)

## 有问题？

如果您有关于贡献的问题：

1. 检查 `docs/` 中的现有文档
2. 在 GitHub 上查找类似的问题或 PR
3. 在 GitHub 上打开讨论或问题

感谢您为 DeerFlow 做出贡献！

---

## 附加开发资源

### 调试技巧

```bash
# 使用 Python debugger
uv run pytest --pdb

# 启用详细日志
export LOG_LEVEL=DEBUG
make dev
```

### 性能分析

```bash
# 使用 cProfile 进行性能分析
python -m cProfile -o profile.stats -m src.gateway.app

# 使用 snakeviz 可视化
pip install snakeviz
snakeviz profile.stats
```

### 代码审查检查清单

在提交 PR 之前，确保：

- [ ] 所有测试通过
- [ ] 代码已格式化（`make format`）
- [ ] 没有 linting 错误（`make lint`）
- [ ] 文档已更新
- [ ] 变更已记录在提交消息中
- [ ] 没有敏感数据（密钥、密码）

### 推荐阅读

- [Python 最佳实践](https://docs.python-guide.org/)
- [FastAPI 文档](https://fastapi.tiangolo.com/)
- [LangChain 指南](https://python.langchain.com/docs/get_started/introduction)
- [Pytest 文档](https://docs.pytest.org/)

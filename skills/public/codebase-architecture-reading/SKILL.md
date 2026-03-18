---
name: "codebase-architecture-reading"
description: "阅读项目核心代码，阐述架构设计思想，检索相关权威论文并附带官方链接，输出结构化 markdown 文档"
author: "DeerFlow"
license: "MIT"
allowed-tools:
  - "Read"
  - "Glob"
  - "Grep"
  - "Bash"
  - "Write"
  - "Edit"
  - "Agent"
tags:
  - "code-reading"
  - "architecture"
  - "documentation"
  - "ai-research"
version: "1.0.0"
---

# Codebase Architecture Reading Skill

## 目的

当用户要求：**"阅读这个项目代码，给我阐述架构设计，检索相关权威论文"** 时，使用此技能。

该技能系统性地：
1. 阅读核心代码文件
2. 识别架构模式和设计思想
3. 检索相关权威论文（OpenAI、Anthropic、Google、Microsoft 等顶界机构）
4. 整理成结构化 markdown 文档，包含：
   - 核心代码摘录 + 分析
   - 权威论文引用 + 官方 URL 链接
   - 方法论总结，可复用到其他项目

## 工作流程

### 第一步：探索代码结构

```
1. 列出目标目录的文件结构
2. 读取核心入口文件（如 main agent, factory function 等）
3. 识别关键组件：patterns, middlewares, config systems
4. 理解组件之间的调用关系
```

### 第二步：识别架构模式

对照常见架构模式：
- 主代理-子代理编排 (Lead-Subagent Orchestration)
- 并行任务分解 (Parallel Task Decomposition)
- 中间件责任链 (Middleware Chain of Responsibility)
- 有状态执行图 (Stateful Execution Graph)
- 上下文管理 (Context Window Management)
- 长期记忆注入 (Long-Term Memory Injection)

### 第三步：关联权威论文

对于识别出的每个架构模式，检索相关权威论文：

**必搜权威来源：**
- OpenAI Research (https://openai.com/research)
- Anthropic Research (https://www.anthropic.com/research)
- arXiv 上的官方论文
- LangChain / LangGraph 官方技术博客

**论文要求：**
- 必须附带**可直接访问的官方 URL**
- 不引用非权威来源
- 每个论文列出作者、年份、核心观点

### 第四步：整理输出结构

输出 markdown 文档必须包含：

```
## [架构模式名称]

### 核心概念
一句话说明这是什么模式

### 项目中的实现

```python
# 核心代码摘录（来自项目源代码）
...
```

**代码分析：**
- 设计亮点
- 对应理论的哪些部分

### 权威论文

| 论文 | 机构 | 核心贡献 | 链接 |
|------|------|---------|------|
| ... | ... | ... | ... |

## 要求

### 对小白友好

- 解释要通俗易懂
- 避免过度晦涩的术语
- 用表格整理清晰
- 配合流程图（mermaid）

### 代码摘录规范

- 只摘录**核心关键代码**（不超过 50 行）
- 注明文件路径和行号
- 必须加上你的分析解释

### 论文引用规范

- **只引用权威机构**：OpenAI, Anthropic, Google DeepMind, Microsoft Research, LangChain
- **必须有可访问 URL**：arxiv 或官方博客
- **不引用**：个人博客、非权威 medium、不知名会议论文

### 输出位置

输出文档保存在项目文档目录，例如：
- `backend/docs/ARCHITECTURE.md`
- `backend/docs/PATTERN_NAME.md`

## 使用示例

用户说："依据这样的逻辑，整理成方法论并记忆到本项目的 memory 里，注意，你作出的回答请整理成标准的 md 文件存储在本地，对于项目中比较核心的代码与注释，请摘出来进行说明和分析，更新记忆，检索论文附带链接，注意不要贴错链接，基于你检索的事实附带链接，要权威论文的url地址"

→ 触发此技能，按上述流程执行。

## 质量检查清单

- [ ] ✓ 已阅读所有核心代码文件
- [ ] ✓ 已识别出主要架构模式
- [ ] ✓ 已摘录关键代码并注明文件路径行号
- [ ] ✓ 已为每个模式找到权威论文/技术报告
- [ ] ✓ 所有链接验证正确（不贴错）
- [ ] ✓ 输出是完整可独立阅读的 markdown 文件
- [ ] ✓ 保存到项目文档目录

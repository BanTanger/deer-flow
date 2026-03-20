# CLAUDE_CN.md

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供中文指导。

## 项目概述

DeerFlow Frontend 是一个 AI 代理系统的 Next.js 16 Web 界面。它与基于 LangGraph 的后端通信，提供基于线程的 AI 对话，具有流式响应ser、工件和技能/工具系统。

**技术栈**: Next.js 16, React 19, TypeScript 5.8, Tailwind CSS 4, pnpm 10.26.2

## 命令

| 命令 | 用途 |
|---------|---------|
| `pnpm dev` | 使用 Turbopack 的开发服务器 (http://localhost:3000) |
| `pnpm build` | 生产构建 |
| `pnpm check` | Lint + 类型检查 (提交前运行) |
| `pnpm lint` | 仅 ESLint |
| `pnpm lint:fix` | ESLint 自动修复 |
| `pnpm typecheck` | TypeScript 类型检查 (`tsc --noEmit`) |
| `pnpm start` | 启动生产服务器 |

未配置测试框架。

## 架构

```
Frontend (Next.js) ──▶ LangGraph SDK ──▶ LangGraph Backend (lead_agent)
                                              ├── Sub-Agents
                                              └── Tools & Skills
```

前端是一个有状态的聊天应用。用户创建**线程** (对话)，发送消息，并接收流式 AI 响应。后端编排可以产生**工件** (文件/代码) 和**待办事项**的代理。

### 源代码布局 (`src/`)

- **`app/`** — Next.js App Router。路由: `/` (着陆页), `/workspace/chats/[thread_id]` (聊天)。
- **`components/`** — React 组件分为:
  - `ui/` — Shadcn UI 基元 (自动生成，ESLint 忽略)
  - `ai-elements/` — Vercel AI SDK 元素 (自动生成，ESLint 忽略)
  - `workspace/` — 聊天页面组件 (消息、工件、设置)
  - `landing/` — 着陆页面部分
- **`core/`** — 业务逻辑，应用的核心:
  - `threads/` — 线程创建、流式传输、状态管理 (hooks + types)
  - `api/` — LangGraph 客户端单例
  - `artifacts/` — 工件加载和缓存
  - `i18n/` — 国际化 (en-US, zh-CN)
  - `settings/` — localStorage 中的用户首选项
  - `memory/` — 持久化用户内存系统
  - `skills/` — 技能安装和管理
  - `messages/` — 消息处理和转换
  - `mcp/` — Model Context Protocol 集成
  - `models/` — TypeScript 类型和数据模型
- **`hooks/`** — 共享 React hooks
- **`lib/`** — 工具函数 (`cn()` 来自 clsx + tailwind-merge)
- **`server/`** — 服务端代码 (better-auth，尚未激活)
- **`styles/`** — 全局 CSS，具有 Tailwind v4 `@import` 语法和 CSS 变量用于主题

### 数据流

1. 用户输入 → 线程 hooks (`core/threads/hooks.ts`) → LangGraph SDK 流式传输
2. 流式事件更新线程状态 (消息、工件、待办事项)
3. TanStack Query 管理服务器状态; localStorage 存储用户设置
4. 组件订阅线程状态并渲染更新

### 关键模式

- **默认使用服务端组件**，交互式组件仅使用 `"use client"`
- **线程 hooks** (`useThreadStream`, `useSubmitThread`, `useThreads`) 是主要 API 接口
- **LangGraph 客户端** 是通过 `core/api/` 中的 `getAPIClient()` 获取的单例
- **环境验证**使用 `@t3-oss/env-nextjs` 和 Zod 模式 (`src/env.js`)。使用 `SKIP_ENV_VALIDATION=1` 跳过

## 代码风格

- **导入**: 强制排序 (builtin → external → internal → parent → sibling)，字母顺序，组间换行。使用内内类型导入: `import { type Foo }`。
- **未使用的变量**: 前缀为 `_`。
- **类名**: 使用来自 `@/lib/utils` 的 `cn()` 进行条件 Tailwind 类。
- **路径别名**: `@/*` 映射到 `src/*`。
- **组件**: `ui/` 和 `ai-elements/` 从注册表生成 (Shadcn, MagicUI, React Bits, Vercel AI SDK) — 不要手动编辑这些。

## 环境

后端 API URL 是可选的; 默认使用 nginx 代理:
```
NEXT_PUBLIC_BACKEND_BASE_URL=http://localhost:8001
NEXT_PUBLIC_LANGGRAPH_BASE_URL=http://localhost:2024
```

需要 Node.js 22+ 和 pnpm 10.26.2+。

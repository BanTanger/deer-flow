# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DeerFlow is a full-stack "super agent harness" that orchestrates sub-agents, memory, and sandboxes to execute complex tasks. It's built with:
- **Backend**: Python 3.12 + LangGraph + FastAPI (in `backend/`)
- **Frontend**: Next.js 16 + React 19 + TypeScript (in `frontend/`)
- **Orchestration**: Docker Compose with nginx reverse proxy for unified access

## Quick Start Commands

From the **project root** directory:

```bash
# First-time setup
make config          # Generate config.yaml and .env from templates
make install         # Install backend + frontend dependencies

# Development
make dev            # Start all services (access at http://localhost:2026)
make stop           # Stop all running services

# Docker development (recommended)
make docker-init     # Build Docker images and install dependencies
make docker-start    # Start Docker services (mode-aware from config.yaml)
make docker-stop     # Stop Docker services

# Prerequisites check
make check           # Verify Node.js 22+, pnpm, uv, nginx are installed
```

## Project Structure

```
deer-flow/
├── Makefile                    # Root-level commands (use this first)
├── config.yaml                 # Main configuration (gitignored)
├── config.example.yaml          # Configuration template
├── extensions_config.json      # MCP servers + skills config (gitignored)
├── backend/                    # Python backend
│   ├── CLAUDE.md             # Backend architecture and commands
│   ├── pyproject.toml          # Python dependencies
│   └── src/
│       ├── agents/              # LangGraph agent system
│       ├── gateway/             # FastAPI API (port 8001)
│       ├── sandbox/             # Sandbox execution
│       ├── subagents/            # Subagent delegation
│       ├── mcp/                # MCP integration
│       ├── skills/              # Skills system
│       └── client.py           # Embedded Python client
├── frontend/                   # Next.js frontend
│   ├── CLAUDE.md             # Frontend architecture and commands
│   ├── package.json            # Node dependencies
│   └── src/
│       ├── app/                 # Next.js routes
│       ├── components/           # React components
│       └── core/                # Business logic
├── docker/                     # Docker configuration
│   ├── docker-compose-dev.yaml # Development stack
│   └── nginx/                 # Nginx reverse proxy configs
└── skills/                     # Agent skill packs
    ├── public/                # Built-in skills
    └── custom/                # Custom skills (gitignored)
```

## Service Architecture

```
Browser → Nginx (2026) ──┬── Frontend (3000)        # UI
                               ├── Gateway API (8001)      # Config, MCP, uploads
                               └── LangGraph Server (2024)  # Agent runtime
```

**Development modes**:
- **Local**: `make dev` - Runs services directly on host
- **Docker**: `make docker-start` - Isolated containers with hot-reload
- **Sandbox**: Three modes - Local, Docker, or Kubernetes (via provisioner)

## When to Work in Which Directory

| Task | Location | Commands |
|------|-----------|-----------|
| Backend features, tools, agents | `backend/` | `make lint && make test` |
| Frontend UI, components | `frontend/` | `pnpm lint && pnpm typecheck` |
| Configuration, orchestration | Root | `make dev` or `make docker-start` |
| MCP/Skills setup | Root | Edit `extensions_config.json` |

## Backend-Specific Commands

From `backend/` directory:
```bash
make dev        # LangGraph server (port 2024)
make gateway    # Gateway API (port 8001)
make test       # All backend tests (pytest)
make lint       # Lint with ruff
make format     # Format with ruff
```

## Frontend-Specific Commands

From `frontend/` directory:
```bash
pnpm dev              # Dev server (port 3000)
pnpm build            # Production build
pnpm lint             # ESLint
pnpm typecheck         # TypeScript type check
pnpm lint:fix         # ESLint with auto-fix
```

## Important Configuration Files

1. **`config.yaml`** (root) - Main application config:
   - Model definitions (API keys, providers)
   - Tool configurations
   - Sandbox settings (local/Docker/Kubernetes)
   - Memory system settings
   - Skills configuration

2. **`extensions_config.json`** (root) - MCP and skills:
   - MCP server definitions and runtime config
   - Skill enable/disable states
   - Modifiable via Gateway API at runtime

3. **`.env`** (root) - Environment variables for API keys

## Key Architectural Patterns

### Agent Orchestration
- Lead agent spawns sub-agents for parallel task execution
- Each sub-agent runs in isolated context
- Middleware chain handles: uploads, sandbox acquisition, memory updates, todos

### Sandbox System
- Virtual filesystem at `/mnt/user-data/` in agent's view
- Maps to physical paths per-thread for isolation
- Supports bash, file read/write, ls tools

### Skills System
- Skills are Markdown files with YAML frontmatter
- Loaded progressively (only when needed)
- Built-in skills in `skills/public/`, custom in `skills/custom/`

### Memory System
- Long-term user memory persists across sessions
- Async background updates with debouncing
- Injected into system prompts as context

## Pre-Commit Checklist

Before submitting changes:
```bash
# Backend
cd backend && make lint && make test

# Frontend (if changed)
cd frontend && pnpm lint && pnpm typecheck

# Optional: frontend build validation
BETTER_AUTH_SECRET=local-dev-secret pnpm build
```

## Documentation

- [Backend Architecture](backend/CLAUDE.md) - Detailed backend documentation
- [Frontend Architecture](frontend/CLAUDE.md) - Frontend-specific documentation
- [Configuration Guide](backend/docs/CONFIGURATION.md) - Config options
- [Contributing Guide](CONTRIBUTING.md) - Development workflow

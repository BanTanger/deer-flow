# 第3课：自主 Agent 架构

## 3.1 AutoGPT 与 BabyAGI：早期自主 Agent 探索

### AutoGPT 架构

AutoGPT 是最早的自主 Agent 之一，它展示了 LLM 作为自主决策者的潜力。

```mermaid
flowchart TB
    subgraph AutoGPT_Arch [AutoGPT 架构]
        User[用户目标] --> Agent[AutoGPT 核心]

        subgraph Core [核心模块]
            Planner[任务规划器]
            Executor[执行器]
            Memory[记忆系统]
            Critic[批评者]
        end

        Agent --> Planner
        Planner -->|分解任务| Executor
        Executor -->|使用| Tools[工具集]
        Executor -->|存储| Memory
        Memory -->|提供上下文| Planner
        Memory -->|提供历史| Critic
        Critic -->|反馈| Planner
    end

    Tools --> Search[网络搜索]
    Tools --> File[文件操作]
    Tools --> Code[代码执行]
    Tools --> API[API 调用]
```

### AutoGPT 任务流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant AutoGPT as AutoGPT
    participant Planner as 规划器
    participant Memory as 记忆
    participant Tools as 工具
    participant Critic as 批评者

    User->>AutoGPT: 目标: "创建一个关于AI的网站"
    AutoGPT->>Planner: 理解目标
    Planner->>Planner: 生成任务列表
    Planner->>Memory: 保存计划

    loop 任务循环
        Planner->>AutoGPT: 选择下一个任务
        AutoGPT->>Tools: 执行任务
        Tools-->>AutoGPT: 结果
        AutoGPT->>Memory: 存储结果
        AutoGPT->>Critic: 评估进度
        Critic-->>AutoGPT: 反馈
        AutoGPT->>Planner: 更新计划
    end

    AutoGPT->>User: 任务完成!
```

### BabyAGI 架构

BabyAGI 是另一个早期自主 Agent，采用了更简洁的设计。

```mermaid
flowchart LR
    subgraph BabyAGI_Arch [BabyAGI 极简架构]
        Objective[目标]
        TaskList[任务列表]
        ExecutionAgent[执行 Agent]
        TaskCreationAgent[任务创建 Agent]
        PrioritizationAgent[优先级 Agent]
        VectorDB[(向量数据库)]
    end

    Objective --> ExecutionAgent
    TaskList --> ExecutionAgent
    ExecutionAgent -->|结果| VectorDB
    VectorDB -->|上下文| TaskCreationAgent
    ExecutionAgent -->|完成| TaskCreationAgent
    TaskCreationAgent -->|新任务| TaskList
    TaskList --> PrioritizationAgent
    PrioritizationAgent -->|重排| TaskList
```

### BabyAGI 核心循环

```python
def babyagi_loop(objective, initial_task):
    """
    BabyAGI 核心循环伪代码
    """
    task_list = deque([initial_task])

    while task_list:
        # 1. 获取下一个任务
        current_task = task_list.popleft()

        # 2. 执行任务
        result = execution_agent(
            objective=objective,
            task=current_task
        )

        # 3. 存储结果
        store_in_vector_db(result)

        # 4. 创建新任务
        new_tasks = task_creation_agent(
            result=result,
            task_list=list(task_list),
            objective=objective
        )

        # 5. 添加新任务
        for task in new_tasks:
            task_list.append(task)

        # 6. 重新排序
        task_list = prioritization_agent(
            task_list=task_list,
            objective=objective
        )
```

---

## 3.2 Stanford Generative Agents

### 论文背景

**Generative Agents: Interactive Simulacra of Human Behavior**
Park et al., 2023 | [arXiv:2304.03442](https://arxiv.org/abs/2304.03442)

Stanford Generative Agents 创建了具有人类行为的交互式 Agent，它们能够记忆、反思、规划和互动。

### 整体架构

```mermaid
graph TB
    subgraph GenAgent [Generative Agent 架构]
        subgraph MemoryLayer [记忆层]
            Sensing[感知] --> MemoryStream[记忆流<br>Memory Stream]
            MemoryStream -->|存储| Events[事件记忆]
            MemoryStream -->|存储| Thoughts[思考记忆]
            MemoryStream -->|存储| Reactions[对话记忆]
        end

        subgraph ReflectionLayer [反思层]
            MemoryStream -->|检索| Retrieval[相关记忆检索]
            Retrieval --> Reflection[反思<br>Reflection]
            Reflection -->|生成| Insights[高层洞察]
            Insights -->|存储| MemoryStream
        end

        subgraph PlanningLayer [规划层]
            Insights -->|指导| Planning[规划<br>Planning]
            Retrieval -->|上下文| Planning
            Planning -->|生成| Plans[长期计划]
            Plans -->|分解| Reactions[反应]
            Plans -->|存储| MemoryStream
        end
    end

    Environment[环境] --> Sensing
    Reactions --> Environment
```

### 记忆流 (Memory Stream)

记忆流是一个时间序列的记忆对象，包含：

```mermaid
flowchart LR
    subgraph MemoryStream [记忆流结构]
        E1[事件1<br>时间: t1<br>重要性: 8]
        E2[事件2<br>时间: t2<br>重要性: 5]
        E3[思考1<br>时间: t3<br>重要性: 9]
        E4[对话1<br>时间: t4<br>重要性: 7]
        E5[事件3<br>时间: t5<br>重要性: 4]
    end

    E1 --> E2
    E2 --> E3
    E3 --> E4
    E4 --> E5
```

**记忆对象包含：**
- 描述文本
- 时间戳（创建时间、最近访问时间）
- 重要性评分（由模型评估）
- 嵌入向量

### 记忆检索机制

检索时综合考虑三个因素：

```mermaid
graph TB
    subgraph Retrieval [记忆检索评分]
        Query[查询]
        Recency[时效性<br>Recency]
        Importance[重要性<br>Importance]
        Relevance[相关性<br>Relevance]
    end

    Query -->|计算| Recency
    Query -->|计算| Relevance
    Memory[记忆] -->|读取| Importance
    Memory -->|嵌入| Relevance

    Recency --> ScoreRecency[分数: 0.7]
    Importance --> ScoreImportance[分数: 0.8]
    Relevance --> ScoreRelevance[分数: 0.9]

    ScoreRecency --> FinalScore[最终分数<br>= α×时效性 + β×重要性 + γ×相关性]
    ScoreImportance --> FinalScore
    ScoreRelevance --> FinalScore
```

**评分公式：**
```
score = α·recency + β·importance + γ·relevance
其中 α + β + γ = 1
```

### 反思 (Reflection) 机制

反思是将记忆整合成高层洞察的过程：

```mermaid
flowchart TD
    Memories[最近的记忆]
    Question1{有什么高层主题?}
    Question2{有什么洞察?}
    Question3{这些记忆如何关联?}

    Memories --> Question1
    Memories --> Question2
    Memories --> Question3

    Question1 --> Insight1[洞察1: 用户对AI很感兴趣]
    Question2 --> Insight2[洞察2: 需要详细的技术解释]
    Question3 --> Insight3[洞察3: 理论和实践结合效果最好]

    Insight1 --> ReflectionNode[反思记忆]
    Insight2 --> ReflectionNode
    Insight3 --> ReflectionNode

    ReflectionNode -->|存储回| MemoryStream[记忆流]
```

### 规划 (Planning) 与反应

```mermaid
stateDiagram-v2
    [*] --> 长期规划
    长期规划 --> 中期计划
    中期计划 --> 短期行动
    短期行动 --> 反应

    反应 --> 感知环境
    感知环境 --> 是否需要调整?
    是否需要调整? -->|是| 重新规划
    是否需要调整? -->|否| 执行行动
    重新规划 --> 短期行动
    执行行动 --> [*]
```

### 完整的 Agent 行为循环

```mermaid
sequenceDiagram
    participant Env as 环境
    participant Agent as Generative Agent
    participant Mem as 记忆流
    participant Ret as 检索
    participant Ref as 反思
    participant Plan as 规划

    Env->>Agent: 感知到事件
    Agent->>Mem: 存储观察

    loop 每一段时间
        Agent->>Ret: 检索相关记忆
        Ret-->>Agent: 相关记忆

        Agent->>Ref: 生成反思
        Ref-->>Mem: 存储洞察

        Agent->>Plan: 制定/更新计划
        Plan-->>Mem: 存储计划
    end

    Agent->>Agent: 决定反应
    Agent->>Env: 执行行动
```

---

## 3.3 三种架构对比

| 特性 | AutoGPT | BabyAGI | Generative Agents |
|------|---------|---------|-------------------|
| **核心目标** | 完成复杂任务 | 任务导向执行 | 模拟人类行为 |
| **记忆系统** | 短期+长期 | 向量数据库 | 记忆流+反思 |
| **规划** | 任务分解 | 优先级排序 | 长期规划+反应 |
| **交互性** | 工具交互 | 工具交互 | 多 Agent 社会交互 |
| **应用场景** | 实用任务 | 任务自动化 | 游戏、模拟 |

```mermaid
graph TB
    subgraph Comparison [架构演进]
        AutoGPT[AutoGPT<br>• 任务规划<br>• 工具使用<br>• 批评反馈]
        BabyAGI[BabyAGI<br>• 极简设计<br>• 任务队列<br>• 优先级]
        GenAgents[Generative Agents<br>• 记忆流<br>• 反思机制<br>• 社会互动]
    end

    AutoGPT -->|简化| BabyAGI
    BabyAGI -->|增强认知| GenAgents
    AutoGPT -->|增强认知| GenAgents
```

---

## 3.4 DeerFlow 中的实现

DeerFlow 融合了这些架构的优点：

```mermaid
flowchart TB
    subgraph DeerFlow_Arch [DeerFlow Agent 架构]
        subgraph MemorySys [记忆系统<br>借鉴 Generative Agents]
            MemoryStream[记忆流]
            Context[上下文摘要]
            Facts[事实记忆]
        end

        subgraph PlanningSys [规划系统<br>借鉴 AutoGPT/BabyAGI]
            TaskDecomp[任务分解]
            Priority[优先级管理]
            Milestone[里程碑追踪]
        end

        subgraph ExecutionSys [执行系统]
            Tools[工具使用]
            Sandbox[沙箱执行]
            SubAgents[子智能体]
        end
    end

    User[用户] --> MemorySys
    MemorySys --> PlanningSys
    PlanningSys --> ExecutionSys
    ExecutionSys -->|结果| MemorySys
```

---

## 3.5 DeerFlow 项目代码导读

### DeerFlow 的架构设计：融合三种架构精华

DeerFlow 吸收了 AutoGPT 的任务规划、BabyAGI 的任务队列、Stanford Generative Agents 的记忆系统，构建了一个企业级的自主 Agent 系统。

### 整体架构视图

```mermaid
graph TB
    subgraph DeerFlow_Arch [DeerFlow 架构]
        subgraph Agent [Agent 核心
            Lead[Lead Agent<br/>
            Tools[工具系统]
            SubAgents[子智能体]
        end

        subgraph Memory [记忆系统层]
            Thread[线程记忆]
            Facts[事实记忆]
            Artifacts[工件存储]
        end

        subgraph Planning [规划层]
            MW[中间件链]
            Todo[任务追踪]
            Sandbox[沙箱管理]
        end
    end

    Lead --> Tools
    Lead --> SubAgents
    Lead --> Memory
    Lead --> Planning
```

### 线程记忆系统：借鉴 Generative Agents

**文件**: `backend/src/agents/memory/`

DeerFlow 的记忆系统借鉴了 Stanford Generative Agents 的三层记忆流设计：

```python
# backend/src/agents/memory/updater.py
class MemoryUpdater:
    """
    基于 LLM 的记忆更新器，提取事实和上下文
    类似 Generative Agents 的反思机制
    """

    def __init__(self, config: MemoryConfig):
        self.config = config
        self.model = None  # 懒加载

    def update_memory(
        self,
        memory_data: MemoryData,
        conversation: list[BaseMessage]
    ) -> MemoryData:
        """
        1. 分析对话，提取新事实
        2. 更新用户上下文
        3. 原子性写入
        """
        # 过滤：只保留用户输入和最终 AI 响应
        filtered = self._filter_conversation(conversation)

        # 使用 LLM 提取事实和上下文更新
        updates = self._extract_updates(filtered)

        # 应用更新
        return self._apply_updates(memory_data, updates)
```

**记忆数据结构** (`backend/src/agents/memory/models.py`):
```python
class MemoryData:
    """
    类似 Generative Agents 的记忆结构
    """
    user_context: UserContext  # workContext, personalContext, topOfMind
    history: History  # recentMonths, earlierContext, longTermBackground
    facts: list[Fact]  # id, content, category, confidence, createdAt, source
```

### 记忆队列：防抖更新机制

**文件**: `backend/src/agents/memory/queue.py`

```python
class MemoryUpdateQueue:
    """
    防抖记忆更新队列，避免频繁 LLM 调用
    """

    def __init__(self, updater: MemoryUpdater, config: MemoryConfig):
        self.updater = updater
        self.debounce_seconds = config.debounce_seconds
        self._queue: dict[str, QueuedUpdate] = {}  # thread_id -> QueuedUpdate
        self._thread: Thread | None = None

    def queue_update(
        self,
        thread_id: str,
        memory_path: Path,
        conversation: list[BaseMessage]
    ):
        """
        队列化更新，去重线程
        防抖等待
        """
        self._queue[thread_id] = QueuedUpdate(...)
        self._ensure_worker()
```

### 任务规划系统：融合 AutoGPT + BabyAGI

**文件**: `backend/src/agents/middlewares/todo_list.py`

```python
class TodoListMiddleware:
    """
    任务追踪中间件，结合 AutoGPT 风格的任务规划
    """

    def __init__(self, is_plan_mode: bool):
        self.is_plan_mode = is_plan_mode

    def before_model(self, state: ThreadState) -> ThreadState:
        if not self.is_plan_mode:
            return state

        # 确保 todos 工具只在计划模式下可用
        state["tools"] = state.get("tools", []) + [self.write_todos_tool]
        return state
```

**Todo 工具** (`backend/src/tools/builtins/todo_list.py`):
```python
def write_todos(
    todos: Annotated[str, "JSON list of todo items"],
) -> Annotated[str, "Result"]:
    """
    Write/update the todo list for plan mode.

    Each todo item should have:
    - id: unique identifier
    - description: what to do
    - status: "pending", "in_progress", or "completed"
    """
    pass
```

### 中间件链：类似 AutoGPT 的模块化设计

**文件**: `backend/src/agents/lead_agent/agent.py`

```python
def _build_middlewares(config: LeadAgentConfig) -> list[AgentMiddleware]:
    """
    构建中间件链，顺序执行，类似 AutoGPT 的模块化架构
    """
    return [
        # 1. 线程数据 (ThreadDataMiddleware
        # 2. 上传 (UploadsMiddleware)
        # 3. 沙箱 (SandboxMiddleware)
        # 4. 悬空工具调用 (DanglingToolCallMiddleware)
        # 5. 摘要 (SummarizationMiddleware)
        # 6. 任务列表 (TodoListMiddleware)
        # 7. 标题 (TitleMiddleware)
        # 8. 记忆 (MemoryMiddleware)
        # 9. 图像 (ViewImageMiddleware)
        # 10. 子 Agent 限制 (SubagentLimitMiddleware)
        # 11. 澄清 (ClarificationMiddleware)
    ]
```

### 线程隔离：每个线程独立环境

**文件**: `backend/src/agents/middlewares/thread_data.py`

```python
class ThreadDataMiddleware:
    """
    为每个线程创建独立的工作目录
    类似 AutoGPT 的工作区隔离
    """

    def __init__(self, threads_dir: Path):
        self.threads_dir = threads_dir

    def _get_thread_dir(self, thread_id: str) -> Path:
        """
        backend/.deer-flow/threads/{thread_id}/user-data/
        ├── workspace/  # 工作区
        ├── uploads/    # 上传文件
        └── outputs/    # 输出文件
        """
```

### 工具系统：可扩展的工具注册

**文件**: `backend/src/tools/__init__.py`

```python
def get_available_tools(
    groups: list[str] | None = None,
    include_mcp: bool = True,
    model_name: str | None = None,
    subagent_enabled: bool = False,
) -> list[BaseTool]:
    """
    组合多个工具源，类似 AutoGPT 的插件系统
    """
    tools = []

    # 1. 配置定义的工具
    tools += get_config_tools(groups)

    # 2. MCP 工具 (懒加载)
    if include_mcp:
        tools += get_cached_mcp_tools()

    # 3. 内置工具
    tools += get_builtin_tools(model_name)

    # 4. 子 Agent 工具
    if subagent_enabled:
        tools += [task_tool]

    return tools
```

### 配置系统

**文件**: `config.yaml`

```yaml
# 记忆系统配置 (Generative Agents 风格)
memory:
  enabled: true
  injection_enabled: true
  storage_path: backend/.deer-flow/memory.json
  debounce_seconds: 30
  max_facts: 100
  fact_confidence_threshold: 0.7
  max_injection_tokens: 2000

# 子 Agent 系统
subagents:
  enabled: true

# 标题生成
title:
  enabled: true
```

### 关键代码文件索引

| 模块 | 文件路径 | 说明 |
|------|----------|------|
| **记忆更新器** | `src/agents/memory/updater.py` | LLM 事实提取 |
| **记忆队列** | `src/agents/memory/queue.py` | 防抖更新 |
| **记忆模型** | `src/agents/memory/models.py` | 数据结构 |
| **Todo 中间件** | `src/agents/middlewares/todo_list.py` | 任务追踪 |
| **线程数据** | `src/agents/middlewares/thread_data.py` | 工作区隔离 |
| **工具加载器** | `src/tools/__init__.py` | `get_available_tools()` |
| **子Agent执行器** | `src/subagents/executor.py` | 并行任务 |

---

## 3.6 小结

**本节课要点：**

1. ✅ **AutoGPT**：开创了任务规划+执行+批评的自主 Agent 模式
2. ✅ **BabyAGI**：极简的任务队列+优先级架构
3. ✅ **Generative Agents**：记忆流+反思+规划的认知架构
4. ✅ 这些架构为现代 Agent 系统奠定了基础

**下节课预告：**
我们将深入学习 Agent 记忆系统的设计与实现。

---

## 参考资料

- [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442)
- [AutoGPT GitHub Repository](https://github.com/Significant-Gravitas/AutoGPT)
- [BabyAGI GitHub Repository](https://github.com/yoheinakajima/babyagi)

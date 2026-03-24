# 第9课：协调与共识

## 9.1 协调机制

### 协调模式分类

```mermaid
graph TB
    subgraph Coordination [协调机制]
        Centralized[集中式控制<br>Centralized]
        Decentralized[分散式协作<br>Decentralized]
        Hybrid[混合模式<br>Hybrid]
    end

    subgraph Centralized_Mechanisms [集中式]
        LeaderElection[领导者选举<br>Leader Election]
        MasterSlave[主从架构<br>Master-Slave]
    end

    subgraph Decentralized_Mechanisms [分散式]
        ContractNet[合同网协议<br>Contract Net]
        Auction[拍卖机制<br>Auction]
        Voting[投票机制<br>Voting]
    end

    Centralized --> LeaderElection
    Centralized --> MasterSlave
    Decentralized --> ContractNet
    Decentralized --> Auction
    Decentralized --> Voting
```

---

## 9.2 集中式 vs 分散式

### 集中式控制

```mermaid
graph TB
    Leader[领导者/协调者]
    Agent1[Agent 1]
    Agent2[Agent 2]
    Agent3[Agent 3]
    Agent4[Agent 4]

    Leader -->|指令| Agent1
    Leader -->|指令| Agent2
    Leader -->|指令| Agent3
    Leader -->|指令| Agent4

    Agent1 -->|状态| Leader
    Agent2 -->|状态| Leader
    Agent3 -->|状态| Leader
    Agent4 -->|状态| Leader
```

| 优点 | 缺点 |
|------|------|
| 决策简单高效 | 单点故障风险 |
| 全局视图 | 领导者成为瓶颈 |
| 易于实现 | 可扩展性有限 |
| 避免冲突 | 缺乏自主性 |

### 分散式协作

```mermaid
graph LR
    A[Agent A]
    B[Agent B]
    C[Agent C]
    D[Agent D]
    E[Agent E]

    A <--> B
    B <--> C
    C <--> D
    D <--> E
    A <--> C
    B <--> D
    C <--> E
```

| 优点 | 缺点 |
|------|------|
| 无单点故障 | 协调复杂 |
| 高度可扩展 | 可能出现冲突 |
| 容错性强 | 达成共识慢 |
| 充分利用自主性 | 缺乏全局最优 |

### 混合模式

```mermaid
graph TB
    subgraph Group1 [小组1]
        L1[领导者1]
        A1[Agent 1]
        A2[Agent 2]
    end

    subgraph Group2 [小组2]
        L2[领导者2]
        A3[Agent 3]
        A4[Agent 4]
    end

    subgraph Group3 [小组3]
        L3[领导者3]
        A5[Agent 5]
        A6[Agent 6]
    end

    L1 <--> L2
    L2 <--> L3
    L1 <--> L3

    L1 --> A1
    L1 --> A2
    L2 --> A3
    L2 --> A4
    L3 --> A5
    L3 --> A6
```

**特点：**
- 分层协调
- 组内集中式，组间分散式
- 兼顾效率和灵活性
- 适合大规模系统

---

## 9.3 角色分配：主从架构 vs 对等网络

### 主从架构 (Master-Slave)

```mermaid
sequenceDiagram
    participant Master as 主 Agent
    participant Slave1 as 从 Agent 1
    participant Slave2 as 从 Agent 2
    participant Slave3 as 从 Agent 3

    Master->>Master: 分解任务
    Master->>Slave1: 分配任务 A
    Master->>Slave2: 分配任务 B
    Master->>Slave3: 分配任务 C

    Note over Slave1,Slave3: 并行执行

    Slave1-->>Master: 结果 A
    Slave2-->>Master: 结果 B
    Slave3-->>Master: 结果 C

    Master->>Master: 合并结果
```

### 对等网络 (Peer-to-Peer)

```mermaid
sequenceDiagram
    participant A as Agent A
    participant B as Agent B
    participant C as Agent C
    participant D as Agent D

    A->>B: 任务公告
    A->>C: 任务公告
    A->>D: 任务公告

    B-->>A: 投标: 我可以做
    C-->>A: 投标: 我可以做
    D-->>A: 投标: 我可以做

    A->>B: 接受投标，任务给你
    B->>B: 执行任务
    B-->>A: 任务完成

    A->>C: 分享结果
    A->>D: 分享结果
```

### 对比总结

| 维度 | 主从架构 | 对等网络 |
|------|---------|---------|
| **决策方式** | 集中决策 | 分布式协商 |
| **通信模式** | 星型 | 网状 |
| **复杂度** | 较低 | 较高 |
| **适应性** | 较差 | 较强 |
| **适用场景** | 结构化任务 | 动态任务 |

---

## 9.4 任务分配算法

### 合同网协议 (Contract Net Protocol)

```mermaid
flowchart TB
    Start[任务发起]
    Announce[任务公告]
    Bid[投标]
    Award[授标]
    Execute[执行]
    Result[结果]

    Start --> Announce
    Announce -->|广播| Bid
    Bid -->|收集投标| Award
    Award -->|选中| Execute
    Execute -->|完成| Result
```

**步骤：**
1. **任务公告**：管理者向所有参与者广播任务
2. **投标**：有能力的参与者提交投标
3. **授标**：管理者选择最佳投标并授标
4. **执行**：中标者执行任务并汇报结果

### 拍卖机制 (Auction Mechanisms)

```mermaid
graph LR
    subgraph AuctionTypes [拍卖类型]
        FirstPrice[第一价格拍卖<br>出价最高者获得]
        SecondPrice[第二价格拍卖<br>出价最高但付第二价]
        English[英式拍卖<br>公开加价]
        Dutch[荷兰式拍卖<br>公开降价]
    end
```

#### 英式拍卖流程

```mermaid
sequenceDiagram
    participant Auctioneer as 拍卖者
    participant Bidder1 as 竞拍者 A
    participant Bidder2 as 竞拍者 B
    participant Bidder3 as 竞拍者 C

    Auctioneer->>Bidder1: 起拍价 10
    Bidder1->>Auctioneer: 15
    Auctioneer->>Bidder2: 15，有人更高吗?
    Bidder2->>Auctioneer: 20
    Auctioneer->>Bidder3: 20，有人更高吗?
    Bidder3->>Auctioneer: 25
    Auctioneer->>Bidder1: 25，有人更高吗?
    Auctioneer->>Bidder2: 25，有人更高吗?

    Note over Auctioneer,Bidder3: 无人加价

    Auctioneer->>Bidder3: 成交！任务归你
```

### 基于能力的任务分配

```mermaid
flowchart TD
    Task[任务描述]
    CapReq[能力需求分析]
    AgentCap[Agent 能力库]
    Match[能力匹配]
    Score[综合评分]
    Assign[分配任务]

    Task --> CapReq
    CapReq --> Match
    AgentCap --> Match
    Match --> Score
    Score --> Assign
```

**评分公式：**

$$
\text{Score}(a, t) = \sum_{i} w_i \cdot \text{match}(c_i^a, c_i^t) + \beta \cdot \text{availability}(a) + \gamma \cdot \text{load}(a)
$$

其中：
- $w_i$ 是各能力的权重
- $\text{match}(c_i^a, c_i^t)$ 是 Agent $a$ 对能力 $i$ 的匹配度
- $\beta, \gamma$ 是可用性和负载的权重

---

## 9.5 冲突解决机制

### 冲突类型

| 冲突类型 | 描述 | 示例 |
|---------|------|------|
| **资源冲突** | 竞争同一资源 | 两个 Agent 同时想编辑同一文件 |
| **目标冲突** | 目标不一致 | 一个想快速完成，一个想保证质量 |
| **意见冲突** | 解决方案分歧 | 对技术选型有不同看法 |
| **优先级冲突** | 任务排序不同 | 对任务重要性有不同判断 |

### 投票机制

```mermaid
flowchart TB
    Conflict[冲突出现]
    Options[生成选项]
    Agents[相关 Agent]
    Vote[投票]
    Count[计票]
    Resolve[选择胜出者]

    Conflict --> Options
    Options --> Agents
    Agents --> Vote
    Vote --> Count
    Count --> Resolve
```

#### 投票方法对比

| 投票方法 | 原理 | 优点 | 缺点 |
|---------|------|------|------|
| **简单多数** | 得票最多者胜出 | 简单 | 可能忽略少数意见 |
| **绝对多数** | 需要超过 50% | 更有共识 | 可能多轮投票 |
| **排序投票** | 排序偏好 | 细致 | 复杂 |
| **认可投票** | 认可多个选项 | 灵活 | 策略性投票 |

### 辩论与说服

```mermaid
sequenceDiagram
    participant A as Agent A (观点1)
    participant B as Agent B (观点2)
    participant M as 调解者

    A->>M: 提出论点和证据
    B->>M: 提出论点和证据
    M->>A: 请回应 B 的论点
    A->>M: 反驳和补充
    M->>B: 请回应 A 的论点
    B->>M: 反驳和补充

    M->>M: 综合评估
    M->>A: 综合结论
    M->>B: 综合结论
```

### 仲裁者设计

```mermaid
flowchart TB
    Dispute[争议]
    Arbiter[仲裁者]
    A_Pos[A 方立场]
    B_Pos[B 方立场]
    Evidence[证据收集]
    Rules[规则/标准]
    Decision[裁决]

    Dispute --> Arbiter
    Arbiter --> A_Pos
    Arbiter --> B_Pos
    Arbiter --> Evidence
    Arbiter --> Rules
    A_Pos --> Decision
    B_Pos --> Decision
    Evidence --> Decision
    Rules --> Decision
```

---

## 9.6 团队反思：集体记忆与经验积累

### 集体记忆架构

```mermaid
graph TB
    subgraph CollectiveMemory [集体记忆]
        Individual[个体记忆]
        Shared[共享记忆]
        Explicit[显性知识]
        Tacit[隐性知识]
    end

    subgraph Processes [过程]
        Share[分享]
        Discuss[讨论]
        Reflect[反思]
        Document[记录]
    end

    Individual --> Share
    Share --> Shared
    Shared --> Discuss
    Discuss --> Reflect
    Reflect --> Explicit
    Reflect --> Tacit
    Explicit --> Document
    Tacit --> Document
    Document --> Individual
```

### 经验学习循环

```mermaid
stateDiagram-v2
    [*] --> 执行任务
    执行任务 --> 观察结果
    观察结果 --> 团队反思
    团队反思 --> 提炼经验
    提炼经验 --> 更新策略
    更新策略 --> 执行任务
```

### 团队反思会议

```mermaid
flowchart TB
    Start[开始反思]
    WhatHappened[发生了什么?]
    Why[为什么会发生?]
    Success[哪些做得好?]
    Improve[哪些可以改进?]
    Actions[行动计划]
    Document[记录文档]
    End[结束]

    Start --> WhatHappened
    WhatHappened --> Why
    Why --> Success
    Success --> Improve
    Improve --> Actions
    Actions --> Document
    Document --> End
```

---

## 9.7 DeerFlow 项目代码导读

### DeerFlow 的协调与共识机制

DeerFlow 采用集中式控制的主从架构，结合了多种协调策略来确保系统的稳定运行。

### 集中式协调架构：Lead Agent + 子 Agent

**文件**: `backend/src/subagents/executor.py`

```python
class SubagentExecutor:
    """
    子 Agent 执行器：集中式任务分配
    主 Agent (Lead Agent) 作为协调者
    """

    MAX_CONCURRENT_SUBAGENTS = 3
    SUBAGENT_TIMEOUT = 900  # 15 minutes

    def __init__(self):
        # 双线程池设计
        self._scheduler_pool = ThreadPoolExecutor(max_workers=3)
        self._execution_pool = ThreadPoolExecutor(max_workers=3)
        self._running_tasks: dict[str, SubagentTask] = {}
        self._lock = threading.Lock()

    def execute(
        self,
        subagent_type: str,
        description: str,
        prompt: str,
        max_turns: int = 10,
    ) -> SubagentResult:
        """
        同步执行子 Agent：主 Agent 等待结果
        """
        subagent = get_subagent(subagent_type)
        return subagent.run(description, prompt, max_turns)

    def get_task_status(self, task_id: str) -> SubagentTask | None:
        """
        获取任务状态：主 Agent 轮询查询
        """
        with self._lock:
            return self._running_tasks.get(task_id)
```

### 任务状态追踪

**文件**: `backend/src/subagents/executor.py`

```python
from dataclasses import dataclass, field
from typing import Any
import time

@dataclass
class SubagentTask:
    """
    子 Agent 任务状态：集中式状态管理
    """
    task_id: str
    status: str  # starting, running, completed, failed, timed_out
    subagent_type: str
    description: str
    result: Any | None = None
    error: str | None = None
    created_at: float = field(default_factory=time.time)
    started_at: float | None = None
    completed_at: float | None = None

class SubagentResult:
    """
    子 Agent 执行结果
    """
    output: str
    success: bool
    error: str | None = None
```

### SubagentLimitMiddleware：并发控制

**文件**: `backend/src/agents/middlewares/subagent_limit.py`

```python
class SubagentLimitMiddleware(AgentMiddleware):
    """
    限制并发子 Agent 数量为 MAX_CONCURRENT_SUBAGENTS
    防止资源耗尽
    """

    def after_model(self, state: ThreadState) -> ThreadState:
        configurable = state.get("config", {}).get("configurable", {})
        if not configurable.get("subagent_enabled"):
            return state

        messages = state["messages"]
        if not messages:
            return state

        last_message = messages[-1]
        if not hasattr(last_message, "tool_calls"):
            return state

        # 分离 task 调用和其他调用
        task_calls = []
        other_calls = []
        for call in last_message.tool_calls:
            if call["name"] == "task":
                task_calls.append(call)
            else:
                other_calls.append(call)

        # 截断超过限制的 task 调用（主从架构的负载控制）
        if len(task_calls) > MAX_CONCURRENT_SUBAGENTS:
            truncated = task_calls[:MAX_CONCURRENT_SUBAGENTS]
            last_message.tool_calls = other_calls + truncated

        return state
```

### 线程数据隔离：避免资源冲突

**文件**: `backend/src/agents/middlewares/thread_data.py`

```python
class ThreadDataMiddleware(AgentMiddleware):
    """
    为每个线程创建独立的工作目录，避免资源冲突
    """

    def __init__(self, threads_dir: Path):
        self.threads_dir = threads_dir

    def before_model(self, state: ThreadState) -> ThreadState:
        """
        在模型调用前：创建或获取线程目录
        """
        thread_id = state.get("thread_id", "unknown")
        thread_dir = self.threads_dir / thread_id
        user_data_dir = thread_dir / "user-data"

        # 创建隔离目录
        workspace_dir = user_data_dir / "workspace"
        uploads_dir = user_data_dir / "uploads"
        outputs_dir = user_data_dir / "outputs"

        workspace_dir.mkdir(parents=True, exist_ok=True)
        uploads_dir.mkdir(parents=True, exist_ok=True)
        outputs_dir.mkdir(parents=True, exist_ok=True)

        # 存储到状态中（所有 Agent 共享）
        state["thread_data"] = {
            "base_dir": str(user_data_dir),
            "workspace": str(workspace_dir),
            "uploads": str(uploads_dir),
            "outputs": str(outputs_dir),
        }

        return state
```

### SandboxMiddleware：资源分配协调

**文件**: `backend/src/sandbox/middleware.py`

```python
class SandboxMiddleware(AgentMiddleware):
    """
    沙箱中间件：协调沙箱环境的获取和释放
    """

    def __init__(self, sandbox_provider: SandboxProvider):
        self.provider = sandbox_provider

    def before_model(self, state: ThreadState) -> ThreadState:
        """
        在模型调用前：获取沙箱环境
        """
        if state.get("sandbox"):
            return state

        # 从 provider 获取沙箱
        sandbox_id = self.provider.acquire()
        sandbox = self.provider.get(sandbox_id)

        state["sandbox"] = {
            "sandbox_id": sandbox_id,
            "provider": type(self.provider).__name__,
        }

        return state

    def after_model(self, state: ThreadState) -> ThreadState:
        """
        在模型调用后：可以选择是否释放沙箱
        当前设计保持沙箱活跃以便重用
        """
        return state
```

### 沙箱 Provider 接口

**文件**: `backend/src/sandbox/sandbox.py`

```python
from abc import ABC, abstractmethod
from pathlib import Path
from typing import Any

class Sandbox(ABC):
    """
    沙箱抽象接口
    """

    @abstractmethod
    def execute_command(
        self, command: str, timeout: int | None = None
    ) -> str:
        """执行命令"""
        pass

    @abstractmethod
    def read_file(self, path: str | Path) -> str:
        """读取文件"""
        pass

    @abstractmethod
    def write_file(self, path: str | Path, content: str, append: bool = False):
        """写入文件"""
        pass

    @abstractmethod
    def list_dir(self, path: str | Path) -> str:
        """列出目录"""
        pass

class SandboxProvider(ABC):
    """
    沙箱提供者：管理沙箱生命周期
    """

    @abstractmethod
    def acquire(self) -> str:
        """
        获取沙箱 ID
        支持多种分配策略：轮询、最少使用等
        """
        pass

    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox:
        """
        获取沙箱实例
        """
        pass

    @abstractmethod
    def release(self, sandbox_id: str):
        """
        释放沙箱
        """
        pass
```

### LocalSandboxProvider：单例沙箱

**文件**: `backend/src/sandbox/local.py`

```python
class LocalSandboxProvider(SandboxProvider):
    """
    本地沙箱提供者：单例模式，简单直接
    开发环境使用，生产环境建议使用 AioSandboxProvider
    """

    _instance: "LocalSandboxProvider" = None
    _lock: threading.Lock = threading.Lock()
    _sandbox: "LocalSandbox" = None

    def __new__(cls):
        with cls._lock:
            if cls._instance is None:
                cls._instance = super().__new__(cls)
            return cls._instance

    def acquire(self) -> str:
        """获取沙箱 ID：总是返回 'local'"""
        return "local"

    def get(self, sandbox_id: str) -> Sandbox:
        """获取沙箱实例：单例"""
        if sandbox_id != "local":
            raise ValueError(f"Unknown sandbox id: {sandbox_id}")

        with self._lock:
            if self._sandbox is None:
                self._sandbox = LocalSandbox()
            return self._sandbox

    def release(self, sandbox_id: str):
        """释放沙箱：本地沙箱不做任何事"""
        pass
```

### 共享状态：ThreadState 作为共识层

**文件**: `backend/src/agents/thread_state.py`

```python
def merge_artifacts(old: list[str] | None, new: list[str]) -> list[str]:
    """
    合并工件列表，去重但保持顺序
    所有 Agent 对 artifacts 的修改都会通过这个 reducer 协调
    """
    combined = (old or []) + new
    seen = set()
    result = []
    for item in combined:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

def merge_viewed_images(
    old: dict | None,
    new: dict | None,
) -> dict | None:
    """
    合并 viewed_images
    - new 为 None: 清除所有
    - 否则: 合并
    """
    if new is None:
        return None
    if old is None:
        return new
    return {**old, **new}

class ThreadState(TypedDict):
    """
    线程状态：作为所有 Agent 的共享共识层
    """
    # 消息历史：使用 LangGraph 的 add_messages
    messages: Annotated[Sequence[BaseMessage], add_messages]

    # 扩展状态：使用自定义 reducers
    sandbox: dict | None
    artifacts: Annotated[list[str] | None, merge_artifacts]
    thread_data: dict | None
    title: str | None
    todos: list[dict] | None
    uploaded_files: list[dict] | None
    viewed_images: Annotated[dict | None, merge_viewed_images]
```

### 子 Agent 注册表：角色分配

**文件**: `backend/src/subagents/registry.py`

```python
from typing import Callable
from .builtins import create_general_purpose_agent, create_bash_agent

SubagentFactory = Callable[[], Subagent]

_subagents: dict[str, SubagentFactory] = {}

def register_subagent(name: str, factory: SubagentFactory):
    """
    注册子 Agent：类似于角色分配
    """
    _subagents[name] = factory

def get_subagent(name: str) -> Subagent:
    """
    获取子 Agent：按角色名称查找
    """
    if name not in _subagents:
        raise ValueError(f"Unknown subagent type: {name}")
    return _subagents[name]()

def list_subagents() -> list[str]:
    """
    列出所有可用的子 Agent 角色
    """
    return list(_subagents.keys())

# 注册内置角色
register_subagent("general-purpose", create_general_purpose_agent)
register_subagent("bash", create_bash_agent)
```

### 配置驱动的协调策略

**文件**: `config.yaml`

```yaml
# 子 Agent 系统配置
subagents:
  enabled: true

# 沙箱配置
sandbox:
  use: src.sandbox.local:LocalSandboxProvider

# 摘要配置（上下文管理）
summarization:
  enabled: true
  trigger:
    type: fraction
    value: 0.8
  keep_policy:
    recent_messages: 10
    summarize_older: true
```

### 关键代码文件索引

| 模块 | 文件路径 | 说明 |
|------|----------|------|
| **子 Agent 执行器** | `src/subagents/executor.py` | `SubagentExecutor` 集中式协调 |
| **子 Agent 限制** | `src/agents/middlewares/subagent_limit.py` | 并发控制 |
| **线程数据** | `src/agents/middlewares/thread_data.py` | 资源隔离 |
| **沙箱中间件** | `src/sandbox/middleware.py` | 沙箱生命周期管理 |
| **沙箱接口** | `src/sandbox/sandbox.py` | `Sandbox` + `SandboxProvider` |
| **本地沙箱** | `src/sandbox/local.py` | `LocalSandboxProvider` 单例 |
| **子 Agent 注册表** | `src/subagents/registry.py` | 角色分配 |
| **线程状态** | `src/agents/thread_state.py` | 共享状态 + reducers |

---

## 9.8 小结

**本节课要点：**

1. ✅ 协调机制分为集中式、分散式和混合模式
2. ✅ 任务分配可以使用合同网协议、拍卖机制或基于能力的分配
3. ✅ 冲突解决可以通过投票、辩论或仲裁
4. ✅ 团队反思帮助积累集体经验和持续改进

**下节课预告：**
我们将学习 Agent 安全与对齐。

---

## 参考资料

- [Contract Net Protocol: High-Level Communication and Control in a Distributed Problem Solver](https://ieeexplore.ieee.org/document/6312940)
- [Multi-Agent Systems: Algorithmic, Game-Theoretic, and Logical Foundations](https://mitpress.mit.edu/books/multi-agent-systems)
- [A Survey of Multi-Agent Coordination](https://arxiv.org/abs/2401.05574)

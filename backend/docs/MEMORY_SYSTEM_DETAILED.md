# DeerFlow Memory 系统详细解析

本文档详细解析 DeerFlow 的 Memory 系统，适合 Python 和 AI Agent 新手阅读。

---

## 一、整体架构概览

Memory 系统由 **4 个核心文件** 组成：

| 文件 | 作用 |
|------|------|
| `prompt.py` | 提示词模板 + 内存格式化工具 |
| `queue.py` | 防抖队列（避免频繁更新） |
| `updater.py` | 内存读写 + LLM 更新逻辑 |
| `__init__.py` | 导出模块 |

还有一个 middleware 文件串联整个流程：
- `memory_middleware.py` - 在对话后触发内存更新

---

## 二、完整调用流程图

```
用户与 Agent 对话
      ↓
[MemoryMiddleware.after_agent()]  ← 对话结束后触发
      ↓
[_filter_messages_for_memory()]    ← 过滤消息（只保留用户输入+最终AI回复）
      ↓
[get_memory_queue().add()]         ← 加入防抖队列
      ↓
（等待 debounce_seconds，默认 30 秒）
      ↓
[MemoryUpdateQueue._process_queue()]
      ↓
[MemoryUpdater.update_memory()]
      ↓
   ┌─────────────────────────────┐
   │ 1. get_memory_data()        │ ← 读取当前内存
   │ 2. format_conversation_for_update()  ← 格式化对话
   │ 3. 调用 LLM 生成更新        │ ← MEMORY_UPDATE_PROMPT
   │ 4. _apply_updates()         │ ← 应用更新
   │ 5. _strip_upload_mentions_from_memory()  ← 去掉上传文件相关
   │ 6. _save_memory_to_file()   │ ← 保存到文件
   └─────────────────────────────┘
```

---

## 三、详细代码解析

### 1. `prompt.py` - 提示词和格式化

#### 两个大提示词模板

**`MEMORY_UPDATE_PROMPT`**（第 15-117 行）：
- 告诉 LLM 如何分析对话并更新内存
- **输入**：当前内存 + 新对话
- **输出**：JSON 格式的更新指令

**输出 JSON 结构**（第 85-100 行）：
```python
{
  "user": {
    "workContext":      {"summary": "...", "shouldUpdate": true/false},
    "personalContext":  {"summary": "...", "shouldUpdate": true/false},
    "topOfMind":        {"summary": "...", "shouldUpdate": true/false}
  },
  "history": {
    "recentMonths":     {"summary": "...", "shouldUpdate": true/false},
    "earlierContext":   {"summary": "...", "shouldUpdate": true/false},
    "longTermBackground": {"summary": "...", "shouldUpdate": true/false}
  },
  "newFacts": [
    {"content": "...", "category": "preference|knowledge|context|behavior|goal", "confidence": 0.0-1.0}
  ],
  "factsToRemove": ["fact_id_1", "fact_id_2"]
}
```

**注意**：这里用双大括号 `{{` `}}` 是因为这是 Python 字符串模板，实际传给 LLM 时会变成单大括号。

---

#### `format_memory_for_injection()` - 把内存注入系统提示词

**入参**：
```python
memory_data: dict[str, Any]  # 内存数据
max_tokens: int = 2000        # 最多用多少 token
```

**出参**：
```python
str  # 格式化好的字符串，用于注入系统提示词
```

**代码逻辑**（第 186-300 行）：

1. **User Context 部分**（第 201-219 行）：
```python
# 输入示例
user_data = {
    "workContext": {"summary": "Python 开发者"},
    "personalContext": {"summary": "喜欢用中文交流"},
    "topOfMind": {"summary": "正在做 DeerFlow 项目"}
}

# 输出示例
"""User Context:
- Work: Python 开发者
- Personal: 喜欢用中文交流
- Current Focus: 正在做 DeerFlow 项目"""
```

2. **History 部分**（第 221-235 行）：类似 User Context

3. **Facts 部分**（第 237-284 行）：这部分比较复杂
   - 按 confidence 从高到低排序
   - 逐个添加，直到达到 token 预算
   - 格式：`- [category | confidence] content`

**举个完整例子**：

输入 `memory_data`：
```python
{
    "user": {
        "workContext": {"summary": "后端开发工程师，主要用 Python"},
        "personalContext": {"summary": "中文交流，喜欢简洁的代码"},
        "topOfMind": {"summary": "正在优化 DeerFlow 的 Memory 系统"}
    },
    "history": {
        "recentMonths": {"summary": "最近在研究 AI Agent 和 LangGraph"}
    },
    "facts": [
        {"id": "fact_1", "content": "用户喜欢用 PostgreSQL", "category": "knowledge", "confidence": 0.95},
        {"id": "fact_2", "content": "用户不喜欢写注释", "category": "preference", "confidence": 0.7}
    ]
}
```

输出 `format_memory_for_injection()` 结果：
```
User Context:
- Work: 后端开发工程师，主要用 Python
- Personal: 中文交流，喜欢简洁的代码
- Current Focus: 正在优化 DeerFlow 的 Memory 系统

History:
- Recent: 最近在研究 AI Agent 和 LangGraph

Facts:
- [knowledge | 0.95] 用户喜欢用 PostgreSQL
- [preference | 0.70] 用户不喜欢写注释
```

---

#### `format_conversation_for_update()` - 格式化对话给 LLM

**入参**：
```python
messages: list[Any]  # LangChain 的消息列表
```

**出参**：
```python
str  # 格式化后的对话文本
```

**代码逻辑**（第 303-339 行）：
1. 遍历每条消息
2. 如果是 human 消息，去掉 `<uploaded_files>` 标签
3. 如果是 ai 消息且没有 tool_calls，保留
4. 格式：`User: xxx\n\nAssistant: xxx`

**输入输出示例**：

输入：
```python
[
    HumanMessage(content="你好，我叫小明"),
    AIMessage(content="你好小明！有什么我可以帮你的吗？"),
    HumanMessage(content="<uploaded_files>...</uploaded_files>\n\n帮我看看这个文件"),
    AIMessage(content="好的，我来看看...", tool_calls=[...]),  # 这个会被过滤掉
    ToolMessage(content="文件内容..."),  # 这个会被过滤掉
    AIMessage(content="这个文件是关于 Python 的")
]
```

输出：
```
User: 你好，我叫小明

Assistant: 你好小明！有什么我可以帮你的吗？

User: 帮我看看这个文件

Assistant: 这个文件是关于 Python 的
```

---

### 2. `queue.py` - 防抖队列

为什么需要防抖？因为用户可能连续发消息，每次都更新内存太浪费了。

#### 数据结构：`ConversationContext`
```python
@dataclass
class ConversationContext:
    thread_id: str           # 对话线程 ID
    messages: list[Any]      # 过滤后的消息
    timestamp: datetime      # 时间戳
    agent_name: str | None   # 可选：按 agent 分开存储
```

#### `MemoryUpdateQueue` 类

**核心方法**：

| 方法 | 作用 |
|------|------|
| `add(thread_id, messages, agent_name)` | 添加到队列 |
| `_reset_timer()` | 重置防抖计时器 |
| `_process_queue()` | 处理队列中的所有更新 |
| `flush()` | 强制立即处理 |

**调用流程图**：

```
用户发消息 → add()
              ↓
         [锁保护]
              ↓
    同一 thread_id 的旧条目删掉，新的加上
              ↓
    取消旧计时器，启动新计时器（30秒）
              ↓
         （30秒后...）
              ↓
         _process_queue()
              ↓
         遍历所有 context
              ↓
         MemoryUpdater.update_memory()
```

**关键点**：
- 第 58 行：`self._queue = [c for c in self._queue if c.thread_id != thread_id]`
  - 如果同一个 thread 已经在队列里，先删掉旧的
  - 这样保证一个 thread 只保留最新的对话

---

### 3. `updater.py` - 内存读写和更新

这是最核心的文件。

#### 内存文件结构

存储在 `backend/.deer-flow/memory.json`：
```python
{
  "version": "1.0",
  "lastUpdated": "2026-03-20T10:30:00Z",
  "user": {
    "workContext":        {"summary": "...", "updatedAt": "..."},
    "personalContext":    {"summary": "...", "updatedAt": "..."},
    "topOfMind":          {"summary": "...", "updatedAt": "..."}
  },
  "history": {
    "recentMonths":       {"summary": "...", "updatedAt": "..."},
    "earlierContext":     {"summary": "...", "updatedAt": "..."},
    "longTermBackground": {"summary": "...", "updatedAt": "..."}
  },
  "facts": [
    {
      "id": "fact_abc123",
      "content": "用户喜欢用 Python",
      "category": "preference",
      "confidence": 0.9,
      "createdAt": "2026-03-20T10:00:00Z",
      "source": "thread_123"
    }
  ]
}
```

---

#### `MemoryUpdater.update_memory()` - 更新内存的主流程

**入参**：
```python
messages: list[Any]           # 对话消息
thread_id: str | None = None  # 线程 ID
agent_name: str | None = None # agent 名称
```

**出参**：
```python
bool  # True 表示更新成功
```

**代码步骤**（第 235-299 行）：

```python
def update_memory(self, messages, thread_id=None, agent_name=None):
    # 步骤 1: 读取当前内存
    current_memory = get_memory_data(agent_name)

    # 步骤 2: 格式化对话
    conversation_text = format_conversation_for_update(messages)

    # 步骤 3: 构建提示词，调用 LLM
    prompt = MEMORY_UPDATE_PROMPT.format(
        current_memory=json.dumps(current_memory, indent=2),
        conversation=conversation_text,
    )
    model = self._get_model()
    response = model.invoke(prompt)
    response_text = str(response.content).strip()

    # 步骤 4: 解析 LLM 返回的 JSON
    if response_text.startswith("```"):
        lines = response_text.split("\n")
        response_text = "\n".join(lines[1:-1])
    update_data = json.loads(response_text)

    # 步骤 5: 应用更新
    updated_memory = self._apply_updates(current_memory, update_data, thread_id)

    # 步骤 6: 去掉上传文件相关内容
    updated_memory = _strip_upload_mentions_from_memory(updated_memory)

    # 步骤 7: 保存到文件
    return _save_memory_to_file(updated_memory, agent_name)
```

---

#### `_apply_updates()` - 应用 LLM 的更新

**入参**：
```python
current_memory: dict[str, Any]   # 当前内存
update_data: dict[str, Any]      # LLM 返回的更新数据
thread_id: str | None = None
```

**出参**：
```python
dict[str, Any]  # 更新后的内存
```

**代码逻辑**（第 301-369 行）：

1. **更新 User 部分**（第 320-328 行）：
```python
user_updates = update_data.get("user", {})
for section in ["workContext", "personalContext", "topOfMind"]:
    section_data = user_updates.get(section, {})
    if section_data.get("shouldUpdate") and section_data.get("summary"):
        current_memory["user"][section] = {
            "summary": section_data["summary"],
            "updatedAt": now,
        }
```
- 只有 `shouldUpdate=True` 才更新

2. **更新 History 部分**（第 330-338 行）：类似 User 部分

3. **删除 Facts**（第 340-343 行）：
```python
facts_to_remove = set(update_data.get("factsToRemove", []))
current_memory["facts"] = [f for f in current_memory["facts"]
                           if f.get("id") not in facts_to_remove]
```

4. **添加新 Facts**（第 345-358 行）：
```python
new_facts = update_data.get("newFacts", [])
for fact in new_facts:
    confidence = fact.get("confidence", 0.5)
    if confidence >= config.fact_confidence_threshold:  # 默认 0.7
        fact_entry = {
            "id": f"fact_{uuid.uuid4().hex[:8]}",  # 生成 ID
            "content": fact.get("content", ""),
            "category": fact.get("category", "context"),
            "confidence": confidence,
            "createdAt": now,
            "source": thread_id or "unknown",
        }
        current_memory["facts"].append(fact_entry)
```

5. **限制 Facts 数量**（第 360-367 行）：
```python
if len(current_memory["facts"]) > config.max_facts:  # 默认 100
    current_memory["facts"] = sorted(
        current_memory["facts"],
        key=lambda f: f.get("confidence", 0),
        reverse=True,
    )[: config.max_facts]  # 只保留置信度最高的 100 个
```

---

### 4. `memory_middleware.py` - 串联流程

#### `MemoryMiddleware.after_agent()` - 对话后触发

```python
def after_agent(self, state, runtime):
    # 1. 获取 thread_id
    thread_id = runtime.context.get("thread_id")

    # 2. 获取消息
    messages = state.get("messages", [])

    # 3. 过滤消息（只保留用户输入+最终AI回复）
    filtered_messages = _filter_messages_for_memory(messages)

    # 4. 加入队列
    queue = get_memory_queue()
    queue.add(thread_id=thread_id, messages=filtered_messages, agent_name=self._agent_name)
```

---

## 四、完整数据流示例

让我们用一个真实场景走一遍：

### 场景：用户第一次对话

```
用户: "你好，我是张三，一名 Python 后端开发者，我正在研究 AI Agent"
AI: "你好张三！很高兴认识你。AI Agent 是个很有意思的方向！"
```

#### 步骤 1: MemoryMiddleware 触发

```python
# 原始消息
messages = [
    HumanMessage(content="你好，我是张三，一名 Python 后端开发者，我正在研究 AI Agent"),
    AIMessage(content="你好张三！很高兴认识你。AI Agent 是个很有意思的方向！")
]

# _filter_messages_for_memory() 过滤后（和原来一样，因为没有 tool calls）
filtered_messages = messages

# 加入队列
queue.add(thread_id="thread_123", messages=filtered_messages)
```

#### 步骤 2: 30 秒后，队列处理

```python
# 读取当前内存（空的，返回 _create_empty_memory()）
current_memory = {
    "version": "1.0",
    "lastUpdated": "...",
    "user": {
        "workContext": {"summary": "", "updatedAt": ""},
        "personalContext": {"summary": "", "updatedAt": ""},
        "topOfMind": {"summary": "", "updatedAt": ""},
    },
    "history": {...},
    "facts": [],
}

# 格式化对话
conversation_text = """
User: 你好，我是张三，一名 Python 后端开发者，我正在研究 AI Agent

Assistant: 你好张三！很高兴认识你。AI Agent 是个很有意思的方向！
"""

# 构建提示词并调用 LLM
prompt = MEMORY_UPDATE_PROMPT.format(
    current_memory=json.dumps(current_memory, indent=2),
    conversation=conversation_text,
)

# LLM 返回（假设）
update_data = {
    "user": {
        "workContext": {
            "summary": "Python 后端开发者",
            "shouldUpdate": true
        },
        "personalContext": {
            "summary": "叫张三，中文交流",
            "shouldUpdate": true
        },
        "topOfMind": {
            "summary": "正在研究 AI Agent",
            "shouldUpdate": true
        }
    },
    "history": {
        "recentMonths": {
            "summary": "开始学习 AI Agent",
            "shouldUpdate": true
        },
        ...
    },
    "newFacts": [
        {
            "content": "用户名叫张三",
            "category": "context",
            "confidence": 1.0
        },
        {
            "content": "用户是 Python 后端开发者",
            "category": "context",
            "confidence": 1.0
        },
        {
            "content": "用户正在研究 AI Agent",
            "category": "goal",
            "confidence": 0.9
        }
    ],
    "factsToRemove": []
}

# 应用更新
updated_memory = _apply_updates(current_memory, update_data, thread_id="thread_123")

# 保存
_save_memory_to_file(updated_memory)
```

#### 最终 memory.json 内容：

```json
{
  "version": "1.0",
  "lastUpdated": "2026-03-20T10:30:00Z",
  "user": {
    "workContext": {
      "summary": "Python 后端开发者",
      "updatedAt": "2026-03-20T10:30:00Z"
    },
    "personalContext": {
      "summary": "叫张三，中文交流",
      "updatedAt": "2026-03-20T10:30:00Z"
    },
    "topOfMind": {
      "summary": "正在研究 AI Agent",
      "updatedAt": "2026-03-20T10:30:00Z"
    }
  },
  "history": {
    "recentMonths": {
      "summary": "开始学习 AI Agent",
      "updatedAt": "2026-03-20T10:30:00Z"
    },
    "earlierContext": {"summary": "", "updatedAt": ""},
    "longTermBackground": {"summary": "", "updatedAt": ""}
  },
  "facts": [
    {
      "id": "fact_abc123de",
      "content": "用户名叫张三",
      "category": "context",
      "confidence": 1.0,
      "createdAt": "2026-03-20T10:30:00Z",
      "source": "thread_123"
    },
    {
      "id": "fact_456fghij",
      "content": "用户是 Python 后端开发者",
      "category": "context",
      "confidence": 1.0,
      "createdAt": "2026-03-20T10:30:00Z",
      "source": "thread_123"
    },
    {
      "id": "fact_789klmno",
      "content": "用户正在研究 AI Agent",
      "category": "goal",
      "confidence": 0.9,
      "createdAt": "2026-03-20T10:30:00Z",
      "source": "thread_123"
    }
  ]
}
```

---

## 五、函数调用关系速查表

```
MemoryMiddleware.after_agent()
    ↓
    ├─→ _filter_messages_for_memory()
    │       └─→ 过滤消息
    │
    └─→ get_memory_queue().add()
            ↓
            └─→ MemoryUpdateQueue._reset_timer()
                    ↓
            （30 秒后...）
                    ↓
            MemoryUpdateQueue._process_queue()
                ↓
                └─→ MemoryUpdater.update_memory()
                        ├─→ get_memory_data()
                        │       └─→ _load_memory_from_file()
                        │
                        ├─→ format_conversation_for_update()  ← prompt.py
                        │
                        ├─→ 调用 LLM (model.invoke())
                        │
                        ├─→ MemoryUpdater._apply_updates()
                        │       ├─→ 更新 user/history 部分
                        │       ├─→ 删除 facts
                        │       ├─→ 添加新 facts
                        │       └─→ 限制 facts 数量
                        │
                        ├─→ _strip_upload_mentions_from_memory()
                        │
                        └─→ _save_memory_to_file()
                                └─→ 原子写入（temp 文件 + rename）


format_memory_for_injection()  ← 独立调用，用于注入系统提示词
    ├─→ _count_tokens()
    └─→ _coerce_confidence()
```

---

## 六、关键设计点

1. **防抖队列**：避免频繁调用 LLM 更新内存
2. **原子写入**：用 temp 文件 + rename，防止写入中断导致文件损坏
3. **缓存机制**：`_memory_cache` 存储内存数据，检查文件 mtime 自动失效
4. **上传过滤**：不上传文件事件记录到内存（因为文件是会话级的）
5. **Token 预算**：`format_memory_for_injection()` 限制注入的 token 数量

---

## 七、文件位置

- `backend/packages/harness/deerflow/agents/memory/prompt.py`
- `backend/packages/harness/deerflow/agents/memory/queue.py`
- `backend/packages/harness/deerflow/agents/memory/updater.py`
- `backend/packages/harness/deerflow/agents/memory/__init__.py`
- `backend/packages/harness/deerflow/agents/middlewares/memory_middleware.py`

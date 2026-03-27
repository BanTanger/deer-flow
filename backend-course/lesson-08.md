# 第 8 课：Memory 为什么难，DeerFlow 怎么做

很多人第一次给 Agent 加记忆，会想得很简单：不就是把对话存下来吗？但真正做起来后很快会发现，memory 的难点根本不在“能不能存”，而在“存什么、什么时候更新、什么时候注入、哪些东西绝对不能带到未来会话里”。

DeerFlow 的 memory 模块一开头就把自己的职责讲得很清楚：

```python
"""Memory module for DeerFlow.

This module provides a global memory mechanism that:
- Stores user context and conversation history in memory.json
- Uses LLM to summarize and extract facts from conversations
- Injects relevant memory into system prompts for personalized responses
"""
```

这段说明已经把坑点说出来了。Memory 不是原样存聊天记录，而是要靠模型把对话抽取、压缩、整理成对未来还有价值的事实。否则存下来的只会是一堆噪声。

DeerFlow 的更新逻辑也很典型：

```python
prompt = MEMORY_UPDATE_PROMPT.format(
    current_memory=json.dumps(current_memory, indent=2),
    conversation=conversation_text,
)

model = self._get_model()
response = model.invoke(prompt)
response_text = _extract_text(response.content).strip()
update_data = json.loads(response_text)
updated_memory = self._apply_updates(current_memory, update_data, thread_id)
updated_memory = _strip_upload_mentions_from_memory(updated_memory)
```

这里你能看到 memory 系统真正难的地方：

- **先把当前记忆和新对话一起交给模型**
- **再让模型产出结构化更新**
- **再把更新合并回 memory**
- **最后还要做清洗，比如去掉上传文件这种会过期的信息**

那句 `_strip_upload_mentions_from_memory` 特别值得你记住，因为它体现了一个经常被忽略的坑：有些信息只在当前会话有效，比如上传文件路径。如果你把这些东西也记成“长期记忆”，后面系统就会在新会话里不断去找早就不存在的文件。

DeerFlow 还把 memory 注入做成了动态行为，而不是每次都硬塞：

- memory 配置关闭时，不注入
- 没有有效内容时，不注入
- 有 token 上限时，按上限裁剪

**Memory 这一层最容易踩的坑**

- **把完整历史原样塞回 prompt**
- **把短期临时信息误当长期记忆**
- **只有存，没有清洗和合并规则**
- **更新和注入没有分开，导致 prompt 越来越重**

**这一课最后你最该记住的点**

- **Memory 难的不是存下来，而是提炼和清洗**
- **长期记忆和当前线程状态不是一回事**
- **DeerFlow 的 memory 本质上是“结构化更新 + 受控注入”**
- **上传文件、临时路径这类会过期信息，不该进长期记忆**

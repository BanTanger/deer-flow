# 第 8 课：Memory 为什么难，DeerFlow 怎么做

很多人第一次给 Agent 加 memory，脑子里都会出现一个很自然的想法：不就是把历史存下来吗？可只要你真的做过两轮，就会发现问题根本没这么简单。因为 Agent 的 memory 难点从来不是“能不能存”，而是下面这几个问题：

- **到底存什么**
- **什么时候更新**
- **哪些内容只是当前线程有效**
- **哪些内容值得带去未来会话**

如果这些边界不清楚，memory 很快就会从“增强体验”变成“制造噪声”。系统一开始也许只是记得太多，后面就会逐渐演化成：把过期路径记成长期事实，把一次性的上传文件记成用户偏好，把整段原始对话重新塞回 prompt，最后上下文越来越重，记忆却越来越不可靠。

DeerFlow 在 memory 模块开头写得很直白：

```python
"""Memory module for DeerFlow.

This module provides a global memory mechanism that:
- Stores user context and conversation history in memory.json
- Uses LLM to summarize and extract facts from conversations
- Injects relevant memory into system prompts for personalized responses
"""
```

这段说明里最重要的词不是 `stores`，而是 `summarize` 和 `extract facts`。也就是说，DeerFlow 从一开始就没有把 memory 理解成“原样存档聊天记录”，而是理解成：

**从历史里提炼对未来还有价值的事实。**

这个判断非常关键。因为真正对后续会话有帮助的，通常不是“用户在某次对话里说过的全部句子”，而是那些稳定、可复用、跨轮仍然成立的信息。比如用户偏好的语言、长期项目背景、持续关注的话题、之前已经确认过的固定约束。相反，像一次性上传的文件路径、某次调试时的临时目录、只在当前线程生效的工作现场，这些东西就不该被当成长久记忆带到未来。

DeerFlow 的更新流程，正好把这件事做得很清楚：

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

这段代码背后其实有四个动作。

第一，DeerFlow 先把“已有记忆”和“新对话”一起交给模型，而不是只让模型盯着最新一轮说话。这样模型在更新 memory 时，看到的是一个增量上下文，而不是从零开始乱猜。

第二，模型返回的不是一段自然语言总结，而是结构化更新。这一点特别重要，因为记忆更新如果只是自由文本，很快就会难以合并、难以裁剪、难以判断冲突。DeerFlow 让模型先产出更新意图，再由系统去合并，这就把“提炼”和“落地”分开了。

第三，系统会把模型产出的更新显式 merge 回当前 memory，而不是每次整份重写。这意味着 memory 不是一轮轮互相覆盖，而是受控地演化。

第四，也是最有后端意识的一步，是 `_strip_upload_mentions_from_memory(updated_memory)`。这句看起来很小，但它正好点中了记忆系统最容易忽略的坑：

**不是所有信息都配得上“长期记忆”这个身份。**

上传文件路径、临时素材、一次性目录，这些内容在当前线程里当然有价值，可一旦线程结束、文件清理、路径变化，它们就会立刻过期。如果你把它们也沉淀进长期 memory，系统后面就会不断引用一个已经不存在的世界。

所以 DeerFlow 的 memory 设计，本质上是在区分两种完全不同的信息：

- **线程态信息**：只在当前任务现场有效
- **长期记忆信息**：跨会话仍然有稳定价值

这个区分一旦不做，memory 就一定会出问题。很多系统并不是“记不住”，而是“记错了该记的层级”。

DeerFlow 在注入阶段也保持了同样的克制。它不是每次都无脑把 memory 塞回 prompt，而是按条件动态注入：

- memory 配置没开，就不注入
- memory 为空，就不注入
- 有 token 限制时，就裁剪到合适大小

这说明 DeerFlow 不是把 memory 当成“越多越好”的附件，而是当成一种受控上下文。只有当前真的有帮助时，它才值得占用上下文预算。

你也能看出这一课和第 4 课之间的关系。ThreadState 解决的是当前线程运行现场，Memory 解决的是跨线程、跨会话还能成立的长期信息。两者都叫“状态”，但它们根本不是同一层。很多系统会把这两层混在一起，结果就是：要么所有短期信息都被错误沉淀，要么长期偏好根本无法被可靠继承。

从工程角度看，memory 最容易坏掉的方式通常有四种：

- **把完整历史原样塞回 prompt**
- **把一次性临时信息误当长期记忆**
- **只有存储，没有清洗和合并规则**
- **把更新和注入混成一个动作**

DeerFlow 避开这些坑的方式很清楚：更新是结构化的，注入是动态受控的，清洗是显式存在的，长期记忆和线程状态是分层处理的。

如果把这一课压成一句话，我会这样说：

**Memory 真正难的，不是把过去留下来，而是只把值得留下的那部分过去留下来。**

DeerFlow 的做法值得学，正是因为它没有把“记住更多”当目标，而是把“提炼对未来仍然有效的信息”当目标。前者只会让上下文越来越重，后者才会让系统越来越像一个真正有记性的后端。

**这一课最后你最该记住的点**

- **Memory 的难点不是存储，而是提炼、清洗和分层**
- **长期记忆和当前线程状态不是一回事**
- **结构化更新比原样回灌更适合生产级 memory**
- **上传文件、临时路径、一次性素材这类信息，不该进入长期记忆**

# 第 9 课：Subagent 为什么会让系统更强，也更复杂

很多复杂任务都不适合让一个 Agent 从头干到尾。原因很简单：任务维度一多，单个 Agent 的上下文就会越来越拥挤，决策也会越来越混乱。于是多 Agent 协作就成了自然选择。但这里也有一个经典误区：只要引入 subagent，系统就一定更强。事实并不是这样，因为 subagent 在带来并行能力的同时，也会把调度、限流、状态共享、结果收敛这些问题全部带进来。

DeerFlow 对 subagent 的处理方式非常工程化。先看它的执行器骨架：

```python
class SubagentExecutor:
    def _create_agent(self):
        model_name = _get_model_name(self.config, self.parent_model)
        model = create_chat_model(name=model_name, thinking_enabled=False)

        middlewares = build_subagent_runtime_middlewares(lazy_init=True)

        return create_agent(
            model=model,
            tools=self.tools,
            middleware=middlewares,
            system_prompt=self.config.system_prompt,
            state_schema=ThreadState,
        )
```

这段代码最值得你注意的地方，不是“子 Agent 也能 create_agent”，而是它和主 Agent 明显共享同一种 runtime 语言：子 Agent 也有模型、工具、中间件、状态 schema。也就是说，在 DeerFlow 里，subagent 不是一个随便开的线程，而是一个缩小版但结构完整的 agent runtime。

DeerFlow 还做了两件很重要的事。第一，子 Agent 默认继承父 Agent 的一部分环境，比如 sandbox 和 thread_data：

```python
state: dict[str, Any] = {
    "messages": [HumanMessage(content=task)],
}

if self.sandbox_state is not None:
    state["sandbox"] = self.sandbox_state
if self.thread_data is not None:
    state["thread_data"] = self.thread_data
```

这意味着子 Agent 不是在真空里工作，它会继承父 Agent 所在的任务环境。第二，DeerFlow 用线程池和状态对象把 subagent 结果托管起来，而不是让主 Agent 自己轮询到怀疑人生。这样主 Agent 看到的是一个可追踪、有限流、有状态的子任务体系，而不是一堆随便飞出去的异步调用。

**Subagent 最容易踩的坑**

- **看见复杂任务就乱开 subagent**
- **不限制并发，结果把系统拖死**
- **子 Agent 不共享必要环境，导致它们各自重新摸索**
- **结果回收和汇总没有统一机制**

DeerFlow 的子 Agent 设计给你的最大启发，其实不是“多 Agent 很高级”，而是“多 Agent 必须比单 Agent 更有纪律”。没有纪律的 subagent 不是增强，是灾难。

**这一课最后你最该记住的点**

- **subagent 是完整 runtime，不是随便开的线程**
- **subagent 的价值在于拆解和并行，不在于“看起来高级”**
- **共享环境、并发限制、结果托管，这三件事缺一不可**
- **多 Agent 系统真正难的部分，是调度和收敛**

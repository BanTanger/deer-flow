# 第 9 课：Subagent 为什么会让系统更强，也更复杂

复杂任务天然会把单个 Agent 推向极限。信息一多、目标一散、步骤一长，单个 Agent 的上下文就会越来越拥挤，决策也会越来越不稳定。于是很多人学到这里，会自然产生一个想法：

**那就拆成多个 Agent 并行做吧。**

这个方向并没有错，甚至常常是必要的。但问题在于，subagent 从来不是“自动增强器”。它确实能带来拆解、并行和任务隔离，却也会同时把调度、限流、状态共享、结果归并这些更难的工程问题一起带进来。很多系统不是死在“不会多 Agent”，而是死在“会开子 Agent，却不会管理它们”。

DeerFlow 对 subagent 的设计很值得学，因为它一开始就没有把子 Agent 当成一个轻飘飘的“异步任务”。先看执行器的骨架：

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

这段代码里最重要的不是“子 Agent 也会 `create_agent(...)`”，而是它透露出的一个明确态度：

**Subagent 不是随便开出去的一段任务，而是一个缩小版但结构完整的 runtime。**

你看它有什么：自己的模型、自己的工具、自己的中间件、自己的系统 prompt，甚至共享同一套 `ThreadState` 语言。也就是说，在 DeerFlow 里，subagent 并不是“主 Agent 的一个函数调用”，而是“主 Agent 派出去的一个小型执行单元”。

这个判断非常关键。因为只要你把 subagent 看轻了，后面就很容易犯两个经典错误。第一，把任何任务都包成子 Agent，导致调度开销远大于收益。第二，子 Agent 之间没有统一 runtime 约束，结果每个子任务都像在野外生长。

DeerFlow 在这件事上显然更克制。它会让子 Agent 继承必要环境，而不是让它们从零开始摸索：

```python
state: dict[str, Any] = {
    "messages": [HumanMessage(content=task)],
}

if self.sandbox_state is not None:
    state["sandbox"] = self.sandbox_state
if self.thread_data is not None:
    state["thread_data"] = self.thread_data
```

这两句继承逻辑非常重要。因为真正好的 subagent，不是在真空里解决问题，而是在父 Agent 已经准备好的任务现场里继续推进。继承 sandbox，意味着它知道自己在哪个受控执行环境里行动；继承 thread_data，意味着它知道上传目录、工作目录、输出目录这些路径上下文，不需要再重新猜一遍。

换句话说，DeerFlow 并没有把 subagent 设计成一群各自为战的小模型，而是设计成：

**共享总体任务环境，但各自处理一块明确子任务。**

这就把“并行”从混乱变成了可管理的拆解。

不过，subagent 真正容易失控的地方，不在“能不能跑起来”，而在“会不会越开越多”。这一点第 2 课里的 prompt 规则已经提前埋了伏笔：DeerFlow 会明确告诉模型，每轮最多开多少个 subagent，简单任务不要包装成 subagent，超出并发上限时要分批执行。很多人以为这些只是 prompt 层提醒，但你现在应该能看出，它背后是在保护整个 runtime 不被过度拆解拖垮。

因为 subagent 最常见的坏法，就是这几种：

- **看见复杂任务就条件反射式乱开子 Agent**
- **简单任务也包装成 subagent，徒增调度成本**
- **没有并发限制，最后线程池、工具、token 一起爆**
- **结果回收和汇总没有统一机制**

DeerFlow 的应对思路很清楚：让 subagent 真正有 runtime 身份，让它继承必要环境，再用线程池、限流和状态托管把它收在一个可管理框架里。主 Agent 最终面对的，不是一堆乱飞的异步调用，而是一组可追踪、可限制、可归并的子任务执行单元。

这也是为什么 DeerFlow 的 subagent 设计很有“后端味道”。它不是在追求“多 Agent 看起来很高级”，而是在解决一个很工程的问题：

**当一个任务大到不适合单 Agent 独立完成时，系统怎么拆，而拆完之后又怎么收回来。**

拆得开是第一步，收得回来才是真功夫。很多多 Agent demo 会给你很强的第一印象，因为它们很会拆，但真正落到生产级系统里，最关键的恰恰是收敛机制。谁负责汇总结果？共享哪些上下文？并发上限怎么定？失败时主 Agent 怎么继续决策？如果这些没处理好，多 Agent 不是增强，而是把单 Agent 的复杂度再乘一遍。

所以你可以把 DeerFlow 对 subagent 的理解总结成一句更朴素的话：

**多 Agent 系统真正难的，不是“派出去”，而是“派出去之后仍然可控”。**

这也是为什么 DeerFlow 会同时强调共享环境、并发限制和结果托管。这三件事缺一不可。没有共享环境，子 Agent 会重复摸索；没有并发限制，资源会失控；没有结果托管，主 Agent 最后根本不知道怎么稳定收束。

如果把这一课压成一句话，我会这样说：

**Subagent 的价值不在于“多开几个模型”，而在于把复杂任务拆到更好管理；而它的难点，则在于拆完之后系统还能不能稳稳收回来。**

DeerFlow 值得你学的地方，正是它从一开始就把 subagent 当成一个要认真治理的 runtime 层，而不是一个为了显得高级而加上的炫技模块。

**这一课最后你最该记住的点**

- **subagent 是完整 runtime 单元，不是随便开的线程**
- **共享必要环境，才能让子 Agent 真正接住父任务上下文**
- **并发限制和结果托管，是多 Agent 稳定性的核心**
- **多 Agent 真正难的不是拆解，而是调度和收敛**

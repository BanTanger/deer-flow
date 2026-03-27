# 第 13 课：Checkpointer 与恢复机制为什么是生产级能力

很多 Agent demo 都有一个共同特点：它们看起来能跑，但一旦进程重启、会话中断、线程跨时段恢复，系统就像失忆了一样。这个问题在短演示里不明显，但只要你把 Agent 当成一个真正的后端服务，它立刻就会变成大问题。DeerFlow 之所以比 demo 更像系统，很重要的一点就是它认真对待了 checkpointer。

你可以先看 DeerFlow 的 checkpointer 工厂：

```python
if config.type == "memory":
    yield InMemorySaver()  # 仅进程内，不持久化

if config.type == "sqlite":
    with SqliteSaver.from_conn_string(conn_str) as saver:
        saver.setup()
        yield saver

if config.type == "postgres":
    with PostgresSaver.from_conn_string(config.connection_string) as saver:
        saver.setup()
        yield saver
```

这段代码其实很直白，但它背后回答的是一个很关键的问题：线程状态到底要不要持久化，以及持久化到什么级别。DeerFlow 没有假设所有场景都一样，而是给了 memory、sqlite、postgres 三种层次。你可以把它粗略理解成：

- **memory**：适合开发和测试，快，但不持久
- **sqlite**：适合单机持久化，轻量
- **postgres**：适合更正式的生产环境

DeerFlow 还同时提供了 singleton 和 context manager 两种用法，这也很实用。前者适合长期运行的进程复用连接，后者适合 CLI 或测试里用完就关。这里你能看到 DeerFlow 的一贯风格：不是只做“能工作”的版本，而是把使用场景也考虑进来。

**如果没有 checkpointer，会先出什么问题**

- **线程跨进程无法恢复**
- **长任务中断后前功尽弃**
- **调试时很难复现“中间状态”**
- **Agent 看起来能跑，但其实很脆**

你可以把 checkpointer 理解成 DeerFlow 给线程状态加的一层“落盘能力”。ThreadState 解决的是“运行中怎么组织状态”，checkpointer 解决的是“状态能不能在运行之外继续活着”。这两个层次加在一起，Agent 后端才真正开始像长期服务，而不是一次性脚本。

**这一课最后你最该记住的点**

- **ThreadState 解决运行中状态，checkpointer 解决运行外状态**
- **memory / sqlite / postgres 对应不同持久化层级**
- **恢复能力不是锦上添花，而是生产级系统的基本面**
- **没有 checkpointer 的 Agent，长任务和跨时段会话都会很脆**

# 第 4 课：ThreadState 为什么是后端的状态背板

新手最容易低估的一件事，就是“状态”到底有多重要。很多人一开始做 Agent，只把 `messages` 当状态，觉得有对话记录就够了。但只要你做过长任务，就会很快发现不够。系统还得知道当前 sandbox 是哪个，工作目录在哪里，输出目录在哪里，用户上传了什么文件，已经生成了哪些 artifacts，甚至哪些图片已经被读过。没有统一状态，这些信息很快就会散落在函数参数、临时变量和工具返回值里。

DeerFlow 用 `ThreadState` 正面解决了这个问题：

```python
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]          # 当前线程的 sandbox
    thread_data: NotRequired[ThreadDataState | None]   # workspace / uploads / outputs 路径
    title: NotRequired[str | None]                     # 线程标题
    artifacts: Annotated[list[str], merge_artifacts]   # 产物列表，更新时做合并
    todos: NotRequired[list | None]                    # todo 或计划状态
    uploaded_files: NotRequired[list[dict] | None]     # 上传文件信息
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]  # 已查看图片
```

这段代码里面最容易卡人的不是 `NotRequired`，而是 `Annotated[...]`。`Annotated` 在这里的意义，不只是说明字段类型，还顺手把“这个字段更新时用什么规则”一起带上。比如 `artifacts` 用的是 `merge_artifacts`，意思就是后面状态更新时，不是简单覆盖掉旧值，而是要合并。这种写法很适合 Agent 场景，因为很多状态天生就是“逐步累积”的，而不是“后写覆盖前写”。

**如果没有统一状态，会先出什么问题**

- **工具层彼此不共享上下文**
- **中间件只能各管各的，没法协作**
- **文件路径到处传，越来越乱**
- **长任务越来越像一堆临时补丁**

ThreadState 的价值，不只是“多放几个字段”，而是它让后端出现了一个统一的状态背板。后面的工具、中间件、sandbox、memory 都可以围绕它协作。这也是为什么 DeerFlow 能做 thread 级工作区、sandbox 绑定、artifact 聚合和图片状态管理，而不是靠到处传参勉强拼起来。

如果你往前回想第 1 课里的 `create_agent(..., state_schema=ThreadState)`，这件事的意义就更清楚了。DeerFlow 不是在运行时偶尔顺手带一点状态，而是从系统最开始就声明：这整个 agent 的状态结构，就是 ThreadState。这个决定非常关键，因为它给了后面所有模块一个共同语言。

**这一课最后你最该记住的点**

- **复杂 Agent 不能只靠 messages，当状态必须显式化**
- **ThreadState 是 DeerFlow 各模块共享的状态背板**
- **`Annotated[...]` 这种写法的关键价值，是把更新规则一起写进状态定义**
- **没有统一状态，后面的 middleware、tools、sandbox 都会越来越乱**

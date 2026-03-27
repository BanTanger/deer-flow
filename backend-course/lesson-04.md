# 第 4 课：ThreadState 为什么是后端的状态背板

很多人刚开始做 Agent 后端时，会天然把“状态”理解成聊天消息。因为从表面看，Agent 不就是一轮一轮收消息、回消息吗？可一旦系统开始真的执行任务，你就会很快发现，`messages` 只能承载“说过什么”，却承载不了“系统此刻处在什么执行现场”。

比如同一个线程里，系统可能已经拿到了 sandbox，已经为用户准备好了 workspace、uploads、outputs 三个目录，已经聚合出一批 artifacts，已经记录了上传文件元信息，还知道哪些图片已经被读取过。这些东西都不是一句对话能自然表达清楚的。如果你不把它们放进统一状态，它们就会散落到参数、局部变量、工具返回值和中间件内部缓存里。系统前期还能勉强跑，后面一定越跑越乱。

DeerFlow 处理这个问题的方式很直接：不要假装“对话历史就是全部状态”，而是显式定义 `ThreadState`，把线程级运行现场收口成统一的状态背板。

```python
class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

这段定义里，最值得你停一下看的，不是字段名本身，而是它透露出来的一种后端姿势：

**系统不是只有对话，还有线程级执行现场。**

`sandbox` 表示当前线程绑定到了哪一个执行环境；`thread_data` 表示这条线程的工作目录、上传目录、输出目录；`artifacts` 表示已经产出的成果物；`uploaded_files` 表示用户实际交给系统的材料；`viewed_images` 则说明哪些图片已经看过，避免系统反复处理同一资源。

换句话说，ThreadState 不是“给对话多挂几个字段”，而是在给 Agent runtime 建一个统一的共享地板。后面的 middleware、tools、subagent、memory，都站在这块地板上协作。

很多人第一次看到这里，会把注意力放在 `NotRequired[...]`。它当然重要，但 DeerFlow 里更有意思的是 `Annotated[...]`。像 `artifacts` 和 `viewed_images` 这种字段，并不是每次都应该“新值覆盖旧值”。它们天然更像“增量积累”。所以 DeerFlow 不是把更新规则散落到每个调用方里，而是直接把“这个字段该怎么 merge”绑定进状态定义：

```python
artifacts: Annotated[list[str], merge_artifacts]
viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

这件事背后的意义非常大。它等于在说：

- **状态字段不只是有类型**
- **状态字段还有自己的更新语义**
- **这些语义应该被集中声明，而不是靠调用方自己猜**

这就是为什么 `ThreadState` 在 DeerFlow 里不是普通数据结构，而是状态协议。你可以把它理解成 DeerFlow 在系统层面明确规定：

**以后大家都围绕这一份线程状态说话。**

这会直接带来几个很现实的好处。

第一，中间件之间终于可以协作。`SandboxMiddleware` 往状态里写了 sandbox，后面的工具层就能直接拿；`ThreadDataMiddleware` 先把目录信息补进去，文件相关工具和 artifact 逻辑就不需要再自己重新推导路径。没有 ThreadState 时，每一层都像在单独打仗；有了它之后，模块之间开始有共同语言。

第二，系统终于有机会在长任务里保持一致性。长任务最怕的不是单点失败，而是系统在不同步骤里对“当前现场”有不同理解。明明前面已经有了 outputs 路径，后面又重新算一遍；明明上传文件已经登记，另一个模块却还在扫目录猜测。ThreadState 的价值，就在于它让“当前到底发生到了哪一步”这件事有了统一依据。

第三，很多原本容易被忽略的状态，也终于能被正式纳管。比如 `viewed_images` 这种字段，很多系统会觉得“临时记一下就行”。但一旦模型会处理图片、会跨轮继续任务，这类状态如果不正式纳入 ThreadState，就很容易反复读取、反复分析、反复浪费 token。DeerFlow 的做法其实是在提醒你：真正长期跑的 Agent 后端，很多细小状态都值得被正式建模。

这里你也能重新理解第 1 课里那个看似普通的配置：

```python
create_agent(
    ...,
    state_schema=ThreadState,
)
```

很多人第一次看，只会把它当成“给 agent 指定一个 state 类型”。但现在你应该能看出它的分量了。DeerFlow 不是运行到一半才想起来“好像需要一些状态”，而是一开始就把线程级状态结构声明进 runtime。这个决定非常关键，因为它意味着后面所有模块都不需要各自发明一套状态模型。

如果没有这一层统一状态，系统最先坏掉的地方通常不是功能本身，而是协作关系。你会看到这些熟悉的问题：

- **工具层彼此不共享上下文**
- **路径信息在多个模块间重复推导**
- **产物列表没有统一归口，谁都能改，谁都说不清**
- **长任务越跑越像一堆“顺手传一下”的临时补丁**

从工程角度看，ThreadState 真正解决的，不是“怎么多存一点数据”，而是“怎么让运行时信息有统一归属”。这也是为什么它特别像“状态背板”这个词。背板的意思不是它最显眼，而是所有重要模块最后都得接到它上面。

你还可以把 DeerFlow 里的状态分成两层来看：

- **对话层状态**：模型说过什么、工具回了什么
- **线程层状态**：当前环境、路径、产物、计划、已处理资源

很多 Agent demo 只有前一层，所以看起来“会聊”；DeerFlow 补上了后一层，所以它开始像一个真正能执行任务的后端。

如果把这一课压成一句话，我会这样说：

**ThreadState 的作用，不是替消息历史做备份，而是给整个 Agent runtime 提供统一的线程级执行现场。**

只有这层现场稳定了，middleware 才能配合，tools 才能共享上下文，sandbox 才能绑定线程，artifacts 才能持续累积。否则系统看起来是一个 Agent，实际上只是很多会说话的零件硬拼在一起。

**这一课最后你最该记住的点**

- **复杂 Agent 不能只靠 `messages`，线程级状态必须显式化**
- **ThreadState 是 DeerFlow 各模块共享的状态背板**
- **`Annotated[...]` 的关键价值，是把更新规则和字段定义绑在一起**
- **统一状态不是锦上添花，而是 runtime 能长期稳定协作的前提**

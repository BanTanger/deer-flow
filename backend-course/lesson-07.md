# 第 7 课：Sandbox 为什么不是可选项，而是底线

很多 AI 初学者第一次给 Agent 接 bash 或文件写入时，会把关注点放在“太酷了，它终于能动起来了”。但从后端视角看，这恰恰是风险开始变大的地方。只要模型能执行命令、改文件、处理上传内容，你就必须开始考虑隔离、清理、生命周期和会话边界。不然系统不是“不够优雅”，而是危险。

DeerFlow 的思路很明确：执行环境不能靠模型自己去猜，也不能默认让模型直接接触宿主环境。它用 `SandboxMiddleware` 在运行时先把当前线程绑定到一个 sandbox：

```python
if "sandbox" not in state or state["sandbox"] is None:
    thread_id = runtime.context["thread_id"]       # 取当前线程 ID
    sandbox_id = self._acquire_sandbox(thread_id)  # 为这个线程申请 sandbox
    return {"sandbox": {"sandbox_id": sandbox_id}} # 再写回状态
```

这段代码的关键不是“多了一层中间件”，而是它明确了 sandbox 的归属：sandbox 是 thread 级资源，不是全局共享资源。这个设计非常重要，因为一旦多个线程共享同一份执行环境，文件污染、命令影响串线、资源回收混乱就都会出现。

DeerFlow 还把工作目录、上传目录、输出目录分开，并通过 `ThreadDataMiddleware` 先把这些路径放进状态里。这样 sandbox 不是孤立存在的，它和 thread 级路径一起构成了完整的执行环境。你可以把它理解成：模型还没开始行动，舞台就已经先搭好了。

**为什么 sandbox 不是可选项**

- **工具一旦能执行命令，就涉及安全边界**
- **文件写入一旦打开，就涉及线程隔离**
- **没有 sandbox，Agent 会越来越依赖宿主环境细节**
- **没有清晰生命周期，资源迟早会泄漏或污染**

DeerFlow 的 sandbox 思路本质上很朴素：不要让 Agent 在宿主机上“裸跑”，而是让它在一个可管理的、线程级别的受控环境里工作。这不是锦上添花，而是底线。

**这一课最后你最该记住的点**

- **执行能力越强，隔离需求越高**
- **sandbox 的关键不是“能跑”，而是“受控地跑”**
- **线程级目录和 sandbox 是一整套环境，不是两个孤立设计**
- **没有 sandbox 的 Agent 后端，越往后越难收场**

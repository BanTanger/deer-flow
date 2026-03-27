# 第 15 课：Guardrails、错误处理与安全边界

系统能力一强，安全问题就一定会跟着来。尤其是 Agent 这种会调工具、会执行命令、会读写文件、还可能调用子 Agent 的后端，它不是“以后再考虑安全”，而是从第一天就得考虑安全。DeerFlow 在这件事上的思路很明确：不要把“安全”理解成一句大口号，而要把它拆成 guardrails、错误处理、循环检测、clarification 中断这些具体运行时机制。

先看 guardrail middleware：

```python
decision = self.provider.evaluate(gr)
if not decision.allow:
    return ToolMessage(
        content=f"Guardrail denied: tool '{tool_name}' was blocked ...",
        tool_call_id=tool_call_id,
        name=tool_name,
        status="error",
    )
```

这段代码最值得你注意的地方，是“被拒绝的工具调用并不会变成后端崩溃”，而是会变成一条错误 ToolMessage 送回 agent。这个思路和 DeerFlow 处理普通工具错误是一致的：系统不只是简单拦掉，还会尽量把这次失败变成 agent 后续可利用的上下文。

DeerFlow 的 guardrail 还支持 `fail_closed`。这意味着如果 guardrail provider 自己出错，系统可以选择：

- **fail closed**：宁可先拦掉，也不冒险放过
- **fail open**：provider 掉了就先放过请求

这其实是很典型的安全系统设计问题：当安全子系统失效时，主系统应该更保守还是更宽松。DeerFlow 没有替所有人做唯一选择，而是把这个策略显式暴露出来。

再往外看，你会发现 DeerFlow 的安全边界不是只靠 guardrails。一整条链都在做同一种事情：

- **ClarificationMiddleware**：不清楚就先停
- **LoopDetectionMiddleware**：避免模型在错误路径上反复打转
- **ToolErrorHandlingMiddleware**：错误不炸链，转成上下文
- **SandboxMiddleware**：把执行环境限制在受控边界内
- **SubagentLimitMiddleware**：防止子任务扩散失控

**安全边界最容易踩的坑**

- **把安全理解成“最后加个鉴权”**
- **只有规则，没有运行时拦截**
- **安全系统出错时没有策略**
- **错误直接崩链，模型无法自我修正**

DeerFlow 给你的最大启发，是安全不应该只是系统外面的一堵墙，而应该是渗透在运行时里的多层边界。真正能长期跑的 Agent 后端，不是“永远不出错”，而是“出错时也不会轻易失控”。

**这一课最后你最该记住的点**

- **安全边界不是单个模块，而是一整套运行时机制**
- **guardrails 负责工具调用前的策略判断**
- **fail_closed / fail_open 是真实的系统策略选择**
- **错误处理、循环检测、澄清中断、sandbox，本质上都在做安全治理**

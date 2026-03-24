# 安全策略

## 支持版本

由于 DeerFlow 尚未提供正式版本，请使用最新版本以获取安全更新。
目前我们维护两个分支：
* main 分支用于 deer-flow 2.x
* main-1.x 分支用于 deer-flow 1.x

## 报告漏洞

请前往 https://github.com/bytedance/deer-flow/security 报告您发现的漏洞。

---

## AI 系统安全最佳实践

### 安全沙箱执行

DeerFlow 使用沙箱化环境来安全地执行 AI 代理的代码。以下是关键的安全措施：

1. **容器隔离**：每个任务在隔离的 Docker 容器中运行
2. **资源限制**：CPU、内存和磁盘使用受到限制
3. **网络限制**：控制容器的网络访问
4. **文件系统隔离**：防止对主机系统的未授权访问

> **AI 知识点 - 沙箱安全**：
> 沙箱化对于 AI 代理安全至关重要，因为：
> 1. **防止代码注入攻击**：隔离环境阻止恶意代码影响主机
> 2. **资源配额**：防止代理耗尽系统资源
> 3. **审计跟踪**：所有操作都记录并可审查
> 4. **临时性**：沙箱可以轻松销毁和重建
>
> 参考论文：[Sandboxing AI Agents](https://www.anthropic.com/research)

### API 密钥安全

1. **永远不要提交 API 密钥**：将密钥存储在 `.env` 文件中（在 .gitignore 中）
2. **使用环境变量**：通过环境变量传递密钥，而不是硬编码
3. **定期轮换密钥**：定期更新 API 密钥以最小化泄露的影响
4. **使用密钥管理服务**：对于生产环境，考虑使用 AWS Secrets Manager 或类似的

### 负责任的 AI 对齐

DeerFlow 支持 Constitutional AI 原则，确保代理输出安全、有用且诚实：

- **内容过滤**：实现安全检查以过滤有害内容
- **透明度**：清楚地标记 AI 生成的内容
- **用户控制**：允许用户控制代理行为和限制
- **可审计性**：维护代理操作的日志

> **AI 知识点 - Constitutional AI**：
> Constitutional AI 是一种通过自我改进训练和宪法约束使 AI 系统更加安全、有用和诚实的方法。关键原则包括：
> 1. **宪法**：系统应遵循的明确原则和规则
> 2. **自我批判**：AI 审查自己的输出是否符合宪法
> 3. **迭代改进**：基于反馈循环持续改进
> 4. **红队测试**：主动测试系统漏洞
>
> 参考论文：[Constitutional AI](https://www.anthropic.com/research/constitutional-ai-harmlessness-ai-feedback)

### 数据隐私

1. **本地存储**：DeerFlow 将数据存储在本地，让用户控制其数据
2. **加密**：在传输中和静止时敏感数据应该加密
3. **最小数据收集**：只收集必要的功能数据
4. **用户同意**：明确征求用户同意用于数据使用

### 审计和日志记录

- **操作日志**：记录所有代理操作和工具使用
- **错误跟踪**：实现全面的错误记录和监控
- **访问日志**：跟踪谁访问了什么内容以及何时访问
- **定期审查**：定期审查日志以发现异常

---

## 威胁模型和缓解措施

| 威胁 | 描述 | 缓解措施 |
|-------|------|-----------|
| 代码注入 | 恶意代码通过提示注入 | 沙箱隔离、输入验证 |
| 资源耗尽 | 代理消耗过多系统资源 | 资源配额、超时 |
| 数据泄露 | 敏感数据暴露给错误方 | 加密、访问控制 |
| 提示注入 | 恶意提示操纵代理行为 | 提示过滤、速率限制 |
| 越狱 | 代理绕过安全限制 | 多层安全检查 |

---

## 报告安全事件

如果您发现安全漏洞，请：

1. **不要公开披露**：私下报告以给时间修复
2. **提供详细信息**：包括复现步骤、影响和建议的修复
3. **给时间响应**：在披露之前给维护者合理的时间
4. **协调披露**：与团队协调负责任的披露时间表

联系方式：https://github.com/bytedance/deer-flow/security

---

## 安全资源

- [OWASP AI Security Guidelines](https://owasp.org/www-project-ai-security/)
- [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework)
- [Anthropic AI Safety Research](https://www.anthropic.com/research#safety)
- [OpenAI Safety Best Practices](https://openai.com/safety-best-practices)

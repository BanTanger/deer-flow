# 第 14 课：模型工厂层为什么不能只写一个 if/else

很多后端在模型接入上，早期都会写成一串 `if provider == ...`。短期当然能跑，但模型一多、能力差异一大、provider 行为一复杂，这种写法很快就会变得又硬又脆。DeerFlow 之所以单独做了模型工厂层，正是因为它不把“创建模型对象”看成一个小动作，而看成一个需要统一抽象的能力层。

核心函数是这个：

```python
def create_chat_model(name: str | None = None, thinking_enabled: bool = False, **kwargs) -> BaseChatModel:
    config = get_app_config()
    if name is None:
        name = config.models[0].name
    model_config = config.get_model_config(name)
    model_class = resolve_class(model_config.use, BaseChatModel)
    ...
    model_instance = model_class(**kwargs, **model_settings_from_config)
```

这里最值得你注意的是 `resolve_class(model_config.use, BaseChatModel)`。它不是在 if/else 里手工判断“这是 OpenAI 还是 Anthropic”，而是把 provider 类路径放进配置，再由工厂层统一解析。这种做法的好处非常明显：模型接入不再靠写死逻辑，而是靠配置 + 反射扩展。

DeerFlow 在这层做的事情也比“new 一个模型对象”多得多。它还会：

- 根据 `thinking_enabled` 和配置决定是否注入 thinking 参数
- 对不支持 reasoning_effort 的模型剔除无效参数
- 对 Codex/Responses 这种特殊 provider 做兼容处理
- 在 tracing 打开时自动挂 LangSmith tracer

**模型工厂层最容易踩的坑**

- **provider 分支全写死在代码里**
- **不同模型能力不做统一收口**
- **thinking / vision / reasoning_effort 参数乱传**
- **tracing、鉴权、provider 特殊兼容散落在各处**

DeerFlow 的模型工厂层给你的启发是：模型接入不是一条 if/else 分支，而是一层适配器架构。你不是在接“一个 API”，而是在接“多家 provider、多种能力、多种参数语义”。

**这一课最后你最该记住的点**

- **模型工厂层解决的是 provider 差异收口问题**
- **`resolve_class(...)` 让模型接入可配置，而不是写死**
- **thinking、vision、reasoning_effort 这些能力应该统一治理**
- **模型抽象层做得好，整个系统才不容易被 provider 差异拖散**

# 第 14 课：模型工厂层为什么不能只写一个 if/else

很多 AI 后端刚起步时，模型接入层都会长得差不多：一个 `if provider == "openai"`，再接一个 `elif provider == "anthropic"`，能跑就先跑。前期当然没问题，甚至很高效。但只要模型一多、provider 一杂、能力差异一拉开，这种写法很快就会变得又硬又脆。

因为你接入的从来不只是“不同接口地址”，而是不同 provider 的一整套差异：

- 参数名字不一样
- thinking / reasoning 支持情况不一样
- vision 能力开关不一样
- tracing 挂接方式不一样
- 鉴权和兼容逻辑不一样

如果这些差异都直接堆进业务代码或者一串分支判断里，后面整个系统都会逐渐被 provider 差异拖散。DeerFlow 之所以专门做模型工厂层，正是因为它没有把“创建模型对象”看成一个小动作，而是看成一个需要统一收口的能力层。

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

这段代码里最关键的一行，是：

```python
model_class = resolve_class(model_config.use, BaseChatModel)
```

它意味着 DeerFlow 并不是在 if/else 分支里手工判断“这次是哪个 provider”，而是把 provider 对应的类路径放进配置，再由工厂层统一解析和实例化。这个设计判断特别重要，因为它把“新增模型接入”这件事从“改核心逻辑”变成了“补配置 + 补适配能力”。

也就是说，DeerFlow 在这层追求的不是“把所有 provider 差异消灭”，而是：

**把 provider 差异收口到同一个工厂层处理。**

这两者差别很大。前者几乎做不到，因为各家模型能力本来就不同；后者是现实可行的，因为你至少可以保证：系统其他地方不必反复感知这些差异。

真正成熟的模型工厂层，也绝不只是“new 一个实例”那么简单。DeerFlow 在这一层还会继续处理很多能力相关的细节，比如：

- 根据 `thinking_enabled` 和模型配置决定是否注入 thinking 参数
- 对不支持 `reasoning_effort` 的模型剔除无效参数
- 对特殊 provider 或特殊 API 语义做兼容处理
- 在 tracing 开启时自动挂接 LangSmith tracer

这些动作单独看都不算巨大，可如果不集中处理，它们就会像藤蔓一样长满整个代码库。今天工具层自己判断一次模型能力，明天 subagent 再判断一次，后天 Gateway 或别的工厂又补一套兼容逻辑。最后不是系统支持了很多 provider，而是系统被 provider 差异切成了很多碎片。

所以 DeerFlow 的模型工厂层，真正解决的不是“如何快速接一个模型”，而是：

**如何让整个系统在接越来越多模型时，仍然只在一个地方面对 provider 差异。**

这个判断对长期维护非常重要。因为 AI 系统最大的外部不确定性，往往不在你自己写的代码里，而在 provider 世界变化太快。新模型会来，旧模型会变，参数语义会更新，能力边界会调整。如果你的代码库里到处散落着 provider 判断，后面每次变化都会牵一发动全身。

反过来，如果工厂层足够清晰，你至少能做到两件事。第一，系统其他模块只和统一抽象打交道，比如 `BaseChatModel`，而不是和某家 provider 的私有细节打交道。第二，新增 provider 或修正兼容逻辑时，改动范围会集中得多。

这也是为什么“模型工厂层为什么不能只写一个 if/else”这个问题，不只是代码洁癖，而是系统边界问题。if/else 本身没错，错的是它会把 provider 差异直接暴露给整个后端。只要差异一多，这条边界就会很快崩掉。

你可以把这一层最容易踩的坑总结成四种：

- **provider 分支写死在业务逻辑里**
- **模型能力差异没有统一收口**
- **thinking / vision / reasoning 参数到处乱传**
- **tracing、兼容逻辑、鉴权逻辑散落在各处**

DeerFlow 的处理方式很明确：把模型接入抽成工厂层，把 provider 类路径放进配置，把能力差异和兼容逻辑尽量集中在一层里治理。这样后面的 prompt、tool binding、subagent、middleware 才不必每次都重新理解一遍“这家模型又有什么特殊脾气”。

如果把这一课压成一句话，我会这样说：

**模型工厂层真正的价值，不是把模型创建出来，而是把 provider 差异挡在系统边界的一处，而不是放任它们渗进整个代码库。**

DeerFlow 值得学，正是因为它不是在“接一个模型 API”，而是在建设一层可以长期承受模型生态变化的适配架构。

**这一课最后你最该记住的点**

- **模型工厂层解决的是 provider 差异的统一收口**
- **`resolve_class(...)` 让模型接入变成配置化扩展，而不是硬编码分支**
- **thinking、vision、reasoning 等能力差异应该集中治理**
- **工厂层越清楚，整个系统越不容易被 provider 变化拖散**

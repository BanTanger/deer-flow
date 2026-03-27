# 第 6 课：MCP 与 Tool Search 为什么能让后端可扩展

工具系统解决的是“模型怎么做动作”，但当系统开始接越来越多外部能力时，很快又会撞上另一个坑：如果每接一个外部系统都手写一套私有集成，后端会越来越碎，配置会越来越乱，工具初始化也会越来越慢。DeerFlow 处理这个问题的方式，是把 MCP 和延迟加载一起纳入工具系统。

先看 MCP 缓存层的核心逻辑：

```python
_mcp_tools_cache: list[BaseTool] | None = None
_cache_initialized = False
_config_mtime: float | None = None

def get_cached_mcp_tools() -> list[BaseTool]:
    if _is_cache_stale():
        reset_mcp_tools_cache()  # 配置变了，先失效缓存

    if not _cache_initialized:
        ...  # 懒初始化 MCP 工具

    return _mcp_tools_cache or []
```

这个设计背后解决的是两个非常现实的问题。第一，MCP 工具初始化通常不轻，不能每次请求都重新加载。第二，Gateway 和 LangGraph 是两个独立进程，MCP 配置可能在一个进程里变了，另一个进程如果一直吃旧缓存，就会出现“明明配置改了，工具列表却没变”的错觉。DeerFlow 用 `mtime` 去判断配置文件有没有变化，本质上是在做一个“够简单、够实用”的缓存失效策略。

DeerFlow 还做了一件非常有意思的事，就是当 `tool_search` 打开时，并不会把所有 MCP 工具 schema 全部直接塞给模型，而是先把它们注册进延迟工具表，需要时再由模型显式搜索和加载。这样做的直接收益是上下文更轻，模型也不会在一堆没必要的工具之间乱选。

**MCP / Tool Search 这一层在解决什么坑**

- **工具越来越多，初始化越来越重**
- **配置改了，但进程内缓存没刷新**
- **把所有外部工具 schema 都塞进 prompt 或 binding，成本太高**

MCP 在 DeerFlow 里不是一个独立的小功能，而是一种扩展哲学：用标准协议把“后端能力怎么接进 Agent”这件事做得更统一。Tool Search 则是在这个基础上继续优化：能力可以很多，但不要一次全给模型。

**这一课最后你最该记住的点**

- **MCP 解决的是后端扩展协议问题**
- **缓存解决的是初始化成本问题**
- **Tool Search 解决的是上下文过重和工具过多的问题**
- **DeerFlow 追求的不是“能力全塞进去”，而是“能力按需暴露”**

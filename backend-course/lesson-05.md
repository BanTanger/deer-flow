# 第 5 课：工具系统是怎么把“会说”变成“能做”的

很多人第一次做 Agent，都会高估 prompt，低估工具层。他们以为只要 prompt 写得足够好，模型自然就会“知道应该怎么做”。但模型真正能做的，其实只有输出文本。它可以说“我建议读取这个文件”“我建议调用搜索工具”“我建议执行这个命令”，可如果系统没有把这些动作真正接成工具，模型就只是把一段意图说出来而已。

DeerFlow 的工具系统，解决的正是这个落差：

```python
def get_available_tools(
    groups: list[str] | None = None,   # 按工具组筛选
    include_mcp: bool = True,          # 是否包含 MCP 工具
    model_name: str | None = None,     # 当前模型，用来判断能力
    subagent_enabled: bool = False,    # 是否加入子 Agent 工具
) -> list[BaseTool]:
```

这里的 `BaseTool` 来自 LangChain 工具体系。你可以把它理解成所有“能被 agent 调用的工具对象”的共同基类。DeerFlow 在这里做的，不是返回一堆配置，而是返回真正可执行的工具实例。

它的核心装配逻辑是这样的：

```python
loaded_tools = [
    resolve_variable(tool.use, BaseTool)  # 把配置里的工具路径解析成真正对象
    for tool in config.tools
    if groups is None or tool.group in groups
]

builtin_tools = BUILTIN_TOOLS.copy()      # DeerFlow 自带工具

if subagent_enabled:
    builtin_tools.extend(SUBAGENT_TOOLS)  # 需要时加入 task 工具

if model_config is not None and model_config.supports_vision:
    builtin_tools.append(view_image_tool) # 视觉模型才给图片工具
```

最关键的坑，不是“工具能不能注册成功”，而是“工具暴露得对不对”。很多系统一开始会把所有工具都暴露给模型，看起来像是增强了能力，实际上往往是在制造混乱。工具越多，模型每轮的选择空间越大，上下文也越重，误调工具的概率也越高。DeerFlow 的做法，是让工具集合跟当前运行模式绑定。只有支持视觉的模型才会拿到 `view_image_tool`，只有允许 subagent 的模式才会拿到 `task_tool`。

这里还有一个特别值得你记住的点：`resolve_variable(...)`。它的作用不是单纯导入模块，而是把配置里的字符串路径解析成真正的 Python 对象。这意味着 DeerFlow 的工具不是全写死在代码里的，而是支持配置化扩展。这也是为什么后面接 MCP、接社区工具、接自定义工具时，系统不会一下子失控。

**工具系统最容易踩的坑**

- **把所有工具都暴露给模型**
- **只注册工具，不处理工具错误**
- **工具层没有和模型能力绑定，比如视觉模型和非视觉模型混用**
- **工具靠硬编码，不靠配置和反射扩展**

**这一课最后你最该记住的点**

- **工具层是模型从“会说”变成“能做”的桥**
- **工具不是越多越好，关键是当前模式该给哪些**
- **DeerFlow 的工具集合是动态装配的，不是写死的**
- **可扩展的工具系统，离不开配置和反射层**

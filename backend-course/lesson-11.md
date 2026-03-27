# 第 11 课：Gateway API 在后端架构里扮演什么角色

很多人第一次看到 DeerFlow 的后端，会误以为“真正重要的是 LangGraph，Gateway 只是附带的 API 壳子”。但如果你从系统工程角度看，Gateway 扮演的是非常关键的角色：它不是直接跑 Agent 主循环，而是给整个后端能力提供一个稳定、清晰、可管理的访问面。

DeerFlow 的 Gateway 启动逻辑并不复杂：

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    get_app_config()  # 启动时先加载配置
    ...
    yield
```

再看应用本体：

```python
app = FastAPI(...)
app.include_router(models.router)
app.include_router(mcp.router)
app.include_router(memory.router)
app.include_router(skills.router)
app.include_router(artifacts.router)
app.include_router(uploads.router)
app.include_router(threads.router)
app.include_router(agents.router)
```

这里的重点不在 FastAPI 语法，而在架构边界。Gateway 把模型管理、MCP 配置、memory 状态、skills 开关、uploads、artifacts、thread 清理这些能力集中成一组 REST 接口，而 Agent 主循环仍然由 LangGraph 负责。这种拆法的好处是很明显的：Agent 运行时和管理接口解耦，系统边界更清楚。

还有一个特别值得你记住的细节：Gateway 并不主动初始化 MCP 工具，因为它不是使用 Agent 工具的那个进程。MCP 工具真正懒加载的地方在 LangGraph 那边。这一点看起来细，但它体现了 DeerFlow 对“进程边界”和“缓存归属”的清晰认识。很多系统一乱，往往不是因为核心算法，而是因为“谁负责什么”没有边界。

**Gateway 这一层最容易被误解的地方**

- **它不是 Agent 主循环本身**
- **它更像后端能力管理面**
- **它负责稳定接口，不负责替 Agent 决策**

**这一课最后你最该记住的点**

- **Gateway 是 DeerFlow 后端的管理入口，不是主推理入口**
- **LangGraph 跑 Agent，Gateway 提供管理和配套接口**
- **进程边界清楚，缓存和初始化责任才不会乱**
- **管理面和执行面分离，是后端可维护性的关键**

# 精读 11：执行环境与服务端（workspace 3067 行 + agent-server 29851 行）

> 位置：`upstream/software-agent-sdk/`
> 文件：`openhands-sdk/openhands/sdk/workspace/{base,workspace,local,remote}.py`、
> `openhands-workspace/openhands/workspace/{docker,apptainer,cloud,agent_sandbox,remote_api}/`、
> `openhands-agent-server/openhands/agent_server/api.py`
> 前置：`notes/11-tool-layer.md`（工具在哪执行）、`notes/05-run-loop.md`（本地会话）

## 这一节在讲什么

前面所有内容都假设"工具执行"这件事会发生。这一节讲**它在哪儿发生**，
以及**怎么把整个 SDK 变成一个能远程调用的服务**。

两个问题：

1. AI 执行命令的地方——你的笔记本、一个 Docker 容器、一台云主机——怎么抽象成同一个东西
2. 29851 行的服务端在解决什么（这个体积本身说明了答案：**不是"包一层 HTTP" 那么简单**）

## 一、工作区的接口：只有五个必须实现的方法

```python
class BaseWorkspace(DiscriminatedUnionMixin, ABC):
    """Workspaces provide a sandboxed environment where agents can execute commands,
    read/write files, and perform other operations."""

    working_dir: str

    @abstractmethod
    def execute_command(self, command, cwd=None, timeout=30.0) -> CommandResult: ...
    @abstractmethod
    def file_upload(self, source_path, destination_path) -> FileOperationResult: ...
    @abstractmethod
    def file_download(self, source_path, destination_path) -> FileOperationResult: ...
    @abstractmethod
    def git_changes(self, path) -> list[GitChange]: ...
    @abstractmethod
    def git_diff(self, path) -> GitDiff: ...
```

**接口小得惊人。** 执行命令、传文件进出、看 git 改动——五个方法就撑起了本地、Docker、
Apptainer、云端、远程 API 五种实现。

> 可迁移的道理：**抽象层的价值和它的接口大小成反比。** 接口越小，实现新后端越容易，
> 各实现之间的行为差异也越少。如果这里有 30 个方法，每加一种后端都是一场噩梦。

注意 `git_changes` / `git_diff` 被提到了这一层——**因为它们必须在代码所在的那一侧执行**。
代码在容器里，本地跑 `git diff` 是没用的。

### 两个非抽象方法：暂停和恢复

```python
def pause(self) -> None:
    """For local workspaces, this is a no-op.
    For container-based workspaces, this pauses the container."""
    raise NotImplementedError(f"{type(self).__name__} does not support pause()")
```

**默认实现是抛异常，而不是静默什么都不做。**

这个选择很讲究：如果默认是空操作，你调用 `pause()` 以为省了钱，实际上容器还在烧钱，
**而且没有任何提示**。抛异常强迫你知道这个后端不支持。

> **可迁移的道理**：**"可选功能"的默认实现应该是明确失败，而不是静默无效。**
> 静默无效会让调用方建立错误的心智模型。

### 上下文管理器：退出时才是重点

```python
def __enter__(self) -> "BaseWorkspace":
    return self

def __exit__(self, exc_type, exc_val, exc_tb) -> None:
    """Default implementation performs no cleanup. Subclasses should override
    to add cleanup logic (e.g., stopping containers, closing connections)."""
```

> **上下文管理器（context manager）**：Python 的 `with` 语句机制，保证"进入"和"退出"
> 成对发生，即使中间抛异常也会执行退出逻辑。

用法：

```python
with workspace:
    result = workspace.execute_command("echo 'hello'")
```

**为什么工作区必须是上下文管理器？** 因为 Docker 容器、SSH 连接、云主机都是
**必须被显式释放的资源**。忘了释放 = 容器一直跑 = 一直计费。`with` 保证了释放。

## 二、一个容易忽略但很实在的设计：完成回调

`BaseWorkspace` 里有个方法专门处理"跑完了要通知谁"：

```python
def _send_completion_callback(self, exc_type, exc_val) -> None:
    """POST completion status to the automation service (best-effort).

    Call this from ``__exit__`` before any cleanup. Does nothing when
    ``AUTOMATION_CALLBACK_URL`` env var is not set."""

    callback_url = os.environ.get("AUTOMATION_CALLBACK_URL")
    if not callback_url:
        return
    ...
    status = "COMPLETED" if exc_type is None else "FAILED"
```

**场景**：自动化系统（定时任务、webhook 触发）派了一个 agent 去干活，
干完要知道成功还是失败。

三个细节值得看：

**① 时机是"清理之前"。** 注释明确要求 `Call this from __exit__ before any cleanup`。
因为清理可能失败、可能很慢，而**通知结果这件事更重要、更该先做**。

**② 错误信息优先用结构化的那一个：**

```python
error = (exc_val.conversation_error if isinstance(exc_val, ConversationRunError) else None)
if error is None:
    cause = (exc_val.original_exception if isinstance(exc_val, ConversationRunError) else exc_val)
    error = ConversationErrorEvent(source="environment", code=type(cause).__name__, detail=str(cause))
```

回想第四节：外层循环抛出的 `ConversationRunError` 里**带着一个结构化的错误事件**。
这里优先用它，没有才降级成泛化的版本。

> **又一次看到同样的原则**：不要用泛化的错误覆盖精确的诊断。这已经是第三次
> 在不同模块看到了（第四节的兜底处理、第五节的异常链、这里）。

**③ 失败只打日志，不影响主流程：**

```python
except Exception as e:
    logger.warning(f"Completion callback failed: {e}")
```

注释里的 `best-effort`（尽力而为）就是这个意思。**通知别人失败了，不该让自己的任务变成失败。**

还有一个跨模块的配合：

```python
def register_cost(self, cost: float) -> None:
    """Called by the conversation on close, once the conversation has finished, so no
    caller opt-in is required. The cost is included in the completion callback"""
```

会话结束时把花费登记到工作区，于是回调里能带上"这次花了多少钱"。

**注意 `no caller opt-in is required`（不需要调用方主动开启）**——这是个有意识的设计选择：
**成本上报默认就有，不用用户记得打开。**

> 回想第九节那个"派生的模型实例要挂在同一份用量统计上"——
> 那条链路的终点就在这里。**一个功能（成本可见）需要三个模块配合：
> LLM 层累加、会话层汇总、工作区层上报。**

## 三、工厂函数：用参数决定实现

```python
class Workspace:
    """Factory entrypoint that returns a LocalWorkspace or RemoteWorkspace.

    Usage:
        - Workspace(working_dir=...) -> LocalWorkspace
        - Workspace(working_dir=..., host="http://...") -> RemoteWorkspace
    """
    def __new__(cls, *, host=None, working_dir="workspace/project", api_key=None):
        if host:
            return RemoteWorkspace(working_dir=working_dir, host=host, api_key=api_key)
        return LocalWorkspace(working_dir=working_dir)
```

**给了 `host` 就是远程，没给就是本地。** 用户只需要记住一个名字 `Workspace`。

而且用 `@overload` 标注了两种签名，**于是类型检查器知道：传了 `host` 得到的是
`RemoteWorkspace`，没传得到的是 `LocalWorkspace`。**

> **这是"工厂 + 重载标注"的漂亮用法**：运行时是一个入口，静态类型上却能精确区分。
> 用户在 IDE 里能看到正确的方法提示。

## 四、Docker 工作区：踩过的坑都写在字段里

```python
class DockerWorkspace(RemoteWorkspace):
    """This workspace creates a Docker container running a pre-built OpenHands agent
    server image, waits for it to become healthy, and then provides remote workspace
    operations through the container's HTTP API."""
```

**注意它继承 `RemoteWorkspace`。** 逻辑是：容器里跑的是一个**完整的 agent server**，
本地通过 HTTP 和它通信。所以"Docker 工作区"本质上是"远程工作区 + 自动启容器"。

> 这个设计的好处：**本地容器和云主机走完全相同的代码路径。**
> 只是前者多了"启动和销毁容器"的生命周期管理。

### 端口选择用了系统随机数

```python
def find_available_tcp_port(min_port=30000, max_port=39999, max_attempts=50) -> int:
    rng = random.SystemRandom()
    ports = list(range(min_port, max_port + 1))
    rng.shuffle(ports)
    for port in ports[:max_attempts]:
        if check_port_available(port):
            return port
    return -1
```

**不是从 30000 往上顺序试，而是把整个范围打乱再取前 50 个。**

为什么？顺序试的话，**同时启动多个容器时大家都会先试 30000**，然后撞车、重试、
再一起试 30001……随机化让它们一次就分散开。

> **可迁移的道理**：**并发场景下的资源分配，随机比顺序好。** 顺序扫描会让所有
> 竞争者走同一条路径，制造不必要的冲突。

还有个细节在 `check_port_available` 里：

```python
except OSError:
    time.sleep(0.1)
    return False
```

**端口不可用时先睡 0.1 秒再返回。** 这是一个粗糙但有效的退避——防止在端口紧张时
疯狂空转。

### 一个"配置项被删掉了"的处理方式

```python
@model_validator(mode="before")
def _reject_removed_mount_dir(cls, data):
    if isinstance(data, dict) and "mount_dir" in data:
        raise ValueError(
            "DockerWorkspace.mount_dir has been removed; use "
            "volumes=['/host/dir:/workspace'] instead.")
```

`mount_dir` 这个参数被删了。**但代码里保留了一个专门的校验器，用来在有人还传它的时候
报一个"这个参数已删除，请改用 volumes"的错误。**

> **这是对使用者非常友好的做法。** 直接删掉的话，用户会得到一个"未知参数"的
> 泛泛错误（或者更糟——Pydantic 默认会静默忽略额外字段），完全不知道该改成什么。
>
> 可迁移的道理：**删掉一个公开参数时，留一个报错指路的墓碑。**

### 哪些环境变量会被转发进容器

```python
forward_env: list[str] = Field(
    default_factory=lambda: ["DEBUG", "SESSION_API_KEY", "OH_SESSION_API_KEYS_0"],
    description=("Environment variables to forward to the container. The session "
                 "API key variables are forwarded so the sandboxed agent server can "
                 "authenticate network-bound requests when it binds 0.0.0.0."))
```

**只转发白名单里的变量**，而不是把整个环境倒进去。

注释解释了为什么必须转发那两个 key：容器里的服务绑的是 `0.0.0.0`
（所有网络接口），**所以它必须能验证请求来源**，否则同一网络里的任何人都能调它。

> **默认绑 `0.0.0.0` 的服务必须有鉴权**，这是个基本安全要求。
> 而鉴权需要密钥，密钥需要传进去——这条链路被写在注释里了。

## 五、服务端：29851 行在干什么

`api.py` 的 `create_app` 是入口。看它挂了多少路由，就明白这个体积从哪来：

```python
api_router.include_router(file_discovery_router)
api_router.include_router(conversation_catalog_router)
api_router.include_router(conversation_router)
api_router.include_router(credential_binding_router)
api_router.include_router(tool_router)
api_router.include_router(bash_router)
api_router.include_router(git_router)
api_router.include_router(file_router)
api_router.include_router(vscode_router)          # ← 容器里跑一个 VSCode
api_router.include_router(skills_router)
api_router.include_router(sub_agents_router)
api_router.include_router(plugins_router)
api_router.include_router(canvas_extensions_router)
api_router.include_router(hooks_router)
api_router.include_router(llm_router)
api_router.include_router(provider_connections_router)
api_router.include_router(mcp_router)
api_router.include_router(settings_router)
api_router.include_router(workspaces_router)
api_router.include_router(profiles_router)
...
app.include_router(openai_router, ...)            # ← 兼容 OpenAI 接口
app.include_router(conversation_registry.sockets_router)   # ← WebSocket
```

**二十多个路由模块。** 前面十节讲的每一个子系统，在这里都有对应的管理接口。

还有一个 `openai_router`——**提供 OpenAI 兼容的接口**，于是任何支持 OpenAI API 的
客户端都能直接连它。

### 三个认证分组，而且理由写清了

这是整个文件设计最细的地方：

```python
# ① /api/init：绕过会话鉴权和休眠门，有自己的 X-Init-API-Key
init_api_router = APIRouter(prefix="/api")

# ② 大部分 /api/*：只认请求头
dependencies = [Depends(check_session_api_key), Depends(require_initialized)]
api_router = APIRouter(prefix="/api", dependencies=dependencies)

# ③ 工作区静态文件：请求头 或 Cookie 都行
workspace_api_router = APIRouter(prefix="/api",
                                 dependencies=[Depends(check_workspace_session)])
```

**为什么第二组"只认请求头，不认 Cookie"？**

```python
# Cookies are NOT honored here so that we don't expand the CSRF surface
# across the whole API.
```

> **CSRF（跨站请求伪造）**：恶意网站诱导你的浏览器向另一个站点发请求。
> 因为浏览器会**自动携带 Cookie**，那个请求看起来就像是你本人发的。
>
> 关键点：**浏览器不会自动携带自定义请求头。** 所以"只认请求头"这件事本身
> 就阻止了 CSRF——恶意网站没法让你的浏览器带上那个头。

**那为什么第三组又必须认 Cookie？**

```python
# The cookie is required so that <iframe src> / <img src> embeds of workspace
# artifacts work — browsers cannot attach custom headers to those requests.
```

**因为 `<img src="...">` 和 `<iframe src="...">` 这种标签没法加自定义请求头。**
要在界面里显示容器里的截图、预览文件，只能靠 Cookie。

> **这是一个教科书级的安全权衡记录**：
> - 默认用更安全的方式（请求头）
> - 只在**技术上做不到**的那一小块放开 Cookie
> - 并且把"为什么"写在代码旁边
>
> 可迁移的道理：**当你必须放宽一个安全限制时，把范围缩到最小，并写明原因。**
> 不要因为一个 `<img>` 标签的需求，就给整个 API 都开 Cookie。

### 延迟初始化：为"温池"设计

```python
# In deferred-init mode the conversation service is *not* entered here — that
# happens later, when POST /api/init delivers the runtime config. We still mark
# the /ready endpoint as ready so a warm-pool orchestrator can tell the pod has
# finished booting and is available to receive its /api/init payload.
```

> **温池（warm pool）**：预先启动一批空闲的服务实例，用户请求来了直接分配一个，
> 省掉冷启动时间。

**问题**：预启动时还不知道这个实例要给谁用、用什么模型、什么密钥。

**解法**：分两阶段。
1. 启动时只把架子搭起来，`/ready` 报告"我启动好了"
2. 真有用户了，`POST /api/init` 把配置送进来，这时才真正创建会话服务

而且未初始化时**所有 `/api/*` 都返回 503**：

```python
# Dormant gate: 503s every /api/* route until POST /api/init completes.
# No-op for non-deferred deployments.
Depends(require_initialized),
```

**注意 `No-op for non-deferred deployments`** —— 不用这个模式的部署完全不受影响。
新功能没有给老路径增加负担。

### 启动时清理上一轮的残留

```python
def _cleanup_stale_tmux_sessions() -> None:
    """Tmux sessions live in a separate process that survives agent-server restarts.
    This function kills all existing sessions on the shared OpenHands tmux socket
    to prevent accumulation of orphaned sessions."""
```

> **tmux**：终端会话管理工具，能让一个 shell 会话在后台一直活着。
> 终端工具用它来实现"命令之间保持状态"（比如 `cd` 之后下一条命令还在那个目录）。

**关键洞察**：tmux 会话活在**独立的进程**里，服务重启杀不掉它们。
不清理的话，每次重启都留下一批孤儿会话，越积越多。

而清理失败不阻止启动：

```python
except Exception as e:
    # Don't let tmux cleanup failures prevent server startup
    logger.warning("Failed to cleanup tmux sessions: %s", e)
```

> **可迁移的道理**：**跨进程的资源不会跟着你的进程一起死。** 启动时清理上一轮残留，
> 是所有会启动外部进程的服务都该做的事。而清理本身要容错——**不能因为打扫不干净就不开门。**

## 六、错误处理：三个很实用的做法

### ① 422 响应要脱敏

```python
def _sanitize_validation_errors(errors) -> list[dict]:
    """FastAPI's default 422 response includes the raw request ``input`` in each
    validation error dict. If the request contained secret-bearing fields
    (e.g. ``agent.llm.api_key``, MCP server ``env``), those values would be
    echoed back to the caller. This helper redacts them.
    """
    for error in errors:
        error = dict(error)      # shallow copy so we don't mutate the original
        if "input" in error:
            error["input"] = sanitize_dict(error["input"])
```

**FastAPI 默认会把出错的请求内容原样回显。** 如果请求里有 API key，
那么一个格式错误就会让密钥出现在响应里（进日志、进监控、进浏览器控制台）。

注释还挂了来源：`Refs: OpenHands/evaluation#385` —— **真实发现过。**

> 回想第十节那个"所有工具输出经过同一个出口脱敏"，和第三节那个"报错只带参数名不带值"。
> **这是同一个主题在第三个位置出现：任何流向外部的通道都要过一次脱敏。**
> 而且注意 `shallow copy so we don't mutate the original` —— 脱敏不改原始数据。

### ② 用关联 ID 把 500 和日志连起来

```python
error_id = uuid.uuid4().hex
# Correlation id that ties the 500 a caller receives to the server-side log line
# (with full traceback) for this failure, so an otherwise opaque 500 can be matched
# to its traceback in the server logs.

content = {"detail": "Internal Server Error", "exception": str(exc), "error_id": error_id}
```

**给客户端一个随机 ID，同时把这个 ID 写进服务端日志。**

于是用户报"我收到一个 500"，你问"error_id 是多少"，就能直接定位到那条带完整堆栈的日志。

**堆栈只在 DEBUG 模式下返回给客户端**：

```python
if DEBUG:
    content["traceback"] = traceback.format_exc()
```

> **可迁移的道理**：**不要为了可调试性而泄露内部细节，用一个关联 ID 替代。**
> 客户端拿到的是无意义的随机串，运维拿它能查到一切。

### ③ 区分"故意抛的 HTTP 错误"和"真崩了"

这段注释是整个文件里最有见地的：

```python
# Log 5xx errors at error level. HTTPException is intentionally raised flow
# control — the route picked this status and detail on purpose — so a stack
# trace adds no information beyond `exc.detail` and makes routine upstream
# blips look indistinguishable from a process crash. Unhandled exceptions
# still get a full traceback via _unhandled_exception_handler above.
```

翻译：`HTTPException` 是**故意抛出来当控制流用的**——路由代码自己选了这个状态码和说明。
所以打堆栈毫无额外信息，**而且会让例行的上游抖动看起来和进程崩溃一模一样。**

**真正没被处理的异常仍然有完整堆栈。**

> **这条洞察值得单独记**：**日志噪音的危害不是占空间，是让真正的问题失去辨识度。**
> 如果每个 503 都带一屏堆栈，那么真的崩溃时你根本注意不到。
>
> 判断标准很清晰：**这个堆栈能告诉我 `exc.detail` 之外的信息吗？** 不能就别打。

4xx 打 info 级别（"预期内的客户端错误，比如鉴权失败"），5xx 打 error 级别。
**日志级别按"是不是我的问题"来分，而不是按"严重不严重"。**

### ④ 遥测上报路由模板，不上报真实路径

```python
"""Sends the *route template* (``/api/conversations/{conversation_id}``) rather than
``request.url.path``, which embeds real identifiers."""

route = request.scope.get("route")
route_template = getattr(route, "path", None)
if not isinstance(route_template, str) or not route_template:
    # Unmatched route: reporting the raw path could leak identifiers.
    route_template = "/unmatched"
```

**上报 `/api/conversations/{conversation_id}` 而不是 `/api/conversations/abc-123`。**

两个理由：**① 不泄露真实 ID；② 能聚合统计**（否则每个请求都是一个独立的路径，
没法算"这个接口的错误率"）。

而且**匹配不上任何路由时，上报 `/unmatched` 而不是原始路径**——
因为那种情况下路径可能是攻击者构造的任意字符串。

整个遥测函数还包在 try/except 里：

```python
"""Fully defensive: an error in the telemetry path must not replace the 500 the
caller is already getting with a different failure."""
```

**遥测出错不能把用户正在收到的 500 换成另一个错误。**

> 这是所有"横切关注点"（日志、监控、埋点）都该守的规矩：
> **辅助设施的失败不能改变主流程的结果。**

## 七、整层的结构

```
    你的代码
       │
   Workspace(...)  ← 工厂：给 host 就远程，不给就本地
       │
  ┌────┴─────────────────────────────────┐
  │ LocalWorkspace   直接在本机执行       │
  ├──────────────────────────────────────┤
  │ RemoteWorkspace  HTTP 调远端 server   │
  │   ├─ DockerWorkspace   自己起容器     │
  │   ├─ ApptainerWorkspace（HPC 场景）   │
  │   ├─ AgentSandboxWorkspace            │
  │   ├─ OpenHandsCloudWorkspace          │
  │   └─ APIRemoteWorkspace               │
  └──────────────────────────────────────┘
       │ （五个抽象方法：执行命令 / 传文件进出 / git 改动 / git diff）
       ▼
  agent-server（FastAPI，29851 行）
   ├─ 20+ 路由模块，覆盖前十节每个子系统
   ├─ WebSocket（事件流实时推送）
   ├─ OpenAI 兼容接口
   ├─ 三个认证分组（header / header+cookie / init 专用）
   ├─ 延迟初始化（温池）
   └─ 容器里还跑着 VSCode
```

**注意 Apptainer** —— 那是高性能计算集群常用的容器方案（不需要 root 权限）。
说明他们在支持学术/HPC 场景。

## 八、这一节的可迁移结论

1. **抽象层的价值和接口大小成反比**：五个方法撑起五种后端。接口越小，
   加新后端越容易，实现间的行为差异越少。

2. **"可选功能"的默认实现应该明确失败，而不是静默无效**：`pause()` 默认抛异常，
   否则调用方会以为省了钱而容器还在烧。

3. **必须释放的资源要做成上下文管理器**：容器、连接、云主机忘了释放就是一直计费。

4. **通知外部要在清理之前做**，而且**失败只打日志**——通知别人失败了不该让自己变成失败。

5. **删掉公开参数时留一个报错指路的墓碑**：直接删会让用户得到一个不知所云的错误。

6. **并发场景的资源分配用随机而非顺序**：顺序扫描会让所有竞争者走同一条路径，
   制造不必要的冲突。

7. **默认绑 0.0.0.0 的服务必须有鉴权**，而且要把"密钥怎么传进沙箱"这条链路写清楚。

8. **环境变量用白名单转发**，不要把整个环境倒进容器。

9. **放宽安全限制时把范围缩到最小并写明原因**：只有 `<img>`/`<iframe>` 那一小块
   放开 Cookie，其余全 API 只认请求头，以此收窄 CSRF 面。

10. **两阶段初始化支持温池**：先搭架子报 ready，配置后到才真正启动。
    并且**新模式对老部署零影响**。

11. **跨进程的资源不会跟着你的进程一起死**：启动时清理上一轮残留，
    而且清理本身要容错。

12. **任何流向外部的通道都要过一次脱敏**：工具输出、错误信息、422 校验响应——
    同一个主题在三个位置出现。

13. **用关联 ID 替代泄露内部细节**：客户端拿随机串，运维拿它查到全部堆栈。

14. **日志噪音的危害是让真正的问题失去辨识度**：故意抛的 HTTP 错误不打堆栈，
    因为它提供不了 `exc.detail` 之外的信息，还会让例行抖动看起来像崩溃。

15. **日志级别按"是不是我的问题"分，而不是按"严重不严重"分**：4xx 是 info，
    5xx 是 error。

16. **遥测上报路由模板而非真实路径**：既不泄露 ID，又能聚合统计；
    匹配不上的路由上报 `/unmatched`。

17. **辅助设施的失败不能改变主流程的结果**：遥测整个包在 try/except 里。

## 下一站

回到 `openhands-tools/`（16839 行）—— 具体工具的实现：
终端工具怎么保持状态、文件编辑器怎么防止改错地方、多 agent 委派怎么落地。
这是前面所有抽象的最终落点。

# 精读 10：工具层（tool/ 3702 行 + mcp/ 2085 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/`
> 文件：`tool/{tool,schema,registry,client_tool}.py`、`tool/builtins/`、`mcp/{tool,client,config}.py`
> 前置：`notes/04-agent-step.md`（并行执行与资源锁）、`notes/10-llm-seam.md`（能力表）

## 这一节在讲什么

工具是 AI 能对世界做的事。这一层要回答：

1. 一个工具怎么定义，才能让 agent、审批层、并行调度器、序列化层都用得上
2. **本地写的工具**、**MCP 服务器提供的工具**、**前端执行的工具**——
   三种来源怎么统一成同一个东西

## 一、一个工具由什么组成

```python
class ToolDefinition[ActionT, ObservationT](DiscriminatedUnionMixin, ABC):
    model_config = ConfigDict(frozen=True, arbitrary_types_allowed=True)

    name: ClassVar[str] = ""                          # 自动从类名推导
    description: str
    action_type: type[Action]                         # 输入的结构定义
    observation_type: type[Observation] | None        # 输出的结构定义
    annotations: ToolAnnotations | None               # 行为提示（下面细讲）
    meta: dict[str, Any] | None

    executor: SkipJsonSchema[ToolExecutor | None] = Field(exclude=True)   # 真正干活的
    response_schema: SkipJsonSchema[...] = Field(exclude=True)
```

**四个关注点被分开了：**

| 部分 | 回答什么 |
|---|---|
| `description` | 给模型看的说明 |
| `action_type` / `observation_type` | 输入输出的结构（自动生成 JSON schema） |
| `annotations` | **这个工具的行为性质**（只读？破坏性？） |
| `executor` | 真正的实现 |

注意 `executor` 标了 `exclude=True`——**序列化时不写出去**。因为它是运行时对象
（可能持有网络连接、子进程），存不了也不该存。**会话存盘时只存"有哪些工具"，
恢复时重新创建实现。**

### 名字自动从类名推导

```python
def __init_subclass__(cls, **kwargs):
    """Automatically set name from class name when subclass is created."""
    if "name" not in cls.__dict__:
        cls.name = _camel_to_snake(cls.__name__).removesuffix("_tool")
```

`TerminalTool` → `terminal`，`FileEditorTool` → `file_editor`。

**省掉一次重复**（类名和字符串名各写一遍），也**消灭了一类 bug**（两处不一致）。

## 二、行为标注：直接采用 MCP 规范

```python
class ToolAnnotations(BaseModel):
    """Based on Model Context Protocol (MCP) spec"""

    readOnlyHint: bool = False
    """If true, the tool does not modify its environment. Default: false"""

    destructiveHint: bool = True
    """If true, the tool may perform destructive updates ...
    (meaningful only when readOnlyHint == false) Default: true"""

    idempotentHint: bool = False
    """If true, calling the tool repeatedly with the same arguments will have no
    additional effect ... Default: false"""

    openWorldHint: bool = True
    """If true, this tool may interact with an 'open world' of external entities...
    For example, the world of a web search tool is open, whereas that of a memory
    tool is not. Default: true"""
```

**四个维度，注意默认值全都是"保守"的一侧：**

- 默认**不是**只读（假设它会改东西）
- 默认**是**破坏性的
- 默认**不**幂等（不能安全重试）
- 默认**是**开放世界（会碰外部系统）

> **写新工具时如果忘了标注，系统会按最危险的情况对待它。** 这和第六节那个
> "失败关闭"是同一个原则，只是应用在了默认值上。
>
> 可迁移的道理：**默认值要选"忘了配置也不会出事"的那个**，而不是选"最常用"的那个。

这些标注的实际用途在第三节见过：`readOnlyHint` 决定要不要跳过审批
（只读工具不问用户，避免审批疲劳）。

**而且他们直接用了 MCP 的字段名**（`readOnlyHint` 这种驼峰命名在 Python 里很别扭），
甚至在注释里贴了规范的具体行号链接。**兼容性优先于代码风格一致性**——
因为这些字段要和 MCP 生态互通。

类里还有一处很实在的注释：

```python
# We need to define the title here to avoid conflict with MCP's ToolAnnotations
# when both are included in the same JSON schema for openapi.json
title="openhands.sdk.tool.tool.ToolAnnotations",
```

自己的 `ToolAnnotations` 和 MCP 库的 `ToolAnnotations` 同名，生成 API 文档时会撞。
用全限定名当标题避开。**这种问题只有真的生成过 OpenAPI 文档才会遇到。**

## 三、"我没想过"和"我想过了，没事"是两回事

第三节提过工具要申报资源。这里看它的完整定义：

```python
@dataclass(frozen=True, slots=True)
class DeclaredResources:
    keys: tuple[str, ...]
    declared: bool
```

注释把一个微妙但重要的区分讲透了：

> The distinction between `declared=True` with empty keys and `declared=False` is
> subtle but important:
>
> - `declared=True, keys=()`: the tool has explicitly analysed its resource usage and
>   determined it touches nothing shared. The executor trusts this and skips locking
>   entirely.
> - `declared=False`: the tool has *not* declared its resources (the default). The
>   executor cannot assume safety, so it falls back to a tool-wide mutex that
>   serializes all calls to this tool.
>
> **In short: `declared=False` means "I haven't thought about it" while
> `declared=True, keys=()` means "I have, and I'm safe."**

**"我没想过"和"我想过了，确实不碰共享资源"在数据上必须能区分开。**

如果用"空列表"同时表示这两种情况，那么所有没申报的工具都会被当成"安全的"，
然后并行执行时数据就坏了。

而现在：没申报 → 整个工具串行执行（慢但安全）；申报了空 → 完全不加锁（快）。
**默认值是 `declared=False`，也就是默认慢而安全。**

> **可迁移的道理**：**"未知"和"已知为空"必须用不同的值表示。**
> 这和第六节那个"UNKNOWN 不能等同于 LOW"是**完全同一个道理**，
> 在两个模块各自出现了一次。

## 四、一个非常具体的经验：字段顺序影响正确性

```python
_prioritize_schema_fields(schema=schema, priority=("security_risk", "summary"))
```

```python
def _prioritize_schema_fields(schema, priority) -> None:
    """Move *priority* fields to the front of ``schema["properties"]``.

    This ensures the LLM generates short metadata fields before large content
    parameters, so output-token truncation does not cut required fields.
    See https://github.com/OpenHands/software-agent-sdk/issues/1911
    """
```

**把 `security_risk` 和 `summary` 这两个短字段挪到参数表最前面。**

原因：模型是**顺序生成**的，而输出有长度上限。如果参数表里"文件内容"（可能几千行）
排在前面、`security_risk` 排在后面，那么一旦输出被截断，**风险等级这个必填字段就没了**。

挪到前面，先生成那两个短字段，再生成大块内容。**即使被截断，元数据是完整的。**

> **这条经验非常具体，但背后的原则通用**：
> **和"顺序生成 + 有长度上限"的系统打交道时，重要的短字段要排在前面。**
> 挂着 issue 编号 #1911 —— 又一个真踩过的坑。
>
> 注意这和第八节那个"时间放最后"看起来相反，其实是同一类思考：
> **顺序不是随意的，它和系统的机制（前缀缓存 / 顺序截断）直接相关。**

## 五、输出处理：漏斗式的统一出口

工具的 `__call__` 做三件事：

```python
def __call__(self, action, conversation=None) -> Observation:
    """Validate input, execute, and coerce output."""
    if self.executor is None:
        raise NotImplementedError(f"Tool '{self.name}' has no executor")

    result = self.executor(action, conversation)          # ① 执行

    # ② 把各种返回值统一成 Observation
    if self.observation_type:
        observation = result if isinstance(result, self.observation_type) \
                      else self.observation_type.model_validate(result)
    elif isinstance(result, Observation):   observation = result
    elif isinstance(result, BaseModel):     observation = Observation.model_validate(result.model_dump())
    elif isinstance(result, dict):          observation = Observation.model_validate(result)
    else: raise TypeError(...)

    # ③ 脱敏
    # Every tool's output funnels through here, so masking once keeps a new
    # tool covered by default instead of by remembering to patch it.
    if conversation is None:
        return observation
    registry = conversation.state.secret_registry
    return registry.mask_secrets_in_model(observation)
```

**第三步的注释是这一节最值得记的一句话：**

> Every tool's output funnels through here, so masking once keeps a new tool covered by
> default instead of by remembering to patch it.
> （每个工具的输出都汇集到这里，所以在这里脱敏一次，就能让新工具默认被覆盖，
> 而不是靠记得去打补丁。）

**场景**：你配置了数据库密码这类密钥。某个工具执行 `env` 命令，输出里就有密码。
如果不处理，密码会进账本、进日志、被发给模型。

**解法不是"每个工具自己记得脱敏"**——那必然有人忘。而是**找到唯一的出口，在那里做一次**。

> **这是安全工程里最重要的一个结构性思路**：
> **不要依赖每个人都记得做正确的事，要让正确的事成为唯一的通路。**
>
> 判断一个项目的安全成熟度，看它是"到处都有脱敏调用"还是"有一个收口的地方"。

## 六、Observation：输出可以不只是文字

```python
class Observation(Schema, ABC):
    ERROR_MESSAGE_HEADER: ClassVar[str] = "[An error occurred during execution.]\n"

    content: list[TextContent | ImageContent] = Field(default_factory=list)
    is_error: bool = False

    @property
    def to_llm_content(self) -> Sequence[TextContent | ImageContent]:
        llm_content = []
        if self.is_error:
            llm_content.append(TextContent(text=self.ERROR_MESSAGE_HEADER))
        llm_content.extend(self.content)
        return llm_content
```

两个点：

**① 内容是"文字块或图片块"的列表**，不是一个字符串。所以工具可以返回截图
（浏览器工具就是这么干的）。

**② 出错时自动在前面加一行标记。** 而且是在 `to_llm_content` 里加，
不是在存储时加——**存的是原始数据，给模型看的时候才加装饰。**

> 回想第一节"原始数据和加工数据分开"，这里是同一个原则的又一次应用：
> `content` 是事实，`to_llm_content` 是为特定用途做的投影。

子类可以重写 `to_llm_content` 提供更丰富的呈现（注释举例：图片、差异）。

## 七、三种工具来源，一个接口

### ① 本地 Python 工具

正常继承 `ToolDefinition`，实现一个 `create` 类方法。文档里的例子很清楚：

```python
class TerminalTool(ToolDefinition[TerminalAction, TerminalObservation]):
    @classmethod
    def create(cls, conv_state, **params):
        executor = TerminalExecutor(working_dir=conv_state.workspace.working_dir, **params)
        return [cls(name="terminal", ..., executor=executor)]
```

注意 `create` 返回的是**列表**——一次可以产出多个工具。注释解释了为什么：
`BrowserToolSet` 这种"一套相关工具"需要一次创建出好几个。

### ② MCP 工具

> **MCP（Model Context Protocol）**：Anthropic 提出的开放协议，让外部服务
> 以标准方式向 AI 提供工具。装一个 MCP 服务器就能给 AI 加一批能力。

```python
class MCPToolDefinition(ToolDefinition[MCPToolAction, MCPToolObservation]):
    """MCP Tool that wraps an MCP client and provides tool functionality."""
    mcp_tool: mcp.types.Tool
```

**就是 `ToolDefinition` 的一个子类。** 于是对上层来说，MCP 工具和本地工具**毫无区别**
——第三节那个 agent 的派发逻辑一行分支都不用写。

名字这样接过来：

```python
@property
def name(self) -> str:
    """Return the MCP tool name instead of the class name."""
    return self.mcp_tool.name
```

#### 处理 MCP 版本差异的一个细节

```python
# mcp 2.x dumps its snake_case attribute names unless by_alias=True; mcp 1.x
# only reads the camelCase wire names. Keys inside inputSchema/outputSchema
# are user JSON Schema and must never be renamed.
_MCP_TOOL_WIRE_KEYS: Final[dict[str, str]] = {
    "input_schema": "inputSchema",
    "output_schema": "outputSchema",
    "meta": "_meta",
}
```

MCP 库 1.x 和 2.x 的字段命名习惯不一样。**而且要特别小心：
`inputSchema` 内部的键是用户自己的 JSON Schema，绝对不能跟着改名。**

序列化时的注释说明了为什么要固定用"协议规定的名字"：

```python
# Persisted events outlive the resolved mcp major; always write the spec's wire
# names so every mcp version can read them back.
```

**存档的寿命比依赖库的大版本长。** 所以落盘一定要用**协议规定的格式**，
而不是当前依赖库恰好用的格式。否则升级库之后，老存档读不出来了。

> **可迁移的道理**：**持久化格式要绑协议/规范，不要绑当前依赖库的实现细节。**
> 这是"存档能不能长期读"的分水岭。

#### 连接断了会自动重连一次

```python
"""If the client's session has been lost (e.g., due to a transient server error
such as HTTP 503), attempt to reconnect once before failing. This prevents a
single transient error from permanently disabling all MCP tools for the
remainder of the conversation."""
```

**防的是：一次瞬时错误导致这个 MCP 服务器的所有工具在这次会话里永久失效。**

而且区分了两种断开：**主动关闭的**（`_closed`）不重连，直接返回错误说明原因；
**意外断开的**才重连。

### ③ 客户端工具（在前端执行）

这是最有意思的一种：

```python
"""Client-defined tools: tools defined via JSON spec, executed by external clients.

These tools allow frontend clients (like Agent Canvas) to register tools purely via
JSON in ``POST /conversations``, with no Python code required. When the agent calls a
client tool, an ActionEvent is emitted over the WebSocket and the client handles
execution. The SDK returns an acknowledgment observation immediately.

This eliminates the need for Python tool code in JavaScript repos and the complex
``tool_module_qualnames`` / ``--import-modules`` plumbing."""
```

**前端用一段 JSON 声明一个工具，AI 调用它时，服务端只发一个事件出去、立刻返回一个
"已收到"的观察，真正的执行在浏览器里发生。**

典型用途：需要访问用户本地环境或 UI 状态的操作（"打开这个文件的编辑器"、
"读取当前选中的文本"）。

文档最后一句在说这个设计解决了什么：**以前要在 JavaScript 仓库里写 Python 工具代码，
还要配一堆模块导入路径。** 现在不用了。

#### 动态生成类带来的一个坑

```python
# ``Action.from_mcp_schema`` creates a *concrete* ``Action`` subclass whose ``kind`` is
# derived from the class name (``ClientAction_<name>``). These subclasses register
# process-globally in the discriminated-union hierarchy, so creating two classes with
# the same name (e.g. when the same client tool is registered twice, or re-created on
# conversation resume) makes ``Action.resolve_kind`` raise a duplicate-class error and
# breaks event deserialization. We therefore cache the generated type per tool name and
# reject same-name/different-schema conflicts explicitly.
_client_action_types: dict[str, type[Action]] = {}
_client_action_schemas: dict[str, dict[str, Any]] = {}
_client_action_lock = threading.RLock()
```

回想第一节那个 `kind` 字段（靠类名恢复类型）。这里是它的副作用：

**从 JSON 动态生成的类，如果注册两次同名的，全局类型注册表就冲突了，
然后整个事件反序列化就坏了。**

触发场景很现实：同一个工具注册了两次、或者**会话恢复时重新创建了一遍**。

解法：**按工具名缓存生成的类型**，同名不同结构的显式报错。加锁是因为可能多线程注册。

> **这是"动态生成类型"这种做法的固有风险**：类型是进程级全局的，
> 而动态生成的时机是不受控的。**用缓存把"生成"变成"幂等"。**

## 八、工具注册表：延迟创建

```python
Resolver = Callable[[dict[str, Any], "ConversationState"], Sequence[ToolDefinition]]

_LOCK = RLock()
_REG: dict[str, Resolver] = {}
_USABILITY_REG: dict[str, UsabilityChecker] = {}
_MODULE_QUALNAMES: dict[str, str] = {}
```

注册的不是工具实例，而是一个**解析器**（给它参数和会话状态，它造出工具）。

**为什么需要延迟？** 因为很多工具的创建依赖会话信息——终端工具要知道工作目录，
文件编辑器要知道工作区路径。注册的时候这些还不存在。

还有一个可用性检查：

```python
@classmethod
def is_usable(cls) -> bool:
    """Return whether the tool can be used in the current environment."""
    return True
```

**工具可以自己说"我在当前环境用不了"**（比如某个工具需要 Docker，而机器上没有）。
于是它不会出现在给模型的工具列表里——**AI 不会看到它其实用不了的工具**。

> 这和第八节那个"功能没开的提示片段不出现"是同一个思路：
> **不要让模型看到它其实没有的能力。**

`_MODULE_QUALNAMES`（记录工具来自哪个模块）是为了会话恢复——
重启进程后要知道去哪里 import 回来。

## 九、这一节的可迁移结论

1. **工具定义要把"说明 / 结构 / 行为性质 / 实现"四件事分开**：
   前三个可序列化，实现是运行时对象，存盘时排除。

2. **默认值要选"忘了配置也不会出事"的那个**：行为标注默认按"会改环境、有破坏性、
   不幂等、碰外部系统"处理。

3. **"未知"和"已知为空"必须用不同的值表示**：`declared=False`（没想过）vs
   `declared=True, keys=()`（想过了，安全）。前者串行，后者免锁。
   和"UNKNOWN 不等于 LOW"是同一个道理。

4. **和"顺序生成 + 有长度上限"的系统打交道，重要的短字段要排在前面**：
   否则截断会切掉必填的元数据。

5. **找到唯一的出口，在那里做一次**：所有工具输出都经过同一个 `__call__`，
   脱敏放在这里，新工具自动被覆盖。**不要依赖每个人都记得做正确的事。**

6. **存的是原始数据，展示时才加装饰**：`content` 是事实，`to_llm_content` 是投影。

7. **让不同来源的东西继承同一个基类**：本地工具、MCP 工具、客户端工具都是
   `ToolDefinition`，于是上层派发零分支。

8. **持久化格式绑协议，不绑依赖库的实现细节**：存档寿命比库的大版本长。

9. **瞬时错误要就地重试一次，而且要区分"主动关闭"和"意外断开"**：
   否则一次 503 会让整个服务的工具在本次会话永久失效。

10. **动态生成类型必须做成幂等的**：按名字缓存，同名不同结构显式报错。
    类型注册表是进程级全局的，而生成时机不受控。

11. **注册解析器而不是实例**：创建时机要推迟到会话信息就绪。

12. **让工具自己声明"当前环境能不能用"**：用不了的工具不出现在列表里，
    避免模型调用不存在的能力。

## 下一站

`openhands-workspace/`（3067 行）+ `openhands-agent-server/`（29851 行）——
执行环境：本地 / Docker / K8s 工作区怎么抽象，
以及 REST/WebSocket 服务端怎么把整个 SDK 变成一个可远程调用的服务。

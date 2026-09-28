# 精读 16：ACP 适配层（agent/acp_agent.py，4681 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/agent/acp_agent.py`
> 相关：`agent/acp_models.py`、`agent/acp_file_credentials.py`、`agent/acp_tracing.py`、
> `settings/acp_providers.py`、`settings/acp_install_catalog.py`
> 前置：`notes/04-agent-step.md`（内置 Agent 的 step）、`notes/13-tool-implementations.md`（委派）

## 这一节在讲什么

这是这个 SDK 最独特的能力：**把 Claude Code、Codex、Gemini CLI 当成子 agent 来驱动。**

```
"""ACPAgent — an AgentBase subclass that delegates to an ACP server.

The Agent Client Protocol (ACP) lets OpenHands power conversations using
ACP-compatible servers (Claude Code, Gemini CLI, etc.) instead of direct LLM
calls. The ACP server manages its own LLM, tools, and execution; the ACPAgent
relays user messages and collects the response."""
```

> **ACP（Agent Client Protocol）**：一个开放协议，让"agent 服务器"和"客户端"分离。
> agent 服务器自己管模型、工具、执行；客户端只负责传消息和收结果。

**4681 行——比内置 Agent（1538 行）大三倍。** 这个比例本身就是结论：
**适配三个厂商的 CLI，比实现一个 agent 本身麻烦得多。**

而且这一节的性质和前面十五节不同：**前面读的是设计，这一节读的基本上是伤疤。**
几乎每个字段和方法的注释里都挂着一个 issue 编号或者"某个版本的某个行为"。

## 一、第一个关键决定：一步 = 一个完整回合

```python
"""Unlike the built-in Agent, one ACP ``step()`` maps to one complete remote
assistant turn. ACPAgent therefore emits a terminal ``FinishAction`` at the end
of each step to delimit that completed turn for downstream consumers."""
```

回想第三节：内置 Agent 的一步是"问一次模型 + 执行工具调用"。

**但 ACP 服务器（比如 Claude Code）自己就是一个完整的 agent** ——
你发一条消息，它内部会跑很多轮、调很多工具，最后返回一个结果。

于是**语义被重新映射**：一个 `step()` = 远端的一整个回合。

而且每步结束都发一个 `FinishAction`。**为什么需要这个？** 因为上层的循环、评审员、
事件消费者全都是按"动作—观察"配对的模型写的（第一、二、三节）。
**一个合成的终结动作，让 ACP 回合能被现有的全部机制无差别地处理。**

> **可迁移的道理**：**接入一个语义粒度不同的外部系统时，在边界上合成缺失的信号，
> 让内部的既有机制不用改。** 比改动所有下游消费者便宜得多。

## 二、能力声明：四个 `False`

```python
@property
def supports_openhands_tools(self) -> bool:
    """``False`` — the ACP server manages its own toolset."""
    return False

@property
def supports_openhands_mcp(self) -> bool:
    """``False`` — OpenHands does not create in-process MCP tools here.
    ACP agents still honor ``mcp_config`` by forwarding configured servers to
    the ACP subprocess at session creation time."""
    return False

@property
def supports_condenser(self) -> bool:
    """``False`` — the ACP server manages its own context window."""
    return False

@property
def agent_kind(self) -> Literal["acp"]:
    return "acp"
```

**前面十五节讲的三大块——工具层、MCP、压缩器——在这里全部被声明为"不适用"。**

这是一个很干脆的边界划分：**远端 agent 自己管这些，OpenHands 不要插手。**

注意 MCP 那条的补充说明：**配置还是会转发过去**，只是不在本地创建工具实例。
**"我不做这件事"和"我连配置都不传"是两回事。**

> **可迁移的道理**：**适配器要显式声明自己不支持什么，而不是让上层去猜或者试。**
> 有了这些属性，上层代码可以做 `if agent.supports_condenser:` 而不是
> `if isinstance(agent, ACPAgent)`。**按能力判断，而不是按类型判断。**

`llm` 字段用了一个假的占位（`_make_dummy_llm()`），因为基类要求有，
但 ACP agent 不直接调模型——**和第十四节 `RouterLLM` 塞假模型名是同一种妥协。**

## 三、提供商注册表：把厂商差异变成数据

`settings/acp_providers.py` 里的 `ACPProviderInfo` 是这一节的核心数据结构。
**每个字段的注释都是一段故事。**

```python
@dataclass(frozen=True)
class ACPProviderInfo:
    """Immutable metadata record for one built-in ACP provider."""
    key: str
    display_name: str
    default_command: tuple[str, ...]
    api_key_env_var: str | None
    base_url_env_var: str | None
    default_session_mode: str | None
    agent_name_patterns: tuple[str, ...]
    supports_set_session_model: bool
    ...
```

### 权限模式：四个厂商四种情况

```python
default_session_mode: str | None
"""ACP session-mode ID set right after ``session/new``, or ``None`` to skip the call.

For servers with a permission-suppressing mode that is the value:
``bypassPermissions`` (claude-agent-acp), ``agent-full-access``
(@agentclientprotocol/codex-acp).
gemini-cli uses ``default`` (its ``yolo`` mode errors at init); the ACP bridge
auto-approves permission requests, so the mode doesn't gate prompts.
``None`` for a server that exposes no permission mode at all: pi-acp maps ACP
modes onto pi's thinking levels (``off``..``xhigh``) and rejects any other id,
so sending one would fail session init to no purpose."""
```

**同一个协议字段，四家四种含义：**
- Claude Code：`bypassPermissions`
- Codex：`agent-full-access`
- Gemini CLI：只能用 `default`（它的 `yolo` 模式**在初始化时就报错**）
- pi-acp：**这个字段被挪用成了"思考力度"**（off..xhigh），传任何权限模式都会失败

**最后一条尤其说明问题**：同一个协议字段被某个实现赋予了完全不同的语义。
**协议规定了字段名，没规定所有人都按同一个意思用。**

### 认证方式也不统一

```python
api_key_env_var: str | None
"""``None`` for providers that authenticate via browser login rather than an
API key (e.g. Claude Code's ``claude-login`` flow)."""

base_url_env_var: str | None
"""Allows routing provider calls through a proxy such as LiteLLM.
``None`` if the provider does not support env-based base-URL override."""
```

**有的用 API key，有的用浏览器登录。** 后者意味着凭证是**一个文件**，
不是一个环境变量——这就引出了后面那一整套"文件凭证"机制。

### 一个被现实推翻的假设

```python
supports_set_session_model: bool
"""``True`` if this provider selects its *initial* model via the
``set_session_model`` protocol call (rather than session ``_meta``).

... ``True`` for every built-in provider, which gets a one-shot
``set_session_model`` call right after the session is created.
claude-agent-acp was ``False`` until 0.30.0 was found to **silently ignore**
the session-``_meta`` selection it relied on (#3654); its ``_meta`` payload is
still sent alongside."""
```

**Claude Code 0.30.0 开始静默忽略了原来那种设置模型的方式。**

处理方式很务实：**改用新方式，但旧的 `_meta` 载荷照样一起发。**
（因为可能还有用户跑着老版本。）

> **可迁移的道理**：**外部系统的行为变更常常是静默的——不报错、不警告，只是不生效。**
> 兼容多个版本时，"两种方式都发一遍"往往比"检测版本再选一种"更可靠。

### 为什么需要一个额外的 `acp_server` 字段

```python
acp_server: str | None = Field(
    description=("...Set by ACPAgentSettings.create_agent() from
    ACPAgentSettings.acp_server so the authoritative key survives onto the agent
    — and thus onto ConversationInfo.agent — **because the launch command in
    acp_command does not reliably reverse-map to a provider.** Informational
    only: consumers use it to resolve a provider brand label / model list; the
    subprocess is still launched from acp_command."))
```

**从启动命令反推"这是哪家的 CLI"不可靠**（用户可以包一层脚本、换个路径、用别名）。
所以单独存一个权威的标识键。

但同时明确说了 `Informational only` —— **这个键只用来查品牌名和模型列表，
真正启动进程还是用命令。** 两个来源各管一件事，不互相假装。

## 四、两种超时，而且理由完全不同

这是整个文件里最值得学的一对设计。

```python
acp_prompt_timeout: float = Field(default=1800.0, description=(
    "Inactivity timeout in seconds for a single ACP prompt() call. "
    "The deadline resets on every update from the ACP server (token, thought, "
    "tool-call progress, usage), so a steadily-progressing agent runs as long "
    "as it keeps making progress; the prompt is only aborted after this many "
    "seconds with no activity at all. Prevents indefinite hangs when the ACP "
    "server stops responding without killing legitimately long-running work."))

acp_startup_timeout: float = Field(default=90.0, description=(
    "Timeout in seconds for ACP server startup: spawning the subprocess, the "
    "initialize/authenticate handshake, and new_session()/load_session(). "
    "**Unlike acp_prompt_timeout, this is a hard deadline rather than an idle "
    "deadline, since startup has no intermediate progress signal to reset it "
    "against.** Prevents an indefinite hang when the ACP server blocks on "
    "authentication (e.g. an expired token) without ever raising."))
```

**对话超时是"空闲超时"（30 分钟没有任何动静才中断），
启动超时是"硬超时"（90 秒必须完成）。**

**判断依据写得很清楚：有没有中间进度信号可以用来重置计时器。**

- 对话过程中：每个 token、每条思考、每次工具进度都是进度信号 → 可以做空闲超时 →
  **一个持续在干活的 agent 可以跑任意久**
- 启动过程中：从发出 initialize 到收到响应，中间什么都没有 → 只能硬超时

而且都说明了**在防什么**：服务器不响应了（但不杀掉合法的长任务）、
认证卡住了（比如 token 过期，而且它**不报错只是挂着**）。

> **这是超时设计的通用方法论**：
> **先问"这个过程有没有可观测的中间进度"。有 → 空闲超时；没有 → 硬超时。**
>
> 很多系统只会给一个硬超时，结果要么砍掉合法的长任务，要么设得太大导致挂死时
> 要等很久。**区分这两种超时，两个问题一起解决。**

## 五、会话 id 是秘密

```python
acp_resume_session_id: str | None = Field(description=(
    "... **Treated as a secret on the wire — possession of the id is enough to
    resume the underlying ACP session**, so default serialization redacts it; pass
    ``expose_secrets='plaintext'`` (trusted backend) or ``expose_secrets='encrypted'``
    plus a cipher (frontend round-trip) when the value must cross a serialization
    boundary."))

@field_serializer("acp_resume_session_id", when_used="always")
def _serialize_acp_resume_session_id(self, value, info):
    """Default ``model_dump`` / ``model_dump_json`` redacts the id so it cannot leak
    into logs, trace exports, or PR review attachments."""
    return serialize_secret(SecretStr(value), info)
```

**"拿到这个 id 就能恢复那个会话"，所以它是凭证，默认序列化时必须打码。**

注释列出了三个泄露渠道：**日志、追踪导出、PR 评审附件**。
最后一个很有意思——**它承认了"调试信息会被贴到 PR 里"这个现实。**

三档暴露级别：默认打码 / 明文（可信后端）/ 加密（前端往返）。

> **可迁移的道理**：**判断一个值是不是秘密，标准是"拿到它能做什么"，
> 而不是"它看起来像不像密码"。** 一个会话 ID、一个分享链接、一个重置令牌，
> 都符合"持有即权限"，都该当秘密处理。

## 六、多会话共享沙箱时的资源隔离

这一段的注释信息密度极高。

```python
def _isolate_acp_data_dir(self, state, env: dict[str, str]) -> None:
    """Relocate the CLI's data/config root to a per-conversation directory.

    ... point the recognised provider's data-dir env var (``CODEX_HOME`` /
    ``CLAUDE_CONFIG_DIR`` / ``HOME``) at ``<persistence_dir>/acp/<provider>`` —
    the same per-conversation tree :meth:`_materialise_file_secrets` seeds auth
    into, so a relocated ``CODEX_HOME`` and a materialised ``auth.json`` always
    agree on one directory. **This stops conversations that share a sandbox from
    racing on a single shared HOME's CLI auth/config/cache/lock files (#1019).**"""
```

**问题**：多个会话可能共享一个沙箱容器。每个会话都启动一个 Claude Code 子进程，
而这些 CLI 默认把配置、认证、缓存、锁文件写到 `~/.claude` 这类共享位置。
**于是它们会互相踩。**

**解法**：把每个 CLI 的"数据目录"环境变量指到**每会话独立的目录**。

### 但三家的"数据目录"杠杆强度不一样

```python
"""``HOME`` (the only lever for gemini-cli, which hard-codes ``~/.gemini`` and
ignores ``XDG``, and for pi-acp, whose session map is hard-coded to
``~/.pi/pi-acp``) has a **wider blast radius** than the surgical ``CODEX_HOME``
/ ``CLAUDE_CONFIG_DIR``: it also relocates the home dir seen by anything the CLI
subprocess itself spawns (``git``, ``npm``, ``node``, shells — e.g.
``~/.gitconfig``, ``~/.npmrc``, the npm cache). **That is accepted as the cost of
isolating Gemini at all**; callers that need a narrower scope can leave isolation
off for Gemini."""
```

- Codex 有 `CODEX_HOME`、Claude 有 `CLAUDE_CONFIG_DIR` → **精准**
- Gemini CLI **硬编码 `~/.gemini` 并且忽略 XDG 标准** → 只能改 `HOME`
- pi-acp 的会话映射也硬编码在 `~/.pi/pi-acp` → 只能改 `HOME`

**改 `HOME` 的副作用**：这个 CLI 再去启动 `git`、`npm`、`node` 时，
它们看到的家目录也变了——`~/.gitconfig` 没了、npm 缓存重新下载。

**作者明确说这是"为了能隔离 Gemini 所接受的代价"，并且给了"关掉隔离"的出路。**

> **可迁移的道理**：**当某个外部程序不提供细粒度的配置入口时，你只剩下粗粒度的杠杆
> （改 `HOME`、改 `PATH`），而粗粒度杠杆一定有副作用。**
> 正确的做法是：**用它、写清副作用、给出退出选项**——
> 而不是假装没有副作用，也不是因为有副作用就放弃这个功能。

### 一个凭证和一个目录必须指向同一处

注释里这句容易被读过去但很重要：

> the same per-conversation tree ``_materialise_file_secrets`` seeds auth into,
> **so a relocated ``CODEX_HOME`` and a materialised ``auth.json`` always agree
> on one directory**

**把认证文件写到哪、和告诉 CLI 去哪找配置，必须是同一个地方。**
这两件事在代码里是两个不同的方法，很容易改一处忘一处。注释把这个耦合点标出来了。

### 顺序也很关键

```python
"""Ordering: this runs *after* the ``secret_registry`` injection in
:meth:`_start_acp_server`. Relocation is now credential-blind (the auth-conflict
strip is keyed on ``CLAUDE_CODE_OAUTH_TOKEN``, not on the config dir), so the
data-dir var it sets never affects auth."""
```

**"现在是凭证无关的"** —— 这句话的潜台词是：**以前不是。**

以前 `_ENV_CONFLICT_MAP` 是按配置目录做键的，所以"为了隔离而设置配置目录"
会意外地把一个能用的 `ANTHROPIC_API_KEY` 剥掉（issue #3588）。
改成按 OAuth token 做键之后，两件事解耦了。

## 七、环境变量冲突：一个厂商的密钥会毁掉另一个

```python
def _strip_conflicting_env(self, env: dict[str, str]) -> None:
    """Remove env vars that would defeat this provider's own credential.

    Scoped to the resolved provider's :attr:`ACPProviderInfo.env_conflicts`:
    the same variable is another provider's *credential* (a provider whose
    ``api_key_env_var`` is ``ANTHROPIC_API_KEY``), and stripping it there leaves
    it with nothing to authenticate with.

    An unrecognised server keeps the conservative union: without an identity we
    cannot tell whose credential a dominant variable belongs to, and a
    directly-constructed ``ACPAgent`` running Claude Code is the case the rule
    was written for (#3588)."""
    provider = self._resolved_provider()
    if provider is not None:
        specs = provider.env_conflicts
    else:
        specs = tuple(spec for info in ACP_PROVIDERS.values()
                      for spec in info.env_conflicts)   # ← 保守的并集
    for spec in specs:
        if spec.dominant in env:
            for name in spec.strip:
                env.pop(name, None)
```

**问题**：环境里同时有 `ANTHROPIC_API_KEY` 和 `CLAUDE_CODE_OAUTH_TOKEN` 时，
CLI 的行为取决于它优先看哪个——**而这可能不是你想要的那个。**

**解法**：声明"当 A 存在时，剥掉 B、C"（`dominant` / `strip`），
但**范围限定在当前提供商的规则内**。

**认不出提供商时取所有规则的并集**（更保守），理由也写了：
没有身份就无法判断某个变量是谁的凭证，而"直接构造一个跑 Claude Code 的 ACPAgent"
正是这条规则当初要解决的场景。

> **可迁移的道理**：**启动外部进程时，环境变量不只是"传进去"，还要考虑"哪些必须拿掉"。**
> 多个凭证共存时的优先级往往是那个程序的内部实现细节，
> 最可靠的做法是**只留你想让它用的那一个**。

## 八、桥接层：把远端的流式更新翻译成本地事件

`_OpenHandsACPBridge` 实现 ACP 的"客户端"一侧。

### 权限请求一律自动批准

```python
async def request_permission(self, session_id, tool_call, options, **kwargs):
    """Auto-approve all permission requests from the ACP server."""
    # Pick the first option (usually "allow once")
    option_id = options[0].option_id if options else "allow_once"
    logger.info("ACP auto-approving permission: %s (option: %s)", tool_call, option_id)
    return RequestPermissionResponse(outcome=AllowedOutcome(outcome="selected",
                                                            option_id=option_id))
```

**远端 agent 问"我能执行这个吗"，桥接层一律说"可以"。**

这看起来很可怕，但逻辑是自洽的：**审批应该发生在 OpenHands 这一层**
（第三、六节那套风险评估 + 确认策略），**而不是在远端 CLI 的交互式提示里**——
那个提示在无头环境里根本没人能回答。

注意它**记了日志**（`logger.info` 带上完整的 tool_call）——**自动批准必须留痕。**

而"询问用户"类的请求直接拒绝：

```python
async def create_elicitation(self, message, mode, **kwargs):
    """Decline elicitation requests; the headless bridge has no user to ask.

    Added to the ``Client`` protocol in agent-client-protocol 0.12.x. None of the
    pinned ACP providers elicit during a headless turn, so this is a defensive
    default rather than an exercised code path."""
    return DeclineElicitationResponse(action="decline")
```

**明确写了这是"防御性默认值，不是被实际走到的代码路径"。**

> 这种自我标注很有价值：**告诉后来的人"这段代码没有被真实流量覆盖"**，
> 于是改它的时候知道要格外小心（没有测试兜着）。

### 脱敏在"摄入时"做，不在"发出时"做

```python
def _mask_tool_call_entry(self, entry: dict[str, Any]) -> None:
    """Applied in place at ingestion (``session_update``) **so the accumulator
    itself never holds plaintext secrets**, and every downstream emitter
    (``_emit_tool_call_event`` and the supersede path in
    ``_cancel_inflight_tool_calls``) carries masked values for free.

    ``title`` is normally a benign server-set label, but **a misbehaving ACP
    server could echo a credential there** (e.g. ``Running: curl -H
    'Authorization: Bearer <token>'``), so it is masked too."""
    for key in ("title", "raw_input", "raw_output", "content"):
        if entry.get(key) is not None:
            entry[key] = self._mask_value(entry[key])
```

**在数据进来的那一刻就脱敏，于是内存里的累加器永远不持有明文。**

对比第十节那个"所有工具输出经过同一个出口脱敏"——**这里选的是入口而不是出口。**

**为什么？** 因为这份数据有**多个出口**（正常发出、取消时的补发）。
在入口做一次，所有出口自动受益。

> **可迁移的道理**：**脱敏要放在"数据流的收窄处"。** 单一出口就放出口，
> 多出口单入口就放入口。**关键是找到那个唯一的点。**

连 `title` 都脱敏的理由很实在：**它通常只是个标签，但一个行为异常的服务器
可能把带 token 的 curl 命令原样写进去。** 不信任外部系统的任何字段。

### 脱敏失败时的取向

```python
"""Defensive: on mask failure, returns the original value unchanged and logs at
DEBUG — **this may transiently leak the credential but prevents a crash**,
matching the regular terminal tool's masking contract."""
```

**脱敏挂了就放原值。** 并且**明确承认这可能泄露凭证**。

这是一个有争议但写清了的取舍——注释说是为了"匹配常规终端工具的脱敏契约"
（保持一致性），而且脱敏内部已经吞掉了凭证解析错误，实践中不该抛异常。

> 值得注意的是他们**把这个风险写在了注释里**，而不是藏起来。
> **一个被记录的已知缺陷，比一个没人知道的正确假设更安全。**

## 九、重试时的事件一致性：两个补偿方法

这是第一、二节那条"每个工具调用必须有恰好一个结果"的铁律，
在一个**不受控的外部系统**上的兑现。

### 情况 A：这一轮失败了要重试

```python
def _cancel_inflight_tool_calls(self) -> None:
    """Emit a terminal ``failed`` ACPToolCallEvent for every tool call in the
    accumulator that has not reached a terminal status yet.

    **ACP servers mint fresh ``tool_call_id``s on a retried turn**, so any
    ``pending`` / ``in_progress`` events already streamed during the failed
    attempt would otherwise be orphaned on ``state.events`` — no later
    notification reuses their id, and consumers that dedupe by ``tool_call_id``
    + "last-seen status wins" **would keep them spinning forever**."""
```

**重试时远端会生成全新的 id，所以上一轮那些"进行中"的事件永远不会被关闭。**
界面上那些卡片会一直转。

解法：**重试前，给所有未终结的调用补一个 `failed` 事件。**

而且顺序很讲究：

```python
on_event = self._client.on_event
self._clear_turn_callbacks()       # ← 先解绑
if on_event is None:
    return
for tc in ...:  on_event(...)       # ← 再发补偿事件
```

> Captures the bridge's ``on_event`` callback, **then unwires the bridge before
> emitting synthetic terminal events so trailing updates from the abandoned
> portal prompt cannot land after these failures.**

**先把回调摘下来再发补偿事件**，否则那个被放弃的请求可能还在往回推更新，
落在补偿事件之后，把状态又改回"进行中"。

### 情况 B：这一轮成功了但服务器少发了收尾帧

```python
def _flush_inflight_tool_calls_as_completed(self) -> None:
    """The prompt returned successfully, so a tool card the server opened but
    never closed (it sent ``ToolCallStart`` but no terminal ``ToolCallProgress``)
    is treated as completed. Since we now persist exactly one early ``started``
    event and one terminal event per call, **this guarantees the
    action->observation pairing holds for *every* call** — without it, a server
    that omits the closing frame would leave the early ``started`` event as the
    last word, and the relaxed canvas render gate would show that card spinning
    forever."""
```

**服务器只发了"开始"没发"结束"，但整个请求成功返回了 → 当作完成。**

推理依据写得很清楚：`conn.prompt` 只在它的工具跑完之后才返回，
所以请求返回了就意味着工具确实跑完了——**服务器只是漏了那一帧。**

> **可迁移的道理**：**对接外部系统时，不能假设它会发完整的状态机转移。**
> 你要在自己这一侧**补齐缺失的终结状态**，依据是你能确定的其他事实
> （"请求返回了" ⇒ "工具跑完了"）。
>
> 而且两种补偿的方向是相反的（失败补 `failed`、成功补 `completed`），
> **因为依据不同。** 这不是随意选的默认值。

## 十、模型标识：不要相信模型对自己的描述

```python
@property
def current_model_id(self) -> str | None:
    """... **This (the protocol-reported session model) is the authoritative model
    id — trust it over what the model says about itself.** Provider models
    introspect their own identity unreliably: **gemini-cli reports a stale
    ``gemini-1.5-flash-latest`` regardless of the active model, and codex won't
    name its tier.** Verified across all three providers; the switch is honored
    even when the model's self-description disagrees."""
```

**这段话是实测出来的：**
- Gemini CLI 不管实际用什么模型，都报一个过时的 `gemini-1.5-flash-latest`
- Codex 拒绝说出自己的档位

**结论：以协议层报告的会话模型为准，不要问模型自己。** 而且明确说了
"三家都验证过，即使模型的自我描述不一致，切换也是生效的"。

> **可迁移的道理**：**"问系统自己是什么"往往不可靠。** 要用带外的、
> 协议层面的信息源。这在 User-Agent 嗅探、语言版本检测、数据库版本判断上
> 都是同一个道理。

### 还有一个诚实的能力声明

```python
@property
def supports_runtime_model_switch(self) -> bool:
    """``True`` only for known providers that explicitly declare support for
    runtime model switching. **Unknown/custom providers use ``set_config_option``
    for *initial* model selection but that RPC is a generic config write, not a
    guaranteed live-switch primitive, so the picker is hidden for them.**"""
```

**能写这个配置项 ≠ 能在对话中途切换生效。**

所以对未知提供商，**界面上直接不显示切换控件**——
而不是显示一个可能静默失效的按钮。

> **不确定一个功能能不能用时，不要提供它。** 一个静默失效的控件
> 比一个不存在的控件更糟。

## 十一、用量统计：拿不到就说拿不到

```python
if not cost_recorded and not input_tokens and not output_tokens:
    # gemini-cli currently returns response.usage=None and response.field_meta=None
    # (ACP SDK strips _meta during serialization). Tracked in
    # google-gemini/gemini-cli#24280.
    logger.debug("No usage data from ACP server %s — token/cost tracking unavailable",
                 self._agent_name or "unknown")
```

**Gemini CLI 目前不返回任何用量数据**，而且原因有两层：
它自己返回 `None`，加上 ACP SDK 在序列化时把 `_meta` 剥掉了。
**上游 issue 编号都记着。**

处理：**打一条 debug 日志说明"追踪不可用"，而不是记 0。**

> 这个区分很重要：**记 0 会让统计看起来正常（"这次没花钱"），
> 而"不可用"是一个诚实的空缺。** 第十四节那个"我没能判断必须可检测"——
> **同一个原则在第六次出现。**

## 十二、这一节的可迁移结论

1. **接入语义粒度不同的外部系统时，在边界上合成缺失的信号**：
   一个 ACP 回合结束时合成 `FinishAction`，让内部所有既有机制不用改。

2. **适配器要显式声明自己不支持什么**：四个 `supports_*` 属性让上层按能力判断，
   而不是按类型（`isinstance`）判断。而且"我不实现"和"我连配置都不转发"是两回事。

3. **把厂商差异收敛成一张数据表，每个字段注明为什么**：
   同一个协议字段在四家有四种含义（甚至被挪用成"思考力度"）。
   **协议规定了字段名，没规定所有人按同一个意思用。**

4. **外部系统的行为变更常常是静默的**：某版本开始"静默忽略"原来的设置方式。
   兼容多版本时，"两种方式都发一遍"往往比"检测版本再选"更可靠。

5. **超时要先问"这个过程有没有可观测的中间进度"**：有 → 空闲超时
   （持续干活就不中断）；没有 → 硬超时。**区分这两种，两个问题一起解决。**

6. **判断一个值是不是秘密，标准是"拿到它能做什么"**：会话 ID 符合"持有即权限"，
   所以默认序列化要打码。泄露渠道包括日志、追踪导出、**PR 评审附件**。

7. **粗粒度的隔离杠杆一定有副作用，正确做法是用它+写清+给退出选项**：
   为了隔离 Gemini 只能改 `HOME`，副作用是它启动的 git/npm 也看到新家目录。

8. **启动外部进程时要考虑"哪些环境变量必须拿掉"**：
   多个凭证共存时的优先级是那个程序的内部细节，只留你想让它用的那一个。
   **认不出身份时取保守的并集。**

9. **脱敏放在数据流的收窄处**：单出口就放出口（第十节），
   **多出口单入口就放入口**（这里）。关键是找到那个唯一的点。
   而且**不要信任外部系统的任何字段**——连"标题"都要脱敏。

10. **已知缺陷要写在注释里**：脱敏失败时放原值"可能瞬时泄露凭证但避免崩溃"。
    **一个被记录的已知缺陷，比一个没人知道的正确假设更安全。**

11. **不能假设外部系统会发完整的状态机转移，要在自己这侧补齐终结状态**：
    重试时补 `failed`、成功但缺收尾帧时补 `completed`——
    **方向相反，因为依据不同。**

12. **发补偿事件之前先解绑回调**，否则被放弃的请求的尾随更新会落在补偿之后，
    把状态改回去。

13. **"问系统自己是什么"往往不可靠**：Gemini 报过时的模型名、Codex 拒绝报档位。
    **用带外的、协议层面的信息源。**

14. **不确定一个功能能不能用时，不要提供它**：未知提供商直接隐藏模型切换控件，
    而不是显示一个可能静默失效的按钮。

15. **拿不到数据要说"不可用"，不要记 0**：记 0 会让统计看起来正常，
    "不可用"是一个诚实的空缺。

16. **给没有真实流量覆盖的防御性代码加标注**：告诉后来人这段没被测试覆盖，
    改动时要格外小心。

## 剩下还没读的

- `openhands-agent-server/` 的具体路由实现（29851 行，只读了入口装配）
- `openhands-tools/` 剩下的工具：`browser_use`、`apply_patch`、`task_tracker`、`workflow`
- `plugin/`、`marketplace/`、`profiles/`、`secret/`、`git/`、`observability/`
- `acp_file_credentials.py`（536 行，浏览器登录型凭证的文件同步机制）

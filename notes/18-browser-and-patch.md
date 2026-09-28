# 精读 17：浏览器工具与补丁工具（browser_use 2652 行 + apply_patch 687 行）

> 位置：`upstream/software-agent-sdk/openhands-tools/openhands/tools/`
> 文件：`apply_patch/{core,definition}.py`、
> `browser_use/{definition,impl,recording,server,event_storage}.py`、`browser_use/js/*.js`
> 前置：`notes/11-tool-layer.md`（工具抽象）、`notes/13-tool-implementations.md`（file_editor 对照）

## 这一节在讲什么

两个工具，正好对应两个前面埋下的话题：

**`apply_patch`** —— 第一节 codeloop 那个"编辑格式是最大杠杆"的假设，
在这里有了第二种实现。和第十三节的 `file_editor` 正好对照。

**`browser_use`** —— 工具里最复杂的一个（2652 行 + 六个注入的 JS 文件）。
它要解决的是"让 AI 操作浏览器"，而这引出了一堆和前面完全不同的问题。

## 一、apply_patch：第二种编辑格式

### 它是别人的格式，作者明确说了

```python
"""Core logic for applying 'apply_patch' text format (OpenAI GPT-5.1 guide).

This module is an adaptation of the reference implementation from
https://github.com/openai/openai-cookbook/blob/main/examples/gpt-5/apply_patch.py
and provides pure functions and data models to parse and apply patches.

Minimal modifications were made to fit within the OpenHands SDK tool ecosystem:
- Types exposed here are used by the ApplyPatch tool executor
- **File I/O is injected via callables so the executor can enforce workspace safety**
"""
```

**这是 OpenAI 官方 cookbook 里的参考实现，移植过来的。**

**为什么要用别人的格式？** 因为 GPT-5.1 是**被训练过这个格式**的。
自己发明一种格式，模型要现学；用它训练时见过的格式，出错率低得多。

> 回想第九节那条结论："让模型产出的格式按'模型容易写对'来选"。
> **最容易写对的格式，就是它训练时见过的那个。**

改动只有一处值得注意：**文件读写被改成了注入的可调用对象**，
这样外层执行器可以强制工作区安全（下面会看到）。

> **可迁移的道理**：**移植第三方实现时，把 I/O 抽出来注入**，
> 这样你的安全策略可以包在外面，而核心逻辑还能跟着上游更新。

### 格式长什么样

```
*** Begin Patch
*** Update File: src/foo.py
@@ 上下文行
 保持不变的行
-删掉的行
+加上的行
*** End Patch
```

支持三种操作：`Add File` / `Delete File` / `Update File`，
`Update File` 还支持 `move_path`（改名）。

### 最有意思的设计：三级模糊匹配 + 分数

```python
def find_context_core(lines, context, start) -> tuple[int, int]:
    if not context:
        return start, 0

    # 第一级：完全相等
    for i in range(start, len(lines)):
        if lines[i : i + len(context)] == context:
            return i, 0                          # fuzz = 0

    # 第二级：忽略行尾空白
    for i in range(start, len(lines)):
        if [s.rstrip() for s in lines[i : i + len(context)]] == [s.rstrip() for s in context]:
            return i, 1                          # fuzz = 1

    # 第三级：忽略首尾空白（即忽略缩进）
    for i in range(start, len(lines)):
        if [s.strip() for s in lines[i : i + len(context)]] == [s.strip() for s in context]:
            return i, 100                        # fuzz = 100

    return -1, 0
```

**三级依次放宽，而且每一级返回一个"模糊分数"。**

- 完全匹配 → 0
- 行尾空白不一致 → 1
- **缩进都不一致 → 100**

分数的跨度很说明问题：**行尾多个空格是无害的笔误（1 分），
而缩进不对是危险信号（100 分）**——在 Python 里缩进就是语义，
如果模型连缩进都记错了，它可能在改一个它没看清的地方。

还有一级更狠的：

```python
def find_context(lines, context, start, eof) -> tuple[int, int]:
    if eof:
        # 先在文件末尾找
        new_index, fuzz = find_context_core(lines, context, len(lines) - len(context))
        if new_index != -1:
            return new_index, fuzz
        # 末尾找不到，退回从 start 全文找，但加 10000 分
        new_index, fuzz = find_context_core(lines, context, start)
        return new_index, fuzz + 10000
    return find_context_core(lines, context, start)
```

**补丁声明"这块在文件末尾"，但实际在别处找到了 → 加 10000 分。**

于是最终的 `fuzz` 总分被暴露在观察结果里：

```python
class ApplyPatchObservation(Observation):
    message: str = ""
    fuzz: int = 0        # number of lines of fuzz used when applying hunks
                         # (0 means exact)
    commit: Commit | None = None
```

**`fuzz` 是一个"这次修改有多不确定"的量化指标。**

> **这个设计非常值得学。** 对比第十三节 `file_editor` 的做法：
> **多次匹配就直接拒绝**（宁可大声失败）。
>
> 这里换了一个思路：**不拒绝，但把不确定性量化并暴露出来。**
>
> 两种都对，适用场景不同：
> - `str_replace` 的歧义是"改哪个"的二元问题 → 必须拒绝
> - 补丁的上下文对不齐是**程度**问题（差个空格 vs 差个缩进 vs 位置完全不对）
>   → 量化更有用
>
> **可迁移的道理**：**当"不确定"有程度差别时，量化它并暴露出来，
> 比简单的"接受/拒绝"更有信息量。** 调用方（或者监控）可以自己决定
> 多少分以上要人工复核。

### 路径逃逸检查

```python
def _resolve_path(self, p: str) -> Path:
    """Resolve a file path into the workspace, disallowing escapes."""
    pth = ((self.workspace_root / p).resolve() if not p.startswith("/")
           else Path(p).resolve())
    if not pth.is_relative_to(self.workspace_root):
        raise DiffError("Absolute or escaping paths are not allowed")
    return pth
```

**绝对路径和 `../` 逃逸都被拒绝。** 而且是先 `resolve()` 再检查——
和第十五节那个 `resolves_within` 同一个思路。

注意三个注入的 I/O 函数都走这个检查：

```python
def open_file(path): fp = self._resolve_path(path); ...
def write_file(path, content): fp = self._resolve_path(path); fp.parent.mkdir(...); ...
def remove_file(path): fp = self._resolve_path(path); fp.unlink(missing_ok=False)
```

**这就是前面"把 I/O 抽出来注入"的收益**：核心解析逻辑完全不知道有工作区这回事，
安全检查集中在三个小函数里。

> 注意 `remove_file` 用的是 `missing_ok=False`——**删一个不存在的文件要报错**，
> 而不是静默成功。因为那说明补丁和实际文件状态不一致。

### 一个针对特定模型的特殊处理

```python
# For OpenAI Responses API with GPT-5.1 models, the tool is server-known.
# Return a minimal function spec so the provider wires its own definition.
def to_responses_tool(self, ...) -> FunctionToolParam:
    """GPT-5.1 tools are known server-side. We return a minimal schema to ensure
    the model includes the canonical 'patch' argument when calling this tool."""
    return {
        "type": "function",
        "name": self.name,
        "parameters": {"type": "object",
                       "properties": {"patch": {"type": "string"}},
                       "required": ["patch"]},
        "strict": False,
    }
```

**这个工具在 OpenAI 服务端是"已知的"**，所以不用发完整的 schema——
发一个最小的就行，服务端会用它自己的定义。

**省下的是 token**：一个完整的补丁格式说明有几百字，每次请求都带着很浪费。

> **可迁移的道理**：**服务端已知的东西不要重复发送。** 但要留一个最小声明
> 确保参数名对上（注释说得很清楚：`to ensure the model includes the
> canonical 'patch' argument`）。

## 二、browser_use：一个工具集，不是一个工具

`definition.py` 里定义了**十几个独立的工具**：

```
navigate / click / type / get_state / get_content / scroll / go_back /
list_tabs / switch_tab / close_tab / get_storage / set_storage /
start_recording / stop_recording ...
```

然后用一个 `BrowserToolSet` 统一创建。回想第十一节那条：
`create()` 返回的是**列表**，注释举的例子正是 `BrowserToolSet`。

**为什么拆成这么多工具而不是一个带 `command` 参数的工具？**
对比终端工具（一个 `command` 字段）和文件编辑器（`command` 枚举 + 一堆可选字段）。

浏览器操作的参数差异太大：点击需要元素索引，输入需要索引+文本，
滚动需要方向和距离，切换标签需要标签 id。**塞进一个工具会得到一个
20 个可选字段的 schema，模型很难填对。**

> **可迁移的道理**：**参数结构差异大的操作要拆成独立工具；
> 参数结构相似的可以用一个 `command` 枚举合并。**
> 判断标准是"合并后有多少字段是互斥的可选项"。

### 截图不能脱敏

```python
class BrowserObservation(Observation):
    screenshot_data: Annotated[str | None, SkipSecretMasking()] = Field(
        default=None,
        description="Base64 screenshot data if available",
    )
```

**`SkipSecretMasking()` 这个标注很有意思。**

回想第十一节：所有工具输出都经过同一个出口做脱敏。
但截图是几十万字符的 base64 —— **对它做字符串替换是纯粹的浪费**
（而且密钥不可能以明文形式出现在 base64 编码的 PNG 里）。

于是用一个类型标注**把这个字段从脱敏流程里排除**。

> **可迁移的道理**：**全局的安全处理需要一个显式的、可审计的例外机制。**
> 用类型标注（而不是硬编码字段名列表）声明例外，
> 于是"哪些字段跳过了脱敏"是可以搜索、可以审查的。

### 查找 Chromium：三级回退，而且有取舍说明

```python
@staticmethod
@functools.cache
def check_chromium_available() -> str | None:
    # 1. 标准安装路径（优先完整的 Chrome）
    for path in _standard_chromium_paths():
        if path.exists(): return str(path)

    # 2. Playwright 装的 Chromium
    # Check Playwright-installed Chromium (preferred over PATH lookups
    # because PATH binaries like homebrew chromium may lack CDP support)
    for playwright_cache in _playwright_cache_dirs():
        ...

    # 3. PATH 里任何 chromium 系的二进制
    for binary in _path_binary_candidates():
        if path := shutil.which(binary): return path

    return None
```

**注释解释了为什么 Playwright 的优先级高于 PATH**：
**homebrew 装的 chromium 可能缺 CDP 支持。**

> **CDP（Chrome DevTools Protocol）**：远程控制 Chrome 的协议。
> 自动化工具靠它操作浏览器。

**"PATH 里找到了一个叫 chromium 的东西"不等于"它能被自动化控制"。**
这是一个只有踩过才知道的事。

`@functools.cache` 让这个查找只做一次——**文件系统探测不便宜，
而且结果在进程生命周期内不会变。**

找不到时的处理：

```python
def _ensure_chromium_available(self) -> str:
    if path := self.check_chromium_available():
        logger.info(f"Chromium is available for browser operations at {path}")
        return path
    # Chromium not available - provide clear installation instructions
    raise Exception(_get_chromium_error_message())
```

**报错信息是安装指引**，不是"找不到 chromium"。

而且这个查找还被用作能力声明：

```python
@classmethod
def is_usable(cls) -> bool:
    return BrowserToolExecutor.check_chromium_available() is not None
```

回想第十一节那条：**工具可以自己说"我在当前环境用不了"**，
于是它不会出现在给模型的工具列表里。**没装 Chromium，AI 就看不到浏览器工具。**

### 以 root 运行时的安全妥协，写得很坦白

```python
# Chromium refuses to run as root with sandboxing enabled.
# Disable the sandbox when running as root so CHROME_DOCKER_ARGS
# (--no-sandbox, --disable-setuid-sandbox, etc.) are applied.
# SECURITY: Running Chrome as root without a sandbox is risky
# - a compromised browser has full root access. Use only in
# controlled environments.
getuid = getattr(os, "getuid", None)
running_as_root = getuid is not None and getuid() == 0
if running_as_root:
    logger.warning(
        "Running as root - disabling Chromium sandbox "
        "(required for root). This reduces security isolation.")
```

**Chromium 以 root 运行时拒绝启用沙箱**（这是 Chromium 自己的策略），
所以只能关掉沙箱。

三层处理：
1. **注释里用 `SECURITY:` 标记**，说明风险（被攻破的浏览器拥有完整 root 权限）
2. **限定使用场景**（`Use only in controlled environments`）
3. **运行时打 warning**，让实际跑起来的人也能看到

而且用 `getattr(os, "getuid", None)` 而不是直接 `os.getuid()`——
**Windows 上没有这个函数。**

> **可迁移的道理**：**被迫做出的安全妥协要三处留痕**：
> 代码注释（给读代码的人）、日志警告（给运维的人）、文档限定（给决策的人）。
> 只写注释不够——跑起来的人看不到代码。

### 每个会话一个独立的浏览器配置目录

```python
"user_data_dir": str(Path.home() / ".config" / "browseruse" / "profiles" / uuid4().hex),
```

**用随机 UUID 做目录名。** 于是多个会话的 cookie、登录状态、缓存完全隔离。

> 和第十六节 ACP 那个"每会话隔离数据目录"是同一个问题的同一种解法。
> **但这里简单得多——因为 browser-use 提供了配置入口，不像 Gemini CLI 只能改 `HOME`。**

### 但浏览器实例是共享的

```python
class BrowserToolSet(ToolDefinition[...]):
    # Shared executor: reuse a single Chromium/CDP instance across parent
    # and subagents to avoid CDP port conflicts in sandbox containers.
    _shared_executor: ClassVar["BrowserToolExecutor | None"] = None
    _shared_executor_lock: ClassVar[threading.Lock] = threading.Lock()
    _shared_executor_creation_lock: ClassVar[threading.Lock] = threading.Lock()
```

**父 agent 和子 agent 共用一个 Chromium 实例**，理由是
"避免沙箱容器里的 CDP 端口冲突"。

> 对比第十三节委派工具里那个"每个执行器有自己的线程池和锁管理器，
> 防止子 agent 把父 agent 卡死"。
>
> **这里的取舍刚好相反：共享一个重资源，因为它的端口是稀缺的。**
> 不是原则不一致，而是**资源性质不同**——线程池可以有很多个，
> 一个固定的 CDP 端口只能有一个监听者。

而且**两把锁**：一把保护"创建"这个过程（防止两个线程同时创建），
一把保护字段读写。

共享带来的一个后果被明确处理了：

```python
@classmethod
def _warn_config_ignored(cls, executor_config: dict[str, object]) -> None:
    if not executor_config:
        return
    _logger.warning(
        "BrowserToolSet.create() called with executor_config but a shared executor "
        "already exists. The config %s will be ignored. **This typically happens "
        "when a subagent requests browser tools — it reuses the parent's browser "
        "session.**", list(executor_config.keys()))
```

**子 agent 想用不同的浏览器配置（比如换个 headless 设置）会被忽略，
而且会打一条说明原因的警告。**

> **可迁移的道理**：**共享资源导致某个配置被忽略时，必须明确警告并解释为什么。**
> 静默忽略会让人以为配置生效了，然后困惑于行为不对。

## 三、会话录制：注入 JS 的完整案例

这是这个工具里最"不像 agent 代码"的部分——**往页面里注入 rrweb 录制整个浏览行为。**

> **rrweb**：一个开源库，能把网页里发生的一切（DOM 变化、鼠标移动、输入）
> 录成事件流，之后可以像看视频一样回放。

### 注入的六个 JS 文件各管一件事

```
rrweb-loader.js            (60 行) 从 CDN 加载 rrweb，处理加载成功/失败
start-recording.js         (17 行)
start-recording-simple.js  (14 行)
stop-recording.js          (15 行)
wait-for-rrweb.js          (16 行) 用 Promise 等加载完成
flush-events.js            ( 6 行) 把浏览器里攒的事件送回 Python
```

**JS 被拆成独立文件而不是写在 Python 字符串里。**

好处：有语法高亮、能被 lint、能被单独测试。加载方式是模板替换：

```python
def get_rrweb_loader_js(cdn_url: str) -> str:
    """Generate the rrweb loader JavaScript with the specified CDN URL."""
    template = _load_js_file("rrweb-loader.js")
    return template.replace("{{CDN_URL}}", cdn_url)
```

> **可迁移的道理**：**要注入到别处执行的代码（JS、SQL、shell 脚本）
> 应该放在独立文件里，用占位符替换参数**，而不是在宿主语言里拼字符串。

### 加载器里的三个状态标志

```javascript
(function() {
    if (window.__rrweb_loaded) return;          // ← 幂等：重复注入直接返回
    window.__rrweb_loaded = true;

    window.__rrweb_events = window.__rrweb_events || [];
    // Flag to indicate if recording should auto-start on new pages (cross-page)
    // This is ONLY set after explicit start_recording call, not on initial load
    window.__rrweb_should_record = window.__rrweb_should_record || false;
    // Flag to track if rrweb failed to load
    window.__rrweb_load_failed = false;
```

**三个标志各有用途**，而且注释解释了最微妙的那个：

`__rrweb_should_record` 是**跨页面延续**用的。因为注入脚本是通过
`Page.addScriptToEvaluateOnNewDocument` 装的——**每打开一个新页面就会重新执行一遍**。

**于是"用户点了开始录制"这个意图必须存在某个地方**，让新页面上的加载器知道
"我应该自动接着录"。注释特别强调 `This is ONLY set after explicit
start_recording call, not on initial load` —— **初始加载时不能自动开录**。

还用了一个 Promise 做"等加载完成"：

```javascript
var resolveReady;
window.__rrweb_ready_promise = new Promise(function(resolve) {
    resolveReady = resolve;
});
```

**先创建 Promise 并把 resolve 函数存起来，等 `onload` 时再调用。**

> 这是等待外部事件的标准模式。好处是 Python 那边可以 `await` 这个 Promise，
> **而不用轮询 `window.__rrweb_ready` 这个标志。**

`onerror` 也处理了（CDN 挂了的情况），设置 `__rrweb_load_failed` 并同样 resolve。
**成功和失败都要 resolve，否则等待方会永远挂着。**

### 分级的错误处理策略，写成了模块文档

这是这一节最值得学的东西。`recording.py` 开头有一整段策略声明：

```python
"""Recording session management for browser session recording using rrweb.

Error Handling Policy
=====================
**Recording is a secondary feature that should never block primary browser
operations.** This module follows a consistent error handling strategy based on
operation type:

1. **User-facing operations** (start, stop):
   - Return descriptive error strings to the user (prefixed with "Error:")
   - Log at WARNING level for unexpected errors
   - Log at INFO level for expected failures (e.g., rrweb load failures)

2. **Internal/background operations** (flush_events, periodic flush, restart):
   - Log at DEBUG level and continue silently
   - **Never raise exceptions that would interrupt browser operations**
   - Return neutral values (0, None) on failure

3. **AttributeError for "not initialized"**:
   - Silent pass - this is expected when recording hasn't been set up
   - Used in the recording_aware decorator in impl.py

This policy ensures that recording failures are observable through logs but never
disrupt the user's primary browser workflow."""
```

**三类操作，三种处理，而且给出了判断依据。**

核心原则是第一句：**录制是次要功能，永远不该阻塞主要的浏览器操作。**

三个观察：

**① 用户主动发起的操作要返回错误给用户**（他在等结果），
**后台操作只打 DEBUG 日志**（没人在等，而且噪音会淹没真问题）。

**② 区分"意外错误"（WARNING）和"预期失败"（INFO）。**
rrweb 从 CDN 加载失败是**预期内**的——网络可能不通，这不是 bug。

> 回想第十二节那条："日志级别按'是不是我的问题'分，而不是按'严重不严重'分。"
> 这里更细一层：**还要按"是不是预期内"分。**

**③ "没初始化"的 AttributeError 静默通过**，因为录制功能默认是关的，
**"没设置过"是正常状态而不是错误。**

> **可迁移的道理**：**一个模块的错误处理策略应该写成文档而不是散落在代码里。**
> 有了这段声明，任何人加新方法时都知道该打什么级别的日志、该不该抛异常。
> 这比在每个 `except` 上写注释有效得多。

## 四、两个工具的对照

```
                  file_editor（第十三节）    apply_patch（本节）
─────────────────────────────────────────────────────────────────
格式来源           Anthropic 定义            OpenAI cookbook
一次改几个文件      一个                      多个（一个补丁含多文件）
定位方式           唯一字符串匹配             上下文行 + 行号
歧义处理           **直接拒绝**              **量化成 fuzz 分数**
不确定性可见性      错误信息里列出行号          观察结果里带 fuzz 数值
撤销               有 undo_edit              无（靠 git）
原子写入           有（临时文件+replace）      无（直接写）
```

**两个都在库里，可以按模型选。** 这正好呼应第一节 codeloop 那个未验证的假设——
**这个项目的做法不是二选一，而是把编辑格式做成可替换的，然后按模型的训练情况选。**

## 五、这一节的可迁移结论

1. **最容易让模型写对的格式，是它训练时见过的那个**：移植 OpenAI cookbook 的
   `apply_patch` 格式给 GPT-5.1 用，而不是自己发明一种。

2. **移植第三方实现时把 I/O 抽出来注入**：核心逻辑跟着上游更新，
   安全检查包在外面（三个小函数统一做路径逃逸检查）。

3. **当"不确定"有程度差别时，量化它并暴露出来**：
   行尾空白 1 分、缩进不一致 100 分、位置完全不对 10000 分。
   比简单的"接受/拒绝"更有信息量，调用方可以自己定阈值。
   **而"改哪个"这种二元歧义（`str_replace`）就该直接拒绝。**

4. **删不存在的文件要报错**（`missing_ok=False`），因为那说明补丁和实际状态不一致。

5. **服务端已知的工具不要重复发完整 schema**，但要留一个最小声明确保参数名对上。

6. **参数结构差异大的操作拆成独立工具**；参数结构相似的用一个 `command` 枚举合并。
   判断标准是"合并后有多少字段是互斥的可选项"。

7. **全局的安全处理需要一个显式、可审计的例外机制**：用类型标注
   （`SkipSecretMasking()`）而不是硬编码字段名列表，于是例外可搜索、可审查。

8. **"PATH 里有个同名二进制"不等于"它能用"**：homebrew 的 chromium 可能缺 CDP 支持，
   所以 Playwright 装的优先级更高。**探测结果要缓存**（文件系统探测不便宜）。

9. **找不到依赖时报错信息应该是安装指引**，而不是"找不到 X"。
   并且把探测结果接到 `is_usable()` 上——**装不了就别让模型看到这个工具。**

10. **被迫做出的安全妥协要三处留痕**：代码注释（`SECURITY:` 标记）、
    运行时日志警告、文档限定使用场景。**只写注释不够——跑起来的人看不到代码。**

11. **共享重资源和隔离轻资源要分开判断**：浏览器配置目录每会话独立（轻），
    Chromium 实例父子共享（重，且端口稀缺）。**资源性质决定策略，不是原则不一致。**

12. **共享资源导致配置被忽略时必须警告并解释原因**：
    子 agent 传的浏览器配置会被忽略，日志里说清"因为复用了父会话"。

13. **要注入到别处执行的代码放在独立文件里，用占位符替换参数**：
    六个 JS 文件各管一件事，有语法高亮、能 lint、能单测。

14. **注入到"每个新页面都执行"的脚本必须幂等**，而且**跨页面的意图需要显式的状态标志**
    （"用户点过开始录制"必须存住，新页面的加载器才知道要自动接着录）。

15. **等待外部事件用 Promise 而不是轮询标志**，而且**成功和失败都要 resolve**，
    否则等待方永远挂着。

16. **一个模块的错误处理策略应该写成文档**：按"用户发起 / 后台操作 / 未初始化"
    分三类，各自规定返回什么、打什么级别的日志、能不能抛异常。
    **比在每个 except 上写注释有效得多。**

17. **日志级别除了按"是不是我的问题"分，还要按"是不是预期内"分**：
    CDN 加载失败是预期内的（INFO），不是 bug（WARNING）。

## 剩下还没读的

- `openhands-agent-server/` 的具体路由实现（29851 行，只读了入口装配）
- `openhands-tools/` 剩下的：`task_tracker`、`workflow`、`ask_oracle`、`tom_consult`、
  `planning_file_editor`、`gemini/`（对齐 Gemini CLI 工具语义的一整套）
- `plugin/`、`marketplace/`、`profiles/`、`secret/`、`git/`、`observability/`
- `acp_file_credentials.py`（536 行，浏览器登录型凭证的文件同步机制）
- `browser_use/server.py`（340 行）和 `event_storage.py`（68 行）的细节

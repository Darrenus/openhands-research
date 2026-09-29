# 精读 18：密钥管理与 git 集成（secret/ 209 行 + git/ 1845 行 + redact/mask 823 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/`
> 文件：`secret/secrets.py`、`conversation/secret_registry.py`、`utils/redact.py`、
> `utils/masking.py`、`utils/command.py`、`git/{utils,cached_repo,git_changes,git_diff,git_commits}.py`
> 前置：前面几乎每一节都引用过脱敏，这一节是它的本体

## 这一节在讲什么

"脱敏"这个词在前面十七节里出现了**七次**：工具输出的统一出口（第十）、
错误信息只带参数名（第三）、422 响应（第十二）、tmux 第三方库日志（第十三）、
ACP 摄入时脱敏（第十六）、截图跳过脱敏（第十七）、会话 ID 当秘密（第十六）。

**这一节读它的本体**，以及一个一直被当作黑盒的东西：`git/`——
"产出补丁给评审员"那条链路的起点。

## 一、密钥的三种形态

### 形态 1：静态值

```python
class StaticSecret(SecretSource):
    """A secret stored locally"""
    value: SecretStr | None = None

    def get_value(self) -> str | None:
        return self.value.get_secret_value() if self.value is not None else None
```

### 形态 2：从 URL 取

```python
class LookupSecret(SecretSource):
    """A secret looked up from some external url"""
    url: str
    headers: dict[str, str] = Field(default_factory=dict)

    def get_value(self) -> str:
        local_value = _resolve_secret_locally(self.url)
        if local_value is not None:
            return local_value
        response = httpx.get(self.url, headers=self.headers, timeout=30.0)
        response.raise_for_status()
        return response.text
```

**密钥不存在本地，每次用的时候去一个 URL 取。** 好处是密钥轮换不需要改配置，
而且**持久化的会话状态里存的是一个 URL，不是密钥本身**。

### 一个自指的死锁，以及它的解法

```python
def register_local_secret_resolver(resolver: LocalSecretResolver) -> None:
    """Register an in-process resolver for ``LookupSecret`` URLs.

    A process that both serves and consumes ``LookupSecret`` URLs — an
    agent-server running an in-process conversation, say — **cannot fetch them
    over HTTP from its own event loop: the request can only be answered by the
    loop that is blocked waiting for it, so it stalls until the client times
    out.** Registering a resolver lets such a process answer its own URLs
    directly and skip the round trip."""
```

**问题**：agent-server 自己提供密钥查询接口，同时自己又要用密钥。
它在事件循环里发一个 HTTP 请求给自己——**而能回答这个请求的，正是那个被阻塞在等待它的循环。**

> 这就是第十二节那个注释提到的同一个问题（`a loopback fetch made from the event loop
> cannot be served by the loop blocked on it`）。**自己请求自己 = 死锁。**

**解法**：注册一个进程内解析器，命中就直接返回，不走网络。

```python
def _resolve_secret_locally(url: str) -> str | None:
    for resolver in list(_local_secret_resolvers):
        try:
            value = resolver(url)
        except Exception:
            logger.warning("Local secret resolver raised; falling back to HTTP",
                           exc_info=True)
            continue
        if value is not None:
            return value
    return None
```

**注意三个细节**：遍历的是 `list(...)` 的拷贝（防止遍历中被修改）、
**解析器抛异常就跳过并回退到 HTTP**（不让一个坏的解析器堵死整条路）、
返回 `None` 表示"我不管这个 URL"，继续下一个。

> **可迁移的道理**：**当一个进程同时是某个服务的提供方和消费方时，
> 必须有一条"绕过网络直接问自己"的短路径。** 否则在单进程部署下会死锁。

### 形态 3：URL 里的凭证占位符

```python
def redact_url_credentials(url: str, *, preserve_placeholders: bool = False) -> str:
    """With ``preserve_placeholders``, a userinfo holding a ``${VAR}`` reference is
    left intact: **it is expanded from the secret registry at fetch time and is not
    itself a secret, so masking it would break the clone.** Inline credentials are
    still masked.

    Examples:
        >>> redact_url_credentials("https://oauth2:SECRET@gitlab.com/repo.git")
        'https://****@gitlab.com/repo.git'
        >>> redact_url_credentials("git@github.com:owner/repo.git")
        'git@github.com:owner/repo.git'
        >>> redact_url_credentials("https://x-token-auth:${TOKEN}@host/r.git",
        ...                        preserve_placeholders=True)
        'https://x-token-auth:${TOKEN}@host/r.git'
    """
```

**`${TOKEN}` 这种占位符不是秘密，打码它反而会让克隆失败。**

而 SSH 形式的 `git@github.com:...` 里那个 `git@` 也不是凭证——**原样保留。**

> **可迁移的道理**：**脱敏函数必须区分"真的秘密"、"指向秘密的引用"、
> 和"长得像但不是"三种情况。** 一刀切地按 `@` 前面全打码，会同时打坏后两种。

## 二、SecretRegistry：注入和脱敏是同一个机制的两面

```python
class SecretRegistry(OpenHandsModel):
    """Manages secrets and injects them into bash commands when needed.

    ... When a bash command is about to be executed, it scans the command for any
    secret keys and injects the corresponding environment variables.

    Additionally, **it tracks the latest exported values to enable consistent masking
    even when callable secrets fail on subsequent calls.**"""

    secret_sources: dict[str, SecretSource] = Field(default_factory=dict)
    _exported_values: dict[str, str] = PrivateAttr(default_factory=dict)
    _exported_values_lock: RLock = PrivateAttr(default_factory=RLock)
    _failed_lookups: dict[str, float] = PrivateAttr(default_factory=dict)
```

### 注入：命令里提到了密钥名，就把它当环境变量传进去

```python
def get_secrets_as_env_vars(self, command: str) -> dict[str, str]:
    found_secrets = self.find_secrets_in_text(command)
    if not found_secrets:
        return {}
    env_vars = {}
    for key in found_secrets:
        try:
            value = self.secret_sources[key].get_value()
            if value:
                env_vars[key] = value
                self.track_exported_values({key: value})   # ← 记下来，为了后面脱敏
        except Exception as e:
            logger.error(f"Failed to retrieve secret for key '{key}': {e}")
            continue
```

```python
def find_secrets_in_text(self, text: str) -> set[str]:
    found_keys = set()
    for key in self.secret_sources.keys():
        if key.lower() in text.lower():      # ← 大小写不敏感的子串匹配
            found_keys.add(key)
    return found_keys
```

**AI 写 `curl -H "Authorization: Bearer $GITHUB_TOKEN" ...`，
系统检测到命令里提到了 `GITHUB_TOKEN` 这个名字，于是把真实值作为环境变量注入。**

**AI 从来不知道密钥的值**——它只知道名字。这是很重要的隔离：
即使 AI 被诱导要泄露密钥，它手上也只有一个变量名。

> **可迁移的道理**：**让 AI 引用秘密而不是持有秘密。** 它写变量名，
> 运行时才绑定真实值。这样"泄露"的上限就是泄露一个名字。

### 脱敏：注入过什么就记住什么

```python
def track_exported_values(self, values: Mapping[str, str]) -> None:
    """Track values for output masking."""
    with self._exported_values_lock:
        self._exported_values.update({k: v for k, v in values.items() if v})
```

**"注入"这个动作顺便建立了"该脱敏什么"的清单。**

类文档解释了为什么要单独记而不是每次重新取：
**`to enable consistent masking even when callable secrets fail on subsequent calls`**
——动态密钥源第二次调用可能失败，但**第一次注入的那个值已经在输出里了，必须继续脱敏**。

> **这个设计很精妙**：注入和脱敏共用一份状态，于是**不可能出现"注入了但没脱敏"**。
> 对比一个错误的设计：脱敏时重新去密钥源取值来做匹配——那么源一挂，脱敏就失效了，
> **而已经泄露的值还在输出里。**

还有一个失败退避：

```python
# Back-off before retrying a failed source; a failed lookup masks nothing anyway.
FAILED_LOOKUP_RETRY_SECONDS: Final[float] = 60.0
_failed_lookups: dict[str, float] = PrivateAttr(default_factory=dict)
```

注释那半句很妙：**`a failed lookup masks nothing anyway`**——
取不到值的密钥源，反正也没法用它来脱敏，所以退避 60 秒完全不损失安全性。

## 三、递归脱敏里的四个坑

`_mask_value` 是一个递归函数，要走遍任意嵌套的数据结构。
**每个 `case` 都在处理一个坑。**

### 坑 1：枚举被脱敏会退化成字符串

```python
case Enum():
    # A str-subclass enum is a str, but masking it would downgrade the member to a
    # plain str and break the field's serialization. **Its vocabulary is fixed, so
    # it can never hold a secret anyway.**
    return value
```

`class Foo(str, Enum)` 这种枚举**同时是字符串**，所以会被 `case str()` 捕获。
但对它做替换会得到一个普通字符串，**枚举身份丢了，序列化就坏了。**

而且给了不需要处理的理由：**枚举的取值范围是固定的，不可能含密钥。**

### 坑 2：具名元组被脱敏会退化成普通元组

```python
case tuple():
    items = [...]
    # Rebuild a NamedTuple through its own constructor; tuple(items) would
    # downgrade it the same way masking an Enum member does.
    return type(value)(*items) if hasattr(value, "_fields") else tuple(items)
```

**用 `hasattr(value, "_fields")` 判断是不是具名元组，然后用它自己的构造器重建。**

> 两个坑是同一类问题：**Python 里很多"带额外语义的类型"是内置类型的子类，
> 对它们做值变换时会丢掉那层语义。** 递归遍历数据结构时必须逐类处理。

### 坑 3：data URL 不能被打断

```python
case str():
    if skip_secret_masking:
        return value
    if preserve_data_urls and value.startswith("data:") and ";base64," in value:
        return value
    return mask(value)
```

配套的类型标注在 `utils/masking.py`（只有 13 行）：

```python
SkipSecretMasking      # 整个字段跳过脱敏（第十七节截图用的那个）
PreserveDataUrls       # 内嵌的 base64 data URL 保持原样
```

**base64 编码的图片里可能碰巧出现和密钥相同的字节序列**，
替换掉会让图片损坏。而 base64 里不可能有明文密钥，所以跳过是安全的。

### 坑 4：用 `model_copy` 而不是重新校验

```python
def _mask_model[ModelT: BaseModel](model: ModelT, mask) -> ModelT:
    """Rebuild ``model`` with every nested string masked.

    **``model_copy`` is used rather than a validate round-trip so private
    attributes survive and no field is re-coerced.**"""
    updates = {}
    for name, field in type(model).model_fields.items():
        metadata = field.metadata
        skip_secret_masking = any(isinstance(item, SkipSecretMasking) for item in metadata)
        preserve_data_urls = any(isinstance(item, PreserveDataUrls) for item in metadata)
        updates[name] = _mask_value(getattr(model, name), mask, ...)
    return model.model_copy(update=updates)
```

**走一遍"序列化再反序列化"会丢掉私有属性，还会把字段重新类型转换一遍。**
用 `model_copy(update=...)` 只替换值。

注意标注是**从字段的 `metadata` 里读的**——这就是第十七节那个
`Annotated[str | None, SkipSecretMasking()]` 起作用的地方。
**例外声明在字段上，处理在这一个函数里。**

## 四、流式输出的脱敏：整个模块最漂亮的一段

这是我在整个项目里看到最精巧的一个小算法。

```python
class StreamOutputMask:
    """Masks text that arrives in pieces.

    **A per-chunk masker cannot see a secret split across chunk boundaries**, so
    this holds back the shortest suffix that could still grow into one: **the
    longest registered value minus one character.**"""
```

### 问题

模型是**流式输出**的，一个字一个字往外吐。假设密钥是 `sk-abc123`，
而输出被切成了 `...sk-ab` 和 `c123...` 两块。

**逐块脱敏的话，两块各自都不包含 `sk-abc123`，于是全都通过——密钥泄露了。**

### 解法

```python
def __init__(self, pattern: re.Pattern[str] | None, max_len: int) -> None:
    self._pattern = pattern
    self._max_len = max_len      # ← 最长密钥的长度
    self._held = ""

def feed(self, text: str) -> str:
    """Return the masked text that is safe to release now."""
    self._held += text
    return self._release(pattern, max(len(self._held) - self._max_len + 1, 0))
```

**始终扣住最后 `max_len - 1` 个字符不放出去。**

为什么是这个数？因为**任何跨界的密钥，至少有 1 个字符落在新来的块里**，
所以它在旧数据里最多占 `max_len - 1` 个字符。**扣住这么多，
就保证任何可能的跨界密钥都还完整地留在缓冲区里。**

释放逻辑：

```python
def _release(self, pattern, cut: int) -> str:
    out, pos, end = [], 0, cut
    for match in pattern.finditer(self._held):
        # A match starting before the cut is already maximal: the cut leaves
        # max_len-1 characters of lookahead behind it.
        if match.start() >= cut:
            break
        out.append(self._held[pos : match.start()])
        out.append("<secret-hidden>")
        pos = match.end()
        end = max(end, match.end())      # ← 匹配可能跨过切点
    out.append(self._held[pos:end])
    self._held = self._held[end:]
    return "".join(out)
```

**注释那句是关键**：一个开始位置在切点之前的匹配**已经是完整的**，
因为切点后面还留着 `max_len - 1` 个字符的前瞻空间——足够让任何密钥匹配完。

而 `end = max(end, match.end())` 处理的是**匹配跨过切点**的情况：
这时要把释放边界推到匹配结束处，否则会把一个已经被替换的密钥的后半截又放出去。

```python
def flush(self) -> str:
    """Return what is still held back; no more input is coming."""
    return self._release(pattern, len(self._held))
```

**流结束时把扣住的全部放出来**（此时 `cut = 全长`，不再需要留前瞻）。

> **这是"流式处理 + 跨块模式匹配"的标准解法**，同样适用于：
> 流式 JSON 解析、日志里的敏感信息过滤、网络流量的关键词检测。
>
> **核心思路：扣住"最长可能模式长度 - 1"的尾巴，等下一块来了再一起判断。**

## 五、按键名猜秘密：一份务实的清单

```python
SECRET_KEY_PATTERNS = frozenset({
    "AUTHORIZATION", "COOKIE", "CREDENTIAL", "KEY",
    "PASSWORD", "SECRET", "SESSION", "TOKEN",
})

def is_secret_key(key: str) -> bool:
    """Performs case-insensitive substring matching against known secret key patterns."""
    return any(pattern in key.upper() for pattern in SECRET_KEY_PATTERNS)
```

**八个词，大小写不敏感的子串匹配。** 于是 `api_key`、`X-Token`、
`SESSION_ID`、`db_password` 全部命中。

`KEY` 这一条会有误报（`primary_key`、`sort_key`），但**这是有意的**——
在"漏一个密钥"和"多打码一个无害字段"之间，选后者。

### 还有一类键要"整棵子树全打码"

```python
# Keys that should have ALL nested values redacted (not just detected secret keys).
# These typically contain environment variables or headers that may include secrets.
REDACT_ALL_VALUES_KEYS = frozenset({"environment", "env", "headers"})
```

**因为环境变量和 HTTP 头的键名是任意的**——你没法预先知道
`MY_COMPANY_INTERNAL_AUTH_THING` 是不是密钥。**整棵子树全打码，只保留键名结构。**

```python
def _redact_all_values(value: Any) -> Any:
    """Recursively redact all values while preserving structure (key names)."""
    if isinstance(value, Mapping):
        return {k: _redact_all_values(v) for k, v in value.items()}
    if isinstance(value, list):
        return [_redact_all_values(item) for item in value]
    return "<redacted>"
```

**保留键名结构而不是整块删掉**——调试时还能看到"有哪些变量"，只是看不到值。

### URL 查询参数有单独一份清单

```python
# Specific URL query parameter names (lowercased) that should always be redacted,
# in addition to any parameter matching SECRET_KEY_PATTERNS via is_secret_key().
SENSITIVE_URL_PARAMS = frozenset({
    "tavilyapikey", "apikey", "api_key", "token", "access_token", "secret", "key",
})
```

**`tavilyapikey` 这种具体的供应商参数名被硬编码进来了。** 说明这是真实遇到的泄露。

### 还有一层：按字面量特征识别

```python
def redact_api_key_literals(text: str) -> str:
    """Replace bare API key literals from common providers with ``<redacted>``.

    Matches known key prefixes (OpenAI, Anthropic, OpenRouter, GROQ, HuggingFace,
    Together AI, GitHub, Sentry, Linear, Tavily, Slack, OpenHands session tokens,
    etc.) anywhere in the text."""
    return _API_KEY_LITERAL_RE.sub("<redacted>", text)
```

**不靠键名，靠密钥本身的前缀特征**（`sk-`、`ghp_`、`xoxb-` 这类）。

这就是第十三节那个 tmux 日志过滤器用的函数——**在没有键名上下文的纯文本里，
只能靠字面量特征。**

> **三层防护各管一段**：
> 1. `SecretRegistry` —— 知道确切值，做精确替换（最可靠）
> 2. `is_secret_key` / `sanitize_dict` —— 有键名，按名字猜
> 3. `redact_api_key_literals` —— 只有纯文本，按前缀特征猜
>
> **可迁移的道理**：**脱敏的可靠性取决于你有多少上下文。** 设计时要问：
> 这个位置我知道确切的秘密值吗？知道键名吗？什么都不知道？**三种情况用三种策略。**

## 六、给 AI 的环境要剥掉密钥

```python
def sanitized_env(env: Mapping[str, str] | None = None) -> dict[str, str]:
    """... **Sensitive environment variables (e.g., ``SESSION_API_KEY``) are stripped
    to prevent LLM-driven agents from accessing credentials via terminal commands.**

    ``AI_AGENT`` defaults to ``openhands`` so downstream tools can select
    agent-friendly output without relying on product-specific heuristics."""
    for key in _SENSITIVE_ENV_VARS:
        base_env.pop(key, None)
```

**防的是：AI 执行 `env` 或 `echo $SESSION_API_KEY` 就能拿到服务器自己的密钥。**

注意**和上面的注入机制配合**：服务器自己的密钥被剥掉，
而用户显式配置的密钥**只在命令提到它的名字时才注入**。**两个方向都管住了。**

还有一个和脱敏无关但有意思的：

```python
"""PyInstaller-based binaries rewrite ``LD_LIBRARY_PATH`` so their vendored libraries
win. This function restores the original value so that subprocess will not use them."""
```

**打包成单文件二进制的程序会改 `LD_LIBRARY_PATH`**，导致它启动的子进程
去加载打包进来的库而不是系统库。这里恢复原值。

> 又一个"只有真的打包发布过才知道"的细节。

最后 `AI_AGENT=openhands` 这个约定很实用：**让下游工具能识别"我在被 AI 调用"**，
从而输出更适合机器读的格式，而不用靠嗅探各种产品特征。

## 七、git 集成：四个务实的决定

### 决定 1：永远不用 shell

```python
def run_git_command(args: list[str], cwd=None, timeout: int = 30, *,
                    expected_failure: bool = False) -> str:
    """Run a git command safely **without shell injection vulnerabilities**.

    Args:
        args: List of command arguments (e.g., ['git', 'status', '--porcelain'])
        expected_failure: Log a non-zero exit at debug level when the caller
            intentionally probes for a fallback condition."""
    redacted_args = [redact_url_credentials(a) for a in args]
    cmd_str = shlex.join(redacted_args)
    ...
    if result.returncode != 0:
        error_msg = f"Git command failed: {cmd_str}"
        # stderr can echo the remote URL (with embedded credentials on some ...
```

**参数列表形式，不经过 shell**——分支名里有 `;` 也不会变成命令注入。

而且**日志里的参数先脱敏**（远程 URL 可能带凭证），错误信息里的 stderr 也要脱敏
（注释说 `stderr can echo the remote URL`）。

`expected_failure` 这个参数很贴心：**有些调用是故意去"试探"某个条件的**
（比如"这个分支存在吗"），失败是预期内的，不该打 error 日志。

> 回想第十七节那条"日志级别要按'是不是预期内'分"——**同一个原则，
> 这里做成了一个显式参数。**

### 决定 2：找仓库根不用 git 命令

```python
def get_closest_git_repo(path: Path) -> Path | None:
    while True:
        git_path = current_path / ".git"
        if git_path.exists():  # Could be file (worktree) or directory
            return current_path
        parent = current_path.parent
        if parent == current_path:  # Reached filesystem root
            return None
        current_path = parent
```

**和第十五节技能加载里那个 `_find_git_repo_root` 是同一个实现思路**
（都注意了 `.git` 可能是文件）。**两处独立实现同一个逻辑，都避开了调用 git。**

### 决定 3：比较基准的选择分两种用途

这是 `git/` 里最有信息量的一段：

```python
def get_valid_ref(repo_dir, override: str | None = None, *,
                  purpose: Literal["export", "display"] = "export") -> str | None:
    """Get a valid git reference to compare against.

    ... Otherwise, auto-detects a reference according to ``purpose``. For
    ``purpose="export"`` (the default), tries multiple strategies:
    1. Current branch's origin (e.g., origin/main)
    2. Default branch (e.g., origin/main, origin/master)
    3. Merge base with default branch
    4. Empty tree (for new repositories)

    Args:
        purpose: What the ref is used for. ``"export"`` (default) **only picks
            bases that are reconstructable from a fresh clone — required by
            workspace export/restore, whose patches are replayed elsewhere.**
            ``"display"`` optimizes for showing a conversation's work: it skips
            a fully-pushed branch's own upstream, prefers the fork point over
            the default-branch tip, and degrades to ``HEAD`` (git status-style)
            instead of the empty tree for repos without a usable origin."""
```

**"这次改了什么"这个问题有两个不同的答案，取决于你要拿它干什么：**

- **导出用**（`export`）：基准必须是**在别处重新克隆也能找到的东西**——
  因为补丁要在另一台机器上重放。所以只能选远程分支、合并基点、空树。
- **展示用**（`display`）：要让人看清"这次对话干了什么"。
  所以跳过已经完全推送的分支的上游（那会显示"没有改动"）、
  优先用分叉点、没有可用远程时退化成 `git status` 那样和 `HEAD` 比。

> **这个区分非常值得学。** 同一个技术操作（算 diff），
> **用途不同，正确的基准就不同。** 很多系统只有一套 diff 逻辑，
> 结果要么导出的补丁在别处应用不了，要么界面上显示"无改动"让用户困惑。
>
> **可迁移的道理**：**当一个计算的结果要被不同的消费者使用时，
> 先问清每个消费者的约束，可能需要不同的参数而不是一套通用逻辑。**

### 决定 4：新仓库的边界情况

```python
"""The ``"HEAD"`` override is treated specially: **if it does not resolve (no commits
on the current branch — e.g. a freshly ``git init``'d workspace, or an orphan branch
in a repo that has commits elsewhere), we fall back to the empty-tree hash so callers
see untracked files as additions** instead of an opaque ``rev-parse --verify``
failure. **Other overrides that do not resolve still raise ``GitCommandError`` so a
typo'd branch/SHA is not silently swallowed.**"""
```

**`HEAD` 解析失败是有意义的情况**（刚 `git init`、还没有任何提交），
降级成"和空树比"，于是所有文件都显示为新增——**这正是用户想看到的。**

**但其他引用解析失败仍然报错**——因为那说明用户打错了分支名或提交号，
静默降级会让他以为自己在和某个分支比较。

> **这是"降级"的正确用法**：**只对有明确语义的失败降级，
> 其他的照样报错。** 和第十五节那个"默认清单读不到才降级，
> 用户指定的读不到就报错"是同一个判断。

### 顺带：改动的分类

```python
status_mapping = {
    "M": GitChangeStatus.UPDATED,
    "A": GitChangeStatus.ADDED,
    "D": GitChangeStatus.DELETED,
    "U": GitChangeStatus.UPDATED,  # Unmerged files are treated as updated
}
if status not in status_mapping:
    raise ValueError(f"Unknown git status: {status}")
```

**认不出的状态码直接抛异常**，而不是归到某个默认类别。

而重命名的处理：

```python
"""Renames are split into DELETED (old path) + ADDED (new path); copies surface only
the new path as ADDED."""
```

**改名拆成"删旧的 + 加新的"**，因为下游（比如评审员）只关心"哪些路径受影响"。

解析时还处理了 git 输出格式的三种变体：

```python
# Depending on git config, format can be either:
# * "A file.txt"
# * "A       file.txt"
# * "R100    old_file.txt    new_file.txt" (rename with similarity percentage)
```

**同一个命令在不同 git 配置下输出格式不同**——这类"外部工具输出不稳定"
的问题，注释把三种形态都列出来了。

### 还有一个大小限制

```python
MAX_FILE_SIZE_FOR_GIT_DIFF = 1024 * 1024  # 1 Mb
```

**超过 1MB 的文件不算 diff。** 因为那大概是构建产物、锁文件、二进制——
diff 出来几万行，对模型毫无价值还很贵。

## 八、这一节的可迁移结论

1. **让 AI 引用秘密而不是持有秘密**：它写变量名，运行时才绑定真实值。
   **"泄露"的上限就是泄露一个名字。**

2. **注入和脱敏共用一份状态**：`track_exported_values` 让"注入过什么"
   直接成为"该脱敏什么"的清单。**于是不可能出现"注入了但没脱敏"。**
   反面设计（脱敏时重新取值来匹配）在密钥源失效时会让脱敏静默失效。

3. **当一个进程同时是某服务的提供方和消费方时，必须有"绕过网络直接问自己"的短路径**，
   否则单进程部署会死锁。而且这个短路径本身要容错（解析器抛异常就回退 HTTP）。

4. **脱敏函数要区分"真秘密"、"指向秘密的引用"、"长得像但不是"**：
   `${TOKEN}` 占位符打码会让克隆失败，SSH URL 的 `git@` 不是凭证。

5. **递归遍历数据结构做值变换时，要逐类处理"内置类型的子类"**：
   枚举和具名元组被替换后会退化，丢掉那层语义。

6. **全局处理需要显式、可审计的例外机制**：`SkipSecretMasking` /
   `PreserveDataUrls` 标注在字段上，处理集中在一个函数里。

7. **流式处理里跨块的模式匹配：扣住"最长可能模式长度 - 1"的尾巴**。
   这个数保证任何跨界匹配都还完整地留在缓冲区。
   同样适用于流式 JSON 解析、日志过滤、流量检测。

8. **脱敏的可靠性取决于你有多少上下文**：知道确切值 → 精确替换；
   知道键名 → 按名字猜；只有纯文本 → 按字面量前缀猜。**三种情况三种策略。**

9. **键名不可预知的地方（环境变量、HTTP 头）整棵子树全打码，但保留键名结构**：
   调试时还能看到"有哪些变量"。

10. **在"漏一个密钥"和"多打码一个无害字段"之间选后者**：
    `KEY` 这个模式会误报 `primary_key`，这是有意的。

11. **给子进程的环境要剥掉自己的密钥**：防止 AI 用 `env` 拿到服务器凭证。
    **和"只在命令提到名字时才注入"配合，两个方向都管住。**

12. **执行外部命令永远用参数列表，不用 shell**，而且**日志和错误信息里的参数要先脱敏**
    （远程 URL 可能带凭证）。

13. **"故意试探某个条件"的调用要能声明"失败是预期的"**：
    做成一个显式参数（`expected_failure`），而不是让调用方自己压日志。

14. **同一个计算给不同消费者用时，可能需要不同的参数**：
    导出用的 diff 基准必须"在别处能重建"，展示用的基准要"让人看清干了什么"。
    **一套通用逻辑会两边都不对。**

15. **只对有明确语义的失败降级，其他照样报错**：`HEAD` 解析不了是"新仓库"
    （降级成空树），其他引用解析不了是"打错了"（报错）。

16. **认不出的外部状态码直接抛异常**，不要归到默认类别。

17. **外部工具的输出格式可能因配置而异**：把见过的所有形态在注释里列出来。

18. **给模型看的 diff 要有大小上限**：1MB 以上大概是构建产物或二进制，
    diff 出来几万行毫无价值还很贵。

## 剩下还没读的

- `openhands-agent-server/` 的具体路由实现（29851 行，只读了入口装配）
- `openhands-tools/` 剩下的：`task_tracker`、`workflow`、`ask_oracle`、`tom_consult`、
  `planning_file_editor`、`gemini/`
- `plugin/`、`marketplace/`、`profiles/`、`observability/`
- `acp_file_credentials.py`（536 行）
- `git/cached_repo.py` 的文件锁细节、`git_commits.py`

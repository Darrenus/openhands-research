# 精读 12：工具的具体实现（openhands-tools/，16839 行）

> 位置：`upstream/software-agent-sdk/openhands-tools/openhands/tools/`
> 文件：`terminal/{definition,metadata,terminal/tmux_terminal}.py`、
> `file_editor/{definition,impl,editor}.py`、`delegate/{definition,impl}.py`
> 前置：`notes/11-tool-layer.md`（工具的抽象层）、`notes/05-run-loop.md`（会话循环）

## 这一节在讲什么

前面十一节讲的都是**抽象**。这一节看三个最重要的具体工具是怎么实现的——
**抽象的价值最终要在这里兑现。**

三个工具，三类不同的难题：

| 工具 | 难题 |
|---|---|
| 终端 | 怎么让一条条独立的命令**共享状态**（`cd` 之后下一条还在那个目录） |
| 文件编辑器 | 怎么**不改错地方**，以及怎么保证写文件不会写坏 |
| 委派 | 怎么让一个 agent 派出子 agent，而不搞乱资源和权限 |

## 一、终端工具：怎么保持状态

### 问题

如果每条命令都用 `subprocess.run()` 单独跑一次，那么：

```bash
cd /tmp          # 第一条命令
ls               # 第二条 —— 回到原来的目录了，cd 白做了
export X=1       # 第三条
echo $X          # 第四条 —— 空的
```

**每条命令都是一个新进程，状态不共享。** 但人用终端的直觉完全不是这样。

### 解法：tmux

> **tmux**：终端会话管理工具。它在后台维持一个**真正的 shell 进程**，你可以往里"打字"、
> 也可以"看屏幕"。关掉终端窗口，那个 shell 还活着。

于是 agent 的每条命令都是**往同一个活着的 shell 里打字**，状态自然就保持了。

> **⚠️ 实跑说明（见 `notes/26-verification-run.md`）**：**tmux 是可选的。**
> 机器上没装 tmux 时会回退到 `SubprocessTerminal`，并打警告 + 给出安装指引
> （`brew install tmux`）。本节讲的 PS1 元数据技巧**只在 tmux 后端上成立**——
> 验证那次因为机器没装 tmux，这段机制一次都没被执行过。

工具的动作定义也因此比想象中复杂：

```python
class TerminalAction(Action):
    command: str          # 要执行的命令，或者一个按键名
    is_input: bool        # True = 这是给正在运行的程序的输入，不是新命令
    timeout: float | None
    reset: bool           # 重开一个会话（会丢掉所有状态）
```

`is_input` 和按键支持是关键：

```
特殊按键：C-c（Ctrl+C）、C-d（EOF）、C-z、任意 C-<字母>
导航键：UP、DOWN、LEFT、RIGHT、HOME、END、PGUP、PGDN
其他：TAB、ESC、BS、ENTER
```

**为什么需要这些？** 因为命令会变成交互式的：`pip install` 问你 yes/no、
`git rebase -i` 打开编辑器、某个脚本卡住了要 Ctrl+C。**AI 必须能像人一样按键。**

`reset` 的描述很诚实：

> Use this only when the terminal becomes unresponsive. Note that all previously set
> environment variables and session state will be lost after reset.

**明确告诉 AI 这是最后手段，而且会丢状态。**

### 最巧妙的一招：把元数据藏在命令提示符里

这是整个终端工具最聪明的设计。

**问题**：往 tmux 里打一条命令，然后"看屏幕"。屏幕上是一堆文本。
**你怎么知道命令的退出码是多少？当前在哪个目录？命令跑完了没有？**

**解法**：把这些信息**塞进 shell 的提示符**里。

```python
@classmethod
def to_ps1_prompt(cls) -> str:
    """Convert the required metadata into a PS1 prompt."""
    prompt = CMD_OUTPUT_PS1_BEGIN
    json_str = json.dumps({
        "pid": "$!",
        "exit_code": "$?",
        "username": r"\u",
        "hostname": r"\h",
        "working_dir": r"$(pwd)",
        "py_interpreter_path": r'$(command -v python || echo "")',
    }, indent=2)
    prompt += json_str.replace('"', r"\"")
    prompt += CMD_OUTPUT_PS1_END + "\n"
    return prompt
```

> **PS1**：shell 的提示符模板（就是你看到的 `user@host:~$` 那一串）。
> 里面可以嵌命令和变量，每次显示提示符时都会重新求值。

**于是每条命令执行完，shell 自动打印出一段带标记的 JSON**，里面有退出码、进程号、
当前目录、Python 解释器路径。程序只要在屏幕文本里找那个标记，就能解析出全部状态。

**这个技巧的妙处**：
- 不需要额外的通信通道
- 不需要修改被执行的命令
- **"命令跑完了没有"这件事也顺便解决了**——出现新的提示符块就说明跑完了

解析时还做了防御：

```python
@classmethod
def matches_ps1_metadata(cls, string: str) -> list[re.Match[str]]:
    """Find all valid PS1 metadata blocks in the string."""
    for match in CMD_OUTPUT_METADATA_PS1_REGEX.finditer(string):
        content = match.group(1).strip()
        try:
            json.loads(content)
            matches.append(match)
        except json.JSONDecodeError:
            logger.debug(f"Failed to parse PS1 metadata - Skipping: ...")
```

**不只匹配正则，还要求 JSON 能解析成功。**

为什么？因为命令的**输出内容里可能恰好包含那个标记**（比如 `cat` 了一个含有标记的文件）。
加一道 JSON 校验，能过滤掉大部分误匹配。

> **可迁移的道理**：**用带内信号（in-band signaling）传递元数据时，必须假设数据本身
> 可能伪造这个信号。** 加一层结构化校验是最便宜的防御。

### 一个很贴心的错误提示

```python
def looks_like_python_literal_argument(command: str) -> str | None:
    """Detect when a tool call has packed structured data into `command`."""
    # Top-level list literals: [{...}], ["..."], ['...']
    if a == "[" and b in ("{", '"', "'"):
        return "list literal"
    # Nested list literals: [[...]] — but bash `[[ -f x ]]` is followed by a
    # whitespace char, so we only flag `[[` followed by non-whitespace.
    if a == "[" and b == "[":
        if len(stripped) >= 3 and stripped[2] not in (" ", "\t"):
            return "nested list literal"
        return None
    # Bash group commands `{ ls; }` always have a space after `{`.
    if a == "{" and b in ('"', "'"):
        return "dict literal"
```

**AI 有个典型错误：把一段 Python 代码或 JSON 直接塞进 `command` 字段。**

这个函数专门检测这种情况。而且**很小心地区分了 bash 的合法语法**：
- `[[ -f x ]]` 是 bash 的条件判断，`[[` 后面一定有空格 → 不报错
- `[[{"a":1}]]` 是嵌套列表，`[[` 后面不是空格 → 报错
- `{ ls; }` 是 bash 的命令组，`{` 后面一定有空格 → 不报错
- `{"key": ...}` 是字典 → 报错

报错信息更是教科书级的：

```
[Tool-argument error] The `command` argument looks like a Python/JSON {literal_kind},
not a shell command. It starts with: {head!r}

The `terminal` tool runs exactly ONE shell command at a time. To pass structured data
or multi-line code:
  - Write a script first with `file_editor` (command="create", path="/tmp/run.py", ...),
    then invoke it: `python /tmp/run.py`.
  - Or use a heredoc inline, e.g.:
        python - <<'EOF'
        DATABASES = {'default': {...}}
        # your code here
        EOF

Do not put a Python list/dict literal into the `command` field; the shell cannot
interpret it.
```

**说清了：错在哪、为什么错、两种正确做法（带完整示例）、最后重申一遍禁令。**

> 这和前面反复看到的模式一致（第三节工具报错带可用列表、第八节截断带取回方法）：
> **给模型的错误信息要包含足够让它自己改对的一切。**
>
> 而且注意这是**一个专门为"模型常犯的错"写的检测器**。不是通用校验，
> 是针对具体失败模式的针对性修补。

### 连日志都要脱敏

```python
class _SecretRedactFilter(logging.Filter):
    """Redact API key literals from libtmux log records."""
    def filter(self, record):
        if record.msg and isinstance(record.msg, str):
            record.msg = redact_api_key_literals(record.msg)
        ...
        tmux_cmd = getattr(record, "tmux_cmd", None)
        if tmux_cmd and isinstance(tmux_cmd, str):
            record.tmux_cmd = redact_api_key_literals(tmux_cmd)
```

**给第三方库（libtmux）的日志器装一个过滤器，把 API key 抹掉。**

因为 libtmux 会把它执行的 tmux 命令打进日志，而那些命令里可能包含 `export API_KEY=...`。

安装时还会扫描所有 `libtmux.*` 的子日志器，并检查是否已装过（避免重复安装）。

> **这是第四次看到"脱敏"这个主题**（工具输出出口、错误信息、422 响应、第三方库日志）。
> 可迁移的道理：**盘点所有"数据可能流出去"的通道，一个都不能漏。**
> 第三方库的日志是最容易被忘掉的那一个。

## 二、文件编辑器：怎么不改错地方

### 核心规则：必须唯一匹配

```python
occurrences = [... for match in re.finditer(pattern, file_content)]

if len(occurrences) > 1:
    line_numbers = sorted(set(line for line, _, _ in occurrences))
    raise ToolError(
        f"No replacement was performed. Multiple occurrences of old_str "
        f"`{old_str}` in lines {line_numbers}. Please ensure it is unique.")
```

**如果要替换的文本在文件里出现了多次，直接拒绝，并告诉 AI 出现在哪几行。**

为什么这条规则如此重要？考虑这个场景：

```python
def foo():
    x = 1        # ← 你想改这个

def bar():
    x = 1        # ← 但这里也有
```

AI 说"把 `x = 1` 换成 `x = 2`"。如果工具替换第一个，**有一半概率改错了地方，
而且悄无声息。** 拒绝掉，强迫 AI 提供更多上下文（比如带上函数名那一行），
就一定改对。

> **可迁移的道理**：**宁可大声失败，不要静默地做可能错的事。**
> 一个被拒绝的编辑，AI 下一轮就修好了；一个改错地方的编辑，可能几小时后才被发现。

### 找不到时会去掉首尾空白重试一次，但只对匹配串

```python
if not occurrences:
    # Strip old_str to retry the *match* only. Do NOT strip new_str: it is the
    # replacement content, and stripping it would silently drop meaningful
    # leading/trailing whitespace (e.g. a Markdown hard line break or
    # intentional indentation) the caller asked to write.
    old_str = old_str.strip()
```

**AI 经常在要替换的文本首尾多带或少带空白。** 所以去掉空白重试一次，提高成功率。

**但绝对不能对替换内容做同样的事。** 注释举了两个具体例子：
Markdown 的硬换行（行尾两个空格）、故意的缩进。**去掉它们会静默破坏 AI 想写的内容。**

> **可迁移的道理**：**"宽容地接受输入"和"忠实地写出输出"是两条不同的原则，不要混用。**
> 匹配时可以模糊，写入时必须精确。

### 原子写入：永远不会留下半个文件

```python
def _atomic_write(self, path: Path, file_text: str, encoding: str) -> None:
    """Write file_text to path atomically, never leaving a truncated file.

    The content is written to a temporary file in the same directory which is then
    Path.replace'd into place, so a failed write can never destroy the original file."""
    with self._temp_file(path, file_text, encoding) as tmp_path:
        # Preserve the original file's permission bits when it already exists.
        if path.exists():
            os.chmod(tmp_path, os.stat(path).st_mode & 0o7777)
        Path.replace(tmp_path, path)
```

**先写到同目录下的临时文件，然后用 `replace` 一步换过去。**

> `Path.replace` 在操作系统层面是**原子的**：要么完全成功，要么完全没发生。
> 不存在"写了一半"的中间状态。

**为什么必须这样？** 如果直接打开原文件写入，写到一半崩溃/断电/磁盘满，
**原文件就被截断了——内容永久丢失**。而这可能是用户几小时工作的成果。

三个配套细节：

**① 临时文件放在同一个目录**。因为 `replace` 跨文件系统不是原子的，
放 `/tmp` 就不安全了。

**② 保留原文件的权限位**：`os.stat(path).st_mode & 0o7777`。
不然一个可执行脚本被编辑后就不可执行了。

**③ 失败时清理临时文件**：

```python
except BaseException:
    tmp_path.unlink(missing_ok=True)
    raise
```

注释还解释了顺序：

> The unlink runs after the file is closed because Windows cannot delete an open file.

**Windows 不能删除打开中的文件**——跨平台的细节。

> **可迁移的道理**：**任何"覆盖已有数据"的操作都应该做成原子的。**
> 写临时文件 + 原子替换是最通用的模式，而且要记得保留权限、同目录、清理残留。

### 编码问题的降级策略

```python
default = self._encoding_manager.default_encoding
if encoding != default and not _is_encodable(file_text, encoding):
    logger.warning(f"Detected encoding '{encoding}' cannot represent the new content "
                   f"for {path}; writing as '{default}' instead.")
    encoding = default
```

注释解释了为什么要这个降级：

> If the file's detected encoding cannot represent the new content, fall back to UTF-8
> so an edit may add characters (arrows, emoji, CJK, ...) the original single-byte
> encoding lacks, instead of failing and truncating the file.

**场景**：一个用老式单字节编码（比如 latin-1）保存的文件，AI 要往里加一个箭头符号
或中文字符。**原编码表示不了**，于是写入失败。

**处理**：降级成 UTF-8，并打警告（注释也诚实地说明了代价：`the fallback transcodes
the whole file to UTF-8`——整个文件都会被转码）。

**取向很明确：能完成编辑 > 保持原编码。** 而且失败的后果（文件被截断）更糟。

### 还有一道可选的白名单

```python
if self.allowed_edits_files is not None and action.command != "view":
    action_path = Path(action.path).resolve()
    if action_path not in self.allowed_edits_files:
        return FileEditorObservation.from_text(
            text=(f"Operation '{action.command}' is not allowed on file '{action_path}'. "
                  f"Only the following files can be edited: {sorted(...)}"),
            is_error=True)
```

**可以限定"只允许改这几个文件"。** 注意 `action.command != "view"`——**查看不受限制，
只限制修改**。

而且用了 `.resolve()`（解析成绝对真实路径），防止用 `../` 或符号链接绕过白名单。

报错时**列出允许的文件清单**——又一次"错误信息要够用"。

### 还有撤销

```python
CommandLiteral = Literal["view", "create", "str_replace", "insert", "undo_edit"]
```

`undo_edit` 配合 `FileHistoryManager`。**AI 改错了可以自己撤销**，不需要人介入。

而观察对象里同时存了改动前后的内容：

```python
old_content: str | None    # 改之前
new_content: str | None    # 改之后
```

于是展示时能生成一个差异视图（`visualize_diff`，带 2 行上下文）。
**给人看的是 diff，不是一整个文件。**

## 三、委派：子 agent 怎么落地

```python
class DelegateExecutor(ToolExecutor):
    """- Spawning sub-agents with meaningful string identifiers (e.g., 'refactor_module')
    - Delegating tasks to sub-agents and waiting for results (blocking)"""

    def __init__(self, max_children: int = 5, confirmation_handler=None):
```

两个命令：`spawn`（造子 agent）、`delegate`（派任务）。**上限默认 5 个。**

### 子 agent 继承什么、不继承什么

这部分是整段最值得看的。创建子 agent 时：

```python
sub_agent_llm = parent_llm.model_copy()
# resetting metrics such that the sub-agent has its own Metrics object
sub_agent_llm.reset_metrics()
```

**模型配置继承，但用量统计重置成自己的一份。** 于是能分别看到每个子 agent 花了多少钱。

```python
# ensuring that the sub-agent LLM has stream deactivated
worker_agent = worker_agent.model_copy(
    update={"llm": worker_agent.llm.model_copy(update={"stream": False})})
```

**关掉流式输出。** 因为子 agent 的输出没人逐字看——只要最终结果。
（和第五节压缩器关流式是同一个理由。）

```python
# Inherit persistence from the parent conversation: if the parent persists its
# conversation, subagents persist theirs under a "subagents" subdirectory.
parent_persistence_dir = parent_conversation.state.persistence_dir
if parent_persistence_dir is not None:
    subagents_persistence_dir = Path(parent_persistence_dir) / _SUBAGENTS_DIR
```

**存盘位置继承，但放在子目录里。** 父会话不存盘，子会话也不存。
**配置的继承是"跟着父亲的选择"，而不是各自默认。**

```python
confirmation_policy = factory.definition.get_confirmation_policy()
if confirmation_policy is None:
    sub_conversation.set_confirmation_policy(parent_conversation.state.confirmation_policy)
else:
    sub_conversation.set_confirmation_policy(confirmation_policy)
```

**审批策略：子 agent 类型自己有定义就用自己的，没有就继承父亲的。**

> **这是权限模型里很关键的一条**：子 agent 不能通过"我没配置"来获得比父亲更宽松的权限。
> 默认继承，而不是默认宽松。
>
> 可迁移的道理：**派生出的执行单元，安全相关的配置要默认继承，而不是默认取默认值。**

### 失败时全部清理

```python
except Exception as e:
    for agent_id, _, sub_conversation in created_sub_agents:
        self._close_sub_agent(agent_id, sub_conversation)
    logger.error(f"Error: failed to spawn agents: {e}", exc_info=True)
```

**要造 5 个子 agent，造到第 3 个失败了，前 2 个也要关掉。**

注意实现方式：先把所有子 agent 造好放进一个**局部列表** `created_sub_agents`，
**全部成功之后才注册进 `self._sub_agents`**：

```python
for agent_id, agent_type, sub_conversation in created_sub_agents:
    previous_conversation = self._sub_agents.get(agent_id)
    if previous_conversation is not None:
        self._close_sub_agent(agent_id, previous_conversation)
    self._sub_agents[agent_id] = sub_conversation
```

**这是"要么全成功要么全不留"的标准写法：先在局部凑齐，再一次性提交。**

而且注册时发现同名的旧 agent，**会先关掉它**——防止泄漏。

> 可迁移的道理：**批量创建资源时，先全部建在局部变量里，成功后才提交到共享状态。**
> 这样异常路径只需要清理局部的那些。

### 派任务是并行的，但等待是阻塞的

```python
# Start all tasks in parallel
for agent_id, task in action.tasks.items():
    thread = threading.Thread(target=run_task, args=(...), name=f"Task-{agent_id}")
    threads.append(thread)
    thread.start()

# Wait for all threads to complete
for thread in threads:
    thread.join()
```

**所有子任务同时开跑，然后等全部跑完。**

`name=f"Task-{agent_id}"` 给线程起了有意义的名字——**调试时能在堆栈里看出这是哪个子 agent。**

### 子 agent 也要处理审批

```python
def _run_until_finished(self, agent_id: str, conversation: LocalConversation) -> None:
    """Run a sub-agent conversation to completion, handling confirmations."""
    conversation.run()
    while conversation.state.execution_status == ConversationExecutionStatus.WAITING_FOR_CONFIRMATION:
        pending = ConversationState.get_unmatched_actions(conversation.state.events)
        if not pending:
            break
        if self._confirmation_handler is None or self._confirmation_handler(agent_id, pending):
            conversation.run()
        else:
            conversation.reject_pending_actions("User rejected the actions")
            conversation.run()
```

回想第三节：确认模式下 `run()` 会停在"等待确认"状态，**再调一次 `run()` 就等于批准**。

这里就是那个机制的使用方：循环检查状态，问审批处理器，批准就再 `run()`，
拒绝就先记录拒绝再 `run()`（让子 agent 知道自己被拒了、换个做法）。

**注意 `if not pending: break`** —— 状态说在等确认但实际没有待确认动作，
说明状态不一致，直接跳出而不是死循环。**防御性地处理"不应该发生"的情况。**

**还有：`self._confirmation_handler is None` 时默认批准。** 这是个值得注意的选择——
没有人能回答审批问题时，选择放行而不是卡死。因为子 agent 的审批策略本身
已经从父亲那里继承了，父亲那一层已经做过把关。

## 四、把三个工具和前面的抽象对上

```
第十节的抽象                      这一节的兑现
──────────────────────────────────────────────────────
declared_resources()       →     终端声明 "terminal:session"
                                 文件编辑器声明 "file:/abs/path"
                                 于是改不同文件能并行，抢终端要排队

readOnlyHint               →     file_editor 的 view 命令只读
                                 于是查看文件不用审批

Observation.content 是列表  →     file_editor 能返回图片（view 一个 png
                                 会返回 base64 的 ImageContent）

统一出口脱敏                →     tmux 日志过滤器补上了第三方库这个漏洞

"错误信息要够用"            →     Python 字面量检测器给出两种正确写法
                                 多次匹配时列出所有行号
                                 白名单拒绝时列出允许的文件
```

## 五、这一节的可迁移结论

1. **需要跨调用保持状态时，维持一个长期存活的进程，而不是每次新建**：
   终端用 tmux 保住 `cd`、环境变量、正在运行的程序。

2. **带内信号传元数据很好用，但必须假设数据本身会伪造信号**：
   PS1 塞 JSON 是个漂亮技巧，配一道 JSON 解析校验过滤误匹配。

3. **交互式场景要支持按键，不只是命令**：`pip` 问 yes/no、程序卡住要 Ctrl+C，
   AI 得能像人一样按。

4. **为"模型常犯的具体错误"写针对性检测器**：不是通用校验，
   是识别"把 Python 字面量塞进 command 字段"这种具体失败模式，并给出完整的正确示例。

5. **宁可大声失败，不要静默地做可能错的事**：替换文本不唯一就拒绝。
   被拒绝的编辑下一轮就修好了；改错地方的编辑可能几小时后才被发现。

6. **"宽容地接受输入"和"忠实地写出输出"是两条不同的原则**：
   匹配时可以去掉首尾空白重试，写入时一个字符都不能动。

7. **任何"覆盖已有数据"的操作都应该做成原子的**：
   同目录临时文件 + 原子替换 + 保留权限位 + 失败清理。直接写入意味着崩溃就丢数据。

8. **降级策略要有明确的价值取向并写出代价**：编码表示不了就转 UTF-8，
   因为"完成编辑"比"保持原编码"重要，而且注释诚实说明整个文件会被转码。

9. **限制修改但不限制查看**：白名单只作用于写操作，`view` 放行。
   而且路径要 `.resolve()` 防止绕过。

10. **给 AI 撤销的能力**：`undo_edit` 让它自己纠错，不需要人介入。

11. **派生执行单元时，安全相关的配置默认继承而非默认取默认值**：
    子 agent 没配审批策略就用父亲的，不能靠"我没配"获得更宽松的权限。

12. **用量统计要各自独立，配置可以共享**：子 agent 复制父亲的模型配置，
    但重置统计对象，于是能分别看花费。

13. **批量创建资源先建在局部变量里，全部成功后才提交到共享状态**：
    异常路径只需清理局部的那些。

14. **防御性处理"不应该发生"的状态**：状态说在等确认但没有待确认动作时直接跳出，
    而不是死循环。

15. **给线程起有意义的名字**：调试时能在堆栈里认出是哪个子任务。

## 全部精读的收尾

十二节走完，这个 SDK 的完整图景是：

```
事件树（不可变账本）                         notes/02
  └─ View（安全切点 + 四条规则）              notes/03
       └─ Agent._step（单步 + 四种错误恢复）   notes/04
            └─ run()（循环 + 预算 + 卡死）     notes/05
                 ├─ Condenser（压缩降级链）    notes/06
                 ├─ Security（五层 + 融合）    notes/07
                 ├─ Critic（迭代精修）         notes/08
                 ├─ Context（提示装配 + 技能）  notes/09
                 ├─ LLM（能力表 + 三层降级）   notes/10
                 ├─ Tool（三种来源一个接口）    notes/11
                 ├─ Workspace/Server          notes/12
                 └─ 具体工具实现               notes/13（本篇）
```

**反复出现、贯穿全部十二节的五条原则：**

1. **不可变 + 只增 + 只读投影** —— 危险操作永远碰不坏真实记录
2. **把报错变成对话** —— 所有失败都写成一条消息给模型，让它自己修
3. **"未知"必须是独立状态** —— 不能等同于"安全"或"为空"（安全等级、资源申报、静态分析三处）
4. **找到唯一出口做一次** —— 脱敏、校验、统计都收口，而不是靠每个人记得
5. **把假设写成机器能执行的规则** —— 校验器、断言、类型，而不是注释里的提醒

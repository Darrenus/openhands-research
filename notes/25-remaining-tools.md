# 精读 24：剩下的工具（grep 451 + glob 444 + gemini 1448 + preset 609 + planning_file_editor 231）

> 位置：`upstream/software-agent-sdk/openhands-tools/openhands/tools/`
> 文件：`grep/`、`glob/`、`gemini/{read_file,write_file,edit,list_directory,file_change}`、
> `preset/{default,gemini,gpt5,planning,task_outcome}.py`、`preset/subagents/*.md`、
> `planning_file_editor/`
> 前置：`notes/13-tool-implementations.md`（file_editor）、
> `notes/18-browser-and-patch.md`（apply_patch）、`notes/21-task-manager.md`（子 agent）

## 这一节在讲什么

把最后几个工具收完。它们本身不复杂，但**合起来回答了一个前面一直悬着的问题：
第一节 codeloop 那个"编辑格式是最大杠杆"的假设，这个项目最终是怎么处理的。**

答案是 `preset/` ——**三套完整的工具组合，按模型家族选。**

## 一、grep：三级后端回退

```python
class GrepExecutor(ToolExecutor[GrepAction, GrepObservation]):
    """This implementation **prefers ripgrep for performance, falls back to the
    system grep binary when available, and finally uses a Python recursive search
    when no grep binary is installed.**"""

    def _select_search_backend(self) -> str:
        if _check_ripgrep_available(): return "ripgrep"
        if _check_grep_available():   return "grep"
        return "python"
```

> **ripgrep（`rg`）**：用 Rust 写的搜索工具，比传统 grep 快很多，
> 而且默认尊重 `.gitignore`。

**三级回退**：ripgrep → 系统 grep → 纯 Python 递归搜索。

**第三级是关键**：**保证这个工具在任何环境下都能用**，
哪怕容器里精简到连 grep 都没装。

### 回退时会打警告，而且说清降级到了哪一级

```python
if self._search_backend == "grep":
    _log_ripgrep_fallback_warning("grep", "system grep")
elif self._search_backend == "python":
    _log_ripgrep_fallback_warning("grep", "system grep, then Python search")
```

**降到第二级说"用系统 grep"，降到第三级说"系统 grep 也没有，用 Python 搜索"。**

> 回想第十七节那条"被迫的妥协要多处留痕"——**这里是性能降级的版本：
> 功能还在，但慢了，而且日志里说清降到了哪一级。**
>
> 对比一个常见的糟糕写法：静默回退。**然后有人报"搜索很慢"，你查不出原因。**

### 后端选择在构造时做一次

```python
def __init__(self, working_dir: str):
    self.working_dir: Path = Path(working_dir).resolve()
    self._search_backend = self._select_search_backend()
```

**不是每次搜索都探测一遍。** 和第十七节那个 Chromium 探测用 `@functools.cache`
是同一个考虑：**文件系统探测不便宜，而结果在进程生命周期内不会变。**

### 正则先编译再分派

```python
try:
    regex = re.compile(action.pattern, re.IGNORECASE)
except re.error as e:
    return GrepObservation.from_text(text=f"Invalid regex pattern: {e}", ..., is_error=True)

if self._search_backend == "ripgrep": return self._execute_with_ripgrep(...)
```

**注意顺序**：无论走哪个后端，**都先用 Python 的 `re` 编译一次做校验。**

**为什么？** ripgrep 和系统 grep 的正则方言和 Python 的不完全一样，
但**先编译能挡住绝大多数明显的语法错误，并给出 Python 风格的清晰错误信息**
（而不是一个 shell 命令的退出码 2）。

> **可迁移的道理**：**把用户输入转交给外部程序之前，先用本地的解析器做一次预检。**
> 外部程序的错误信息通常没有你自己的好。
>
> 代价是方言差异可能让某些合法的 ripgrep 正则被误拒——这是一个取舍，
> 换来的是绝大多数情况下更好的报错。

### 结果上限 100，而且在观察里标记了

```python
_MAX_MATCHES = 100

class GrepObservation(Observation):
    truncated: bool = Field(default=False,
                            description="Whether results were truncated to 100 files")
```

**而且工具描述里明确告诉模型怎么办**：

```
* Only the first 100 results are returned. **Consider narrowing your search with
  stricter glob patterns or provide path parameter if you need more results.**
```

> 又一次"错误/限制信息要包含下一步怎么办"（索引第 6 条）。
> 而 `truncated` 字段让**程序**也能知道结果不完整——不只是靠模型读文字。

## 二、glob：按修改时间排序

```
* Returns matching file paths **sorted by modification time**
```

**为什么按修改时间而不是按字母？**

因为 AI 找文件通常是在找"最近在动的那些"。`src/a.py`、`src/b.py`……
按字母排没有信息量；**按修改时间排，最相关的在最前面**——
而在只返回前 100 个的情况下，**排序方式直接决定了截断掉什么。**

> **可迁移的道理**：**有结果上限时，排序方式决定了你保留了什么。**
> 按"和当前意图的相关性"排，而不是按"实现起来最方便"排。

### 工作目录被拼进工具描述

```python
enhanced_description = (
    f"{TOOL_DESCRIPTION}\n\n"
    f"Your current working directory is: {working_dir}\n"
    f"When searching for files, patterns are relative to this directory."
)
```

**每个会话的工具描述里都带上实际的工作目录。**

> 回想第九节那个提示装配——**工具描述也是上下文的一部分，也可以动态填。**
> 比让模型猜"相对路径相对于哪里"可靠得多。

## 三、gemini 工具组：为什么要第四套编辑工具

```python
"""Gemini-style file editing tools.

This module provides gemini-style file editing tools **as an alternative to the
claude-style file_editor tool.** These tools are designed to **match the tool
interface used by gemini-cli.**

Tools:
    - read_file: Read file content with pagination support
    - write_file: Full file overwrite operations
    - edit: Find and replace with validation
    - list_directory: Directory listing with metadata
"""
```

**四个独立的工具，对齐 gemini-cli 的接口。**

回想第十节那条结论："**最容易让模型写对的格式，是它训练时见过的那个。**"

**这是那条原则的第三次落地**：
- `file_editor` → Anthropic 定义的接口（Claude 训练时见过）
- `apply_patch` → OpenAI cookbook 的格式（GPT-5.1 训练时见过）
- **`gemini/` 四件套 → gemini-cli 的接口（Gemini 训练时见过）**

> **这是对第一节那个假设的最终回答。** codeloop 的作者猜"编辑格式是最大杠杆"
> 并把它做成了可切换的变量，准备做 A/B 实验。
>
> **这个项目的做法不是做实验找出"最好的格式"，而是承认根本没有单一最优解——
> 因为最优格式取决于模型训练时见过什么。** 于是维护三套，按模型选。

### 一个小而漂亮的复用

```python
"""Shared observation behaviour for the Gemini tools that write files.

``edit`` and ``write_file`` both report a file that was either created or changed,
and both render the result as a diff. **The only part that differs is the line
announcing a change to an existing file**, so the rendering lives here and each
tool supplies that one line."""

class FileChangeObservation(Observation, ABC):
    @abstractmethod
    def change_summary(self) -> str:
        """Return the line shown when an existing file was changed."""
```

**两个工具唯一的差别是一行文字**，所以抽出一个基类，
**让子类只实现那一行。**

而且注释解释了为什么字段不放在基类：

```python
"""Subclasses declare ``file_path``, ``is_new_file``, ``old_content`` and
``new_content`` themselves **so that each tool keeps its own field descriptions**,
and implement ``change_summary``."""
```

**字段留给子类声明，是为了让每个工具保留自己的字段描述文字**
——因为那些描述会进给模型的 schema。

> **可迁移的道理**：**抽共同基类时，要区分"共享的行为"和"各自的措辞"。**
> 渲染逻辑可以共享，但**给模型看的字段描述应该按工具各自写**——
> 一句通用的描述对两个工具都不够准。
>
> 用 `if TYPE_CHECKING:` 声明字段类型让类型检查器满意，但运行时不定义它们。

## 四、preset：三套工具组合，一个 planning 模式

```
preset/default.py    终端 + file_editor + task_tracker + 浏览器
preset/gpt5.py       终端 + apply_patch  + task_tracker + 浏览器
preset/gemini.py     终端 + gemini 四件套 + ...
preset/planning.py   只读分析模式
```

`gpt5.py` 的文档说明了它的定位：

```python
"""This preset uses ApplyPatchTool for file edits instead of the default
claude-style FileEditorTool. **It mirrors the Gemini preset pattern by providing
optional helpers without changing global defaults.**"""
```

**"提供可选的辅助函数，而不改全局默认值。"**

> **可迁移的道理**：**针对特定环境的优化配置应该是"额外提供的一组辅助函数"，
> 而不是"改掉全局默认"。** 后者会让不知情的用户行为突变。

### planning 预设：第九节那个只读组合的来源

回想第九节那个 `PromptPreset.PLANNING` —— 静态提示块**只有一个片段**。
这里是它配套的工具组合。

而 `planning_file_editor` 是专门为它做的工具：

```python
class PlanningFileEditorAction(FileEditorAction):
    """Inherits from FileEditorAction but **restricts editing to PLAN.md only.
    Allows viewing any file but only editing PLAN.md.**"""
```

**能看任何文件，只能改 `PLAN.md`。**

> 回想第十七节那条"限制修改但不限制查看"——**这里是它的极端形态：
> 白名单只有一个文件。**

而且 PLAN.md 的位置有讲究：

```python
# PLAN.md is now stored in .agents_tmp/ to keep workspace root clean
# and **separate agent temporary files from user content**
DEFAULT_CONFIG_DIR = ".agents_tmp"
PLAN_FILENAME = "PLAN.md"
```

**放在 `.agents_tmp/` 而不是工作区根目录**，理由是"保持根目录干净、
**把 agent 的临时文件和用户内容分开**"。

> 对比第十九节那个 `TASKS.md`（人要看的，放在人找得到的地方）。
> **两个选择相反，但判断标准一致：这个文件是给人看的，还是 agent 的中间产物？**
> - `TASKS.md` → 人要看 → 放显眼处
> - `PLAN.md` → agent 的工作草稿 → 放临时目录

### 内置子 agent 也是 Markdown 文件

```
preset/subagents/default.md
preset/subagents/bash_runner.md
preset/subagents/code_explorer.md
preset/subagents/web_researcher.md
```

**内置的子 agent 和用户自定义的用同一套格式**（第二十一节那个 YAML 头 + 正文）。

> **可迁移的道理**：**内置的东西应该用和用户自定义完全相同的机制实现。**
> 于是格式的表达力被自己验证过，用户能照着内置的改，
> 而且**不存在"内置的能做但用户做不到"的功能。**

## 五、`bash_runner.md`：一份写得很好的子 agent 提示

这个文件值得整段读，因为它是"如何让子 agent 返回有用的东西"的范本。

### 描述部分先定位它的价值

```yaml
description: >-
   USE THIS to execute shell commands and get a concise report of the results.
   Runs tests, builds, linters, git operations, system inspection, dependency
   installation, or any other CLI task. **Returns only what matters: pass/fail
   counts, specific failures with reasons, and actionable errors — never raw output.**
```

**"只返回重要的东西——从不返回原始输出。"**

### 正文里一句话点明了它存在的理由

```
Your most important job is to **distill command output into a concise report**.
**The caller does not see raw terminal output — they only see what you write back.**
Never dump raw output. Always summarize.
```

**"调用方看不到原始终端输出——他们只看到你写回去的东西。"**

> **这是子 agent 的核心价值，也是这段提示最重要的一句。**
> 派一个子 agent 去跑测试的收益不在于"它会跑命令"（主 agent 也会），
> **而在于原始输出（可能几万行）不会进主 agent 的上下文。**
>
> 回想第二十节那条结论"要求委派方说明回报什么格式"——**这里是在被委派的那一侧
> 把格式要求写死。**

### 按输出类型分别规定格式

```
For **test suites**, report:
- Total passed / failed / skipped / errored counts
- For each failure: test name, short reason, and the file:line where it failed
- **Nothing else — no passing test names, no full tracebacks, no captured stdout**

For **builds and linters**, report:
- Success or failure
- For each error/warning: file:line, the message, and a one-line summary
- **Nothing else — no "compiling X..." progress lines**

For **git operations**, report:
- What changed (branch, commit hash, files affected)
- Any conflicts or errors
```

**四种输出类型，每种都明确列出"要什么"和"不要什么"。**

那三个 `Nothing else` 尤其重要——**光说"要什么"不够**，
模型会倾向于"顺便多给点"。**必须明确否定。**

> **可迁移的道理**：**给模型规定输出格式时，"不要包含什么"和"要包含什么"一样重要。**
> 而且要按输入类型分别规定——一条通用的"请简洁"毫无约束力。

### 三条操作纪律

```
1. **Be precise.** Run exactly what was requested. Do not add extra flags or
   steps unless they are necessary for correctness.
2. **Chain when appropriate.** Use `&&` to chain dependent commands so later
   steps only run if earlier ones succeed.
3. **Avoid interactive commands.** Do not run commands that require interactive
   input (e.g., `vim`, `less`, `git rebase -i`). Use non-interactive alternatives.
```

**第一条防的是模型"自作主张加个 `-v`"**；
**第三条防的是第十三节那个终端会话卡在交互式程序里**——
**在提示层面就避免，而不是等它卡住再靠按键去救。**

## 六、`task_outcome.py`：一个标注为"实验性"的结构化结果

```python
"""**Experimental** structured task outcome models for conversation finish actions."""

TaskOutcomeStatus = Literal["success", "partial_success", "blocked", "failed", "unknown"]

class TaskOutcome(BaseModel):
    """Latest semantic outcome reported for a conversation task.

    **Experimental: this structured response model may change as task outcome
    reporting is refined.**"""
    status: TaskOutcomeStatus = Field(
        description="Agent's semantic assessment of task completion.")
    summary: str = Field(alias="outcome_summary", ...)
```

**五档状态，注意中间那两个**：`partial_success`（部分成功）和 `blocked`（被阻塞）。

回想第二十节那条"非 FINISHED 一律算错误"——**那是执行层面的二元判断**，
而这是**语义层面的五档自评**。两者并存：
- 执行层面：跑完了没有（程序判断）
- 语义层面：做成了多少（模型自评）

`blocked` 这一档尤其有用：**"我没做成，但不是因为我做错了，是因为环境不允许"**
——这和"失败"应该被区别对待（回想第七节那个"区分环境的问题和 AI 造成的问题"）。

> **而且这个模型被明确标注了两次 `Experimental`**，
> 说明作者知道这个分类可能不对、可能会改。
>
> **可迁移的道理**：**不确定的设计要标注为实验性，并说明"可能会变"。**
> 否则下游会把它当成稳定契约来依赖。

## 七、这一节的可迁移结论

1. **性能相关的依赖要有多级回退，而且最后一级必须是"一定能用"的**：
   ripgrep → 系统 grep → 纯 Python。**保证工具在任何环境下都可用。**

2. **降级要打警告，并说清降到了哪一级**：静默回退会导致"搜索很慢"这种
   报障查不出原因。

3. **能力探测在构造时做一次并缓存**：文件系统探测不便宜，
   而结果在进程生命周期内不变。

4. **把用户输入转交给外部程序之前，先用本地解析器做一次预检**：
   外部程序的错误信息通常没有你自己的好。（代价是方言差异可能误拒少数合法输入。）

5. **有结果上限时，排序方式决定了你保留了什么**：
   按"和当前意图的相关性"排（修改时间），而不是按实现方便（字母序）。
   **而且要在观察里加一个 `truncated` 标记，让程序也能知道结果不完整。**

6. **工具描述也是上下文的一部分，可以动态填**：把实际的工作目录拼进去，
   比让模型猜"相对路径相对于哪里"可靠。

7. **没有单一最优的模型交互格式**：最优格式取决于模型训练时见过什么。
   **所以维护多套并按模型选，而不是做实验找"最好的那个"。**

8. **抽共同基类时要区分"共享的行为"和"各自的措辞"**：
   渲染逻辑共享，但给模型看的字段描述按工具各自写。

9. **针对特定环境的优化配置应该是"额外提供的辅助函数"，而不是改全局默认**：
   后者会让不知情的用户行为突变。

10. **判断一个中间文件放哪里：它是给人看的，还是 agent 的工作草稿？**
    `TASKS.md` 放显眼处，`PLAN.md` 放 `.agents_tmp/`。

11. **内置的东西应该用和用户自定义完全相同的机制实现**：
    格式的表达力被自己验证过，而且不存在"内置能做但用户做不到"的功能。

12. **子 agent 的核心价值是"原始输出不进主 agent 的上下文"**，
    所以它的提示里最重要的一句是"调用方只看到你写回去的东西"。

13. **给模型规定输出格式时，"不要包含什么"和"要包含什么"一样重要**，
    而且要**按输入类型分别规定**——一条通用的"请简洁"毫无约束力。

14. **在提示层面避开已知的陷阱**，而不是等它发生再靠机制去救：
    "不要运行交互式命令"比"卡住了再按 Ctrl+C"便宜得多。

15. **结果状态需要"部分成功"和"被阻塞"这两档**：
    "被阻塞"是"我没做成但不是我的错"，和"失败"应该被区别对待。

16. **不确定的设计要标注为实验性并说明可能会变**，
    否则下游会把它当稳定契约依赖。

## 收尾：`openhands-tools` 的完整工具清单

读到这里，这个包提供的工具可以完整列出来了：

```
终端类     terminal（tmux 保状态，第 12 节）
编辑类     file_editor（Anthropic 风格，第 12 节）
           apply_patch（OpenAI 风格，第 17 节）
           gemini/{read_file,write_file,edit,list_directory}（Gemini 风格，本节）
           planning_file_editor（只能改 PLAN.md，本节）
搜索类     glob（按修改时间排序）、grep（三级后端回退）
浏览器     browser_use（十几个工具 + rrweb 录制，第 17 节）
任务类     task_tracker（清单，第 19 节）、task（派子 agent，第 20 节）
           delegate（spawn + delegate，第 12 节）、workflow（写编排代码，第 19 节）
顾问类     ask_oracle（更强的模型，第 21 节）
           tom_consult（用户模型，第 21 节）
内置       think / finish / switch_llm / classify_and_switch_llm /
           vision_inspect / invoke_skill（第 3、14 节）
```

**三套编辑工具 + 四个内置子 agent + 四套预设组合**，
这个"可替换性"的程度，正是第一节那个问题的答案。

## 剩下还没读的

- `openhands-agent-server/` 的细节：`event_service.py`（2021 行）、
  `persistence/store.py`（990 行）、`file_router.py`（1083 行）、
  `docker/build.py`（1297 行）
- `acp_file_credentials.py`（536 行）
- `settings/model.py`（2477 行）的具体字段
- `clients/typescript/`（给前端的 TS 客户端）

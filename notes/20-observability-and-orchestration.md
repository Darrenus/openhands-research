# 精读 19：可观测性与任务编排（observability/ 618 行 + task/workflow 2061 行）

> 位置：`upstream/software-agent-sdk/`
> 文件：`openhands-sdk/openhands/sdk/observability/{laminar,utils}.py`、
> `openhands-tools/openhands/tools/task_tracker/definition.py`、
> `openhands-tools/openhands/tools/task/{manager,definition,impl}.py`、
> `openhands-tools/openhands/tools/workflow/{definition,impl}.py`
> 前置：`notes/13-tool-implementations.md`（委派）、`notes/05-run-loop.md`（会话循环）

## 这一节在讲什么

两个一直被跳过的东西：

**`@observe` 装饰器** —— 前面十八节里它出现了几十次（`@observe(name="agent.astep")`、
`@observe(span_type="TOOL")`），但从没读过它是什么。

**任务编排** —— AI 怎么管理长任务，以及一个很激进的功能：
**让 AI 写 Python 代码来编排一群子 agent。**

## 一、可观测性的第一个问题：不能强制依赖

```python
def observe[**P, R](*, name=None, span_type="DEFAULT", ignore_inputs=None, ...):
    """Lazy-resolving observe decorator.

    **When observability is not enabled, decorated functions run as pass-throughs
    with no `lmnr` import.** The first call after observability becomes enabled
    imports `lmnr` and caches the wrapped function."""
```

> **追踪 / span（跨度）**：把一次请求拆成一棵嵌套的"时间段"树，
> 每段记录耗时、输入输出、父子关系。用来看"这次运行到底在哪儿花了时间"。

**核心约束**：这个 SDK 是个库，不能强迫每个用户装一个追踪后端。

于是装饰器做成**三态懒加载**：

```python
def decorator(func):
    wrapped: Any = None

    @functools.wraps(func)
    async def async_wrapper(*args, **fkwargs):
        nonlocal wrapped
        if wrapped is not None:                        # ① 已经包装过 → 直接用
            with _maybe_use_root_span(args):
                return await wrapped(*args, **fkwargs)
        if not should_enable_observability():           # ② 没开 → 直接透传
            return await func(*args, **fkwargs)
        wrapped = _build_wrapped(func)                  # ③ 第一次开启 → 现在才 import
        with _maybe_use_root_span(args):
            ...
```

**没开追踪时，`@observe` 的开销就是一次布尔判断。** `lmnr` 这个包根本不会被 import。

### 而且分了同步/异步两条包装路径

```python
# Branch on async-ness at decoration time so that
# inspect.iscoroutinefunction(decorated) matches the original. **A sync wrapper
# around an async function would hide its asyncness from callers like run_async
# that introspect the function.**
if inspect.iscoroutinefunction(func):
    @functools.wraps(func)
    async def async_wrapper(...): ...
```

**在装饰的时刻就判断被装饰的函数是不是协程**，分别生成 async / sync 包装器。

**为什么不能统一用一个同步包装器？** 因为代码里有地方会用
`inspect.iscoroutinefunction(f)` 去判断"这个函数要不要 await"。
**一个同步包装器会把异步性藏起来**，那些判断就全错了。

> **可迁移的道理**：**装饰器必须保持被装饰函数的可内省特征**（是否协程、签名、名字）。
> `functools.wraps` 只解决名字和文档，**异步性要靠分支解决。**

### 开关状态是"一旦开启就永久开启"

```python
# Cache of positive results for should_enable_observability. Once observability is
# enabled (via env vars or a user-side Laminar.initialize() call), **it stays
# enabled for the lifetime of the process.**
_observability_enabled: bool = False

def should_enable_observability() -> bool:
    global _observability_enabled
    if _observability_enabled:
        return True
    if any(get_env(key) for key in _OBSERVABILITY_ENV_KEYS):
        _observability_enabled = True
        return True
    # **Only probe Laminar.is_initialized() if the user has already imported lmnr
    # themselves — otherwise importing it here defeats the purpose of lazy loading.**
    if "lmnr" in sys.modules:
        from lmnr import Laminar
        if Laminar.is_initialized():
            _observability_enabled = True
            return True
    return False
```

**`"lmnr" in sys.modules` 这个检查非常聪明**：想知道"用户是不是自己初始化了 Laminar"，
但**直接 import 就破坏了懒加载的意义**。于是先看模块有没有被别人 import 过——
如果没有，用户显然没在用它。

> **可迁移的道理**：**检测一个可选依赖的状态时，用 `sys.modules` 判断它是否已被加载，
> 而不是直接 import 它。** 这是"只缓存肯定结果"（单向开关）配合懒加载的标准手法。

## 二、跨线程/跨任务的追踪：一个实测出来的结论

这段类文档是整个模块最有价值的部分。

```python
class RootSpan:
    """A long-lived Laminar span owned by a single object (e.g. a Conversation).

    The span is created via ``Laminar.start_span`` (which does NOT attach the span to
    the current OpenTelemetry context). To make the span the parent of nested
    ``@observe``-decorated calls, the ``observe`` wrapper **re-attaches the span via
    ``Laminar.use_span`` at every entry point.** This allows the root span to span
    across asyncio tasks, threads, and processes **where naive ``contextvars``
    propagation breaks down.**

    The ``Laminar.start_active_span`` API was previously used for this purpose but its
    docstring explicitly warns:

        "ending the started span in a different async context yields unexpected
         results. … Use Laminar.start_span + Laminar.use_span where possible."

    **Empirically, ``start_active_span`` produced trace-context loss for ~60% of
    conversations** (orphan ``conversation.send_message`` / ``conversation.run``
    traces with no ``session_id``), so we switched to the recommended pattern."""
```

**"实测下来，60% 的会话丢失了追踪上下文。"**

> **contextvars**：Python 里"跟着当前执行流走"的变量。
> 问题是它在**跨线程、跨 asyncio 任务**时不会自动传播（第三节那个
> `contextvars.copy_context()` 就是在处理这个）。

**为什么会丢 60%？** 因为这个 agent 到处都在跨执行流：并行工具调用跑在线程池里、
异步路径跑在 asyncio 任务里、子 agent 跑在独立线程里。
**依赖 contextvars 自动传播的方案在这种架构下必然大面积失效。**

**解法**：span 创建时**不挂到当前上下文**，而是让每一个 `@observe` 的入口
**主动把它重新挂上**：

```python
@contextlib.contextmanager
def _maybe_use_root_span(args: tuple[Any, ...]) -> Iterator[None]:
    """If the first positional arg owns a ``RootSpan``, re-attach it.

    **This is what ties ``@observe``-decorated methods (called from arbitrary asyncio
    tasks or threads) back to the conversation's long-lived root span.**"""
    root = _root_span_from_args(args)
```

**从第一个位置参数上找 `RootSpan`** ——因为 `@observe` 装饰的基本都是方法，
第一个参数是 `self`（Conversation 或 Agent）。

> **可迁移的道理**：**跨线程/跨任务的上下文传播不要依赖语言的自动机制。**
> 把上下文挂在一个**明确的对象**上，在每个入口主动恢复。
> 这个模式叫"显式上下文传递"，虽然啰嗦但可靠。
>
> **而且这一段给了一个很有说服力的论证方式**：不是说"文档建议这样"，
> 而是"我们实测发现 60% 的会话丢了上下文，所以换了"。

## 三、一个和第三方库抢时机的问题

```python
@contextlib.contextmanager
def llm_call_span(name: str) -> Iterator[Any]:
    """Open a span that stays current for the duration of one LLM call.

    **lmnr's LiteLLM instrumentation builds its span with ``Laminar.start_span``
    (detached from the OTel context) and ends it in a ``finally`` before
    ``litellm.completion`` returns, so attributes cannot be attached to it
    afterwards — OTel silently drops writes to an ended span.** This span is
    *current* while the call runs, so lmnr's span nests underneath it, and it stays
    open long enough for Telemetry to attach the authoritative cost.

    Yields ``None`` when observability is disabled, so callers stay branch-free."""
```

**问题**：追踪库自己给 LLM 调用建了一个 span，但**在函数返回前就关掉了**。
而真正的花费数据（第九节那个 `Metrics`）是在返回之后才算出来的。
**往一个已关闭的 span 写属性会被 OTel 静默丢弃。**

**解法**：自己先开一个更外层的 span，让库的 span 嵌在里面，
**自己这个 span 活得够久，等花费数据算出来再挂上去。**

最后那句也值得注意：**关闭追踪时 yield `None`，让调用方不用写 if 分支。**

```python
with llm_call_span("llm.completion") as span:
    # span 可能是 None，但 with 语句本身总能用
```

> **可迁移的道理**：**可选功能的上下文管理器应该在"关闭"时也能正常进入**，
> yield 一个空值，而不是让每个调用点写 `if enabled:`。

## 四、子 agent 的追踪要主动切断，而且理由极其具体

这段是整个文件里最"深水区"的一段：

```python
@contextlib.contextmanager
def detached_delegate_context() -> Iterator[dict[str, TraceMetadataValue]]:
    """Clear the ambient span so a conversation constructed inside this block starts a
    genuinely new Laminar trace, instead of ``RootSpan`` silently joining whatever span
    is currently active.

    ``Laminar.start_span`` without an explicit ``context=`` parents onto the ambient
    span — its "isolated context" helper just returns whatever is current. **Laminar
    tracks that "current" span via its OWN isolated ``ContextVar``
    (``lmnr...tracing.context._ISOLATED_RUNTIME_CONTEXT``), separate from the standard
    ``opentelemetry.context`` one**, so attaching an empty/no-parent context through
    the standard API (as ``RootSpan`` does for cross-thread re-attachment) **has no
    effect here** — ``Laminar.use_span`` is the one call that updates both. **Pushing
    ``INVALID_SPAN`` through it is what actually severs the link**, so a sub-agent
    conversation constructed synchronously inside its parent's ``task`` TOOL span
    starts its own trace instead of inheriting the parent's (#4365)."""
```

**问题**：追踪库维护了**自己的一份**"当前 span"的 ContextVar，
和标准 OpenTelemetry 的那份是分开的。**用标准 API 清空上下文对它无效。**

**解法**：往 `Laminar.use_span` 里塞一个 `INVALID_SPAN`
（唯一能同时更新两份状态的调用）。

**为什么要切断？** 因为子 agent 是一个独立的长任务，
挂在父 agent 的某一个工具调用 span 下面，追踪界面上会变成一棵畸形的树
（父 span 的耗时包含了整个子 agent 的运行）。

但**切断之后又要保留可追溯性**：

```python
link["delegate.parent_trace_id"] = str(parent.trace_id)
link["delegate.parent_span_id"] = str(parent.span_id)
tool_call_id = (parent.metadata or {}).get(_TOOL_CALL_ID_META_KEY)
if tool_call_id:
    link[_TOOL_CALL_ID_META_KEY] = tool_call_id
```

**把父 span 的 id 作为元数据存到子 trace 上**——于是两条独立的 trace 之间有一条
可查询的链接。注释说目的是
`so the originating tool call is visible from the delegate's trace alone`
——**光看子 agent 的 trace 就能知道它是被哪个工具调用触发的。**

还有一个异常处理的细节很讲究：

```python
# Only guard *entering* Laminar.use_span (a caller exception raised inside the
# ``with`` block below must propagate normally, not be swallowed here).
with contextlib.ExitStack() as stack:
    try:
        stack.enter_context(Laminar.use_span(otel_trace.INVALID_SPAN, ...))
    except Exception:
        logger.debug("Failed to detach ambient span for delegate", exc_info=True)
    yield link
```

**只给"进入上下文"这个动作套 try**，而不是把整个 `with` 块套进去——
否则调用方在块内抛的异常会被这里吞掉。

> 用 `ExitStack` 而不是嵌套的 `with`，正是为了把"进入"和"块体"分开。
> **这是一个很容易写错的地方**：随手写 `try: with ...: yield` 就会把调用方的异常也吞掉。

> **可迁移的道理**：**辅助设施（追踪、日志、指标）的容错只能包住它自己的操作，
> 绝不能包住业务代码。** 这和第十二节那条"遥测失败不能改变主流程结果"是同一条，
> 但这里是它在语法层面的体现。

## 五、任务清单工具：提示词比代码长

`task_tracker` 的代码逻辑极简：一个 `view` 一个 `plan`，
任务只有 `title` / `notes` / `status`（todo / in_progress / done）三个字段。

**但工具描述有 150 多行。** 值得看它在教什么。

### 明确说了什么时候不要用

```
## Situations Where Tool Usage Is Unnecessary

Avoid using this tool when:
1. Single atomic tasks that require no decomposition
2. Trivial operations where tracking adds no organizational value
3. Simple activities completable in minimal steps
4. Pure information exchange or discussion

Note: For single straightforward tasks, proceed with direct implementation
**rather than creating tracking overhead.**
```

> **给模型的工具描述里，"什么时候不要用"和"什么时候用"一样重要。**
> 否则模型会对每个请求都建一个任务清单——**这不只是浪费 token，
> 还会让真正需要规划的时候失去信号。**

### 一条纪律被写进描述

```
6. Work commencement - Update task status to in_progress before beginning
   implementation. **Maintain focus by limiting active work to one task**
7. Task completion - Update status to done and identify any additional work
   **that emerged during implementation**
```

**"同时只能有一个任务处于进行中"** 是一条强制的专注纪律。

**第 7 条更有意思**：完成一项时要识别"实施过程中浮现出来的新工作"。
这承认了一个现实——**计划在执行中一定会变，而且新发现的工作必须被记录，
否则会被遗忘。**

> 回想第七节那个失败模式清单里的 `incomplete_implementation`（实现不完整）
> 和 `scope_creep`（范围蔓延）。**这两条纪律正好在防这两件事：
> 限制并发防蔓延，记录新发现防遗漏。**

描述里还有完整的**使用场景示例**（"Scenario A: Feature Development with Validation"），
带着 AI 应该怎么回应的示范。

### 持久化到 TASKS.md

```python
def __init__(self, save_dir: str | None = None):
    """Args:
        save_dir: Optional directory to save tasks to. If provided, tasks will be
                 persisted to save_dir/TASKS.md"""
    self.save_dir = Path(save_dir) if save_dir else None
    self._task_list: list[TaskItem] = []
    if self.save_dir:
        self._load_tasks()
```

**存成 Markdown 而不是 JSON**，而且文件名是 `TASKS.md`——**人可以直接读和改。**

> **可迁移的道理**：**AI 的中间状态如果人也需要看，就存成人类可读的格式
> 放在人找得到的地方。** 一个 `.json` 藏在缓存目录里，和一个 `TASKS.md`
> 放在项目根目录，对协作的意义完全不同。

标注也很准确：`idempotentHint=True`（重复执行同样的 plan 结果一样）、
`destructiveHint=False`（不破坏什么）、`openWorldHint=False`（不碰外部）。

## 六、workflow 工具：让 AI 写编排代码

这是整个 SDK 里最激进的一个功能。

```python
class WorkflowAction(Action):
    name: str
    script: str = Field(description=(
        "Python workflow script to run. It must define `async def main(wf):` "
        "and coordinate work only through the provided `wf` object."))
    max_concurrency: int = Field(default=8, ge=1, le=64, description=(
        "Maximum number of sub-agent tasks to run concurrently. "
        "**Consider 2–4 for LLM-heavy workflows to avoid hitting API rate limits.**"))
```

**AI 写一段 Python，这段代码负责编排一群子 agent。**

### 职责边界划得很清楚

```
The script coordinates sub-agents through the `wf` object. **It should not read or
write files, run shell commands, or perform the engineering work directly.
Sub-agents should do that work through their normal OpenHands tools and security
policy.**
```

**编排脚本只做编排，不干活。** 真正的工作由子 agent 通过正常的工具和安全策略做。

> **这是一条关键的安全边界**：如果编排脚本能直接读写文件，
> 那前面十八节讲的所有审批、风险评估、脱敏机制**全部被绕过了**。
> 限制它只能调 `wf` 的方法，就把它关在了编排这一层。

### 四个编排原语

```
- await wf.run_agent(prompt, subagent_type=..., description=None)
- await wf.map_agents(items, prompt, subagent_type=..., max_concurrency=None, ...)
- await wf.reduce_agent(items, prompt, subagent_type=..., description=None)
- await wf.pipeline(items, *stages)
- wf.flatten(values)
```

**`map` / `reduce` / `pipeline`** ——就是数据处理里那套原语，
只不过每个"计算单元"是一个 agent。

`pipeline` 和 `map_agents` 的区别写得很清楚：

```
- `await wf.pipeline(items, *stages)` — run each item through all stages **with no
  barrier between stages (a fast item reaches a later stage while a slow item is
  still in an earlier one)**. ... Prefer this over chained `map_agents` calls when
  per-item stages are independent, **since `map_agents` fully drains each stage
  before the next.**
```

> **屏障（barrier）**：并行计算里指"所有任务都到这儿了才能继续"的同步点。

**`map_agents` 链式调用有屏障**（第一阶段全做完才开始第二阶段），
**`pipeline` 没有**（快的条目可以先走到后面阶段）。

**在 agent 编排里这个差别很大**：agent 的耗时差异可能是 10 倍，
有屏障的话所有人都要等最慢的那个。

### 异常处理的一个语言限制被写进了文档

```
If one or more `map_agents` items fail, the whole call raises an `ExceptionGroup`.
**The name `ExceptionGroup` is not available by name in the workflow sandbox, so
scripts cannot use `except*` for selective group handling.** A plain
`except Exception` will still catch the entire group. **To handle partial failures
and collect all results, design sub-agent prompts to return an error sentinel value
instead of raising.**
```

因为沙箱不给内置名字（下面会看到），所以 `except*` 用不了。
**给出的替代方案是"让子 agent 返回一个错误哨兵值而不是抛异常"**——
这是一个很实际的建议。

### `print` 的行为也说清了

```
`print()` is available for debugging but **writes to the server logs, not to the
workflow observation seen by the LLM**; use the return value of `main()` to surface
results.
```

**否则 AI 会用 `print` 输出结果，然后困惑于自己看不到。**

### 最后一句最重要

```
This MVP executes generated Python in-process after best-effort validation.
**Treat running a workflow as approving generated code execution.**
```

**"把运行一个工作流当作批准执行生成的代码。"**

这句话写在工具描述里，也就是说**模型自己也看到了这个警告**。
而 `MVP` 和 `best-effort validation` 两个词都在降低读者对安全性的预期。

## 七、AST 校验：一个诚实到罕见的安全声明

```python
def validate_workflow_script(script: str) -> None:
    """Perform best-effort validation for generated workflow scripts.

    **Note: The private-attribute guard checks the literal name ``wf``, so aliasing
    (e.g. ``x = wf; x._attr``) can bypass the check.** The attributes accessible
    through ``WorkflowContext`` do not expose dangerous capabilities, so **this is a
    documentation gap rather than a security gap.**"""
```

**作者直接写出了绕过方法**（`x = wf; x._attr`），然后论证为什么这不构成安全问题
（`wf` 上没有危险能力可访问）。

> **这是我在整个项目里看到最诚实的一段安全说明。** 大部分项目会把这种已知绕过
> 藏起来，或者假装校验是完备的。写出来的好处：
> **后来加 `wf` 方法的人会知道"这个校验挡不住别名，所以不能在 `wf` 上放危险能力"。**
>
> 可迁移的道理：**基于 AST 的校验永远不是完备的沙箱。** 如果你用它，
> 就必须写清它挡不住什么，并保证"挡不住的那部分不重要"。

### 校验的五道检查

```python
if len(script) > _MAX_SCRIPT_CHARS:  raise ...     # ① 长度
tree = ast.parse(script)                            # ② 语法

main_defs = [n for n in tree.body
             if isinstance(n, ast.AsyncFunctionDef) and n.name == "main"]
if len(main_defs) != 1:  raise ...                  # ③ 有且只有一个 main

if ([a.arg for a in main_args.args] != ["wf"] or main_args.kwonlyargs
        or main_args.vararg or main_args.kwarg
        or main_args.defaults or main_args.posonlyargs):
    raise WorkflowScriptError("Workflow entry point must be `async def main(wf):`")
```

**第 4 条检查签名时把六种参数形式全列了一遍**：
关键字参数、`*args`、`**kwargs`、默认值、仅位置参数——
**一个都不允许，签名必须精确是 `main(wf)`。**

> 这种"把所有变体都堵上"的写法，对比只检查 `args == ["wf"]`——
> 后者会漏掉 `async def main(wf, *, extra=1)` 这种。

然后遍历整棵树：

```python
for node in ast.walk(tree):
    if isinstance(node, (ast.Import, ast.ImportFrom)):
        raise WorkflowScriptError("Workflow scripts may not import modules")
    if isinstance(node, ast.Name) and node.id.startswith("__"):
        raise WorkflowScriptError("Workflow scripts may not access dunder names")
    if isinstance(node, ast.Attribute) and node.attr.startswith("__"):
        raise WorkflowScriptError("Workflow scripts may not access dunder attributes")
    if (isinstance(node, ast.Attribute) and _attribute_root_name(node) == "wf"
            and (node.attr.startswith("_") or node.attr == "close")):
        raise WorkflowScriptError("Workflow scripts may not access private wf "
                                  "attributes or call wf.close()")
    ...
```

**禁 import、禁双下划线名字、禁双下划线属性、禁 `wf` 的私有属性和 `close()`。**

> **为什么禁双下划线？** 因为 Python 的沙箱逃逸经典路径就是
> `().__class__.__bases__[0].__subclasses__()` 这类——**通过双下划线属性
> 一路爬到任意类，然后拿到 `os` 之类的模块。** 禁掉双下划线是最基本的一道。

而 `_attribute_root_name` 会**一路剥到最外层的名字**：

```python
def _attribute_root_name(node: ast.Attribute) -> str | None:
    value = node.value
    while isinstance(value, ast.Attribute):
        value = value.value
    return value.id if isinstance(value, ast.Name) else None
```

于是 `wf.a.b._secret` 也能被识别出根是 `wf`。

### 还有一处针对格式化字符串的防御

```python
# (e.g. "{item._manager}"), which would bypass the AST private-attribute guard.
```

**f-string 或 `.format()` 里的属性访问在 AST 上不是 `Attribute` 节点**
（在旧版 Python 里 `.format()` 的模板是个纯字符串），所以会绕过检查。
这里专门处理了。

### reducer 输入要截断

```python
_MAX_REDUCE_INPUT_CHARS
...
chunk = jsonlib.dumps(dict([item]) if is_dict else item, default=str)
if kept and used + len(chunk) > _MAX_REDUCE_INPUT_CHARS:
    break
kept.append(item)
used += len(chunk)
while len(kept) > 1 and len(render(kept)) > _MAX_REDUCE_INPUT_CHARS:
    kept.pop()

# Safety net: a single oversized element can still exceed the budget.
return _truncate_text(render(kept))
```

**三道防线**：逐条累加时超了就停、渲染后还超就往回弹、
**最后还有一道整体截断兜底**（注释说明了原因：单个元素就可能超预算）。

工具描述里也提前说了：`Large reducer inputs may be truncated before being sent
to the reducer sub-agent.` ——**让 AI 知道它的 reduce 输入可能被截断。**

## 八、这一节的可迁移结论

1. **库里的可观测性必须是零成本可选的**：没开时装饰器只做一次布尔判断，
   追踪库根本不被 import。**"只缓存肯定结果"的单向开关 + 懒加载。**

2. **检测可选依赖的状态用 `sys.modules` 判断它是否已被加载**，
   而不是直接 import——否则就破坏了懒加载。

3. **装饰器必须保持被装饰函数的可内省特征**：在装饰时刻分支判断是否协程，
   分别生成 async / sync 包装器。`functools.wraps` 只解决名字，
   **异步性会骗过 `inspect.iscoroutinefunction`。**

4. **跨线程/跨任务的上下文传播不要依赖语言的自动机制**：
   把上下文挂在明确的对象上，在每个入口主动恢复。
   **实测数据（60% 的会话丢上下文）比"文档建议"有说服力得多。**

5. **和第三方库抢时机的问题，用"更外层、活得更久"的 span 解决**：
   库的 span 在函数返回前就关了，往关闭的 span 写属性会被静默丢弃。

6. **可选功能的上下文管理器在"关闭"时也要能正常进入**，yield 空值，
   让调用方不用写 if 分支。

7. **辅助设施的容错只能包住它自己的操作，绝不能包住业务代码**：
   用 `ExitStack` 把"进入上下文"和"块体"分开，
   否则调用方的异常会被吞掉。**随手写 `try: with ...: yield` 就错了。**

8. **切断追踪继承后要留一条可查询的链接**：把父 span 的 id 作为子 trace 的元数据，
   于是光看子 trace 就知道它的来源。

9. **给模型的工具描述里，"什么时候不要用"和"什么时候用"一样重要**：
   否则模型会对每个请求都建任务清单，**真正需要规划时反而失去信号。**

10. **把纪律写进工具描述**："同时只能有一个任务进行中"（防范围蔓延）、
    "完成时要识别实施中浮现的新工作"（防遗漏）。

11. **AI 的中间状态如果人也需要看，就存成人类可读格式放在人找得到的地方**：
    `TASKS.md` 在项目里，而不是一个 json 藏在缓存目录。

12. **让 AI 写编排代码时，把编排和执行严格分开**：脚本只能调编排原语，
    不能读写文件或执行命令——**否则所有审批和安全机制都被绕过。**

13. **并行编排要区分"有屏障"和"无屏障"**：agent 的耗时差异可能有 10 倍，
    有屏障时所有人都等最慢的那个。

14. **沙箱的语言限制要写进文档并给出替代方案**：`except*` 用不了，
    所以建议"让子 agent 返回错误哨兵值而不是抛异常"。

15. **说清 `print` 去了哪里**，否则 AI 会用它输出结果然后困惑于看不到。

16. **基于 AST 的校验永远不是完备的沙箱**：必须写清它挡不住什么
    （别名绕过），并保证"挡不住的那部分不重要"（`wf` 上没有危险能力）。
    **写出已知绕过，比假装完备安全得多。**

17. **检查函数签名要把所有参数形式都堵上**：关键字参数、`*args`、`**kwargs`、
    默认值、仅位置参数，只检查 `args` 列表会漏。

18. **禁双下划线是 Python 沙箱的基本一道**：
    `().__class__.__bases__[0].__subclasses__()` 是经典逃逸路径。
    而且属性根要一路剥到最外层的名字。

19. **给下游模型的输入要多道截断防线**：逐条累加、渲染后回弹、
    整体兜底——单个超大元素能穿透前两道。

## 剩下还没读的

- `openhands-agent-server/` 的具体路由实现（29851 行，只读了入口装配）
- `openhands-tools/` 剩下的：`ask_oracle`、`tom_consult`、`planning_file_editor`、
  `gemini/`、`glob`、`grep`
- `plugin/`、`marketplace/`、`profiles/`、`settings/`
- `acp_file_credentials.py`（536 行）
- `task/manager.py`（496 行，workflow 底下真正跑子 agent 的那一层）

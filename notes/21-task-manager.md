# 精读 20：任务管理器（task/，828 行）

> 位置：`upstream/software-agent-sdk/openhands-tools/openhands/tools/task/`
> 文件：`manager.py`（496 行）、`definition.py`（263 行）、`impl.py`（69 行）
> 前置：`notes/13-tool-implementations.md`（delegate 工具）、
> `notes/14-hooks.md`（agent 钩子的子会话）、`notes/20-observability-and-orchestration.md`（workflow）

## 这一节在讲什么

**这是"跑子 agent"的第三种实现。** 前面已经看过两种：

| 实现 | 位置 | 特点 |
|---|---|---|
| `delegate` 工具 | 第十三节 | `spawn` 先造一批，`delegate` 再派任务；子 agent 持久存在 |
| agent 钩子的子会话 | 第十四节 | 一次性，用完即弃，禁用钩子防递归 |
| **`TaskManager`** | 本节 | **一个任务一个生命周期，可暂停、可恢复、可驱逐** |

而且 `TaskManager` 是第十九节那套 `map_agents` / `pipeline` 编排原语底下**真正干活的那一层**。

**三种实现并存不是冗余**——它们的生命周期模型不同，下面会看到这个差别体现在哪。

## 一、任务是一个有状态机的对象

```python
class TaskStatus(StrEnum):
    RUNNING = "running"
    """The task is currently being processed by an agent."""
    COMPLETED = "completed"
    """The task completed successfully and returned a valid result or response."""
    ERROR = "error"
    """The task failed to complete due to an unhandled exception or system fault."""

class Task(BaseModel):
    id: str
    status: TaskStatus
    conversation_id: uuid.UUID
    result: str | None = None
    error: str | None = None
    conversation: LocalConversation | None = Field(default=None, exclude=True, ...)
```

**注意 `conversation` 字段标了 `exclude=True`** —— 序列化时不写出去。
和第十一节工具的 `executor` 一样：**运行时对象不进存档。**

但 `conversation_id` 是存的——**这就是"可恢复"的钥匙**（下面会看到）。

### 结果和错误互斥

```python
def set_result(self, result: str) -> None:
    """Set task as successful."""
    self.result = result
    self.error = None            # ← 清掉错误
    self.status = TaskStatus.COMPLETED

def set_error(self, error: str) -> None:
    """Set task as failed with an error."""
    self.error = error
    self.result = None           # ← 清掉结果
    self.status = TaskStatus.ERROR
```

**设置一个的同时清掉另一个。** 于是不可能出现"既有结果又有错误"这种矛盾状态。

> **可迁移的道理**：**互斥的字段要在同一个方法里一起设置**，
> 而不是让调用方分两步赋值（那样中间会有一个不一致的瞬间，而且容易忘掉清另一个）。
> 状态机的转移应该是原子的。

## 二、驱逐：完成之后立刻释放会话对象

```python
def _evict_task(self, task: Task) -> None:
    if task.conversation:
        task.conversation.pause()
        task.conversation.close()
    with self._tasks_lock:
        self._tasks[task.id] = task.model_copy(update={"conversation": None})
```

**任务一跑完，就 `pause()` + `close()` 会话，并把 `conversation` 字段置空。**

`_run_task` 的 `finally` 里调用它：

```python
finally:
    self._update_parent_metrics(parent, task)
    self._evict_task(task)
```

**顺序很重要：先同步用量统计，再驱逐。** 因为驱逐会销毁会话对象，
统计数据就拿不到了。

> **这是和 `delegate` 工具最大的区别。** 第十三节那个 `delegate` 把子 agent
> 留在 `self._sub_agents` 字典里持久存在（于是可以连续派多个任务）；
> **这里是跑完就释放**，因为一个"任务"的生命周期就是一次运行。
>
> **可迁移的道理**：**按"这个对象的自然生命周期"决定何时释放，而不是统一等到最后。**
> 一个会话对象持有子进程、文件句柄、网络连接——留着几十个已完成的会话会很贵。

而 `pause()` 在 `close()` 之前调用也有讲究：回想第四节，`pause()` 是
**协作式取消**（设标记让对方在安全点停）。**先让它优雅地停，再关。**

## 三、任务的"可恢复"靠什么

```python
"""The conversation linked to a completed task is persisted in a temporary
directory, ensuring the state can be restored if the task is resumed for further
work later."""
```

**会话被驱逐了，但它的事件日志已经落盘。** 恢复时用同一个 `conversation_id`
重建一个会话，它会从存档里读回全部历史：

```python
def _resume_task(self, resume: str, subagent_type: str) -> Task:
    with self._tasks_lock:
        if resume not in self._tasks:
            raise ValueError(f"Task '{resume}' not found. "
                             f"Available tasks: {', '.join(sorted(self._tasks))}")
        ...
        conversation_id = self._tasks[resume].conversation_id   # ← 复用同一个 id
        with detached_delegate_context() as link:
            conversation = LocalConversation(
                agent=worker_agent,
                conversation_id=conversation_id,      # ← 按 id 恢复
                persistence_dir=self._persistence_dir,
                ...
            )
```

**这正是第一、二节那个"不可变事件日志 + 可重放"设计的兑现。**
会话对象可以被销毁，因为它的全部状态在账本里。

**任务找不到时的报错列出了所有可用任务** ——又一次"错误信息要够用"。

### 存档位置：跟着父亲的选择

```python
def _ensure_parent(self, conversation: LocalConversation) -> None:
    if self._parent_conversation is None:
        self._parent_conversation = conversation
        parent_persistence_dir = conversation.state.persistence_dir
        if parent_persistence_dir is not None:
            self._persistence_dir = Path(parent_persistence_dir) / _SUBAGENTS_DIR
            self._persistence_dir.mkdir(parents=True, exist_ok=True)
        else:
            self._persistence_dir = Path(tempfile.mkdtemp(prefix="openhands_tasks_"))
```

**父会话持久化 → 子任务放它的 `subagents/` 子目录；
父会话不持久化 → 用一个临时目录。**

而清理逻辑正好对应：

```python
def close(self) -> None:
    """Clean up temporary directory (if used) and remove all created tasks."""
    # Only clean up when using a temp dir (parent had no persistence).
    # **When the parent persists, subagent data lives under its directory.**
    parent_persists = (self._parent_conversation is not None
                       and self._parent_conversation.state.persistence_dir is not None)
    if (not parent_persists and self._persistence_dir is not None
            and self._persistence_dir.exists()):
        shutil.rmtree(self._persistence_dir, ignore_errors=True)
```

**只删自己创建的临时目录，绝不删父会话的存档目录。**

> **可迁移的道理**：**"谁创建谁销毁"。** 如果资源是借来的（父会话的目录），
> 就不要在自己的清理逻辑里删它。这里用一个布尔判断把两种情况分开，
> 而不是"反正 rmtree 一下"。

## 四、配置继承：三处两级回退

创建任务时有三个配置走同一个模式：

```python
"""The iteration limit is resolved with the following precedence:
1. ``factory.definition.max_iteration_per_run`` (from the agent definition)
2. The parent conversation's ``max_iteration_per_run``"""
effective_max_iter = (factory.definition.max_iteration_per_run
                      if factory.definition.max_iteration_per_run
                      else self.parent_conversation.max_iteration_per_run)

# Sub-agent budget: definition value, else inherit the parent's.
effective_max_budget = (factory.definition.max_budget_per_run
                        or self.parent_conversation.max_budget_per_run)
```

```python
def _set_confirmation_policy(self, conversation, confirmation_policy) -> None:
    """Apply permission_mode: explicit mode from definition or inherit the
    parent's policy when None."""
    if confirmation_policy is None:
        conversation.set_confirmation_policy(
            self.parent_conversation.state.confirmation_policy)
    else:
        conversation.set_confirmation_policy(confirmation_policy)
```

**步数上限、预算上限、审批策略——三个都是"agent 定义里配了就用它，没配就继承父亲"。**

> 这和第十三节 `delegate` 工具的处理完全一致（那里也是审批策略默认继承）。
> **两处独立实现同一条规则**：子 agent 不能通过"我没配置"来获得比父亲更宽松的权限
> 或更大的预算。
>
> **可迁移的道理**：**派生单元的资源上限和权限，默认继承而不是取系统默认值。**
> 否则一个子 agent 可能拿到比父亲更大的预算，父亲的预算控制就形同虚设。

顺带注意 `max_budget_per_run` 用的是 `or`，而 `max_iteration_per_run` 用的是
`if ... else`。**两者对 0 的处理不同**（`0 or x` 会取 `x`，而显式判断会保留 0）——
不过步数上限为 0 本身无意义，所以这个差别在实践中不会触发。

## 五、非成功的终止一律算错误，但要保住部分结果

这是这个文件里最值得学的一段。

```python
status = task.conversation.state.execution_status
if status == ConversationExecutionStatus.FINISHED:
    result = get_agent_final_response(task.conversation.state.events)
    task.set_result(result)
    logger.info(f"Task '{task.id}' completed.")
else:
    # **Any non-FINISHED terminal status (run-limit, stuck, paused, ...) is
    # surfaced as an error, not an empty "completed"; the detail keeps partial
    # output so the parent can use/retry it.**
    task.set_error(self._run_stop_detail(task.conversation, status))
    logger.warning(f"Task '{task.id}' stopped: status '{status.value}'.")
```

**只有 `FINISHED` 算成功。** 超步数、卡死、被暂停——全部算错误。

**为什么不能算"完成但结果为空"？** 因为父 agent 收到一个空结果时无法区分
"子 agent 认真做了但确实没什么可报告的"和"子 agent 撞上步数上限被砍掉了"。
**前者可以继续，后者应该重试或换策略。**

> 这是"未知/失败必须可检测"这条原则的**第七次出现**（前六次在第六、七、
> 十一、十四、十五、十六节）。

### 但错误里要带上部分结果

```python
@staticmethod
def _run_stop_detail(conversation, status) -> str:
    """Why a sub-agent stopped without finishing (run-limit, stuck, paused, ...),
    **plus any partial output so the parent isn't left with nothing to use.**"""
    errors = [e for e in conversation.state.events
              if isinstance(e, ConversationErrorEvent)]
    reason = (errors[-1].detail if errors
              else f"Sub-agent stopped without finishing (status: {status.value}).")
    partial = get_agent_final_response(conversation.state.events)
    return f"{reason}\nPartial result:\n{partial}" if partial else reason
```

**三个细节：**

**① 优先用最后一条结构化错误事件的详情**（`errors[-1].detail`），
没有才用一个泛化的描述。

> 回想第四节那个"兜底错误处理要先确认没人报过更好的错"——
> **同一个原则的第四次出现。**

**② 把部分输出拼进错误信息里。** 子 agent 跑了 400 步撞上上限，
它前面做的工作不该被完全丢弃。

**③ 只在真有部分输出时才拼**（`if partial else reason`），
避免产生一个 `Partial result:\n` 后面空着的字符串。

> **可迁移的道理**：**失败的结果里要带上已完成的部分。**
> "失败"和"什么都没有"是两件事——一个跑了 90% 才失败的任务，
> 那 90% 对调用方可能仍然有用。

## 六、子 agent 的用量统计：替换而不是合并

```python
def _update_parent_metrics(self, parent, task) -> None:
    """Sync sub-agent metrics into parent before eviction destroys the conversation.
    **Replace (not merge) because sub-agent metrics are cumulative across resumes.**"""
    if task.conversation is not None:
        parent.conversation_stats.usage_to_metrics[f"task:{task.id}"] = (
            task.conversation.conversation_stats.get_combined_metrics())
```

**注释解释了为什么是替换：子 agent 的统计本身已经是跨多次恢复累积的了。**

想一下：任务 A 第一次跑花了 $1，父亲记 $1。恢复后又跑了 $0.5，
**子会话的累计统计是 $1.5**（因为它从存档里读回了历史）。
**如果合并，父亲会记成 $1 + $1.5 = $2.5 —— 多算了一倍。**

> **这是"从存档恢复 + 累积统计"组合下的一个经典陷阱。**
> 判断标准：**被同步的那个值是"增量"还是"累计值"？**
> 增量要加，累计值要替换。
>
> 而且注意键名是 `f"task:{task.id}"` —— 按任务 id 分桶，
> 于是"每个任务花了多少"可以单独看到。这和第十四节钩子的 `usage_id`、
> 第十五节路由分类器的 `classifier:<名字>` 是同一套命名约定。

## 七、和前两种实现的三处相同

### 相同点 1：关掉流式输出

```python
llm_updates: dict = {"stream": False}
sub_agent_llm = parent_llm.model_copy(update=llm_updates)
sub_agent_llm.reset_metrics()

sub_agent = factory.factory_func(sub_agent_llm)

# ensuring that the sub-agent LLM has stream deactivated
sub_agent = sub_agent.model_copy(
    update={"llm": sub_agent.llm.model_copy(update={"stream": False})})
```

**关了两次。** 第一次是给工厂函数的输入，第二次是**工厂函数可能自己换了一个 LLM**，
所以出来之后再关一遍。

> **可迁移的道理**：**给工厂/回调传配置之后，要重新检查结果是否仍满足约束。**
> 工厂可以忽略你传的东西。

### 相同点 2：可视化器要新建子实例

```python
visualizer = None
if parent_visualizer is not None:
    label = description or task_id
    visualizer = parent_visualizer.create_sub_visualizer(label)
```

**和第十三、十四节完全一样的处理。** 标签优先用人类可读的描述，退化到任务 id。

### 相同点 3：审批循环一模一样

```python
def _run_until_finished(self, task_id: str, conversation) -> None:
    """Run a sub-agent conversation to completion, handling confirmations."""
    conversation.run()
    while conversation.state.execution_status == WAITING_FOR_CONFIRMATION:
        pending = ConversationState.get_unmatched_actions(conversation.state.events)
        if not pending:
            break
        if self._confirmation_handler is None or self._confirmation_handler(task_id, pending):
            conversation.run()
        else:
            conversation.reject_pending_actions("User rejected the actions")
            conversation.run()
```

**和第十三节 `delegate` 工具里那个函数几乎逐字相同**（只有第一个参数名从
`agent_id` 变成 `task_id`）。

> **三处独立实现里有一处逐字重复的代码，这是一个真实的设计信号**：
> 说明"驱动一个子会话跑到完成并处理审批"这件事应该被提取成一个共享函数。
> 现在的状态是可维护性的隐患——**改一处忘一处会导致两种子 agent 行为不一致。**
>
> 这也是读源码的价值之一：**重复代码往往标记出一个还没被抽象出来的概念。**

## 八、三个只有这里有的东西

### ① 切断追踪继承

```python
with detached_delegate_context() as link:
    return LocalConversation(..., observability_metadata=..., observability_tags=["delegate"])
```

**这是第十九节那个 `detached_delegate_context` 的唯一使用方。**
子任务开一条独立的 trace，同时把父 span 的 id 作为元数据带上。

元数据的构造：

```python
def _delegate_observability_metadata(self, task_id, subagent_type, link):
    return {
        "is_delegate": True,
        "task_id": task_id,
        "subagent_type": subagent_type,
        "parent_session_id": str(self.parent_conversation.state.id),
        **link,          # ← delegate.parent_trace_id / parent_span_id / tool_call_id
    }
```

**四个自己的字段 + 追踪层给的链接。** 于是在追踪界面上可以按
`is_delegate=True` 过滤出所有子任务、按 `subagent_type` 分组看哪种子 agent 最慢、
按 `parent_session_id` 找到某次对话派出的所有子任务。

> **可迁移的道理**：**给追踪加元数据时，要想清"我以后会按什么维度查"。**
> 布尔标记（能过滤）、类型（能分组）、父 id（能关联）——三类各有用途。

### ② 共享提示缓存分片

```python
prompt_cache_key=str(parent.state.id),
```

回想第九节那个提示缓存（前缀相同就走缓存，价格降到 1/10）。

**子任务用父会话的 id 作为缓存键**，于是**父子共享同一个缓存分片**。
因为它们跑在同一个工作区、系统提示的静态部分基本相同——**共享缓存能省一大笔钱。**

> 第九节 `LLM` 的字段注释里正好解释了这个：
> `Sub-conversations set this to the parent's ID to share the same cache shard.`
> **这里是那句话的使用方。**

### ③ 申报"我不碰任何共享资源"

```python
def declared_resources(self, action: Action) -> DeclaredResources:
    return DeclaredResources(keys=(), declared=True)
```

回想第十一节那个微妙区分：`declared=True, keys=()` 是**"我想过了，我是安全的"**，
不是"我没想过"。

**于是多个 task 工具调用可以完全并行，不需要任何锁。**

这个申报是对的吗？**是的**——`TaskTool` 本身只是启动一个子会话，
它自己不碰文件、不碰终端。**真正的资源竞争发生在子 agent 内部**，
而子 agent 有自己的资源锁管理器（第三节那个"每个执行器有自己的锁管理器"）。

> **可迁移的道理**：**资源申报的粒度要和锁的作用域对齐。**
> 这一层不碰资源，就诚实地申报空——把竞争交给真正发生竞争的那一层去管。

## 九、工具描述：教模型什么时候不要用

`TASK_TOOL_DESCRIPTION` 的结构和第十九节 `task_tracker` 一样，
**先说什么时候用，再说什么时候不要用。**

```
Subagents are autonomous agents that work independently and return results to you.
They are your primary tool for understanding codebases and running tests, **but each
delegation has overhead — use them when the task genuinely benefits from a separate
agent, not for simple lookups.**

When NOT to use the task tool:
- **A single grep, find, or cat command would answer your question — just run it yourself**
- You are making a file edit (use file_editor directly)
- You already have the context needed
```

**"每次委派都有开销"** 这句话把成本讲明了。三条禁用情形都很具体。

### 四条使用建议，最后一条很微妙

```
When using the task tool:
- Write a detailed prompt describing exactly what you need
- Include specific file paths, class names, or error messages from the issue
- **Tell the agent what to report back (file paths, line numbers, code snippets)**
- **The agent's results are authoritative — verify subagent results only when the
  task involves judgment or interpretation.**
```

**第三条**：要告诉子 agent **回报什么格式**。因为子 agent 的输出会变成父 agent
上下文里的一段文字——**如果它回报了一大段散文而不是具体的文件路径和行号，
父 agent 还得再派一次。**

**第四条最微妙**：`The agent's results are authoritative` ——
**默认相信子 agent 的结果，只在涉及判断或解释时才验证。**

> **为什么要这么说？** 因为模型有一种倾向：派出去之后自己再去核对一遍，
> **于是委派的成本节约完全消失了**（还多花了一次子 agent 的钱）。
>
> 但也不是无条件相信——"涉及判断或解释"时要验证。
> **事实查找（这个函数在哪个文件）可以信，判断（这个设计好不好）要核。**
>
> **可迁移的道理**：**委派机制必须明确"结果的可信度"，否则调用方会重复劳动。**
> 而且可信度要分类：事实性结论可信，判断性结论需要复核。

### 描述是动态拼出来的

```python
TASK_TOOL_DESCRIPTION = """...
Available agent types and the tools they have access to:
{agent_types_info}
...
{task_tool_examples}
"""
```

**可用的子 agent 类型列表和它们各自的工具是运行时填进去的**
（从 `get_registered_agent_definitions()` 取）。

而示例是**按子 agent 类型分别准备的**：

```python
TASK_TOOL_EXAMPLES: Final[dict[str, str]] = {
    "code-explorer": """Example — Multi-step exploration (good use of code-explorer): ...""",
    "bash-runner": """Example — Running tests (good use of bash-runner): ...""",
    "web researcher": """...""",
    "general purpose": """...""",
}
```

**注册了哪些子 agent，就只放哪些示例。** 于是提示里不会出现"用 web researcher
干这个"而实际上这个类型没注册。

> 回想第九节那条"功能没开的提示片段完全不出现"——**同一个原则在工具描述层面的应用。**

`bash-runner` 那个示例特别值得看：

```
prompt="Run: cd /workspace/django && python tests/runtests.py utils_tests.test_dateformat
-v 2. Provide a summary including the total tests run, the final status, and a list of
any failing test names. For each failure, include the specific cause or assertion error,
**but do not include the full stack trace or the verbose setup/teardown output.**"
```

**示例本身在演示"怎么要求子 agent 裁剪输出"** ——跑测试的完整输出可能几万行，
明确说"要失败原因，不要完整堆栈和啰嗦的初始化输出"。

> **这是一个很聪明的教学方式**：不是在描述里写一条规则"记得让子 agent 裁剪输出",
> 而是**在示例里示范出来**。模型模仿示例比遵守抽象规则更可靠。

## 十、一个被废弃但保留的参数

```python
max_turns: SkipJsonSchema[int | None] = Field(
    default=None,
    description="Deprecated: This field is ignored and will be removed in version 2. "
    "Maximum iterations are now determined by the agent definition or parent "
    "conversation.",
    deprecated=True,
    ge=1,
)
```

**三重标记**：`SkipJsonSchema`（不出现在给模型的 schema 里）、
`deprecated=True`（工具链能识别）、描述里写明"被忽略，v2 会删"。

**`SkipJsonSchema` 是关键**：模型看不到这个参数，所以不会再传它；
但如果有老的调用方（或者存档里的老事件）带着这个参数，**反序列化不会失败。**

> **可迁移的道理**：**废弃一个模型可见的参数，要分两步**：
> 先从 schema 里隐藏（模型不再用），再等一个大版本删掉字段（老数据还能读）。
> 直接删会让历史事件反序列化失败——回想第一节那个 `extra="forbid"`。

## 十一、这一节的可迁移结论

1. **互斥的字段要在同一个方法里一起设置**：`set_result` 清掉 error，
   `set_error` 清掉 result。状态机的转移应该是原子的。

2. **按"这个对象的自然生命周期"决定何时释放，而不是统一等到最后**：
   任务跑完立刻 `pause()` + `close()` + 置空引用。
   **一个会话持有子进程、文件句柄、网络连接，留着几十个已完成的很贵。**

3. **释放前先同步需要的数据**：`finally` 里先 `_update_parent_metrics`
   再 `_evict_task`，因为驱逐会销毁数据源。

4. **"谁创建谁销毁"**：只删自己 `mkdtemp` 出来的临时目录，
   绝不删父会话借来的存档目录。用一个布尔判断分开，而不是"反正 rmtree 一下"。

5. **派生单元的资源上限和权限默认继承，不取系统默认值**：
   步数、预算、审批策略三处都是"定义里配了用它，没配继承父亲"。
   **否则子 agent 可能拿到比父亲更大的预算。**

6. **只有明确的成功状态才算成功**：超步数、卡死、被暂停全算错误。
   **空结果无法区分"认真做了没什么可说"和"被砍掉了"。**

7. **失败的结果里要带上已完成的部分**：跑了 90% 才失败的任务，
   那 90% 对调用方可能仍然有用。而且**只在真有部分输出时才拼**。

8. **同步统计时要区分"增量"还是"累计值"**：子 agent 的统计跨恢复累积，
   所以是**替换**而不是合并——合并会多算一倍。
   **这是"从存档恢复 + 累积统计"组合下的经典陷阱。**

9. **给工厂/回调传配置之后，要重新检查结果是否仍满足约束**：
   工厂可能自己换了一个 LLM，所以出来之后再关一次流式。

10. **资源申报的粒度要和锁的作用域对齐**：这一层不碰资源就诚实申报空，
    把竞争交给真正发生竞争的那一层（子 agent 自己的锁管理器）。

11. **给追踪加元数据时要想清"以后会按什么维度查"**：
    布尔标记能过滤、类型能分组、父 id 能关联。

12. **委派机制必须明确"结果的可信度"**，否则调用方会重复劳动、
    抵消委派的收益。而且**可信度要分类**：事实性结论可信，判断性结论需要复核。

13. **要求委派方说明"回报什么格式"**：否则子 agent 返回一段散文，
    调用方还得再派一次。

14. **在示例里示范规则，而不是在描述里陈述规则**：
    "让子 agent 裁剪输出"写成一个具体的 prompt 示例，模型模仿示例比遵守抽象规则可靠。

15. **动态拼装工具描述，只放实际注册了的东西**：
    没注册的子 agent 类型不出现在示例里。

16. **废弃模型可见的参数要分两步**：先 `SkipJsonSchema` 从 schema 隐藏
    （模型不再用），再等大版本删字段（老存档还能反序列化）。

17. **重复代码往往标记出一个还没被抽象出来的概念**：
    三处子 agent 实现里那段"跑到完成并处理审批"的循环逐字重复，
    **这是可维护性隐患——改一处忘一处会导致行为不一致。**

## 剩下还没读的

- `openhands-agent-server/` 的具体路由实现（29851 行，只读了入口装配）
- `openhands-tools/` 剩下的：`ask_oracle`、`tom_consult`、`planning_file_editor`、
  `gemini/`、`glob`、`grep`
- `plugin/`、`marketplace/`、`profiles/`、`settings/`、`subagent/`
- `acp_file_credentials.py`（536 行）
- `workflow/impl.py` 剩下的部分（`WorkflowContext` 的具体实现）

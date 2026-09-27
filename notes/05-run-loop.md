# 精读 04：外层循环（conversation/impl/local_conversation.py 的 run）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/conversation/impl/local_conversation.py:1903`
> 相关：`conversation/stuck_detector.py`、`conversation/state.py`、`conversation/fifo_lock.py`
> 前置：`notes/04-agent-step.md`（单步决策）

## 这一节在讲什么

上一节的 `_step()` 只管"走一步"。这一节管**"要不要再走一步"**。

听起来是个 while 循环的事，但真正要处理的是：钱花超了怎么办、用户中途按暂停怎么办、
AI 原地打转怎么办、AI 刚说完"我做完了"而你同时发来新消息怎么办。

**这一节是全项目最能看出"跑过生产"的地方。**

## 一、骨架

```python
def run(self) -> None:
    self._ensure_agent_ready()
    self._cancel_token = CancellationToken()

    iteration = 0
    while True:
        with self._state:                 # ← 每一轮都要拿状态锁
            if 暂停或卡死:      break
            if 已完成:          询问停止钩子，允许才 break
            if 卡死检测要处理:  continue
            清除等待确认标记
            self.agent.step(...)          # ← 上一节讲的那一步
            iteration += 1
            if 等待确认:        break
            if 超预算:          报错 break
            if 超步数上限:      报错 break
```

注意循环里**没有**"检查是否完成"这一项放在 step 之后——这是故意的，第五节细说。

## 二、`with self._state`：为什么每轮都要拿锁

```python
with self._state:
    ...
```

> **锁（lock）**：多个线程同时改一份数据会出乱子，锁保证"同一时刻只有一个人能改"。

这个会话可能被多个线程同时操作：循环在跑、你在另一个线程发消息、界面在另一个线程读状态。
所以每轮都要独占状态。

用的锁是 `FIFOLock`（`conversation/fifo_lock.py`）—— **先进先出锁，先等的先拿到**。

> Python 自带的锁**不保证顺序**。谁抢到算谁的，理论上可能有某个请求一直抢不到，
> 这叫**饥饿**（starvation）。

**为什么这里非要先进先出？** 因为它影响的是**用户体验的正确性**，不只是公平。
第五节那个"用户消息不能丢"的设计，正是建立在"先来的操作一定先被处理"这个保证上的。

另外有个很实在的坑，代码里用一个字段专门记着：

```python
# True while run()/arun() executes an agent step while holding the state
# lock. A state-mutating tool (e.g. switch_llm) runs on a worker thread, so
# it must skip re-acquiring the lock the run loop holds while blocked
# awaiting that tool (#3485).
_step_holds_state_lock: bool
```

翻译：循环拿着锁去执行一步；这一步里某个工具（比如"切换模型"）要改状态，
而它跑在另一个线程上——如果它也去拿锁，就会**等一个永远不会被释放的锁**（因为持锁的
循环正在等它执行完）。

> 这就是**死锁**：A 等 B 完成，B 等 A 放锁。

解法是留一个标记："现在锁在循环手里，你别去抢了，直接改。"注释还挂了 issue 编号 #3485
——说明这是**真的踩到过**才加的。

## 三、暂停：为什么能"从任何线程"喊停

```python
def pause(self) -> None:
    """This method can be called from any thread to request that the agent
    pause execution. The pause will take effect at the next iteration
    of the run loop (between agent steps).

    Note: If called during an LLM completion, the pause will not take
    effect until the current LLM call completes.
    """
```

实现只是**把状态改成 PAUSED**，然后循环下一轮开头自己发现、自己退出。

**这个设计叫"协作式取消"** —— 不强行杀线程，而是设一个标记让对方在安全的地方自己停。

> 强行杀线程的问题：可能停在一个"改了一半"的地方，数据就坏了。
> 协作式取消保证只在明确安全的点停下来。

文档字符串诚实地写明了代价：**如果正在等模型返回，暂停要等这次调用结束才生效。**
把限制写清楚，而不是让用户以为点了就立刻停。

## 四、两道硬闸门：步数和钱

### 步数

```python
max_iteration_per_run: int = 500
```

超了就报错并结束。但注意这里有个细节：

```python
if iteration >= self.max_iteration_per_run:
    # If the agent finished on this final iteration, preserve the FINISHED
    # status rather than overwriting it with ERROR.
    if self._state.execution_status == ConversationExecutionStatus.FINISHED:
        break
    ... 设成 ERROR ...
```

**如果 AI 正好在最后一步完成了任务，不能报错。** 不加这个判断，一次成功的运行会被
标记成失败——只因为它刚好用满了配额。这种边界处理是区分"能用"和"好用"的地方。

### 钱

```python
def _budget_exceeded_detail(self) -> str | None:
    """Error detail if the run has hit its cost budget, else None.

    Bounds total spend across all of the run's LLMs (agent, condenser, ...),
    complementing the iteration cap which only bounds step count.
    """
    spent = self.conversation_stats.get_combined_metrics().accumulated_cost
    if spent < self.max_budget_per_run:
        return None
    return f"Agent reached maximum budget limit (${...:.4f}); accumulated cost ${spent:.4f}."
```

注释点明了两道闸门的分工：**步数管次数，预算管钱**。为什么两个都要？因为一步的花费差别
可以很大——读一个 20 万字的文件和读一行，都算一步。

关键在 `get_combined_metrics()`：**把这次运行用到的所有模型的花费加在一起**
（主模型、压缩用的模型、评审用的模型……）。只算主模型的花费会漏掉一大块。

两道闸门都走同一个收尾函数：

```python
def _emit_run_limit_error(self, code: str, detail: str) -> None:
    logger.error(detail)
    self._state.execution_status = ConversationExecutionStatus.ERROR
    self._on_event(ConversationErrorEvent(source="environment", code=code, detail=detail))
```

**注意错误也是记进账本的一条事件。** 于是"为什么这次运行停了"是可查的历史，
不是只存在于日志里的东西。

## 五、最精妙的地方：故意不检查"已完成"

循环里 `step()` 之后检查了一堆状态，但**没有检查 FINISHED**。代码里有一大段注释解释：

```python
# Check for non-finished terminal conditions
# Note: We intentionally do NOT check for FINISHED status here.
# This allows concurrent user messages to be processed:
# 1. Agent finishes and sets status to FINISHED
# 2. User sends message concurrently via send_message()
# 3. send_message() waits for FIFO lock, then sets status to IDLE
# 4. Run loop continues to next iteration and processes the message
# 5. Without this design, concurrent messages would be lost
```

### 拆开讲这个竞态

> **竞态条件（race condition）**：两件事同时发生，结果取决于谁先谁后，于是行为不确定。

场景：AI 刚刚判断"任务完成了"，几乎同一瞬间你打字发来"等等，还要加个功能"。

**如果在 step 之后立刻检查 FINISHED 就退出**，会发生什么？

```
时刻 1：step() 里 AI 说完成了，状态 = FINISHED
时刻 2：你的 send_message() 在另一个线程排队等锁
时刻 3：循环检查到 FINISHED → 直接 break，退出
时刻 4：你的消息终于拿到锁，把状态改成 IDLE……但循环已经没了
        → 你的消息静静地躺在账本里，永远没人处理
```

**你会以为 AI 无视了你。** 这种 bug 极难复现——只在你刚好在那个瞬间发消息时出现。

作者的解法是：**step 之后不退出，让循环再转一圈**。这样：

```
时刻 3：循环转到下一轮开头，要拿锁
时刻 4：因为是先进先出锁，你的 send_message() 先排队，所以它先拿到
        → 状态被改成 IDLE，消息进账本
时刻 5：循环拿到锁，发现状态不是 FINISHED 了，继续处理你的消息 ✓
```

**注意这里两个设计是咬合在一起的**：
- 循环多转一圈，给并发操作留出插入的窗口
- 先进先出锁保证"先来的 send_message 一定先被处理"，而不是碰运气

少任何一个都不成立。这就是为什么前面说 FIFO 锁影响的是**正确性**，不只是公平。

`send_message` 那边的对应代码（`local_conversation.py:1832`）：

```python
with self._state:
    if self._state.execution_status in (ConversationExecutionStatus.FINISHED, ...):
        self._state.execution_status = ConversationExecutionStatus.IDLE
```

**"已完成"不是终点，是可以被新消息唤醒的一个状态。** 这才符合人的直觉——
对话框还开着，我当然可以接着说话。

## 六、钩子可以否决"结束"

```python
if self._state.execution_status == ConversationExecutionStatus.FINISHED:
    if self._hook_processor is not None:
        should_stop, feedback = self._hook_processor.run_stop(reason="agent_finished")
        if not should_stop:
            logger.info("Stop hook denied agent stopping")
            if feedback:
                prefixed = f"{ACP_STOP_HOOK_FEEDBACK_PREFIX} {feedback}"
                feedback_msg = MessageEvent(source="environment", ...)
                self._on_event(feedback_msg)
            self._state.execution_status = ConversationExecutionStatus.RUNNING
            continue        # ← 不让走，接着干
    break
```

你可以配一个"停止钩子"，在 AI 宣布完成时自动跑。典型用法：跑一遍测试，没过就不许结束。

**钩子不只能说"不行"，还能附上理由（feedback）**，这条理由会作为一条消息进账本，
AI 下一轮就看到了"测试没过，错误是 xxx"。

注意反馈消息的 `source="environment"` —— 上一节讲过，来源身份是如实记录的，
不会伪装成你说的话。

> 这个机制的价值：**把"什么叫做完了"的定义权交给使用者。** AI 自己觉得完成了不算，
> 你的测试通过了才算。

## 七、卡死检测：先推一把，再判死

循环里这一行：

```python
if self._check_stuck_or_nudge():
    continue
```

它的实现体现了一个很好的分级思路（`local_conversation.py:734`）：

```python
def _check_stuck_or_nudge(self) -> bool:
    """Nudge once on a repeating action-error streak, else apply is_stuck()."""
    nudge = self._stuck_detector.get_action_error_nudge()
    if nudge is not None:
        self._on_event(MessageEvent(source="environment", ...nudge...))
        return False          # ← 不算卡死，给一次机会

    if self._stuck_detector.is_stuck():
        logger.warning("Stuck pattern detected.")
        self._state.execution_status = ConversationExecutionStatus.STUCK
        return True           # ← 判死，停
    return False
```

**两级处理：先提醒，提醒无效才判死。**

### 那句提醒写得很好

```python
return (
    f"You've called `{action.tool_name}` with the same arguments "
    f"{threshold} times in a row and gotten the same error each "
    f"time: {error.error}. Repeating the exact same call again "
    "will not work — review the error message and either correct "
    "the arguments or try a different approach."
)
```

翻译："你用同样的参数调用 `xxx` 已经连续 N 次、每次都得到同样的错误：`yyy`。
再调一次同样的不会有用——看一下错误信息，要么改参数，要么换个思路。"

**三个要素**：说清事实（几次、什么错）、明确否定当前做法（再来一次没用）、给出两条出路。
这比干巴巴一句"你卡住了"有用得多。

### 一个防重复的细节

```python
# Id of the AgentErrorEvent already nudged for, so a frozen streak
# (e.g. an empty/reasoning-only response that adds no new action)
# doesn't re-emit the same nudge every iteration.
self._last_nudged_error_event_id: str | None = None
```

场景：提醒发出去了，但 AI 这一轮只输出了思考内容、没产生新动作（上一节讲的
`REASONING_ONLY` 那种）。于是连错次数没变，下一轮又满足提醒条件——**同一句提醒会一直刷**。

解法：记住"已经为哪条错误提醒过了"，同一条不再提醒。

> 这类 bug 的共同形态：**触发条件基于"状态"而非"变化"，状态不动就会反复触发。**
> 凡是"满足条件就发通知"的代码都该想一下这个问题。

### 检测的四种模式

```python
1. Repeating action-observation cycles     重复的动作-结果循环
2. Repeating action-error cycles           重复的动作-报错循环
3. Agent monologue                         自言自语（连续多条 AI 消息、无用户介入）
4. Repeating alternating action-observation 交替往复（A→B→A→B）
```

扫描窗口的定义也讲了理由：

```python
# Maximum recent events to scan for stuck detection.
# This window should be large enough to capture repetitive patterns
# (4 repeats × 2 events per cycle = 8 events minimum, plus buffer for user messages)
MAX_EVENTS_TO_SCAN_FOR_STUCK_DETECTION: int = 20
```

**20 不是随手写的**：4 次重复 × 每轮 2 个事件 = 至少 8 个，再给用户消息留点余量。
所有阈值都可以配置（`StuckDetectionThresholds`）。

第 4 种"交替往复"值得单独说：AI 可能不是傻傻重复同一件事，而是在两件事之间来回
（改 A 导致 B 坏、改 B 导致 A 坏）。这种"看起来一直在做新事情"的死循环更难发现。

## 八、异常兜底：不要覆盖更好的错误信息

```python
except Exception as e:
    with self._state:
        self._state.execution_status = ConversationExecutionStatus.ERROR

        # Add an error event — unless the agent already surfaced a typed,
        # detailed one for this failure (e.g. ACPAgent._emit_turn_error),
        # which a generic str(e) duplicate would otherwise clobber.
        if not _agent_already_surfaced_error(self._state.events, _run_start_event_count):
            self._on_event(ConversationErrorEvent(code=e.__class__.__name__, detail=str(e)))

    raise ConversationRunError(
        self._state.id, e,
        persistence_dir=self._state.persistence_dir,
        conversation_error=_latest_conversation_error(...),
    ) from e
finally:
    self._cancel_token = None
```

三个点：

**① 先查有没有人已经报过更好的错。** 内层可能已经发出了一条带类型、带细节的错误事件；
外层再补一条 `str(e)` 的泛化版本，会把好的那条**盖掉**（注释用的词是 clobber）。
所以先检查。

> 这个模式很值得记：**兜底的错误处理要先确认"还没人处理过"**，否则兜底会毁掉上游的
> 精确诊断。很多项目的日志之所以难读，就是每一层都往上包一层没信息量的错误。

**② 重新抛出时附上"接下来去哪看"。** `ConversationRunError` 带上了会话 ID 和
**存档目录**。注释说 `for better UX` —— 出错时直接告诉你现场在哪个目录，
而不是让你自己去翻。

**③ `raise ... from e`** 保留原始异常链，不丢原始堆栈。

**④ `finally` 里清掉取消令牌。** 无论怎么退出都要清，否则下次运行会拿到一个上次遗留的、
可能已经被标记为"已取消"的令牌。

## 九、顺带看到的两个东西

### 协作式取消的对外接口

```python
@property
def cancel_token(self) -> CancellationToken | None:
    """Tools that want cooperative cancellation can check this during execution::

        if conversation and conversation.cancel_token:
            if conversation.cancel_token.is_cancelled:
                return Observation(output="Cancelled")
    """
```

长时间运行的工具（比如跑一个十分钟的测试）可以**自己检查"是不是该停了"**，
然后干净地返回一个"已取消"的结果——而不是被强行掐断。

注意它返回的是一个**正常的 Observation**，不是异常。于是上一节那条铁律
（每个调用必须有恰好一个结果）依然成立，历史不会坏。

### fork：分支的对外接口

```python
def fork(self, *, conversation_id=None, from_event_id=None, ...) -> "LocalConversation":
    """Deep-copy this conversation with a new ID.

    Events are copied so the source remains immutable. The fork starts
    in ``execution_status='idle'``; calling ``run()`` resumes from the ...
    """
```

第一节讲的"事件树可以分叉"，在这里有了用户能直接调用的形式：**从某个事件开始，
复制出一条新的会话线**。原会话一个字不动。

`from_event_id` 参数意味着你可以说"回到第 37 个事件那时候，从那里换条路走"。

## 十、这一节的可迁移结论

1. **协作式取消，不要强杀**：设标记让对方在安全点自己停，并且把"什么时候才会真的停"
   写进文档。

2. **多道闸门各管一维**：步数管次数，预算管钱，而且预算要**把所有模型的花费加总**。

3. **"完成"不是终点，是可被唤醒的状态**：对话框还开着，用户当然可以接着说话。
   为此宁愿多转一圈循环。

4. **并发的正确性可能依赖锁的公平性**：先进先出锁 + 多转一圈，两个设计咬合才保住
   "用户消息不会丢"。任何一个换掉都不成立。

5. **判死之前先给一次带信息的提醒**：说清事实 + 否定当前做法 + 给出出路。
   并且**记住已经提醒过什么**，否则状态不变会一直刷。

6. **兜底的错误处理要先确认没人处理过**：否则泛化的 `str(e)` 会盖掉上游的精确诊断。

7. **出错时告诉用户现场在哪**：附上会话 ID 和存档目录，而不是让人自己翻。

8. **边界情况要照顾成功路径**：最后一步刚好完成，不能因为用满配额就被判失败。

## 下一站

`context/condenser/llm_summarizing_condenser.py` —— 压缩器的具体策略：
什么时候压、压多少、保留哪些、压不动了怎么办。
它会用到第二节讲的"可操作位置"来决定切口。

# 精读 03：AI 决策的一步（agent/agent.py 的 _step）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/agent/agent.py:645`
> 相关：`agent/utils.py:581`、`agent/parallel_executor.py`、`conversation/resource_lock_manager.py`
> 前置：`notes/02-event-model.md`（账本）、`notes/03-view-and-properties.md`（视图）

## 这一节在讲什么

前两节讲的是**数据**：事情怎么记（账本）、怎么整理成能给 AI 看的形式（视图）。

这一节讲**动作**：AI 走一步，具体发生了什么。

一句话概括这一步：**整理历史 → 问模型 → 模型要调工具 → 校验 → 执行 → 把结果记回账本。**
难的地方全在"但是"里。

## 一、先注意一件事：循环不在这里

```python
def _step(self, conversation, on_event, stream) -> None:
```

注意**返回值是 None**，而且函数名叫"一步"。这个函数**只走一步就返回**，循环在外面
（`conversation.run()`，下一节讲）。

**为什么这样切？** 因为"走一步"和"什么时候该停"是两件不同的事。前者关心怎么和模型交互，
后者关心预算够不够、用户有没有按暂停、是不是卡死了。混在一起写，两边都会变得没法测试。

> 这种切分叫**关注点分离**（separation of concerns）——每段代码只操心一件事。
> 说起来简单，能在一个 1500 行的文件里坚持住不容易。

## 二、动手之前的四道门

在真正问模型之前，`_step` 先做四件事，每件都是一道"可能直接返回、这一步就不问模型了"的门。

### 门 1：有没有上次批准了但还没执行的动作

```python
pending_actions = ConversationState.get_unmatched_actions(state.active_branch())
if pending_actions:
    logger.info("Confirmation mode: Executing %d pending action(s)")
    self._execute_actions(conversation, pending_actions, on_event)
    return        # ← 执行完就返回，这一步不问模型
```

这是**确认模式**（confirmation mode）的实现方式，很聪明：

> **确认模式**：AI 每要做一件（有风险的）事，先停下来问你同不同意。

传统做法是在代码里阻塞住、等一个 yes/no 的输入。这里的做法是：
1. 第一次走这一步 —— 生成动作，**但不执行**，状态改成"等待确认"，返回
2. 你看完觉得可以，**再调一次 `run()`**
3. 第二次走这一步 —— 发现有"生成了但没执行"的动作，**把"你又调了一次"本身当成批准**，执行它们

`get_unmatched_actions` 的名字点明了原理：**找那些"有调用、没结果"的动作**。
上一节讲的"调用必须配对结果"那条规则，在这里反过来被当成了状态的判断依据——
配不上对的，就是还没执行的。

**同一个数据结构，既用于保证正确性，又用于表达状态。** 这是设计得好的信号。

### 门 2：这条用户消息是不是被钩子拦了

```python
reason = state.pop_blocked_message(state.last_user_message_id)
if reason is not None:
    state.execution_status = ConversationExecutionStatus.FINISHED
    return
```

你配置的钩子脚本（比如"检查用户消息里有没有密钥"）可以拦下一条消息。拦了就直接结束，
不浪费一次模型调用。

注意 `pop` 这个词——取出并删除。**拦截理由只消费一次**，不会在后面的步骤里重复触发。

### 门 3：先搞清楚模型的真实容量

```python
self.llm.resolve_runtime_metadata()
```

注释解释了为什么这一行必须在压缩判断之前：

> before the condenser decides a token threshold, so a routed model's real endpoint
> limit drives condensation on the first step
> （在压缩器决定 token 阈值之前执行，这样被路由的模型的真实端点上限，从第一步就能主导压缩判断）

**什么意思**：现在很多配置会做"模型路由"——你写一个名字，实际请求可能被转发到不同的后端，
而不同后端的容量不一样。如果不先问清楚，压缩器就会拿一个猜的数字去判断"要不要压缩"，
第一步就可能压错。

注释还标注了 `(cached, no I/O on a hit)` —— 缓存命中时不产生网络请求。**这是在替读者
回答"每步都调一次不会很慢吗"。**

### 门 4：整理历史，但可能变成"先去压缩"

```python
_messages_or_condensation = prepare_llm_messages(
    state.view, condenser=self.condenser, llm=self.llm
)
if isinstance(_messages_or_condensation, Condensation):
    on_event(_messages_or_condensation)
    return        # ← 这一步只做压缩，不问模型
```

这个函数的返回类型是 `list[Message] | Condensation` —— **要么给你整理好的消息，
要么告诉你"太长了，我先压缩一下"**。

压缩记录被 `on_event` 记进账本，然后返回。下一轮循环重新进来时，视图已经是压缩后的了。

**关键点：压缩自己也占用一步。** 它不是偷偷在后台做掉的，而是账本上明明白白一行。
这正是上一节"压缩是可重放的记录"在流程上的体现。

顺带一个性能细节，注释里带着 issue 链接：

> Callers should pass the cached `ConversationState.view`, which is maintained
> incrementally as events are appended. This avoids paying the O(n)
> `View.from_events` (with `enforce_properties`) cost on every step.

翻译：视图是**增量维护**的（新事件来了就往上加），不是每一步都从整个账本重算一遍。
否则对话越长每一步越慢。

> **O(n)**：计算量和数据量成正比的说法。每步都 O(n) 重算，整个对话就是 O(n²)，
> 长对话会明显变慢。

## 三、一个容易被跳过但很实在的分支：模型不认识图片

```python
if _should_handle_non_multimodal_image_input(self.llm, _messages):
    if VISION_INSPECT_TOOL_NAME in self.tools_map:
        _messages = _replace_latest_user_images_with_references(_messages)
    else:
        on_event(MessageEvent(... "这个模型看不了图片" ...))
        state.execution_status = FINISHED
        return
```

> **多模态（multimodal）**：模型能同时理解文字和图片。不支持的叫单模态。

你贴了张截图，但当前模型看不了图。两种处理：

- **有"看图工具"** → 把消息里的图片换成**引用**（"图片1"这样的占位），AI 想看时主动调工具去看
- **没有** → 明确告诉你这个模型不行，然后结束

**第一种做法值得记**：模型本身没有的能力，可以**变成一个工具**补上。
不是改模型，是给它一个"去查"的手段。

第二种做法的价值在于**失败得清楚**——不是把图片悄悄丢掉然后让 AI 答得莫名其妙。

## 四、问模型

```python
llm_response = self.llm.generate(
    messages=_messages,
    tools=list(self.tools_map.values()),
    store=False,
    add_security_risk_prediction=True,   # ← 注意这个
    on_token=stream.token_callback,
    call_context=call_context,
)
```

`add_security_risk_prediction=True` 的意思是：**在工具的参数表里额外插一个字段，
要求模型每次调用工具时，自己给这次操作打一个风险等级**（低/中/高）。

这个自评分数后面会被用来决定"要不要停下来问用户"。

**为什么让模型自评而不是用规则判断？** 因为规则判断不了意图。`rm -rf ./build` 和
`rm -rf /` 长得很像，危险程度天差地别。模型知道自己想干什么，让它自己说。
当然这不是唯一防线——后面还有 shell 语法分析等好几层，这只是最便宜的第一层。

## 五、四种报错，四种活法

这是整个函数最值得学的部分。**四种异常，没有一个是简单抛出去的。**

### ① 模型把函数调用格式写错了

```python
except FunctionCallValidationError as e:
    error_message = MessageEvent(source="user",
        llm_message=Message(role="user", content=[TextContent(text=str(e))]))
    on_event(error_message)
    return
```

**把错误信息伪装成一条用户消息，塞回对话里。**

于是 AI 下一轮看到的是："（用户说）你刚才那个调用格式不对，缺了 xxx 参数"。
模型通常自己就改对了。

> 这是全项目的核心思路之一：**能让模型自己修的，就别让程序崩。**

### ② 输出被内容过滤拦了

```python
except LLMContentPolicyViolationError as e:
    on_event(MessageEvent(source="user", llm_message=Message(role="user",
        content=[TextContent(text=(
            "Your previous response was blocked by the model's content filter. "
            "Please continue, rephrasing to avoid the flagged content."))])))
    return
```

注释说明了为什么可以这样处理：

> Content-policy blocks are deterministic; nudge the model and let the run loop continue
> instead of emitting a fatal error.
> （内容策略拦截是确定性的；推一下模型让循环继续，而不是抛一个致命错误。）

**"确定性"是关键判断**：同样的输入一定会被同样地拦，所以原样重试毫无意义，
必须让模型换个说法。于是注入一条"换个说法继续"。

### ③ 对话历史结构坏了

```python
except LLMMalformedConversationHistoryError as e:
    if self.condenser is not None and self.condenser.handles_condensation_requests():
        state.rebuild_view()          # ← 重建视图
        on_event(CondensationRequest())
        return
    logger.warning(
        "... This usually indicates an upstream event-stream or resume bug: ...")
    raise e
```

服务方说"你发来的历史结构不合法"（比如上一节讲的调用/结果没配对）。

处理有两层，注释解释得很好：

> The incremental view may itself be the source of the malformed history. Re-derive with
> full enforcement so the condenser operates on a clean view.
> （增量维护的视图本身可能就是坏历史的来源。用完整强制重新推导一遍，让压缩器在干净的视图上工作。）

回想上一节：视图是**增量**维护的（快），但完整重建会跑一遍**所有规则的强制检查**（慢但可靠）。
出问题时就掏出慢而可靠的那条路。**这是"快路径 + 慢路径兜底"的标准形态。**

**更值得学的是没有 condenser 时的那条分支**：它不是静默失败，而是打一条明确的警告
——"这通常说明上游的事件流或恢复逻辑有 bug" ——然后抛出去。

> 作者知道这个异常有两种来源：一种是历史太长（该压缩），一种是自己代码有 bug（该修）。
> 他**没有用压缩把第二种掩盖掉**，而是留下了一条能让 bug 浮出水面的路径。
> 这种克制在工程上很少见——用 try/except 把所有问题吞掉太容易了。

### ④ 超出上下文窗口

```python
except LLMContextWindowExceedError as e:
    if self.condenser is not None and self.condenser.handles_condensation_requests():
        on_event(CondensationRequest())
        return
    self._log_context_window_exceeded_warning()
    raise e
```

最直白的一种：太长了，触发压缩重试。没配压缩器的话，打一条**有帮助的**提示再抛
（去看那个函数会发现它在告诉你"你应该配一个压缩器"）。

### 小结这四个 except

| 异常 | 本质 | 策略 |
|---|---|---|
| 格式错 | 模型手滑 | 把错误当用户消息，让它自己改 |
| 被过滤 | 确定性拦截 | 注入"换个说法" |
| 历史坏 | 可能是长、也可能是 bug | 重建视图 + 压缩；无压缩器则报警抛出 |
| 超长 | 确实太长 | 触发压缩 |

**共同点：程序不替模型做决定，而是把情况告诉模型，让它自己走下一步。**

## 六、模型答完之后：三种分类

```python
response_type = classify_response(message)
match response_type:
    case LLMResponseType.TOOL_CALLS:      # 要调工具
        self._handle_tool_calls(...)
    case LLMResponseType.CONTENT:         # 只说了话
        self._handle_content_response(...)
    case LLMResponseType.REASONING_ONLY | LLMResponseType.EMPTY:   # 只思考了 / 啥也没说
        self._handle_no_content_response(...)
```

第三种值得注意：**模型可能只输出了思考内容、或者什么都没输出。** 这在推理模型上是真实
存在的情况。很多实现会在这里崩掉或者死循环，这里把它当成一种正常分类来处理。

## 七、工具批次的生命周期：_ActionBatch

模型一次可能要调好几个工具。这批调用的处理被封装成了一个类（`agent.py:185`），
它的文档字符串说明了设计意图：

> Agent-specific logic (iterative refinement, state mutation) is injected via callables
> so the batch stays decoupled from the Agent class.
> （agent 特有的逻辑通过可调用对象注入，这样批次类和 Agent 类保持解耦。）

四个阶段：`_truncate_at_finish` → `prepare` → `emit` → `finalize`

### 阶段 1：finish 之后的调用全部丢掉

```python
@staticmethod
def _truncate_at_finish(action_events) -> tuple[list[ActionEvent], bool]:
    """Return (events[:finish+1], True) or (events, False).
    Discards and logs any calls after FinishTool."""
```

模型有时会一边说"我做完了"（调用 `finish` 工具），一边又要求再执行几个命令。
**一旦出现 `finish`，后面的调用全部丢弃**，并打日志列出丢了哪些。

这是个很实在的防御。不然"已经宣布完成"和"还在动手"两个状态会打架。

### 阶段 2：先分流被拦的，再执行剩下的

```python
for ae in action_events:
    reason = state.pop_blocked_action(ae.id)
    if reason is not None:
        blocked_reasons[ae.id] = reason      # 钩子拦下的，不执行
    else:
        executable.append(ae)                # 可以执行的

executed_results = executor.execute_batch(executable, tool_runner, ...)
results_by_id = dict(zip([ae.id for ae in executable], executed_results))
```

注意**结果是按编号存成字典的**，不是按顺序存成列表。因为下一步要按**原始顺序**发出事件，
而执行时被拦掉的那些不在结果里——用字典就不会错位。

### 阶段 3：按原始顺序发出，被拦的补一条"被拒"

```python
def emit(self, conversation, on_event) -> None:
    """Emit all events in original action order."""
    for ae in self.action_events:
        reason = self.blocked_reasons.get(ae.id)
        if reason is not None:
            rejection = UserRejectObservation(..., rejection_source="hook")
            on_event(rejection)
        else:
            for event in self.results_by_id[ae.id]:
                on_event(event)
```

**被钩子拦下的动作，也会生成一条"被拒"的观察事件。**

为什么必须这样？回想上一节那条铁律：**每个工具调用必须有恰好一个结果。** 被拦了也得有
个结果，否则历史就不合法了。而且 AI 能看到"我这个操作被钩子拦了，理由是 xxx"，
下一轮有机会换个做法。

### 阶段 4：收尾——完成了，还是再来一轮

```python
should_continue, followup = check_iterative_refinement(self.action_events[-1])
if should_continue and followup:
    on_event(MessageEvent(source="user", llm_message=... followup ...))
else:
    mark_finished()
```

模型说完成了，**但不一定真的结束**。这里会问"评审员"（critic，后面章节细讲）：
质量够不够？不够就注入一条追加要求，继续干。

还有个细节：`if not self.has_finish or self.action_events[-1].id in self.blocked_reasons`
—— 如果那个 `finish` 调用本身被钩子拦了，就不算完成。**钩子可以否决"结束"。**

## 八、并行执行与资源锁

多个工具可以同时跑（`parallel_executor.py`）。但同时跑有个明显的问题：**两个工具同时
改同一个文件怎么办？**

解法是让每个工具**申报自己要用什么资源**（`tool/tool.py:513`）：

```python
def declared_resources(self, action: Action) -> DeclaredResources:
    """Declare the resources this tool accesses for a given action.
    Keys should use the format "<type>:<identifier>", e.g.
    "file:/absolute/path" or "terminal:session"."""
    return DeclaredResources(keys=(), declared=False)
```

然后由一个锁管理器按资源上锁（`resource_lock_manager.py`）：

```python
mgr = ResourceLockManager()
with mgr.lock("file:/a.py", "file:/b.py"):
    # 独占这两个文件
```

**效果**：改不同文件的两个工具**可以并行**；都要用终端的两个工具**排队**。
比"全部串行"快，比"全部并行"安全。

> 注意锁的类型叫 `FIFOLock`（先进先出锁）—— 先等的先拿到。普通锁不保证顺序，
> 高竞争下可能有请求一直抢不到（叫"饥饿"）。

模块开头有一条很诚实的警告：

> .. warning:: Thread safety of individual tools
>    ... tools must correctly implement ``declared_resources()`` for this to be effective.
>    （……工具必须正确实现 declared_resources()，这套机制才有效。）

**这是把限制写在明面上**：机制本身是对的，但依赖每个工具如实申报。写新工具的人如果偷懒
不申报，并行就不安全——文档直接告诉你了，而不是等你踩坑。

还有一个防死锁的设计：

> Each instance has its own thread pool, concurrency limit, and ResourceLockManager, so
> nested execution (e.g., subagents) cannot deadlock the parent.
> （每个实例有自己的线程池、并发上限和锁管理器，所以嵌套执行（比如子 agent）不会
> 把父级卡死。）

> **死锁（deadlock）**：两方各持一把锁又在等对方的锁，永远僵住。
> 子 agent 和父 agent 共用一个线程池是典型的死锁来源——这里通过"各自独立"避开了。

还有一个只有实际踩过才会写的细节：

```python
futures = [executor.submit(contextvars.copy_context().run, self._run_safe, ...)]
# submit() itself propagates no contextvars; a fresh copy per task
# because one Context cannot be entered by two threads.
```

> **contextvars**：Python 里"跟着当前执行流走"的变量，常用来存追踪 ID、当前用户之类。

丢到线程池里执行时它不会自动跟过去，所以要手动复制。而且**每个任务必须复制一份新的**
——同一个 Context 不能被两个线程同时进入。

## 九、参数校验：把模型的输出当成不可信输入

`_get_action_event`（`agent.py:1227`）负责把模型的一次工具调用变成一个可执行的动作。
它对模型输出的态度很明确：**当成不可信的外部输入。**

处理链条：

```python
arguments = parse_tool_call_arguments(tool_call.arguments)      # 1. 解析 JSON
tool_name, arguments = normalize_tool_call(...)                 # 2. 名字归一化（别名、回退）
tool = self.tools_map.get(tool_name)
if tool is None:
    err = f"Tool '{tool_name}' not found. Available: {available}"   # 3. 不存在 → 告知模型
    self._emit_tool_error(...)
    return
arguments = fix_malformed_tool_arguments(arguments, tool.action_type)   # 4. 修常见畸形
...
action = tool.action_from_arguments(arguments)                  # 5. 严格校验成对象
```

几个点：

**第 2 步的"归一化"**：模型可能用了别名、或者用了一个已经改名的旧工具名。先尽量映射到
真实工具，而不是直接报错。

**第 3 步的错误信息带上了可用工具列表** —— `f"Tool '{tool_name}' not found. Available: {available}"`。
给模型的报错要包含**足够让它自己改对的信息**，这是个反复出现的模式。

**第 4 步 `fix_malformed_tool_arguments`**：专门修模型常犯的格式毛病（该是数组给了字符串之类）。
容错，但容错完了照样走第 5 步的严格校验。

**错误信息里只放参数名，不放参数值**：

```python
# Build concise error message with parameter names only (not values).
keys = list(arguments.keys()) if isinstance(arguments, dict) else None
params = f"Parameters provided: {keys}" if keys is not None else "Arguments: unparseable JSON"
```

**为什么不放值？** 参数值里可能有密钥、token、用户数据。报错信息会进日志、进账本、
可能上报到遥测系统。只报参数名足够让模型改对，同时不泄露内容。

还有一行断言：

```python
assert "security_risk" not in arguments, (
    "Unexpected 'security_risk' key found in tool arguments")
```

还记得第四节那个"让模型自评风险"的开关吗？那个字段是**临时插进参数表让模型填的**，
取出来之后**必须从真正的执行参数里拿掉**——否则会被当成工具的真实参数传下去。
这行断言就是守这个不变量。

> 又一次看到同样的手法：**把"我以为这里一定成立"写成断言，而不是写成注释。**

## 十、这一节的可迁移结论

1. **"走一步"和"何时停"要分开**：一个管怎么和模型交互，一个管预算/暂停/卡死。
   混在一起两边都测不了。

2. **把报错变成对话，而不是崩溃**：格式错了、被过滤了，都做成"告诉模型发生了什么"，
   让它自己修。前提是**错误信息要足够让它修对**（比如附上可用工具列表）。

3. **区分"该兜底的错"和"该暴露的 bug"**：同一个异常有两种来源时，不要用兜底逻辑把
   自己的 bug 一起掩盖掉——留一条打警告并抛出的路径。

4. **快路径 + 慢路径兜底**：视图增量维护（快），出问题时完整重建（慢而可靠）。

5. **能力缺失可以用工具补**：模型看不了图，就给它一个"看图工具"，而不是改模型。

6. **并行的前提是申报资源**：让每个工具声明自己碰什么，按资源上锁——
   比全串行快，比全并行安全。并且**把"依赖如实申报"这个前提写进文档**。

7. **把模型输出当不可信输入**：解析 → 归一化 → 容错修复 → 严格校验，四步都不能省。
   报错只带参数名不带值，因为值里可能有密钥。

## 下一站

`conversation/impl/local_conversation.py:1903` 的 `run()` —— 外层循环：
预算控制、暂停、卡死检测、以及那个"故意不检查已完成状态"的并发设计。

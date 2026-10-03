# 精读 05：压缩器（context/condenser/）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/context/condenser/`
> 文件：`base.py`（252 行）、`llm_summarizing_condenser.py`（517 行）、`utils.py`、`prompts/summarizing_prompt.j2`
> 相关：`event/condenser.py`
> 前置：`notes/03-view-and-properties.md`（可操作位置）、`notes/04-agent-step.md`（谁调用它）

## 这一节在讲什么

第二节讲了"哪些位置可以下剪刀"，这一节讲**到底剪哪一刀、剪掉的内容怎么处理**。

两条线在这里汇合：**先算出想剪多少（数量问题），再找最近的合法切口（结构问题）。**

## 一、先看接口设计：一个返回类型说明了一切

```python
def condense(self, view: View, agent_llm: LLM | None = None) -> View | Condensation:
```

**要么给你一个（可能更短的）视图，要么给你一条"压缩记录"。** 类文档解释了这两种情况的区别：

> If the condenser returns a `Condensation` instead of a `View`, the agent should return
> `Condensation.action` instead of producing its own action. On the next agent step the
> condenser will use that condensation event to produce a new `View`.

翻译：返回压缩记录时，这一步**不问模型**，而是把这条记录写进账本，下一步再来。

这就是上一节看到的"压缩自己占用一步"。**代价是多花一轮，收益是压缩变成了账本上可回放的一行。**

### 一条写给实现者的重要警告

```python
view: A view of the history containing all events that should be condensed.
    **Implementations must treat this view as read-only.** The view may be
    a cached projection owned by ``ConversationState``, and mutating it in
    place will corrupt that cache. Return a new ``View``
    (e.g. ``View(events=view.events[k:])``) or a ``Condensation`` instead.
```

上一节讲过视图是**增量缓存**的（为了性能）。这里的警告是那个决定的后果：**缓存是共享的，
谁就地改坏了，所有人都受影响。** 所以要求实现者只能返回新对象。

> 这是"为性能引入缓存"必然要付的税：多了一条不能违反的约定。作者的处理是把约定
> **加粗写进文档、并给出正确写法的示例**。这是在没有语言层强制手段时能做的最好的事。

## 二、硬压和软压：不是所有"该压缩"都同等紧急

```python
class CondensationRequirement(Enum):
    HARD = "hard"
    """Indicates that a condensation is required right now, and the agent cannot
    proceed without it."""

    SOFT = "soft"
    """Indicates that a condensation is desired but not strictly required."""
```

**硬压 = 不压就走不下去。软压 = 压一下更好，但不压也能继续。**

这个区分的价值在失败时体现（后面第六节）。

## 三、三个触发原因，各自映射到硬/软

```python
class Reason(Enum):
    REQUEST = "request"   # 有人明确要求压缩
    TOKENS = "tokens"     # 超过 token 上限
    EVENTS = "events"     # 事件条数太多
```

判断逻辑（`get_condensation_reasons`）：

```python
# 原因 1：有未处理的压缩请求（视图在处理事件流时已经检测到了）
if view.unhandled_condensation_request:
    reasons.add(Reason.REQUEST)

# 原因 2：配了 token 上限且已超过
max_tokens = self._effective_max_tokens(agent_llm)
if max_tokens is not None and agent_llm is not None:
    total_tokens = get_total_token_count(view.events, agent_llm)
    if total_tokens > max_tokens:
        logger.info("Condenser token limit exceeded: total_tokens=%d max_tokens=%d events=%d", ...)
        reasons.add(Reason.TOKENS)

# 原因 3：事件条数超过 max_size
if len(view) > self.max_size:
    reasons.add(Reason.EVENTS)
```

注意**返回的是一个集合**，可以同时成立多个原因。

### token 上限取两个值里更严的那个

```python
def _effective_max_tokens(self, agent_llm) -> int | None:
    """Takes the stricter (i.e. smaller) of the condenser's configured max_tokens
    and the agent LLM's effective input limit."""
    limits = [limit for limit in (
        self.max_tokens,
        agent_llm.effective_max_input_tokens if agent_llm is not None else None,
    ) if limit is not None]
    return min(limits) if limits else None
```

两个来源：**你配置的上限**、**模型自己的真实上限**。取小的那个。

这解释了上一节那个"先 `resolve_runtime_metadata()` 再压缩"的顺序——模型的真实上限
必须先查清楚，否则这里的 `min()` 会算错。

### 原因到紧急程度的映射，注释写了理由

```python
if Reason.TOKENS in reasons:
    return CondensationRequirement.HARD
```

> Token pressure is a hard requirement in benchmark runs that use a fixed local model
> context: sending the next request can fail before the recovery path has a chance to run.

翻译：token 超了必须**立刻**压。因为如果放行，下一次请求会直接失败——**在补救逻辑有机会
运行之前**。所以不能等。

```python
resource_reasons = {Reason.EVENTS}
if reasons.issubset(resource_reasons):
    return CondensationRequirement.SOFT
```

> Treat event-count pressure as soft because that threshold is only a history-management
> heuristic.

条数超了是**软**的，因为"最多 240 条"只是一个管理历史的经验值，不是硬约束。

```python
if Reason.REQUEST in reasons:
    return CondensationRequirement.HARD
```

> Requests -- whether they come from the user or the agent -- are always hard requirements.
> We need to condense now because: 1. the user expects it 2. the agent has no more room
> in the context window and can't continue

明确请求是硬的，两个理由：用户在等；或者 agent 自己发的请求（上一节那个"超上下文异常 →
发压缩请求"的路径），说明它真的走不下去了。

> **值得学的地方**：同样是"该压缩了"，来源不同紧急程度不同，而每一档都写清了
> **为什么是这一档**。很多代码只有 if/else 没有 why，半年后没人敢改。

## 四、压多少：三个原因各算一个数，取最严的

这是 `_get_forgotten_events` 的第一半，很值得逐行看：

```python
suffix_events_to_keep: set[int] = set()     # "尾部保留多少条"的候选值

if Reason.REQUEST in reasons:
    target_size = len(view) // 2                              # 目标：砍到当前的一半
    suffix_events_to_keep.add(target_size - self.keep_first - 1)

if Reason.EVENTS in reasons:
    target_size = self.max_size // 2                          # 目标：砍到上限的一半
    suffix_events_to_keep.add(target_size - self.keep_first - 1)

if Reason.TOKENS in reasons:
    total_tokens = get_total_token_count(view.events, agent_llm)
    tokens_to_reduce = total_tokens - (max_tokens // 2)        # 目标：砍到上限的一半
    suffix_events_to_keep.add(get_suffix_length_for_token_reduction(...))

# 多个原因时取最严的
events_from_tail = min(suffix_events_to_keep)
```

### 为什么都是"砍到一半"而不是"砍到刚好不超"

因为砍到刚好不超，下一步就又超了，于是**每一步都要压缩一次**——每次压缩都是一次额外的
模型调用，很贵。

**砍到一半，就能撑很多步不用再压。** 这是典型的"批量处理换取摊销成本"，和数据结构里
动态数组扩容时翻倍而不是加一是同一个道理。

> **⚠️ 实跑修正（见 `notes/26-verification-run.md`）**：这条摊销逻辑**有一个最小规模要求**。
> 用 `max_size=10` 实跑时，压缩后视图只剩 5 个事件、每轮加 2 个，**只撑 2–3 轮就又触发**，
> 5 条用户消息的对话里压缩了 **7 次**，最终**压缩器的花费（$0.0220）超过了主 agent（$0.0185）**。
>
> 可以算出来：`撑住的轮数 = (max_size - 压缩后长度) / 每轮增加的事件数`。
> `max_size=10` 时约 2.5 轮；默认的 `max_size=240` 时约 59 轮——**差 24 倍。**
>
> **结论**：摊销成立的前提是 `max_size` 远大于"每轮产生的事件数"。
> 低于那个量级，压缩会从省钱手段变成最大的开销项。

### `min()` 的含义

三个原因各算出"尾部该保留多少条"。保留得越少 = 删得越多 = 越激进。**取 `min` 就是取最激进
的那个方案**，这样能同时满足所有约束。

### token 那条路用了二分查找

`get_suffix_length_for_token_reduction` 背后是 `get_shortest_prefix_above_token_count`
（`utils.py`），它做的是：

> performs a binary search to efficiently find the shortest prefix of events that ...
> has a total token count greater than the specified target token count

**为什么需要二分？** 因为"删多少条能减掉这么多 token"没有公式——每条事件的长度天差地别。
只能试。逐条累加是 O(n) 次 token 计算，而**每次 token 计算本身也不便宜**。二分把它降到
O(log n) 次。

`get_total_token_count` 里还有个细节：

```python
tools = next((event.tools for event in events if isinstance(event, SystemPromptEvent)), None)
return llm.get_token_count(messages, tools=tools or None,
    # Security-risk tokens are always included in real tool requests.
    add_security_risk_prediction=bool(tools))
```

**算 token 时把工具定义也算进去了**，而且把那个"让模型自评风险"的额外字段也算上——
注释说明理由：真实请求里一定有它。

> 这是很容易漏的地方：工具定义（几十个工具的完整 JSON schema）常常比对话本身还长。
> 只算消息不算工具，估出来的数字会严重偏小，然后在真实请求时炸掉。

## 五、剪哪一刀：数量算完，交给结构

这是 `_get_forgotten_events` 的第二半，也是**本节和第二节汇合的地方**：

```python
# 按数量算出的"天真"终点（没考虑原子边界）
naive_end = len(view) - events_from_tail

# 真正的起点：>= keep_first 的最小可操作位置
forgetting_start = view.manipulation_indices.find_next(self.keep_first)

# 真正的终点：>= naive_end 的最小可操作位置
forgetting_end = view.manipulation_indices.find_next(naive_end)

forgotten_events = view[forgetting_start:forgetting_end]
```

**四行代码，把两个世界连起来了：**

1. `naive_end` 是**数量**的诉求："我想从这里开始保留"
2. `find_next` 是**结构**的裁决："最近的合法切口在这里"

变量名 `naive_end`（天真的终点）本身就在说明这件事——**先算个理想值，再让约束修正它。**

> 这个模式值得单独记住：**需求和约束分开算，最后让约束去调整需求。** 硬写在一起
> （"边算数量边判断能不能切"）会得到一坨没法测试的代码。

注意 `find_next` 是**向后找**（`>= threshold`）。意味着实际删的**可能比想删的少**
（切口往后挪了，保留的更多）。宁愿压得不够，也不破坏结构。

## 六、压不动的时候怎么办：三级降级

`get_condensation` 有三道检查，任何一道不过就抛 `NoCondensationAvailableException`：

```python
if not forgotten_events:
    raise NoCondensationAvailableException(
        "Cannot condense 0 events. This typically occurs when a tool loop "
        "spans almost the entire view, leaving no valid range for forgetting "
        "events. Consider adjusting keep_first or max_size parameters.")

if len(forgotten_events) < len(view) * self.minimum_progress:
    raise NoCondensationAvailableException(
        "Cannot apply condensation: events forgotten below minimum progress threshold.")
```

**第一个错误信息是教科书级的**：说了现象（0 条可删）、解释了原因（一个"思考回合"
占了几乎整个视图，所以没有合法切口——正是第二节讲的第 4 条规则造成的）、给了两个
可调参数。

**第二个是 `minimum_progress`（默认 0.1）**：一次至少要删掉 10%，否则视为失败。
为什么？删 1% 毫无意义，但照样付了一次摘要的模型调用费。**不划算的进展等于没进展。**

### 然后是分级处理（`base.py` 的 `condense`）

```python
except NoCondensationAvailableException as e:
    if request == CondensationRequirement.SOFT:
        # For soft requests, we can just return the uncondensed view. This request
        # will _eventually_ be handled, but it's not critical that we do so immediately.
        return view                                    # ← 软的：算了，继续跑

    elif request == CondensationRequirement.HARD:
        # The agent has found itself in a situation where it cannot proceed without
        # condensation, but the condenser cannot provide one.
        try:
            hard_reset_condensation = self.hard_context_reset(view, agent_llm=agent_llm)
            if hard_reset_condensation is not None:
                return hard_reset_condensation         # ← 硬的：掏出核选项
        except Exception as hard_reset_exception:
            # And if something goes wrong with the hard reset make sure we keep
            # both errors in the stack
            raise hard_reset_exception from e
    raise e
```

**这就是第二节那个硬/软区分的用处**：软压失败就放过，硬压失败要动用更激进的手段。

注意 `raise hard_reset_exception from e` —— **两个异常都保留在链上**。不然你只会看到
"重置也失败了"，看不到"最初为什么需要重置"。调试时这是天壤之别。

### 核选项：hard_context_reset

```python
def hard_context_reset(self, view, agent_llm=None) -> Condensation | None:
    """Perform a hard context reset by summarizing all events in the view.

    Depending on how the hard context reset is triggered, this may fail (e.g., if the
    view is too large for the summarizing LLM to handle). In that case, we keep
    trimming down the contents until a summary can be generated."""

    max_event_str_length = None
    attempts_remaining = self.hard_context_reset_max_retries      # 默认 5

    while attempts_remaining > 0:
        try:
            return self._generate_condensation(
                forgotten_events=view.events,      # ← 全部事件，一个不留
                summary_offset=0,
                max_event_str_length=max_event_str_length)
        except Exception as e:
            if max_event_str_length is None:
                # 第一次失败：把上限设为"最长那条事件的长度"
                max_event_str_length = max(len(str(event)) for event in view.events)

            # 每次失败就把每条事件的长度上限砍掉 20%
            max_event_str_length = int(max_event_str_length * self.hard_context_reset_context_scaling)
            logger.warning(f"Hard context reset summarization failed with exception: {e}. "
                           f"Reducing max event size to {max_event_str_length} and retrying.")
        attempts_remaining -= 1

    logger.error("Hard context reset summarization failed after multiple attempts.")
    return None
```

**这是"压缩自己也太长了"这个死局的解法。**

场景：历史太长要压缩 → 但要压缩就得把历史发给摘要模型 → 而历史长到摘要模型也读不下。

解法很朴素但有效：**每条事件只保留前 N 个字符，N 每次失败砍 20%，最多试 5 次。**

于是最多 5 次尝试后，N 会缩到原来的 0.8⁵ ≈ 33%。如果还失败，返回 `None` 并打 error 日志。

> 注意第一次的 `max_event_str_length` 不是随便定的，而是取**最长那条事件的长度**。
> 也就是说第一次重试等价于"把最长的那条截到它自己的 80%"——从最大的那块开始削，
> 而不是一刀切。

## 七、压缩记录本身：一条可回放的账

`Condensation` 事件（`event/condenser.py`）只有四个字段：

```python
forgotten_event_ids: set[EventID]   # 哪些事件被忘记了
summary: str | None                 # 摘要文本
summary_offset: int | None          # 摘要插在哪个位置
llm_response_id: EventID            # 是哪次模型调用生成的
```

应用方式：

```python
def apply(self, events: list[LLMConvertibleEvent]) -> list[LLMConvertibleEvent]:
    """...removes events that are marked to be forgotten and returns a NEW list..."""
    output = [event for event in events if event.id not in self.forgotten_event_ids]
    if self.has_summary_metadata:
        summary_event = self.summary_event
        output.insert(self.summary_offset, summary_event)
    return output
```

**注意三件事：**

**① 记的是"忘记哪些 id"，不是"保留哪些"。** 于是这条记录在任何事件列表上都能应用，
而不依赖当时的下标。下标会随着新事件追加而漂移，id 不会。

**② 返回新列表，原列表不动。** 第一节那条"只增不改"的契约在这里兑现。

**③ 摘要事件是"算出来的"，不存进账本。** 看 `summary_event` 属性：

```python
@property
def summary_event(self) -> CondensationSummaryEvent:
    """Since summary events are not part of the main event store and are generated
    dynamically, this property ensures the created event has a unique and consistent
    ID based on the condensation event's ID."""
    summary_id = f"{self.id}-summary"    # ← 确定性地派生 id
    return CondensationSummaryEvent(id=summary_id, summary=self.summary, source=self.source)
```

**摘要不是独立的一行账，而是从压缩记录派生出来的。** id 用后缀确定性生成，所以每次
重放都得到同一个 id——幂等。

> **幂等（idempotent）**：同样的操作做一次和做多次结果一样。
> 重放账本这件事必须幂等，否则每次加载会话都会长出新东西。

而 `CondensationSummaryEvent.to_llm_message()` 把摘要变成一条 **user 角色**的消息：

```python
return Message(role="user", content=[TextContent(text=self.summary)])
```

**为什么是 user 而不是 system 或 assistant？** 因为它是"环境提供给模型的背景信息"，
既不是模型自己说过的话（assistant 会让模型以为那是它的原话），也不该占用系统提示的位置。

## 八、摘要提示词：它在保什么

`prompts/summarizing_prompt.j2` 值得整段读，因为它定义了"压缩时什么信息不能丢"。

开头就定了角色：

```
You are maintaining a context-aware state summary for an interactive agent.
You will be given a list of events ..., which will include previous summaries.
```

**"which will include previous summaries"** —— 摘要会包含之前的摘要。这是**滚动压缩**：
第二次压缩时，第一次的摘要也在待压缩的内容里，会被重新摘要一遍。

> 这带来一个真实风险：**反复摘要会逐渐失真**（像复印件的复印件）。
> 提示词里那些"必须保留"的硬性要求，主要就是在对抗这个。

要求保留的结构：

```
USER_CONTEXT: (保留用户的核心需求、目标和澄清)
TASK_TRACKING: {进行中的任务、它们的 ID 和状态 - 必须保留任务 ID}
COMPLETED: (已完成的任务，附简要结果)
PENDING: (还需要做的任务)
CURRENT_STATE: (当前的变量、数据结构、相关状态)

代码任务额外要求：
CODE_STATE: {文件路径、函数签名、数据结构}
TESTS: {失败的用例、错误信息、输出}
CHANGES: {代码改动、变量更新}
DEPS: {依赖、导入、外部调用}
VERSION_CONTROL_STATUS: {仓库状态、当前分支、PR 状态、提交历史}
```

几个观察：

**① 任务 ID 被两次强调必须原样保留**（开头一句 + `PRESERVE TASK IDs`）。因为 ID 一变，
后续"把任务 3 标记为完成"就找不到目标了。**摘要可以丢细节，不能丢标识符。**

**② "已完成"和"待办"必须分开。** 混在一起，AI 会重做已经做完的事——这是长任务里最
常见的失败模式之一。

**③ 分支名、提交哈希、失败的测试用例名**都点名要保留。这些都是**不能靠推理还原的
具体事实**。

**④ 给了两个完整范例**（一个代码任务、一个写俳句的任务），而且明确说
`Adapt tracking format to match the actual task type`——格式要随任务类型调整，
不是死板套模板。

> 可迁移的道理：**写摘要提示词时，要按"能不能被推理还原"来分类信息。**
> 能还原的（"我在重构认证模块"）可以压；不能还原的（提交哈希、任务 ID、错误原文）
> 必须逐字保留。

## 九、几个配置项的含义

```python
max_size: int = 240              # 事件条数上限
max_tokens: int | None = None    # token 上限（和模型真实上限取更严的）
keep_first: int = 2              # 开头几条永不压缩
minimum_progress: float = 0.1    # 一次至少压掉 10%
hard_context_reset_max_retries: int = 5
hard_context_reset_context_scaling: float = 0.8
```

### `keep_first = 2` 保的是什么

开头两条通常是：**系统提示** + **用户的原始任务**。这两条一旦丢了，AI 就不知道自己是谁、
在干什么。所以永远保留。

### 一个启动时就检查的约束

```python
@model_validator(mode="after")
def validate_keep_first_vs_max_size(self):
    events_from_tail = self.max_size // 2 - self.keep_first - 1
    if events_from_tail <= 0:
        raise ValueError("keep_first must be less than max_size // 2 to leave room for condensation")
    return self
```

如果 `keep_first` 配得太大（接近 `max_size` 的一半），就没有空间可压了。
**这个矛盾在创建对象时就报错，而不是等到运行到一半才发现压不动。**

> 又一次看到同一种手法：**能在启动时检查的配置错误，绝不留到运行时。**

### 摘要用的模型会关掉流式输出

```python
@model_validator(mode="after")
def _disable_streaming_for_summary(self):
    # Summaries are consumed whole with no on_token callback, which a streaming LLM
    # requires. Disable streaming once so every summary path is covered.
    # model_copy is non-mutating and shares usage_id/metrics, so summary tokens stay
    # attributed to the conversation.
```

> **流式输出（streaming）**：模型一个字一个字地吐，而不是全部生成完再返回。
> 适合给人看，不适合程序等一个完整结果。

摘要是整段一次性使用的，不需要流式。关掉。

注释里那句 `shares usage_id/metrics, so summary tokens stay attributed to the conversation`
很关键：**改配置用的是 `model_copy`（不改原对象），而且仍然共享用量统计**——
所以摘要花的钱**照样计入这次会话的总花费**。

这正好接上上一节那个 `get_combined_metrics()`：预算能管住所有模型的花费，
前提是每个派生出来的模型实例都还挂在同一个统计上。**一个设计要在两个模块同时配合才成立。**

## 十、这一节的可迁移结论

1. **紧急程度要分级，并写清为什么**：硬（不做就崩）和软（做了更好）在失败时走完全
   不同的路径。每一档都注明理由，否则半年后没人敢改。

2. **需求和约束分开算**：先算"理想上想删到哪"（`naive_end`），再让约束修正成
   "实际能删到哪"（`find_next`）。硬写在一起会得到没法测试的代码。

3. **一次做够，别每步都做**：压到一半而不是压到刚好不超，用一次昂贵操作换很多步的空间。
   和动态数组翻倍扩容同理。

4. **降级要有梯度，且到底要认输**：正常压缩 → 软的放过/硬的重置 → 重置也分 5 次
   逐步削减 → 最终返回 None 并记 error。**每一级都比上一级更激进，最后一级是明确的失败。**

5. **异常链不能断**：`raise B from A`，否则只看到"补救失败"、看不到"最初为什么需要补救"。

6. **记"忘了什么"而不是"留了什么"**：用不会漂移的 id，这条记录才能在任何时候重放。

7. **派生数据要确定性生成**：摘要事件的 id 由压缩记录的 id 加后缀得来，
   保证重放幂等。

8. **摘要提示词按"能否被推理还原"分类信息**：ID、哈希、错误原文必须逐字保留；
   概括性描述可以压。

9. **配置的自相矛盾在启动时就报错**：`keep_first` 和 `max_size` 打架，创建对象时就拒绝。

10. **派生出的模型实例要保持挂在同一份用量统计上**，否则上层的预算控制会漏账。

## 下一站

`security/` 目录（3619 行）—— 三层防护：模型自评风险、确认策略、
以及真正解析 shell 语法的纵深防御。这是 codeloop 标为"未做"的那一块。

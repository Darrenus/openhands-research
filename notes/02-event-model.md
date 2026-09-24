# 精读 01：事件模型（event/base.py，199 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/event/base.py`
> 这 199 行是整个 SDK 的地基，其余所有模块都建在它上面。

## 先建立直觉：什么是"事件"

想象一个银行账本。你每笔存取款都记一行，**记上去就不许擦改**。想知道余额，就把所有行加一遍算出来——
而不是维护一个"余额"数字然后每次去改它。

这个 SDK 对 AI 的对话历史就是这么干的。AI 说了一句话、调用了一个工具、工具返回了结果、用户批准了某个
操作——**每一件事都是账本上的一行，叫一个"事件"（Event）**。要把历史喂给 AI 模型时，就把这些行
重新算成一份对话记录。

为什么不像大多数程序那样直接维护一个"消息列表"、需要时改一改？因为改过的历史就查不清了。
账本模式的好处：任何时候都能回答"当时到底发生了什么"。

## 一、Event 类：五个字段

```python
class Event(DiscriminatedUnionMixin, ABC):
    model_config = ConfigDict(extra="forbid", frozen=True)
    id: EventID                    # 这行账的编号，默认随机生成
    timestamp: str                 # 什么时候记的
    source: SourceType             # 谁干的
    parent_id: EventID | None      # 这行账接在哪一行后面
```

### `frozen=True` —— 写上去不许改

`frozen`（冻结）是 Pydantic 这个库的功能，意思是对象创建之后任何字段都不能再被赋值，改了直接报错。

这正是"账本不许擦改"的技术落地。代码里有一处注释把这个约定写明了：

> events are immutable once created and appended, so merges build new content lists
> rather than mutating the events' own messages.
> （事件一旦创建并追加就不可变，所以合并操作是新建内容列表，而不是去改事件自己的消息。）

**这带来什么好处？** 后面有个很危险的功能叫"上下文压缩"——对话太长了要删掉一部分旧内容。
因为原始账本改不动，压缩再怎么出错，都只是算出了一份不好的"摘要视图"，真实历史一个字都没丢。
可以重算，可以换策略重来。

### `extra="forbid"` —— 多一个字段就报错

正常程序读到不认识的字段一般会忽略。这里选择直接报错。

原因：这些事件会存成文件、日后再读回来。如果代码升级后字段名改了，**宁愿在读取的瞬间大声炸掉**，
也不要静默地丢掉一个字段、然后在很久以后以某个莫名其妙的形式出问题。

### `source` —— 谁干的，有四种

```python
SourceType = Literal["agent", "user", "environment", "hook"]
```

- `agent` —— AI 自己
- `user` —— 你
- `environment` —— 环境（工具执行的返回结果算这一类）
- `hook` —— 钩子

**"钩子"（hook）**是什么：你可以配置一些脚本，在特定时刻自动运行，比如"AI 每次要执行命令前
先让我的脚本检查一下"。这种脚本注入的内容也会进账本。

注意它是独立的第四种来源。意思是：**钩子塞进去的话，永远标着"钩子"的身份，不会伪装成用户说的话。**
事后追查"这句提示是谁加的"永远查得清。

## 二、账本是一棵树，不是一条线

`parent_id` 是最关键的字段。每行账记着"我接在哪一行后面"。

大多数 AI 程序的历史是一条**直线**：消息 1 → 2 → 3 → 4。这里是一棵**树**：

```
       消息1
         │
       消息2
      ╱     ╲
   消息3a   消息3b     ← 两条分支，同一个父节点
     │        │
   消息4a   消息4b
```

**有什么用？** AI 试了一条思路走不通，可以回到分叉点换一条重来，而**不用**丢掉失败那条的记录。
两条路都在账本里，可以对比、可以回溯。这在"让 AI 多试几种解法再挑最好的"这类玩法里是必需的。

### 一个值得停下来看三十秒的细节

分支相关的定义在 `event/types.py`：

```python
ROOT_PARENT_ID: Final = "__root__"
```

这是一个**哨兵值**（sentinel，一个被约定赋予特殊含义的固定值）。当一行账的 `parent_id` 等于
`"__root__"` 时，意思是"我是树根，我没有父节点"。

这种写法有个经典的坑：**万一真有某行账的编号（id）就叫 `__root__` 呢？** 那它的所有子节点看起来
就都变成了"我是树根"，整棵树会默默断成好几截，而且不报错——这种 bug 能查一个星期。

作者的处理不是"应该不会撞吧"，而是写了一个校验器**从类型上禁止这个编号存在**：

```python
@field_validator("id")
def _reject_reserved_id(cls, v):
    if v == ROOT_PARENT_ID:
        raise ValueError(f"Event id may not equal reserved sentinel {v!r}")
```

**校验器**（validator）就是"存进来之前先检查一遍，不合格就拒收"的函数。

于是这个哨兵的含义是**无歧义**的，不再是"大家约定好别用这个名字"。

> 这是判断一份代码值不值得读的好办法：看作者有没有把自己心里的假设，
> **变成机器能强制执行的规则**，而不是写在注释里提醒别人小心。

## 三、用"类型"当过滤器

第二个类只有三行有效代码：

```python
class LLMConvertibleEvent(Event, ABC):
    @abstractmethod
    def to_llm_message(self) -> Message: ...
```

意思是：**继承了这个类的事件，才有办法变成"喂给 AI 的消息"。**

`abstractmethod`（抽象方法）= 只声明名字不写实现，强制每个子类必须自己实现一份，否则连
创建对象都不允许。

为什么需要这个区分？因为账本里有很多行是**不该给 AI 看的**：钩子的执行记录、流式输出的
零碎片段、压缩操作的记录、会话状态的变更。这些直接继承 `Event`，**没有** `to_llm_message` 方法。

于是"哪些内容进 AI 的上下文"这件事，靠的是**类型**——不是靠一串 if 判断，也不是靠维护一张
字段白名单。后面负责组装上下文的 `View` 类，它的类型签名写的就是 `list[LLMConvertibleEvent]`，
想把不该给 AI 看的东西塞进去，工具在你运行代码之前就会报错。

**可迁移的道理**：能用类型表达的约束，就不要用运行时检查。前者在写代码时就拦住你，后者要等到线上出事。

## 四、全文唯一的算法：还原并行工具调用

### 先解释问题

**"并行工具调用"（parallel function calling）**：现代 AI 模型一次回复可以同时要求执行多个工具，
比如"同时读这三个文件"。

麻烦在于：账本是扁平的，三个工具调用被记成了**三行独立的账**。但要把历史喂回给 AI 时，
必须把这三行**重新合并回原本那一条回复**。

**为什么必须合并？** 因为 AI 模型的接口有严格规定：一次回复就是一条消息。如果你把它拆成三条
发回去，接口会认为你在伪造对话轮次，直接报错。

### 怎么合并

靠每行账上的 `llm_response_id`（这行账是哪一次 AI 回复产生的）：

```python
if isinstance(event, ActionEvent):
    batch_events = [event]
    response_id = event.llm_response_id
    j = i + 1
    while j < len(events) and isinstance(events[j], ActionEvent):
        if events[j].llm_response_id != response_id:
            break                      # 编号变了，说明换了一次回复，这批到此为止
        batch_events.append(events[j])
        j += 1
    msg = _combine_action_events(batch_events)
    i = j                              # 整批一起跳过
```

朴素的向前扫描：编号相同的连续几行归成一批，合成一条消息。

### 合并时的一个隐藏约定

AI 的"思考内容"（模型在动手之前的推理文字）在一次回复里只有一份。那么拆成三行账之后，
这份思考放在哪一行？约定是：**放在第一行，后两行必须为空。**

作者没有把这个约定写在注释里，而是写成了断言：

```python
for e in events[1:]:
    assert len(e.thought) == 0, (
        "Expected empty thought for multi-action events after the first one"
    )
```

**断言**（assert）= "我认为这里一定成立，如果不成立就立刻崩溃"。

为什么要这么凶？因为如果哪天有人不小心把思考内容存到了第二行，合并时那段推理会被**悄悄丢掉**，
AI 的表现会莫名变差，而且没有任何报错。断言让这种错在第一时间暴露。

### 顺带一个细节：什么样的消息能合并

```python
def _is_plain_user_message(message):
    return (message.role == "user"
        and message.tool_calls is None
        and message.tool_call_id is None
        and message.name is None)
```

连续的**纯**用户消息可以合成一条。但"纯"的条件卡得很死——带任何工具调用相关信息的都不合。

原因：工具返回的结果必须和"它在回答哪个工具调用"严格一一对应（靠 `tool_call_id` 配对）。
合并会破坏这个配对，AI 接口会直接拒收。

**这是"上下文不能随便乱动"这条铁律在最底层的第一次露面。** 后面 `View.manipulation_indices`
是它的系统化升级版，把这类约束变成了一套可组合的规则。

## 五、存档怎么恢复成正确的类型

`Event` 继承的 `DiscriminatedUnionMixin`（在 `utils/models.py:197`）加了一个自动字段：

```python
@computed_field
@property
def kind(self) -> str:
    return self.__class__.__name__
```

存成文件时自动带上类名，读回来时按这个名字找到对应的类去还原。

**为什么重要**：一条 JSON 格式的账本，能完整恢复成原来的各种事件类型。这是"会话能存盘、能
恢复、能通过网络传给另一台机器"的前提，也是这个 SDK 能做到"AI 跑在服务器上、界面在你电脑上"
（`RemoteConversation`）的根本原因。

## 六、两个子类，验证上面的设计

### `ActionEvent`（AI 要执行一个动作）

它把同一个工具调用**存了两份**，字段注释解释了为什么：

> `tool_call` 可能包含 LLM 预测的 `security_risk` 字段（当启用 LLM 风险分析器时），
> 而 `action` 没有。

- `tool_call` = AI 原始输出的忠实副本，包含 AI 给自己这个动作打的**风险等级**
- `action` = 校验解析之后、真正拿去执行的参数

**为什么要冗余？** 两个目标冲突了：喂回给 AI 时要**一字不差**（否则模型会困惑于自己的话被改了），
真正执行时要**干净安全**（必须校验过）。分开存，两个目标就都能满足，不用互相妥协。

它还有个 `summary` 字段，要求 AI 用大约 10 个词概括自己在干什么，注释里给了五个范例，
比如 `"running tests to verify bug fix"`（跑测试验证修复）。纯粹为了让人看日志时看得懂。

### `ObservationEvent`（工具返回了结果）

有个 `extended_content` 字段：

> Content added by agent context (e.g. path-scoped rules triggered by the touched file),
> appended after the tool result in to_llm_message.
> （由 agent 上下文追加的内容，例如被改动文件所在路径触发的规则，拼在工具结果之后。）

翻译：你让 AI 改了 `src/api/` 下的文件，项目里专门针对这个目录的规矩（比如"这里禁止直接发
网络请求"）会**自动拼在工具返回结果的后面**给 AI 看。

**这个设计很值得学**：规则不是一次性全塞进开头的系统提示里，而是**绑在相关的观察上、用到时才出现**。
好处是长期占用的上下文小，且规则出现的时机正是最需要它的时候。

### `UserRejectObservation`（动作被拒了）

```python
rejection_source: RejectionSource   # "user" 或 "hook"
```

区分是**你本人**拒绝的，还是**钩子脚本**自动拦下的。拒绝本身也是一行账，AI 能看到自己被拒了、
以及是被谁拒的——于是它有机会换个做法，而不是一脸茫然地重试。

## 七、这一节的四个可迁移结论

1. **只增不改的账本 + 只读的派生视图**：让"压缩历史"这种危险操作永远不可能损坏真实记录。
2. **把假设变成机器能执行的规则**：校验器（禁止保留字做编号）、断言（思考只存第一行），
   而不是写在注释里提醒别人。
3. **能用类型表达的约束就别用运行时判断**：`LLMConvertibleEvent` 让"不该给 AI 看的东西"
   在编译期就进不去。
4. **原始数据和加工数据分开存**：`tool_call`（忠实副本）vs `action`（校验后的执行参数），
   忠实度和安全性就不必二选一。

## 下一站

`context/view/view.py` + `context/view/properties/tool_loop_atomicity.py`
—— 第四节末尾那条"消息不能随便合并"的约束，在那里被系统化成一套可组合的规则。

# 精读 02：上下文视图与四条规则（context/view/）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/context/view/`
> 文件：`view.py`（160 行）、`manipulation_indices.py`（60 行）、`properties/` 下五个文件
> 前置：先读 `notes/02-event-model.md`（事件模型）

## 这一节要解决的问题

上一节讲了：所有发生的事都记进一个**只增不改的账本**（事件流）。

但账本会越来越长。AI 模型能一次读进去的内容是有上限的（叫**上下文窗口**，
context window——可以理解成模型的"一次能看多少字"的硬性额度，超了就报错）。

所以必须删掉一部分旧内容。**问题是：不能乱删。**

## 一、为什么"乱删"会出事

AI 接口对消息的结构有硬性规定。最典型的一条：

> 每一个"工具调用"（AI 说"帮我执行这个"），必须紧跟着**恰好一个**"工具结果"
> （执行完的返回值）。

如果你删掉了一个工具调用、却留下了它的结果，接口收到的就是一个凭空出现的返回值——
**直接报错，整个对话崩掉**。反过来也一样。

还有更隐蔽的。Anthropic 的模型开启"思考模式"时，一串连续的工具往返被当成**一个回合**，
模型会用校验和（checksum，一小段用来验证数据完整性的指纹）检查这个回合的第一条消息里
带着思考内容。中间删掉任何一条，校验和对不上，接口同样拒收。

**所以"删哪里"不是随便选的，某些位置根本碰不得。** 这一节就是这个问题的系统化解法。

## 二、核心概念：View（视图）

```python
class View(BaseModel):
    events: list[LLMConvertibleEvent]
    unhandled_condensation_request: bool = False
```

**View = 账本的一份加工结果，专门用来喂给 AI。**

用之前那个银行账本的比方：账本是全部流水，View 是"最近三个月的对账单，更早的合并成一行期初余额"。
对账单可以重新生成无数份，流水一个字不动。

注意它的类型是 `list[LLMConvertibleEvent]` —— 上一节讲的那个"只有能变成 AI 消息的事件
才继承这个类"，在这里兑现了：不该给 AI 看的东西，类型上就进不来。

## 三、关键抽象：ManipulationIndices（可操作位置）

```python
class ManipulationIndices(set[int]):
    """A set of indices where events can be safely manipulated."""
```

这是一个**整数集合**，表示"哪些位置是可以动的"。注释给了精确定义：

1. 如果 `i` 在集合里，可以在位置 `i` **插入**任何事件
2. 如果 `i` 和 `j` 都在集合里，可以**删除** `events[i:j]` 这一段

### 用剪纸条来理解

把事件列表想成一条纸带，每两个事件之间有一个"缝隙"。缝隙编号 0、1、2……

```
 [事件0] [事件1] [事件2] [事件3]
0       1       2       3       4      ← 缝隙编号
```

**ManipulationIndices 就是"哪些缝隙可以下剪刀"。** 要删掉中间一段，必须**两刀都落在允许的缝隙上**。

起手是全部允许：

```python
@staticmethod
def complete(events) -> ManipulationIndices:
    manipulation_indices.update(range(0, len(events)))
    manipulation_indices.add(len(events))     # 末尾那个缝隙也算
```

然后每条规则各自**划掉**一些不许下剪刀的缝隙。

还有个实用方法：

```python
def find_next(self, threshold: int) -> int:
    """找到 >= threshold 的最小可操作位置"""
```

压缩器说"我想从第 50 个开始删"，这个方法回答"最近的合法切口在第 53 个"。找不到就抛异常，
而不是硬删。

## 四、四条规则，每条都是一段血泪

规则定义在 `properties/base.py`，每条必须实现两个方法：

```python
class ViewPropertyBase(ABC):
    @abstractmethod
    def enforce(...) -> set[EventID]:
        """这个视图已经坏了，返回该删掉哪些事件来修好它"""

    @abstractmethod
    def manipulation_indices(...) -> ManipulationIndices:
        """返回在这条规则下，哪些缝隙可以下剪刀"""
```

**两个方法是一攻一守**：
- `manipulation_indices` 是**预防**：一开始就不让你剪错地方
- `enforce` 是**急救**：万一还是坏了（脏数据、崩溃恢复、没想到的情况），把坏的部分切掉

类注释把这个分工说得很清楚：

> Enforcement is intended as a fallback mechanism to handle edge cases, bad data, or
> unforeseen situations.
> （enforce 是兜底机制，用来处理边界情况、坏数据或没预料到的状况。）

注册顺序（`properties/__init__.py`）本身有含义，后面会说：

```python
ALL_PROPERTIES = [
    ObservationUniquenessProperty(),
    BatchAtomicityProperty(),
    ToolCallMatchingProperty(),
    ToolLoopAtomicityProperty(),
]
```

### 规则 1：ObservationUniqueness（一个工具调用只能有一个结果）

**问题从哪来**，注释交代得很具体：

> Crash recovery can synthesize an AgentErrorEvent for an in-flight tool call and then
> the original ObservationEvent may still arrive late.
> （崩溃恢复时，会给一个执行到一半的工具调用补一条"错误事件"；然后原本那个结果可能又姗姗来迟。）

于是同一个工具调用就有了**两个**结果——接口只允许一个。

处理：**先到的赢**（"the first occurrence wins because the agent has likely already seen it"
——因为 AI 大概率已经看过它了，后到的丢掉）。

这条规则**不限制任何缝隙**，它只负责清理重复。但如果清理漏了，它的 `manipulation_indices`
会打一条警告日志：

> log a warning so the regression is visible without crashing condensation
> （打警告让这个回归问题可见，但不至于让压缩崩溃）

**这是很成熟的工程判断**：知道自己可能有 bug，于是留一个"出问题时能看见、但不炸"的观察点。

### 规则 2：BatchAtomicity（同一次回复里的工具调用，要么全留要么全删）

上一节讲过：AI 一次回复可以同时发起多个工具调用，被拆成了多行账，靠 `llm_response_id` 关联。

这条规则说：**这些行是一个整体。删掉其中一个，就必须删掉全部。**

因为喂回给 AI 时要把它们合并还原成原本那一条回复——缺了一块就还原不出来。

> **原子性（atomicity）**：借自数据库的词，指一组操作要么全部生效、要么全部不生效，
> 不存在做了一半的状态。

划缝隙的逻辑很简洁——用 `pairwise` 两两相邻地看：

```python
for index, (left, right) in enumerate(pairwise(current_view_events)):
    if (isinstance(left, ActionEvent) and isinstance(right, ActionEvent)
        and left.llm_response_id == right.llm_response_id):
        manipulation_indices.remove(index + 1)   # 这两个之间不许下剪刀
```

`enforce` 的判断方式也值得看：拿**视图里的这一批**和**完整账本里的这一批**比对，
只要不是一模一样，就说明已经缺了东西，整批清掉。

### 规则 3：ToolCallMatching（调用和结果必须配对）

就是本节开头那条铁律。

划缝隙的思路很聪明——**维护一个"欠账集合"**：

```python
pending_tool_call_ids = set()
for index, event in enumerate(current_view_events):
    match event:
        case ActionEvent():
            pending_tool_call_ids.add(event.tool_call_id)      # 发起调用，记一笔欠账
        case ObservationBaseEvent():
            pending_tool_call_ids.discard(event.tool_call_id)  # 结果来了，销账

    if pending_tool_call_ids:      # 还有欠账 = 我正处在"调用和结果之间"
        manipulation_indices.remove(index + 1)
```

一次线性扫描。只要"欠账集合"非空，就说明当前位置夹在某个调用和它的结果中间，这个缝隙锁死。

`enforce` 则是：收集所有调用的编号和所有结果的编号，**两边各自找孤儿**——有调用没结果的删掉，
有结果没调用的也删掉。

### 规则 4：ToolLoopAtomicity（思考回合不能拆）

最隐晦的一条，注释直接点名了触发场景：

> This property is important to enforce for Anthropic models with thinking enabled.
> They expect the first element of such a tool loop to have a thinking block, and use
> some checksums to make sure it is correctly placed.
> （对开启思考模式的 Anthropic 模型必须强制这条。它们要求这种工具回合的第一个元素带有
> 思考块，并用校验和确保它放对了位置。）

**什么叫一个"工具回合"（tool loop）**：从一个**带思考内容**的动作开始，后面紧跟的一连串
动作和结果，中间不夹杂别的东西——这一整段被模型当作**一个回合**。

识别代码是一个小状态机：

```python
for event in events:
    match event:
        case ActionEvent() if event.thinking_blocks:
            # 带思考内容的动作 = 新回合的开始
            开始新回合()
        case ActionEvent() | ObservationBaseEvent():
            # 普通动作或结果 = 如果在回合里就算进去，不在就不开新的
            if 在回合里: 加入当前回合()
        case _:
            # 别的任何东西（比如用户插话）= 回合结束
            结束当前回合()
```

回合内部的所有缝隙全部锁死。`enforce` 则检查：如果一个回合只有部分还在视图里，
**把它剩下的部分也全删掉**。

## 五、组合方式：取交集

回到 `view.py`，四条规则是这样合起来的：

```python
@property
def manipulation_indices(self) -> ManipulationIndices:
    results = ManipulationIndices.complete(self.events)
    for property in ALL_PROPERTIES:
        results &= property.manipulation_indices(self.events)
    return results
```

`&=` 是集合的**交集**运算。

**含义：从"全部缝隙都可以剪"出发，每条规则划掉自己不允许的，剩下的才是真正安全的切口。**

这个设计的漂亮之处在于**可扩展**：明天出现一个新模型、有一条新的结构约束，你只要写一个新的
Property 类塞进 `ALL_PROPERTIES` 列表，**其它代码一行都不用改**，压缩器自动就会绕开新的禁区。

> 这在设计上叫"开闭原则"——对扩展开放（能加新规则），对修改封闭（不用改老代码）。
> 很多人背过这个词，很少有人见过这么干净的实现。

## 六、急救逻辑：递归收敛

`enforce_properties` 的写法有个细节值得看：

```python
for property in ALL_PROPERTIES:
    events_to_forget = property.enforce(self.events, all_events)
    if events_to_forget:
        logger.warning(f"Property {property.__class__} enforced, "
                       f"{len(events_to_forget)} events dropped.")
        self.events = [e for e in self.events if e.id not in events_to_forget]
        break          # ← 注意这里是 break，不是 continue
else:
    return             # ← for-else：一轮下来没人触发，说明干净了，收工

self.enforce_properties(all_events)   # ← 有人触发过，从头再来一遍
```

**为什么一旦有规则生效就 break，然后整个重来？**

因为**规则之间会互相影响**。规则 2 删掉了一批工具调用之后，那批调用对应的结果就变成孤儿了，
于是规则 3 现在有活干了——而规则 3 在这一轮里可能已经检查过、当时是干净的。

所以只能：**删一次 → 全部重新检查 → 直到一整轮都没人有意见**。这是一个**收敛到稳定状态**
的过程。

> 这里用到了 Python 的 `for...else` 语法：`else` 分支只在循环**没有被 break** 时执行。
> 用来表达"一整轮下来平安无事"非常贴切，是个冷门但正确的用法。

还有一点：每次 `enforce` 生效都会**打一条 warning 日志**。因为按设计，预防机制
（manipulation_indices）应该已经挡住了一切，急救逻辑本不该被触发。**一旦触发，
说明预防那边有漏洞，日志是留给开发者的线索。**

## 七、View 怎么造出来的

```python
@staticmethod
def from_events(events) -> View:
    result = View()
    for event in events:
        result.append_event(event)      # 一条一条加
    result.enforce_properties(events)   # 加完统一体检
    return result
```

`append_event` 里有个关键分支，处理"压缩"这件事：

```python
match event:
    case Condensation():          # 一条"压缩记录"
        self.events = event.apply(self.events)     # 直接应用它的效果
        self.unhandled_condensation_request = False
    case CondensationRequest():   # 一条"请求压缩"的标记
        self.unhandled_condensation_request = True
    case LLMConvertibleEvent():   # 普通的、该给 AI 看的事件
        self.events.append(event)
    case _:                       # 其它内部事件，跳过
        logger.debug(...)
```

**注意"压缩"本身也是账本上的一行。** 这就是上一节"只增不改"的兑现方式：压缩不是把旧事件
删掉，而是**追加一条"从这里开始，前面那些用这段摘要代替"的记录**。

重放账本时，读到这一行就应用它的效果。于是：
- 真实历史永远完整
- 压缩策略换了，重放一遍就得到新的视图
- 想查"当时到底发生了什么"，账本里全都在

## 八、这一节的可迁移结论

1. **把"约束"变成一等公民**：不要让"不能删这里"散落在各处的 if 判断里，
   把它做成一个可以独立测试、独立增删的对象。

2. **预防和急救分开，且急救要报警**：预防机制保证正常路径正确，急救机制处理意外，
   但每次急救生效都记一条警告——因为它生效就意味着预防那边漏了。

3. **多个约束用交集组合**：每条规则只管自己那一摊、返回自己的允许集合，交集就是最终答案。
   加新规则零成本。

4. **互相影响的规则要循环到收敛**：删一次就全部重查，不要假设一轮能搞定。

5. **不可逆操作要做成可重放的记录**：压缩不是删除，是追加一条"这里开始用摘要代替"的账。

## 下一站

`agent/agent.py` 的 `_step()`（645-830 行）—— AI 决策的单步流程，
以及四种 LLM 报错各自的恢复策略。那里会看到本节的视图被真正用起来。

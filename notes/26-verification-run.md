# 实跑验证：笔记里的断言 vs 真实行为

> 2026-10-03。用 `deepseek/deepseek-v4-flash` 实跑 `examples/01_standalone_sdk/`
> 里的四个例子，核对前面 25 篇笔记里写下的具体断言。
>
> **总花费 ≈ $0.052。** 工作目录全在临时目录，没有碰仓库。
>
> 这篇的价值不在"跑通了"，而在**三类结果**：被证实的、被证伪/需要修正的、
> 以及**笔记里完全没预料到的**。

---

## 一、被实跑证实的断言

### 1. 压缩器的切点算术（`notes/06-condenser.md`）

例子用的配置是 `max_size=10, keep_first=2`。笔记里的推算是：

```
Reason.EVENTS → target_size = max_size // 2 = 5
suffix_events_to_keep = target_size - keep_first - 1 = 2
naive_end = len(view) - 2
forgetting_start = find_next(keep_first) = 2
summary_offset == forgetting_start
```

**实跑结果（7 次压缩的落盘数据）：**

| 第几次 | forgotten_event_ids 数量 | summary_offset |
|---|---|---|
| 1 | **8** | **2** |
| 2–7 | **7** | **2** |

- `summary_offset = 2` 七次全部命中 —— **笔记里"摘要插入位置等于遗忘起点、
  而遗忘起点是 `find_next(keep_first)`"完全正确。**
- 遗忘 7 个 = 区间 `[2, 9)` ⇒ `naive_end = 9` ⇒ 触发时 `len(view) = 11`。
  第一次遗忘 8 个 ⇒ `len(view) = 12`。
  **这和"一轮加两个事件（动作+观察），所以视图长度按偶数跳"吻合**：
  压缩后视图是 `2(保留) + 1(摘要) + 2(尾部) = 5`，然后 5→7→9→11 触发。

### 2. 摘要事件不进账本（`notes/06-condenser.md`）

笔记说：`CondensationSummaryEvent` 是从压缩记录**派生**出来的，不独立存盘。

**实跑的事件类型统计（58 个落盘事件）：**

```
ActionEvent: 20    ObservationEvent: 20    MessageEvent: 10
Condensation: 7    SystemPromptEvent: 1
```

**没有一个 `CondensationSummaryEvent`。** ✓

### 3. 动作和观察严格配对（`notes/03-view-and-properties.md`）

`ActionEvent: 20` / `ObservationEvent: 20` —— **数量完全相等**。
这正是 `ToolCallMatchingProperty` 要维护的不变量。✓

而且 `SystemPromptEvent: 1` 验证了第 6 节那句
"a view is expected to have one system prompt event"。✓

### 4. 摘要提示词真的在按模板产出（`notes/06-condenser.md`）

笔记分析了 `summarizing_prompt.j2` 要求的段落结构。**实际产出的摘要：**

```
USER_CONTEXT: User requested `math_utils.py` with basic arithmetic functions,
then requested adding `factorial`; now requested adding a function to check
if a number is prime.

TASK_TRACKING: No formal task IDs assigned.
- Add `factorial` function to `math_utils.py`: completed (added and tested).

COMPLETED:
- Created `/private/tmp/.../math_utils.py` ...
```

`USER_CONTEXT` / `TASK_TRACKING` / `COMPLETED` 三个段落名逐字出现，
而且**"已完成"和"待办"确实被分开了**（笔记里强调的那条防重做的要求）。✓

### 5. `ActionEvent.summary`（`notes/02-event-model.md`）

笔记说这个字段要求模型用约 10 个词概括动作，并列了注释里的五个范例。

**实跑输出里每个动作都带着这一行：**

```
Summary: Create FACTS.txt with 3 project facts
Summary: fetch: {"url": "https://github.com/OpenHands/OpenHands"}
```

✓ 而且**连 MCP 工具调用也有**。

### 6. 系统提示的片段顺序（`notes/09-context-engineering.md`）

笔记里从 `preset.py` 读出的静态片段顺序是 `SoulSection(), RoleSection(), ...`。

**实跑 dump 出的第 0 条消息：**

```
role='system' content=[... text='<SOUL>\nYou are OpenHands agent, a helpful AI
assistant that can interact with a computer to solve tasks.\n</SOUL>\n\n<ROLE>\n...
```

`<SOUL>` 在最前、`<ROLE>` 紧随其后 —— **装配顺序和注册顺序一致。** ✓

### 7. 提示缓存真的在生效（`notes/09-context-engineering.md` / `notes/10-llm-seam.md`）

单次运行的终端输出里直接打了命中率：

```
Tokens: ↑ input 8.39K (total 31.92K) • cache hit 97.61% (total 73.79%)
```

而落盘的累计数据更有说服力（下面第三节会用到）：

| 用量桶 | prompt_tokens | cache_read_tokens | 命中率 |
|---|---|---|---|
| `agent` | 221,798 | 201,344 | **90.8%** |
| `condenser` | 9,368 | 2,304 | 24.6% |

**主 agent 九成的输入走了缓存** —— 第 9 节那套"静态块在前、时间放最后"的
排布是真的在省钱。而**压缩器命中率低**也完全符合预期：
每次摘要的提示内容都不同，没有可复用的前缀。✓

### 8. 卡死检测与"推一把"的分工（`notes/05-run-loop.md`）

实跑日志：

```
WARNING  Action, Observation loop         stuck_detector.py:182
WARNING  Stuck pattern detected.          local_conversation.py:753
Final stuck status: True
```

- 命中的是四种模式里的第一种（重复的动作-观察循环）✓
- 日志文本 `"Stuck pattern detected."` 和笔记引用的源码逐字一致 ✓
- **没有出现"推一把"的提醒** —— 这正是笔记里写的：
  提醒只针对"同一动作反复**报错**"，而这次是"同一动作反复**成功**"的循环。
  **一个本来可能被当成遗漏的现象，其实是设计使然。** ✓

### 9. 按 usage_id 分桶记账（`notes/15-routing.md` / `notes/19-secrets-and-git.md`）

落盘的 `stats.usage_to_metrics` 里有**两个独立的桶**：

```
agent        cost=$0.018503
condenser    cost=$0.022001
```

笔记里那条"派生的模型实例要挂在同一份统计上、但按 usage_id 分桶"
**得到了直接证据**。✓

### 10. 会话状态里的那些去重字段（`notes/09`/`notes/04`/`notes/02`）

`base_state.json` 的顶层键里逐一对上了：

```
activated_path_rules        ← 第 9 节：路径规则每条只注入一次
activated_knowledge_skills  ← 第 9 节：技能触发去重
invoked_skills
blocked_actions             ← 第 4 节：钩子拦下的动作
blocked_messages            ← 第 4 节：钩子拦下的用户消息
leaf_event_id, head_is_empty ← 第 2 节：事件树的当前叶子节点
confirmation_policy          ← 第 7 节（实跑值是 NeverConfirm）
```

**笔记里从源码读出的每一个状态字段，都在真实存档里存在。** ✓

### 11. MCP 工具被统一成同一个接口（`notes/11-tool-layer.md`）

```
INFO  Created 1 MCP tools        utils.py:449
...
Action: MCPToolAction
  kind: "MCPToolAction"
```

笔记说 `MCPToolDefinition` 就是 `ToolDefinition` 的子类、
上层派发零分支 —— **实跑里它和本地工具走的是完全相同的动作/观察流程**
（连 `Summary:` 字段都有）。✓

---

## 二、需要修正的地方

### 修正 1：`max_iterations` 是存档里的字段名

笔记（`notes/05-run-loop.md`）一直用构造参数名 `max_iteration_per_run`。
**落盘的 `base_state.json` 里字段名是 `max_iterations`。**

这和笔记里读到的那一行是一致的（`max_iterations=max_iteration_per_run`），
但**对着存档找字段时会找不到**，值得标一下。

### 修正 2：MCP 的版本兼容问题是真实存在的，而且现在就在发生

`13_get_llm_metrics.py` 跑到调用 MCP 工具时失败：

```
ERROR  Failed to parse JSONRPC message    __init__.py:157
       ValidationError: 1 validation error
```

例子里固定的是 `mcp==1.29.0` + `mcp-server-fetch==2026.7.10`，
和当前环境里装的 `mcp` 库对不上。

**第 11 节笔记说"MCP 库 1.x 和 2.x 的字段命名习惯不一样，所以落盘要用协议规定的名字"**
—— 这条判断是对的，但笔记的语气像是在讲一个"已经被处理掉的历史问题"。
**实跑说明这个生态的版本漂移现在仍然会让集成直接失败。**

---

## 三、笔记完全没预料到的发现

### 压缩器比被压缩的 agent 还贵

```
agent        cost=$0.018503   completion_tokens=9,299
condenser    cost=$0.022001   completion_tokens=16,557
```

**压缩器花了 $0.0220，主 agent 花了 $0.0185 —— 压缩占了总花费的 54%。**
而且压缩器产出的 token 数（16,557）几乎是主 agent 的**两倍**。

**为什么？** 这个例子把 `max_size` 设成了 10（真实默认值是 **240**）。
于是：

- 压缩后视图只剩 5 个事件
- 每轮对话加 2 个事件
- **只撑 3 轮就又触发压缩**
- 5 条用户消息的对话里触发了 **7 次**压缩

**笔记里（`notes/06-condenser.md`）的原话是**：

> **为什么都是"砍到一半"而不是"砍到刚好不超"**……砍到一半，就能撑很多步不用再压。
> 这是典型的"批量处理换取摊销成本"。

**这句话本身没错，但它隐含了一个前提：`max_size` 足够大，摊销才成立。**
实跑证明了反面：**当 `max_size` 相对于"每轮产生的事件数"太小时，
摊销完全失效，压缩从一个省钱手段变成最大的开销项。**

**这给出一个可以量化的配置约束**：

```
压缩后的视图长度 = keep_first + 1(摘要) + 尾部保留数
每轮增加         = 2（动作 + 观察），多工具并行时更多
撑住的轮数       = (max_size - 压缩后长度) / 每轮增加
```

代入这个例子：`(10 - 5) / 2 = 2.5 轮`。代入默认值 `max_size=240`：
压缩后约 `2 + 1 + 119 = 122`，`(240 - 122) / 2 = 59 轮` —— **差 24 倍。**

> **可迁移的道理**：**任何"攤销式"的优化都有一个最小规模要求。**
> 低于那个规模，优化本身的开销会超过它节省的。
> 设计这类机制时应该**把"撑住多少轮"算出来写进文档**，
> 而不只是说"砍到一半是为了摊销"。
>
> 而这个例子本身是**演示用的**（故意把阈值调低好让压缩可见），
> 所以这不是 SDK 的 bug —— 但它说明**文档应该警告"不要把 max_size 调到接近
> 每轮事件数的量级"。**

---

## 四、这次验证的方法论收获

1. **落盘数据比终端输出可靠得多。** 终端输出是给人看的渲染结果（会截断、会省略），
   而 `base_state.json` + `events/*.json` 是**程序读写的真实状态**。
   验证一个库的行为，优先去读它的持久化格式。

2. **"没有发生的事"也是证据。** 事件类型统计里**没有** `CondensationSummaryEvent`，
   日志里**没有**"推一把"的提醒 —— 两者都验证了笔记里的具体断言。
   **验证时要明确列出"按笔记推断，什么不应该出现"。**

3. **数量关系最容易暴露理解错误。** `ActionEvent: 20 / ObservationEvent: 20`
   相等、`summary_offset` 七次都是 2、遗忘数量 8/7/7/7/7/7/7 ——
   **这些数字要么全对要么全错，比读代码的"看起来对"强得多。**

4. **成本数据会暴露文档里的隐含前提。** 如果不看那两个用量桶的对比，
   "砍到一半是为了摊销"这句话会一直看起来是对的。

---

# 补记（2026-10-08）：哪些结论其实是"模型相关"的

上面那 11 条写得像"全部证实",**这不够诚实**。按"什么决定了这个观察结果"重新分类：

## A. 真正与模型无关的（6 条）

由代码结构决定,模型只是触发器：没有 `CondensationSummaryEvent` 落盘、
`SystemPromptEvent: 1`、`<SOUL>` 在 `<ROLE>` 前、`base_state.json` 的字段集、
MCP 工具走同一流程、MCP 的 JSONRPC 失败（库版本问题,与模型无关）。

## B. 被我过度归因的（5 条）

### ① 缓存命中率 90.8% —— 这几乎完全是厂商特性

我写的是"静态块在前、时间放最后的排布**是真的在省钱**"。

**但 DeepSeek 有自动前缀缓存,不需要调用方做任何事。** 而 dump 出的消息里
`cache_prompt=False` —— **SDK 的显式缓存断点机制根本没被用上。**

那 90.8% 可能是 DeepSeek 的自动缓存贡献的,**不是 SDK 的排布设计贡献的**。
要分清需要一个对照（把时间戳挪到动态块开头看命中率掉多少）,**我没做这个对照**。
换到需要显式 `cache_control` 断点的 Anthropic 结论可能完全不同。

### ② "压缩器比 agent 还贵" —— 定价结构在起作用

agent 的开销主要是输入（九成走便宜的缓存）,压缩器的开销主要是输出。
**DeepSeek 的输出单价远高于缓存输入单价,这个比例是它的定价表决定的。**

而且 7 次摘要产出 16,557 token ≈ 每次 2,365 token —— 相当啰嗦。
另一个模型给 400 token 的摘要,**结论就反过来了**。

**但"摊销有最小规模要求"那条推论是靠算术成立的,与模型无关。**
教训：**结论和证据要分开写,我当时混在一起了。**

### ③ 卡死检测触发 —— 几乎纯粹是模型能力

**一个强模型可能根本不会陷入那个循环。** 而且命中哪一种模式也由模型决定——
这次命中第 1 种（动作-观察循环）,所以"推一把"没出现。

我那句"没有提醒是设计使然"的论证来自**读源码**,
**这次实跑没有证明它,只是没有反驳它。**

### ④ `ActionEvent: 20 / ObservationEvent: 20` 相等 —— 是运气

不变量是"每个工具调用有恰好一个**观察类**事件"。模型发畸形调用时会生成
`AgentErrorEvent` 而非 `ObservationEvent`,**两类计数就不等,但不变量仍满足**。
**我验证的是一个特例,不是不变量本身。**

### ⑤ `summary` 字段被填了 —— 填不填是模型的事

字段在 schema 里要求,弱模型可能省略,然后 `_extract_summary` 返回 `None`。

## C. 完全没被触及的代码路径（比"结论可能错"更严重）

日志里的 `reasoning 0` 和"一次只调一个工具"说明：

| 机制 | 笔记 | 需要什么 |
|---|---|---|
| `ToolLoopAtomicityProperty` | 第 3 节,**整个 Property 类** | 带 thinking blocks 的模型 |
| "thought 只存首个事件"的断言 | 第 2 节 | 推理模型 + 并行调用 |
| `BatchAtomicityProperty` | 第 3 节 | **并行工具调用** |
| `_combine_action_events` | 第 2 节,**全文唯一的算法** | 并行工具调用 |
| `ParallelToolExecutor` + 资源锁 | 第 3 节 | 并行工具调用 |
| `fn_call_converter`（963 行 XML 协议） | 第 9 节 | 不支持原生函数调用的模型 |
| Responses API 分支 | 第 9、17 节 | OpenAI 系模型 |
| 显式提示缓存断点 | 第 8 节 | Anthropic |
| **tmux 终端后端 + PS1 元数据技巧** | **第 12 节** | **装了 tmux 的机器** |

**最后一条是这次新发现的**：这台机器没装 tmux,日志里有

```
WARNING  tmux is not installed. Falling back to subprocess-based terminal,
         which may be less stable. For best agent performance, install tmux
```

**所以之前所有的终端调用走的是 `SubprocessTerminal` 回退路径。**
第 12 节笔记里那个"把元数据藏在 PS1 提示符里"的漂亮技巧,
**一次都没有被执行过。**（而这个回退**打了警告并给出安装指引**,
符合索引第 2 条和第 6 条。）

---

# 补记二：三个在"付费调用之前"免费拿到的发现

尝试跑 Claude 时连续失败三次,**每次都在发出 API 请求之前**,所以花费 $0。
但三次失败各自是一个真实发现：

## 发现 1：压缩器的配置校验器真的在构造时就拦（证实索引第 39 条）

把 `max_size=6, keep_first=2` 传进去,**对象创建就失败**：

```
ValidationError: Value error, keep_first must be less than max_size // 2
                 to leave room for condensation
```

代入第 6 节笔记引的公式：`6 // 2 - 2 - 1 = 0` → `<= 0` → 拒绝。**完全一致。**
最小可用值是 `max_size=8`。

**这正是"配置的自相矛盾在创建时就报错"的实况,而且它省下了一次无意义的付费运行。**

## 发现 2：预算硬闸门在公开入口上够不着 ⚠️ 需要修正第 5 节

```
TypeError: Conversation.__new__() got an unexpected keyword argument
           'max_budget_per_run'
```

第 5 节笔记把步数上限和预算上限并列成"两道硬闸门"。
**但 `Conversation` 这个工厂的签名里没有 `max_budget_per_run`** ——
它只存在于 `LocalConversation.__init__` 上（注释说它"追加在参数表末尾
以免挪动已有位置参数"）。

**也就是说：走官方推荐的入口 `Conversation(...)` 设不了预算上限,
必须直接构造 `LocalConversation`。**

对比一下：`max_iteration_per_run: int = 500` **是**在工厂签名里的。
**两道闸门的可达性并不对称** —— 笔记里"并列"的写法掩盖了这一点。

> **可迁移的道理**：**一个"追加在参数表末尾以保持兼容"的参数,
> 很容易漏掉上层的转发层。** 加参数时要把整条调用链上的工厂/包装都过一遍,
> 否则这个功能对大多数用户等于不存在。

## 发现 3：出错时真的带上了会话 ID（证实第 5 节）

余额不足那次,最终抛出的是：

```
ConversationRunError: Conversation run failed for id=9d59b549-6a39-43aa-...
```

第 5 节笔记说"重新抛出时附上会话 ID 和存档目录,注释写的是 `for better UX`"。
**实跑逐字对上。** 而且在抛出之前先发了一条 `ConversationErrorEvent`
（日志里有 `Event type ConversationErrorEvent is ...`）—— 
**"错误也是账本上的一条事件"也被证实。**

---

# 补记三：能力表落后于模型发布,而且失败是静默的

为了选模型,我离线查了 SDK 的能力表：

| 模型 | thinking_mode | supports_extended_thinking |
|---|---|---|
| `anthropic/claude-sonnet-5` | **none** | **False** |
| `anthropic/claude-haiku-4-5-20251001` | manual | True |
| `anthropic/claude-sonnet-4-5` | manual | True |

因为 `EXTENDED_THINKING_MODELS` 这个列表里只有三项：

```python
EXTENDED_THINKING_MODELS: list[str] = [
    "claude-sonnet-4-5", "claude-sonnet-4-6", "claude-haiku-4-5",
]
```

**最新的 Sonnet（`claude-sonnet-5`）不在里面。** 而匹配是子串匹配,
所以用它跑**根本不会开启扩展思考,也不会有任何警告** ——
`thinking_mode` 静默地变成 `none`。

**这和第 9 节笔记里称赞的那条"验证过的模型列表"纪律是同一张表的两面**：
纪律让列表保持精简可信,**但列表落后于模型发布时,新模型会静默地失去能力**。

> **可迁移的道理**：**"按名单启用能力"的设计,在名单落后时是静默降级的。**
> 如果这个能力对效果影响很大（扩展思考就是),
> 应该在"模型支持推理但不在名单里"时打一条警告 ——
> 现在的代码在那种情况下直接返回 `"unknown"` 然后 `"none"`,一声不响。
>
> 顺带：**这次是靠先离线查能力表才避免了一次白花钱的运行。**
> 对着一个库做付费验证时,**先用它自己的能力查询接口做预检**。

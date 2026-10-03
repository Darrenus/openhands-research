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

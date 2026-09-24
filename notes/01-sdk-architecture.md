# OpenHands software-agent-sdk 架构拆解

对象：`OpenHands/software-agent-sdk`（MIT，Python，SWE-bench Verified 77.6，技术报告 arXiv:2511.03690）
拆解日期：2026-09-24（shallow clone of main）

## 0. 先搞清楚仓库关系

OpenHands 现在是多仓库结构，很多人会拉错仓库：

| 仓库 | 内容 | 学架构看这里？ |
|---|---|---|
| `OpenHands/OpenHands`（89k★） | Agent Canvas —— React 前端控制台 | ✗ 只有 UI |
| **`OpenHands/software-agent-sdk`（1.1k★）** | **agent / tools / events / conversation / agent-server** | **✓ 全部核心** |
| `OpenHands/extensions` | skills、automations、MCP 集成 | 辅助 |
| `OpenHands/automation` | 调度、webhook、运行历史 | 辅助 |

星数和含金量在这里完全不成正比。

## 1. 规模

| 包 | 行数 | 职责 |
|---|---|---|
| `openhands-sdk` | 74,638 | 核心：agent 循环、事件、上下文、LLM 接缝、安全 |
| `openhands-agent-server` | 29,851 | REST/WebSocket 服务端 |
| `openhands-tools` | 16,839 | 工具实现（终端、文件编辑、浏览器、委派…） |
| `openhands-workspace` | 3,067 | 本地 / Docker / K8s 工作区 |

合计约 12.4 万行。作为对比，codeloop 的循环是 110 行——差的不是循环，是循环周围的一切。

## 2. 四层抽象

```
Conversation  ← 会话：状态、事件日志、run 循环、分支
   └─ Agent   ← 单步决策：组装消息 → 调 LLM → 分类响应 → 派发工具
        ├─ LLM      ← 模型接缝（litellm 之上，含路由、遥测、fn-call 转换）
        ├─ Tool     ← 工具接缝（本地工具 / MCP 工具统一成同一个 schema）
        └─ Workspace← 执行接缝（本地 / Docker / 远程）
```

关键划分：**Conversation 拥有循环和状态，Agent 只负责"走一步"**。这让同一个 Agent 既能被本地 CLI 驱动，也能被服务端 WebSocket 驱动（`LocalConversation` 3128 行 vs `RemoteConversation` 1846 行，同一接口两套实现）。

## 3. 核心抽象一：Event（事件流）

`event/base.py`

```python
class Event(DiscriminatedUnionMixin, ABC):
    model_config = ConfigDict(extra="forbid", frozen=True)   # 不可变
    id: EventID
    timestamp: str
    source: SourceType            # user / agent / environment
    parent_id: EventID | None     # ← 关键：事件是树，不是列表
```

三个设计要点：

1. **不可变（frozen）** —— 事件写进日志就不能改，历史可审计、可回放。
2. **有 parent_id，构成树** —— 同一个 parent 下的多个事件是**兄弟分支**。这意味着会话可以分叉（试一条路不行，回退换一条），`state.active_branch()` 取当前活跃分支。这是比"线性消息列表"强得多的建模。
3. **`LLMConvertibleEvent` 子类才有 `to_llm_message()`** —— 事件流里很多东西（hook 执行、条件压缩记录、token 流）不该进模型上下文，靠类型分开而不是靠字段过滤。

## 4. 核心抽象二：View（上下文视图）

`context/view/view.py`

事件日志是全量、只增的；喂给模型的是 **View** —— 一个线性化的、被压缩过的子集。

最精妙的是 `manipulation_indices`：

```python
results = ManipulationIndices.complete(self.events)
for property in ALL_PROPERTIES:
    results &= property.manipulation_indices(self.events)
```

含义：**不是所有位置都能随便删**。比如一个 `tool_use` 和它对应的 `tool_result` 必须成对出现，删掉一半模型 API 会直接报错。每条 property（如 `tool_loop_atomicity.py`）各自算出"哪些下标可以安全切割"，取交集才是真正可动的位置。

这是"上下文压缩"这件事里最容易踩坑、也最少有项目认真处理的部分。

## 5. 核心抽象三：Condenser（压缩器）

`context/condenser/llm_summarizing_condenser.py`

```python
max_size: int = 240          # 事件数上限
max_tokens: int | None       # token 上限
keep_first: int = 2          # 开头 N 条永不压缩（系统提示、原始任务）
minimum_progress: float = 0.1  # 一次至少压掉 10%，否则视为失败
hard_context_reset_max_retries: int = 5
hard_context_reset_context_scaling: float = 0.8  # 压不动就把每条事件截短 20% 再试
```

触发方式有三种（`Reason` 枚举）：`REQUEST`（显式请求）/ `TOKENS`（超 token）/ `EVENTS`（超条数）。

用**独立的 LLM** 做摘要（可以配便宜模型），且注释明确说不要假设它和 agent 的 LLM 是同一个。

## 6. 主循环：两层

### 外层 `LocalConversation.run()`（local_conversation.py:1903）

```
while True:
    if status in (PAUSED, STUCK): break
    if status == FINISHED:
        run_stop_hook()           # ← hook 可以否决"结束"，塞反馈继续跑
        if allowed: break
    if _check_stuck_or_nudge(): continue   # ← 卡死检测
    agent.step(...)
    if status == WAITING_FOR_CONFIRMATION: break
    if budget_exceeded: break
```

有意思的细节（源码注释里写了）：**故意不在 step 后检查 FINISHED**，因为用户可能在 agent 刚宣布完成的瞬间并发发来新消息；留一个循环周期让 `send_message()` 把状态改回 IDLE，消息才不会丢。这是真跑过生产才会遇到的问题。

### 内层 `Agent._step()`（agent/agent.py:645）

顺序：

1. **未执行的待确认动作优先** —— 有 pending action 就先执行（"再调一次 run" 即视为用户批准）
2. 检查 hook 是否拦截了用户消息
3. `llm.resolve_runtime_metadata()` —— 先确定真实端点的上下文窗口，再让 condenser 判断阈值
4. `prepare_llm_messages(state.view, condenser, llm)` —— 返回消息，**或者**返回一个 `Condensation` 事件（这一步不调模型，直接 return，下一轮再来）
5. 非多模态模型收到图片时：有 `vision_inspect` 工具就把图片换成引用，没有就告知用户并结束
6. `llm.generate(..., add_security_risk_prediction=True)` —— **让模型在输出动作时自评风险等级**
7. 按响应类型分派：`TOOL_CALLS` / `CONTENT` / `REASONING_ONLY | EMPTY`

### 错误处理是这段代码的精华

四种异常四种策略，全部不是简单 raise：

| 异常 | 处理 |
|---|---|
| `FunctionCallValidationError` | 把错误当成 user 消息塞回去，让模型自己改 |
| `LLMContentPolicyViolationError` | 注入"你被内容过滤拦了，换个说法继续" |
| `LLMMalformedConversationHistoryError` | `state.rebuild_view()` 重建视图 + 触发压缩（注释指出这通常暴露上游事件流 bug） |
| `LLMContextWindowExceedError` | 触发压缩重试；没有 condenser 才抛 |

## 7. 安全与审批

三道独立的关卡：

**① 模型自评风险**（`security/risk.py`）
`SecurityRisk` = UNKNOWN / LOW / MEDIUM / HIGH，作为工具调用参数由模型自己填。

**② 确认策略**（`agent.py:1046 _requires_user_confirmation`）
```
单个 FinishAction / ThinkAction → 永不确认
其余 → security_analyzer 给出风险 → confirmation_policy.should_confirm(risk)
```
策略与风险评估解耦：可以配"只有 HIGH 才问我"。

**③ 纵深防御**（`security/defense_in_depth/`，3619 行）
- `shell_semantics.py` + `_shell_ast.py` —— **真的解析 shell AST**，不是正则匹配危险词
- `pattern.py` / `policy_rails.py` —— 规则护栏
- `toolshield_llm_analyzer.py` —— 用 LLM 做工具调用分析
- `grayswan/` —— 对抗性检测

这是 codeloop 标为「Stage 6 未做」的东西，这里是 3600 行的完整子系统。

## 8. 卡死检测（`conversation/stuck_detector.py`）

四种模式：
1. 重复的 action-observation 循环
2. 同一动作反复报同样的错（有 `_action_error_streak`）
3. Agent 自言自语（无用户输入的重复消息）
4. 交替往复模式

检测到不只是停 —— `get_action_error_nudge()` 会生成一条"推一把"的提示注入回去。

## 9. Critic（自评 / 迭代精修）

`critic/` 1431 行。`IterativeRefinementConfig`：

```python
success_threshold: float = 0.6
max_iterations: int = 3
```

`conversation.run()` 结束后跑 critic 打分，低于阈值自动带着反馈重试。内置实现有 `agent_finished` / `empty_patch`（空 diff 直接判失败）/ `pass_critic`，还有一个走 API 的模型 critic（`impl/api/`，带 taxonomy）。

**这是 SWE-bench 77.6 分那种成绩里很关键的一环**：不是一次做对，是做完自己检查再改。

## 10. 多 Agent 委派

`openhands-tools/openhands/tools/delegate/` + `sdk/subagent/`

委派是一个**工具**，两个命令：
- `spawn` —— 按 id 列表创建子 agent，可指定 `agent_types`（researcher / programmer / …）
- `delegate` —— 把 `{id: task}` 字典派下去

子 agent 定义在 `tools/preset/subagents/`，注册表在 `subagent/registry.py`。

## 11. 工具层

`openhands-tools/openhands/tools/` 提供的工具：

terminal（含独立的 terminal session 管理）、file_editor、apply_patch、planning_file_editor、glob、grep、browser_use（带注入 JS）、task / task_tracker、delegate、ask_oracle、tom_consult、workflow，另有一整套 `gemini/`（read_file / write_file / edit / list_directory —— 对齐 Gemini CLI 的工具语义）。

内置工具在 SDK 侧 `tool/builtins/`：`think`、`finish`、`switch_llm`、`classify_and_switch_llm`（**按任务难度自动切模型**）、`vision_inspect`、`invoke_skill`。

MCP 工具（`sdk/mcp/`，2085 行）被转换成与本地工具同一个 `ToolDefinition`，agent 侧无分支。

## 12. 对比 codeloop：差距在哪

| 维度 | codeloop | OpenHands SDK |
|---|---|---|
| 历史结构 | 线性消息列表 | 不可变事件树（可分支） |
| 上下文管理 | 无 | View + property 安全切点 + LLM 摘要压缩 |
| 审批 | 二元（只读跳过 / 其余询问） | 模型自评风险 × 策略 × shell AST 纵深防御 |
| 卡死处理 | 步数上限 | 四种模式检测 + 主动 nudge |
| 结果校验 | 无 | Critic + 迭代精修 |
| 多 agent | 无 | spawn/delegate 工具 + subagent 注册表 |
| 远程 | 无 | agent-server（REST/WS）+ TS 客户端 |
| 实测 | **零次跑分** | SWE-bench Verified 77.6 + 技术报告 |

## 13. 建议的精读顺序

1. `sdk/event/base.py`（199 行）—— 先理解事件模型，其余都建立在它上面
2. `sdk/context/view/view.py` + `view/properties/tool_loop_atomicity.py` —— 上下文安全切割
3. `sdk/agent/agent.py:645-830` —— 单步决策与错误恢复
4. `sdk/conversation/impl/local_conversation.py:1903-2050` —— 外层循环
5. `sdk/context/condenser/llm_summarizing_condenser.py` —— 压缩策略
6. `sdk/security/defense_in_depth/shell_semantics.py` —— shell AST 安全分析
7. `sdk/critic/base.py` —— 迭代精修
8. `examples/01_standalone_sdk/` —— 跑起来验证理解

> 注意：`agent/acp_agent.py` 有 4681 行，是 Agent-Client Protocol 适配层（让这个 SDK 能驱动 Claude Code / Codex / Gemini 等外部 agent）。**它不是核心架构**，第一遍读可以整个跳过。

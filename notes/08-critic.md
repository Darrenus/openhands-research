# 精读 07：评审员与迭代精修（critic/，1431 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/critic/`
> 文件：`base.py`、`result.py`、`impl/{empty_patch,agent_finished,pass_critic}.py`、
> `impl/api/{critic,client,taxonomy,chat_template}.py`
> 相关：`agent/critic_mixin.py`
> 前置：`notes/04-agent-step.md`（第七节 `finalize` 那段）、`notes/05-run-loop.md`

## 这一节在讲什么

AI 说"我做完了"——**凭什么信它？**

这一节是这个问题的答案。核心机制叫**迭代精修**（iterative refinement）：
做完自己检查一遍，不合格带着反馈重做。

> 这是 SWE-bench 77.6 那个分数里很关键的一环。**不是一次做对，是做完自查再改。**

## 一、评分的基本结构（`result.py`）

```python
class CriticResult(BaseModel):
    THRESHOLD: ClassVar[float] = 0.5
    score: float = Field(description="A predicted probability of success between 0 and 1.",
                         ge=0.0, le=1.0)
    message: str | None
    metadata: dict[str, Any] | None
```

注意 `score` 的定义不是"质量分"，而是**"预测的成功概率"**（predicted probability of success）。

**这个措辞的区别很重要。** "质量 8 分"是个主观标签；"成功概率 80%"是一个**可以被验证**
的预测——真跑一遍就知道对不对。于是评审员本身的准确率是可以被度量、被改进的。

> 可迁移的道理：**让评价指标变成可验证的预测，而不是主观标签。**
> 前者能形成"预测 → 验证 → 校准"的闭环，后者只能吵架。

## 二、什么时候评审：两种模式

```python
mode: Literal["finish_and_message", "all_actions"] = "finish_and_message"
# - 'finish_and_message': Evaluate on FinishAction and agent MessageEvent
#   (default, minimal performance impact)
# - 'all_actions': Evaluate after every agent action
#   (WARNING: significantly slower due to API calls on each action)
```

默认只在**两个时刻**评审：AI 宣布完成时、AI 说话时。可选的 `all_actions` 是**每个动作后
都评审**，注释明确警告会显著变慢。

**把成本写进配置项的文档里**，而不是让用户自己踩坑发现变慢了。

## 三、三个简单实现：从最朴素的信号开始

### `PassCritic` —— 永远通过

```python
def evaluate(self, events, git_patch=None) -> CriticResult:
    return CriticResult(score=1.0, message="PassCritic always succeeds")
```

看起来毫无用处，但它有两个真实作用：
1. **基线对照**：跑评测时用它做"不开评审"的对照组
2. **接口占位**：需要一个评审员但不想真评时用它，避免代码里到处 `if critic is not None`

### `EmptyPatchCritic` —— 只看有没有改动

```python
if not git_patch or not git_patch.strip():
    return CriticResult(score=0.0, message="Git patch is empty or missing")
return CriticResult(score=1.0, message="Git patch is non-empty")
```

> **git patch（补丁）**：这次任务改了哪些代码的差异文本。空的意味着**一个字都没改**。

**这是一个极其便宜但极其有效的检查。** AI 有一种典型失败模式：分析了半天、
解释了一大段"我认为应该这样改"，然后宣布完成——但**一行代码都没动**。

只要一行判断就能抓住这种情况，不需要任何模型调用。

> 可迁移的道理：**最有价值的检查往往是最便宜的那个。** 先把"什么都没做就说做完了"
> 这类硬失败挡住，再考虑复杂的质量评估。

### `AgentFinishedCritic` —— 改动 + 正确收尾

两个条件都要满足：

```python
# 1. 补丁非空
if not git_patch or not git_patch.strip():
    return CriticResult(score=0.0, message="Agent did not produce a non-empty git patch. ...")

# 2. 最后一个动作是 FinishAction
if not self._has_finish_action(events):
    return CriticResult(score=0.0, message="Agent did not finish properly. ...")
```

第二个条件的实现有个细节：

```python
def _has_finish_action(self, events) -> bool:
    for event in reversed(events):
        if isinstance(event, ActionEvent):
            if event.action and isinstance(event.action, FinishAction):
                return True
            return False        # ← 注意：找到的第一个动作不是 finish，立刻返回 False
    return False
```

**从后往前找第一个动作事件，只看它是不是 `finish`。**

为什么不是"历史里有没有出现过 finish"？因为那样的话，AI 中途说过一次"完成"、
后来又干了别的活、最后崩了——也会被判成正常完成。**必须是"最后一个动作"。**

## 四、真正的评审员：`APIBasedCritic`

这个走一个专门训练的评审模型 API。它的输出不是一个分数，而是**一组带概率的标签**。

### 标签体系（`taxonomy.py`）

四大类，这份清单本身就是一份"AI 编程失败模式图谱"，值得逐条看：

**① AI 的行为问题**（`agent_behavioral_issues`）

```python
"misunderstood_intention"        # 误解了意图
"did_not_follow_instruction"    # 没按指令做
"insufficient_analysis"         # 分析不足
"insufficient_clarification"    # 该问清楚的没问
"improper_tool_use_or_setup"    # 工具用得不对
"loop_behavior"                 # 陷入循环
"insufficient_testing"          # 测试不足
"insufficient_debugging"        # 调试不足
"incomplete_implementation"     # 实现不完整
"file_management_errors"        # 文件管理错误
"scope_creep"                   # 范围蔓延（改了不该改的）
"risky_actions_or_permission"   # 危险操作/权限问题
```

**② 预测的用户后续行为**（`user_followup_patterns`）—— 这一类是最有意思的

```python
"follow_up_timing"                # 多快会有追问
"clarification_or_restatement"    # 用户会要求澄清/重述
"correction"                      # 用户会来纠正
"direction_change"                # 用户会改方向
"vcs_update_requests"             # 用户会要求改提交/PR
"progress_or_scope_concern"       # 用户会担心进度或范围
"frustration_or_complaint"        # 用户会表达不满
"removal_or_reversion_request"    # 用户会要求撤销
```

**停下来想想这一类在干什么：它不是在评价代码质量，而是在预测"人看到这个结果会做什么反应"。**

`frustration_or_complaint`（用户会不会抱怨）和 `removal_or_reversion_request`
（用户会不会要求撤销）这两个标签，是把**用户满意度**当成了质量的代理指标。

> **这是一个很聪明的转向。** "这段代码好不好"很难客观评价；但"用户下一句会不会是
> 『不对，我不是这个意思』"是一个**有大量真实数据可以训练**的预测任务
> ——因为真实产品里每一次对话的后续都被记录下来了。
>
> 可迁移的道理：**当目标难以直接度量时，找一个和它高度相关、但有大量自然标注数据的
> 代理指标。** 用户的下一句话就是天然的标注。

**③ 基础设施问题**（`infrastructure_issues`）

```python
"infrastructure_external_issue"       # 外部原因（服务挂了之类）
"infrastructure_agent_caused_issue"   # AI 自己搞坏的
```

**区分"环境本来就有问题"和"AI 把环境搞坏了"。** 这个区分对改进模型很关键——
前者不是模型的错，不该算进失败率。

**④ 一般上下文**（`general_context`）：用户目标摘要、整体情绪。

### 概率归一化用了 softmax

```python
def _softmax_normalize(probs: dict[str, float]) -> dict[str, float]:
    """Apply softmax normalization to convert logits to probabilities."""
    exp_values = [math.exp(v) for v in values]
    exp_sum = sum(exp_values)
    normalized = [exp_v / exp_sum for exp_v in exp_values]
```

> **softmax**：把一组任意大小的数字转换成一组和为 1 的概率。
> 模型原始输出（叫 logits）不是概率，要这样转一下才能当概率解释。

### 一个额外的触发条件

`APIBasedCritic` 重写了"是否需要重做"的判断：

```python
def should_refine(self, critic_result) -> bool:
    """Use API critic taxonomy signals in addition to the score threshold."""
    if super().should_refine(critic_result):      # 先看总分
        return True
    if self.iterative_refinement is None:
        return False
    return bool(_get_high_probability_agent_issues(critic_result, self.issue_threshold))
```

`issue_threshold` 默认 **0.75**。含义：

**即使总分过了线，只要有任何一个"AI 行为问题"标签的概率超过 75%，也要重做。**

> **为什么需要这个？** 因为总分是一个加权平均，可能掩盖单项的严重问题。
> 比如"整体看起来 70% 会成功，但有 90% 的概率是完全没写测试"——
> 平均之后看不出来，单项一看就很清楚。
>
> 可迁移的道理：**总分之外要保留单项的一票否决。** 只看加权总分会漏掉"某一项极差"
> 这种情况。

## 五、迭代精修的控制流（`agent/critic_mixin.py`）

> **⚠️ 实跑修正（见 `notes/26-verification-run.md`）**：**迭代精修只在 AI 通过 `finish`
> 工具收尾时才参与。** `_check_iterative_refinement` 唯一的到达路径是
> `_ActionBatch.finalize`，而它开头就是 `if not self.has_finish: return`；
> 而"模型只说话"那条路（`_handle_content_response`）**没有任何 critic / 精修钩子**。
>
> 实跑里 AI 直接用一条消息回答了问题（简单任务上很常见），
> **整个精修回路被绕过**：计数器为空、没有注入任何追加要求。
>
> critic 的 `mode="finish_and_message"` 描述的是**打分**时机，
> **重试**只挂在 `FinishAction` 上——这两件事不是一回事，本节原文把它们混为一谈了。

### 反馈信息怎么写

基类版本（`base.py`）：

```python
return (
    f"The task appears incomplete (iteration {iteration}, "
    f"predicted success likelihood: {score_percent:.1f}%).\n\n"
    "Please review what you've done and verify each requirement is met.\n"
    "List what's working and what needs fixing, then complete the task.\n"
)
```

翻译："任务看起来没完成（第 N 轮，预测成功率 X%）。请回顾你做了什么，逐项验证每个要求
是否满足。列出哪些能用、哪些需要修，然后完成任务。"

**注意它要求 AI "列出哪些能用、哪些需要修"** —— 强制它先做一次显式盘点，而不是直接
一头扎进去继续改。这是让模型自我纠错的常用手法：**先复述现状，再动手。**

`APIBasedCritic` 的版本更具体，把标签都带上：

```python
lines.append(f"Potential agent issues: {_format_feature_list(agent_issues)}")
lines.append(f"Predicted user follow-up needs: {formatted}")
```

于是 AI 收到的是："潜在问题：测试不足 (85%)、实现不完整 (78%)"——
**具体到哪一项、多大概率，而不是一句"做得不好"。**

### 计数器只在真正要继续时才加

```python
# Check if we've exceeded max iterations BEFORE incrementing
if iteration >= config.max_iterations:
    return False, None
...
if not self.critic.should_refine(critic_result):
    logger.info(f"success threshold met with score {critic_result.score:.3f}")
    return False, None

# Refinement is needed and we haven't hit max iterations
# NOW we increment the counter since we're actually continuing
new_iteration = iteration + 1
state.agent_state = {**state.agent_state, ITERATIVE_REFINEMENT_ITERATION_KEY: new_iteration}
```

注释用大写强调了 `BEFORE` 和 `NOW`。**顺序是：先检查上限 → 再看要不要重做 → 确定要重做
了才加计数。**

**为什么这么在意顺序？** 如果一进来就加计数，那么"一次就做对"的情况也会消耗一个配额。
更糟的是，如果评审结果缺失、走了提前返回的路径，计数已经加了但什么都没做——
**配额被白白浪费，而且很难查。**

> 可迁移的道理：**副作用（改计数器、写状态）要放在所有提前返回之后。**
> 这类 bug 不会崩溃，只会让行为悄悄跑偏。

还有一行注释解释了为什么用重新赋值而不是原地改：

```python
# Use reassignment pattern to trigger autosave
state.agent_state = {**state.agent_state, KEY: new_iteration}
```

**改字典要整个替换，而不是 `state.agent_state[KEY] = x`** —— 因为自动保存是靠监听
"这个属性被赋值"触发的，原地改字典不会触发，状态就不会落盘。

> 这是用 Pydantic 这类框架时常见的坑：**容器的内部变化不会被侦测到。**

### 评审失败不影响主流程

```python
except Exception as e:
    logger.error(f"✗ Critic evaluation failed: {e}", exc_info=True)
    return None
```

**评审员自己挂了，返回 `None`，主流程继续。**

对比上一节安全模块的处理：**检查器挂了算高危（fail-closed）**。

**为什么这里是相反的取向？** 因为两者的后果不同：
- 安全检查挂了 → 可能执行危险操作 → **必须倒向保守**
- 质量评审挂了 → 可能少一次优化机会 → **不该因此让整个任务失败**

> 可迁移的道理：**fail-closed 还是 fail-open，取决于"失败的代价"是什么。**
> 没有一刀切的答案。安全用前者，增强性功能用后者。同一个项目里两种都出现是正常的。

## 六、和主循环怎么接上

回想第三节 `_ActionBatch.finalize`：

```python
should_continue, followup = check_iterative_refinement(self.action_events[-1])
if should_continue and followup:
    on_event(MessageEvent(source="user", llm_message=... followup ...))    # 注入反馈
else:
    mark_finished()                                                        # 真的结束
```

**评审的反馈是以一条消息的形式注入回对话的**，`source="user"`。

于是 AI 下一轮看到的就是一条"（用户说）任务似乎没完成，潜在问题是……"。
**整个精修过程不需要任何特殊的控制流——就是往对话里加一条消息，然后循环自然转下去。**

> 这和第三节那四种报错的处理手法**完全一致**：把情况写成一条消息告诉模型。
> **一个项目里反复出现同一个手法，说明它是真的想清楚了的架构选择，而不是巧合。**

## 七、三层"什么叫做完了"的判定

把前面几节串起来看，这个系统对"完成"的判定其实有三道：

```
① AI 自己调用 finish 工具                      （AI 的主张）
        ↓
② Critic 评分 + 标签检查                        （模型的质量判断）
   不过线 → 注入反馈，继续干（最多 N 轮）
        ↓
③ Stop hook（第四节）                           （你的客观标准）
   否决 → 注入反馈，继续干
        ↓
     真正结束
```

**三道关卡的性质完全不同**：第一道是自称，第二道是概率预测，第三道是确定性验证
（比如"测试必须通过"）。

> 可迁移的道理：**"完成"的定义权应该分层，而且最终裁决权要留给能做确定性验证的那一层。**
> 只靠 AI 自称必然虚高；只靠模型评审会被概率误差影响；而一个跑测试的钩子是确定的。

## 八、一个题外但值得注意的工程细节

`impl/api/chat_template.py`（232 行）的文档：

> This module provides a lightweight implementation of chat template rendering that is
> compatible with HuggingFace transformers but **removes the dependency on the full
> transformers library.**

**为了渲染一个聊天模板，他们自己实现了一份，就为了不依赖整个 transformers 库。**

而且用的是 Jinja2 的**沙箱环境**：

```python
from jinja2.sandbox import ImmutableSandboxedEnvironment
```

> **沙箱环境**：限制模板里能执行什么，防止模板本身变成代码执行的入口。

因为模板是**从网上下载的** tokenizer 配置里来的：

```python
CACHE_DIR = Path.home() / ".cache" / "chat_templates"
```

**下载来的东西当成不可信输入处理**——这和第三节"把模型输出当不可信输入"是同一个原则。

## 九、这一节的可迁移结论

1. **让评价指标变成可验证的预测**（"成功概率"），而不是主观标签（"质量 8 分"）。
   前者能形成"预测→验证→校准"的闭环。

2. **最有价值的检查往往最便宜**：`EmptyPatchCritic` 一行判断就挡住了"解释了一大堆
   但一个字没改"这种典型失败。先挡硬失败，再谈质量评估。

3. **判"完成"要看最后一个动作，而不是历史里有没有出现过**。

4. **目标难以直接度量时，找有自然标注的代理指标**：预测"用户会不会抱怨/要求撤销"
   来代理"做得好不好"，因为真实产品里每次对话的后续都被记录了。

5. **总分之外要保留单项一票否决**：加权平均会掩盖"某一项极差"。
   任何行为问题标签超过 75% 就重做。

6. **反馈要具体到项和概率**，并要求模型先显式盘点现状再动手。

7. **副作用要放在所有提前返回之后**：计数器只在真正要继续时才加，
   否则配额会被悄悄浪费。

8. **fail-closed 还是 fail-open 取决于失败的代价**：安全检查挂了算高危；
   质量评审挂了就跳过。同一项目里两种取向并存是正常的。

9. **区分"环境的问题"和"AI 造成的问题"**：否则失败率统计会把不是自己的锅算进来。

10. **"完成"的定义权分层，最终裁决交给能做确定性验证的那一层**：
    AI 自称 → 模型评审 → 跑测试的钩子。

11. **同一个手法反复出现说明是架构选择**：报错、卡死提醒、评审反馈，
    全部都是"写成一条消息注入对话"。不需要特殊控制流。

## 下一站

`context/agent_context.py`（619 行）+ `skills/`（2952 行）—— 系统提示怎么组装、
技能（skills）怎么按需加载、路径规则怎么绑到观察上。
这是"上下文工程"里除压缩之外的另一半。

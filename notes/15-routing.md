# 精读 14：模型路由（router/ 244 行 + 元配置路由 ~1000 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/`
> 文件：`llm/router/{base,impl/multimodal,impl/random}.py`、
> `llm/meta_profile_store.py`、`llm/llm_profile_store.py`、
> `tool/builtins/{switch_llm,classify_and_switch_llm}.py`
> 前置：`notes/10-llm-seam.md`（模型接缝、三层降级）、`notes/14-hooks.md`（用 LLM 做判断时的隔离）

## 这一节在讲什么

第九节末尾提到过"路由"是三层降级的第三层，但没展开。这一节把它拆开。

**核心问题**：任务难度差别巨大——改一个拼写错误和重构一个模块都是"任务"。
**用最强的模型干所有活，账单会很难看；用便宜模型干所有活，难任务做不完。**

这个 SDK 提供了**两套完全不同的路由机制**，值得对照看：

| 机制 | 决定时机 | 决定者 | 粒度 |
|---|---|---|---|
| `RouterLLM` | 每次调用模型前 | 代码（确定性规则） | 单次请求 |
| 元配置 + 分类工具 | AI 自己决定调用时 | 一个分类模型 | 整个会话（后续都用新模型） |

## 一、RouterLLM：一组模型伪装成一个模型

```python
class RouterLLM(LLM):
    """Base class for multiple LLM acting as a unified LLM.
    ...
    - Works with multiple LLMs configured via llms_for_routing
    - Delegates all other operations/properties to the selected LLM
    - Provides routing interface through select_llm() method
    """
    router_name: str = "base_router"
    llms_for_routing: dict[str, LLM] = Field(default_factory=dict)
    active_llm: LLM | None = None
```

**它继承 `LLM`。** 第九节提过这一点，这里看它是怎么做到的。

核心只有一个抽象方法：

```python
@abstractmethod
def select_llm(self, messages: list[Message]) -> str:
    """Returns: The key/name of the LLM to use from llms_for_routing dictionary."""
```

**输入是消息列表，输出是一个名字。** 就这么简单。

然后 `completion` 拦截调用、选模型、转发：

```python
def completion(self, messages, tools=None, ...) -> LLMResponse:
    selected_model = self.select_llm(messages)
    self.active_llm = self.llms_for_routing[selected_model]
    logger.info(f"RouterLLM routing to {selected_model}...")
    return self.active_llm.completion(messages=messages, tools=tools, ...)
```

### 两个为了"伪装成一个模型"而做的技巧

**① 给自己塞一个假的模型名**

```python
@model_validator(mode="before")
def set_placeholder_model(cls, data):
    """Guarantee `model` exists before LLM base validation runs."""
    # In router, we don't need a model name to be specified
    if "model" not in d or not d["model"]:
        d["model"] = d.get("router_name", "router")
    return d
```

`LLM` 基类要求必须有 `model` 字段，但**路由器本身没有"一个模型"**。
于是在基类校验之前，先用路由器的名字填进去。

注意 `mode="before"` —— **必须在基类校验运行之前塞进去**，否则会因为缺字段而失败。

**② 其他属性一律转发**

```python
def __getattr__(self, name):
    """Delegate other attributes/methods to the active LLM."""
    fallback_llm = next(iter(self.llms_for_routing.values()))
    logger.info(f"RouterLLM: No active LLM, using first LLM for attribute '{name}'")
    return getattr(fallback_llm, name)
```

> `__getattr__` 是 Python 的兜底钩子：**只有在正常属性查找失败时才会被调用。**

于是外面问路由器"你的上下文窗口多大"这类问题时，**答案来自第一个模型**。

**这是个务实但有点危险的妥协**：如果几个模型的窗口大小不同，
这个回答就是错的（回想第五节压缩器要靠 `effective_max_input_tokens` 判断阈值）。
`MultimodalRouter` 的文档承认了这个前提：

> The primary model is expected to support multimodal content, while the secondary
> model is typically a text-only model with a **lower context window**.

**而它的路由逻辑正好用这个差异来做决定**（见下），所以在这个特定实现里是自洽的。
但换一个路由策略就要小心。

> **可迁移的道理**：**用 `__getattr__` 做透明代理很方便，但代理的是"一组行为不同的对象"时，
> 任何被代理的属性查询都可能给出错误答案。** 要么明确列出可代理的属性，
> 要么在文档里写清前提。这里选了后者。

### 一个启动即检查的约束

```python
@field_validator("llms_for_routing")
def validate_llms_not_empty(cls, v):
    if not v:
        raise ValueError("llms_for_routing cannot be empty - at least one LLM "
                         "must be provided")
```

空的路由器是无意义的配置。**又一次"配置错误在创建时就报错"。**

## 二、`MultimodalRouter`：确定性规则，两个条件

这是唯一一个有实际业务逻辑的路由器：

```python
def select_llm(self, messages: list[Message]) -> str:
    route_to_primary = False

    # 条件 1：消息里有图片
    for message in messages:
        if message.contains_image:
            logger.info("Multimodal content detected in messages. "
                        "Routing to the primary model.")
            route_to_primary = True

    # 条件 2：超出次要模型的上下文窗口
    secondary_llm = self.llms_for_routing.get(self.SECONDARY_MODEL_KEY)
    if secondary_llm and (
        secondary_llm.effective_max_input_tokens
        and secondary_llm.get_token_count(messages)
        > secondary_llm.effective_max_input_tokens
    ):
        logger.warning(f"Messages having ... tokens, exceeded secondary model's max "
                       f"input tokens (...). Routing to the primary model.")
        route_to_primary = True

    return self.PRIMARY_MODEL_KEY if route_to_primary else self.SECONDARY_MODEL_KEY
```

**默认走便宜的次要模型，只有两种情况升级到主模型：有图片、或者内容太长装不下。**

三个观察：

**① 条件是"能力"而非"难度"。** 它判断的不是"这个任务难不难"（那需要模型判断），
而是"次要模型**能不能处理**"（图片、长度都是硬性能力）。
**确定性规则只适合判断硬性约束。**

**② 两个条件都判断完才决定**，没有提前 `return`。看代码：第一个循环设了标志位
但继续往下走。**于是日志里会同时出现两条原因**——排查时能看到"既有图片也超长"。

**③ 键名是类变量，而且启动时检查存在**：

```python
PRIMARY_MODEL_KEY: ClassVar[str] = "primary"
SECONDARY_MODEL_KEY: ClassVar[str] = "secondary"

@model_validator(mode="after")
def _validate_llms_for_routing(self) -> "MultimodalRouter":
    if self.PRIMARY_MODEL_KEY not in self.llms_for_routing:
        raise ValueError(f"Primary LLM key '{self.PRIMARY_MODEL_KEY}' not found "
                         "in llms_for_routing.")
    if self.SECONDARY_MODEL_KEY not in self.llms_for_routing:
        raise ValueError(...)
```

**配少了一个模型，创建时就报错**，不会等到运行时 `KeyError`。

### `RandomRouter`：22 行的对照组

```python
class RandomRouter(RouterLLM):
    """A simple implementation of RouterLLM that randomly selects an LLM from
    llms_for_routing for each completion request."""
    def select_llm(self, messages):
        selected_llm_name = random.choice(list(self.llms_for_routing.keys()))
```

**看起来没用，但它是做 A/B 测试的基础设施**——随机把请求分给几个模型，
然后比较结果。（和第七节那个 `PassCritic` 一样，是评测用的对照组。）

## 三、第二套机制：元配置（meta-profile）

这是完全不同的思路：**让一个模型来决定用哪个模型**。

### 先理解两层配置

**第一层：LLM 配置（profile）** —— 一个保存好的模型配置（模型名、密钥、参数）。
存在 `~/.openhands/profiles/` 下，有名字。

**第二层：元配置（meta-profile）** —— 一份"路由规则表"：

```python
class MetaProfileClass(BaseModel):
    """A single task category and the LLM profile that should handle it."""
    description: str = Field(
        description="Natural-language description of the kind of task this class "
        "covers (e.g. 'task is UI oriented or requires looking at images').")
    model: str = Field(
        description="Name of the saved LLM profile to switch to for tasks "
        "matching this class.")

class MetaProfile(BaseModel):
    """A declarative model-routing configuration."""
    classifier_model: str        # 用哪个模型来做分类
    classes: list[MetaProfileClass]   # 类别 → 目标配置
    prompt_template: str | None       # 或者：直接路由的提示模板
    model_table: str | None
```

**一份元配置说的是：用 X 模型来判断任务属于哪一类，每一类分别用哪个模型干。**

类别的描述是**自然语言**（"任务偏 UI 或需要看图"），因为判断者是一个模型。

### 关键不变量：所有引用都是"配置名"，不是模型名

模块文档第一句就点明：

> Key invariant: every model reference in a meta-profile (``classifier_model``, each
> class's ``model``, and direct-routing prompt outputs) is the *name of a saved LLM
> profile* ..., not a raw model string — credentials/provider settings resolve through
> that store.

**为什么必须这样？** 因为一个模型要能用，需要模型名 + 密钥 + 服务地址 + 参数。
如果元配置里直接写模型名，那这些东西没地方放。**通过"配置名"间接引用，
凭证和参数都由配置存储统一管理。**

> **可迁移的道理**：**配置引用配置，而不是引用裸值。** 一旦某个概念需要携带凭证或多个参数，
> 就必须给它一个名字并单独存储，让别处按名字引用。

### 两种路由模式，而且互斥

```python
@model_validator(mode="after")
def validate_routing_mode(self) -> MetaProfile:
    """Require either structured classes or a direct prompt, not both."""
    if self.prompt_template is None:
        return self
    if re.search(r"{{\s*instance_text\s*}}", self.prompt_template) is None:
        raise ValueError("Direct-routing meta-profiles must include "
                         "{{ instance_text }} in prompt_template.")
    if self.classes:
        raise ValueError("Direct-routing meta-profiles cannot also define classes.")
```

- **结构化模式**：给几个类别，分类模型回答"第几类"
- **直接模式**：给一个自由提示模板，分类模型直接回答"用哪个配置"

**两种不能同时配。** 而且直接模式**必须包含 `{{ instance_text }}` 占位符**
——否则任务内容根本传不进去，分类器在凭空猜。**这个检查很实在：
它检查的是"这个模板有没有用到必需的输入"。**

> 又一次看到同一个模式（第十四节钩子的类型字段检查）：
> **不只检查字段存在，还检查配置的组合是否自洽。**

## 四、分类工具的六个自我约束

`classify_and_switch_llm.py`（503 行）是这套机制的执行者。它的模块文档已经把
最重要的取向写出来了：

> When the classifier produces no usable answer, the tool **fails loudly** (returns an
> error observation) so the miss is visible and retryable, instead of silently routing
> to a default model.

### ① 不做静默降级

两处都是这个处理：

```python
index = parse_class_index(reply, len(meta.classes))
if index == 0:
    return ClassifyAndSwitchLLMObservation.from_text(
        text=("Classifier did not pick a valid category "
              f"(reply: {reply!r}); refusing to silently route to a default. "
              "Retry the tool or fix the meta-profile classes."),
        is_error=True)
```

```python
if target_profile is None:
    return ...from_text(
        text=("Classifier reply did not resolve to a saved LLM profile "
              f"(reply: {reply!r}); refusing to silently route to a default. "
              "Retry the tool or extend the model table."),
        is_error=True)
```

**分类器给不出有效答案 → 返回错误，而不是"那就用默认模型吧"。**

注意两点：**错误信息里带上了分类器的原始回复**（`reply!r`），
以及**两条不同的修复建议**（改类别定义 / 扩充模型表）。

> 为什么"静默用默认"很糟？因为**它会让路由功能看起来在工作，实际上从未生效**。
> 你以为难任务被路由到了强模型，其实分类器一直在失败、一直用默认模型。
> 账单正常、日志正常、效果不对，而且极难发现。
>
> 这和第十四节钩子那个"我没能判断"必须可检测是**同一个原则的第五次出现**。
> 但注意这里更激进：钩子选择了"放行 + 标记不成功"，这里选择了**直接报错**。
> 差别在于后果：钩子放行只是少一道检查；路由静默降级会让整个功能**长期无声失效**。

### ② 分类器的花费必须记账

这段注释是这个文件里最有信息量的：

```python
# 1) Load the classifier LLM (a saved profile), respecting at-rest cipher, and
#    register it in the conversation's LLM registry under a stable usage id.
#    Registration is what makes the classifier completion's tokens/cost flow into
#    ``conversation.conversation_stats`` and count against ``max_budget_per_run`` —
#    calling an unregistered LLM would spend off the books (mirrors the
#    ``ask-agent-llm`` pattern). Caching by usage id means repeated routing calls
#    reuse one metrics bucket.
usage_id = f"classifier:{meta.classifier_model}"
try:
    classifier_llm = conversation.llm_registry.get(usage_id)
except KeyError:
    ...
    conversation.llm_registry.add(classifier_llm)
    conversation._bind_conversation_context(classifier_llm)
```

**`calling an unregistered LLM would spend off the books`（调用未注册的模型会
产生账外开销）** —— 这句话点出了一个很容易犯的错误。

回想第四节那个预算控制、第九节那个"派生的模型实例要挂在同一份统计上"、
第十二节那个"成本上报到工作区"。**这是那条链路的第四个环节**：
任何临时创建的模型都必须注册进登记表，否则它花的钱不受预算约束。

而且**按 `usage_id` 缓存**（`classifier:<配置名>`）：重复路由复用同一个统计桶，
既省掉重复创建，又让"分类一共花了多少"可以单独统计。

> **可迁移的道理**：**预算/配额系统的完整性，取决于"所有花钱的地方都必须先登记"
> 这条纪律有没有被每一个新功能遵守。** 一个绕过登记的调用点就能让预算失效，
> 而且不会报错。所以要在注释里写清"为什么必须注册"。

### ③ 元配置要延迟解析

```python
# Resolve the meta-profile lazily (at invocation), not at construction, so a
# missing/renamed file under ~/.openhands/meta-profiles produces a tool error
# instead of breaking conversation startup.
```

**配置文件不见了，应该是"这个工具报错"，而不是"整个会话起不来"。**

> **可迁移的道理**：**可选功能的配置解析要推迟到使用时。** 在启动时解析，
> 一个可选功能的配置问题就会变成整个系统的启动失败。

### ④ 名字和内联数据谁说话算数，写得很清楚

```python
def _resolve_meta_profile(self) -> MetaProfile:
    """When an ``active_meta_profile`` name is set, the store is authoritative
    (loaded by name) so the inline ``meta_profile`` blob and the store cannot
    diverge after a name change (``PATCH /api/settings``) or a file overwrite
    (``POST /api/meta-profiles/{name}``). The inline blob is only consulted when
    the store cannot resolve the name — e.g. cloud runtimes whose ephemeral
    filesystem has no meta-profile store."""
```

**有名字时以存储为准**（防止改名/覆盖后两边不一致），
**存储读不到才用内联的那份**（云端临时文件系统没有配置目录）。

注意最后那个降级路径还有一句：

```python
# ``list()`` is alphabetically sorted, so this is the alphabetically-first
# meta-profile, not the most recently saved.
name = available[0]
logger.info("No active meta-profile set; falling back to first available: %r", name)
```

**明确说明"这是按字母序第一个，不是最近保存的那个"。**

> 这类注释的价值：**读代码的人会自然地假设"第一个"是"最新的"。**
> 一句澄清省掉一个误解，也防止后人"修"出一个 bug。

### ⑤ 只给分类器看消息，不给看工具调用

```python
def _recent_messages_text(conversation, limit=_RECENT_MESSAGE_LIMIT) -> str:
    """Only ``MessageEvent`` is included on purpose: user/assistant messages are the
    task signal the classifier needs. Action/observation events (tool calls, outputs)
    are deliberately excluded — they are noisy, can be large, and don't describe the
    task any better than the surrounding messages."""
```

`_RECENT_MESSAGE_LIMIT = 6` —— **只看最后 6 条消息。**

三个理由都写了：**噪音大、可能很长、而且对描述任务并不比周围的消息更有用。**

> **这是很克制的上下文设计。** 一个分类任务不需要完整历史，只需要"最近在聊什么"。
> 给太多反而稀释信号，还要为 token 付钱。
>
> 可迁移的道理：**辅助性的模型调用要主动裁剪输入，按"这个判断真正需要什么"来选，
> 而不是把手头有的全塞进去。**

### ⑥ 解析分类器输出：宽容，但边界明确

结构化模式的解析简单粗暴：

```python
def parse_class_index(text: str, num_classes: int) -> int:
    """Returns 0 (no usable answer) when no in-range integer is found."""
    match = re.search(r"-?\d+", text)
    if match is None:
        return 0
    index = int(match.group())
    if 1 <= index <= num_classes:
        return index
    return 0
```

**找第一个整数，检查是否在范围内，越界一律返回 0（=无效）。**
注意正则是 `-?\d+`（允许负号）——**把负数也抓出来然后判越界**，
而不是让 `-1` 被当成合法索引去反向取列表（Python 里 `classes[-1]` 是合法的，
那会静默选中最后一类）。

**这是一个很具体的防御**：Python 的负索引语义在这里是个陷阱。

直接模式的解析更宽容：

```python
def parse_direct_model(text: str, available_models: Sequence[str]) -> str | None:
    """The expected contract is JSON with a ``model`` field, but the parser also scans
    the raw response for an allowed profile name to tolerate fenced JSON or small
    formatting mistakes. Matching is case-insensitive, but the returned value is the
    canonical saved profile name."""
    raw = re.sub(r"^```[a-zA-Z]*\n?", "", raw)    # 去掉代码围栏
    raw = re.sub(r"\n?```$", "", raw).strip()
    profile_by_casefold = {model.casefold(): model for model in available_models}
    ...
    # 先试 JSON
    if candidate:
        return profile_by_casefold.get(candidate.casefold())
    # JSON 失败就扫描原文里有没有出现某个允许的配置名
    for model in sorted(available_models, key=len, reverse=True):
        if model.casefold() in scan:
            return model
    return None
```

三个细节：

**① 三级降级**：JSON 解析 → 扫描原文 → 返回 None（报错）。

**② 扫描时按名字长度**从长到短排序。防的是：配置名有 `gpt` 和 `gpt-mini` 两个，
回复里写的是 `gpt-mini`，按短的先匹配会错误命中 `gpt`。**长的优先是正确的。**

**③ 大小写不敏感匹配，但返回规范名。** `profile_by_casefold` 这个映射就是为了
"宽容地认，精确地返回"。

> **这和第十三节文件编辑器那条原则完全一致**：
> **宽容地接受输入，精确地输出。** 匹配时可以模糊，产生的值必须是规范的。

### 切换本身也分两路

```python
inline_target = self._meta_profile_llms.get(target_profile)
if inline_target is None:
    conversation.switch_profile(target_profile)      # 从存储加载
else:
    conversation.switch_llm(inline_target.model_copy(
        update={"usage_id": f"profile:{target_profile}"}))    # 用内联的
```

**内联的模型也要改 `usage_id`**（`profile:<名字>`），保持统计口径一致。

而且返回给 AI 的消息明确说了生效时机：

> "Future agent steps will use this profile."

**切换在下一步生效**，不是当前这一步。工具描述里也再说了一遍
（`The switch takes effect on the next LLM call.`）——**让 AI 知道时序，
避免它以为当前这一步已经换了。**

## 五、两套机制的对比与取舍

```
RouterLLM（代码决定，每次调用）
  ✓ 零额外成本（没有分类调用）
  ✓ 确定性、可测试
  ✓ 适合硬性能力约束（有图片？超长？）
  ✗ 判断不了"任务难不难"
  ✗ 透明代理（__getattr__）在模型能力不一致时会给出错误属性

元配置 + 分类工具（模型决定，会话级）
  ✓ 能理解意图和难度（自然语言类别描述）
  ✓ 决定是持久的（后续步骤都用新模型，一次分类摊销到整个任务）
  ✓ 用户可声明式配置，不改代码
  ✗ 每次分类是一次额外的模型调用（所以只在任务开始或任务变化时调）
  ✗ 依赖分类器可靠——所以失败时宁可报错也不静默降级
```

**注意第二套机制的调用时机是 AI 自己决定的**（它是一个工具）。工具描述里建议：

> Use this near the start of a task (or when the task changes) to route the work
> to the best model.

**一次分类的成本被摊销到整个任务上。** 这和第五节压缩器"砍到一半而不是刚好不超"
是同一种成本思维：**用一次昂贵操作换很多步的收益。**

## 六、这一节的可迁移结论

1. **让"容器"继承"元素"的接口，一组东西就能伪装成一个东西**：
   `RouterLLM(LLM)` 让上层代码零感知。（和第六节安全融合器、第九节提到的是同一手法。）

2. **透明代理（`__getattr__`）在被代理对象行为不一致时会给出错误答案**：
   要么明确列出可代理属性，要么在文档里写清前提（这里选了后者，
   并让路由逻辑刚好依赖那个差异）。

3. **确定性规则只适合判断硬性能力约束**（有图片、超长），
   判断"任务难不难"需要模型。

4. **两个条件都判断完再决定，不要提前返回**：这样日志里能看到全部原因。

5. **配置引用配置，而不是引用裸值**：一旦某个概念要携带凭证或多个参数，
   就给它一个名字单独存储，让别处按名字引用。

6. **不只检查字段存在，还检查配置组合是否自洽**：两种路由模式互斥、
   直接模式必须用到必需的占位符。

7. **绝不静默降级到默认值**：分类失败就报错，并附上分类器原始回复和两条修复建议。
   **静默降级会让一个功能长期无声失效——账单正常、日志正常、效果不对，极难发现。**

8. **预算系统的完整性取决于"所有花钱的地方都先登记"这条纪律**：
   未注册的模型调用会产生账外开销，而且不会报错。新功能必须遵守，
   所以要在注释里写清为什么。

9. **可选功能的配置解析要推迟到使用时**：启动时解析会让一个可选功能的
   配置问题变成整个系统起不来。

10. **多来源配置要明确谁权威，并写清降级路径**：有名字以存储为准（防止改名后不一致），
    读不到才用内联的。

11. **辅助性的模型调用要主动裁剪输入**：只给分类器最近 6 条消息，
    明确排除工具调用（噪音大、可能很长、对判断没帮助）。

12. **注意语言自身的陷阱**：解析索引时用 `-?\d+` 把负数也抓出来判越界，
    否则 `-1` 会被 Python 的负索引语义静默接受。

13. **名字匹配要从长到短**：否则 `gpt` 会错误命中 `gpt-mini` 的回复。

14. **宽容地接受输入，精确地输出**：大小写不敏感匹配，但返回规范名。
    （和第十三节文件编辑器同一原则。）

15. **告诉模型状态变更的生效时机**："切换在下一次调用生效"——
    避免它以为当前这一步已经换了。

16. **看起来没用的实现可能是评测基础设施**：`RandomRouter` 22 行，
    用于 A/B 比较。（和第七节 `PassCritic` 同理。）

## 下一站

`skills/` 的市场加载机制 —— SKILL.md 怎么解析、公共技能从哪拉、
用户/项目/公共三级技能怎么合并去重。

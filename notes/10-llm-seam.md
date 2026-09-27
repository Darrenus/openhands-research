# 精读 09：模型接缝（llm/，12753 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/llm/`
> 文件：`llm.py`（3306 行）、`mixins/fn_call_converter.py`（963 行）、`mixins/non_native_fc.py`、
> `utils/{model_features,retry_mixin,metrics,telemetry}.py`、`fallback_strategy.py`、`router/`
> 前置：`notes/09-context-engineering.md`（提示缓存）、`notes/05-run-loop.md`（预算控制）

## 这一节在讲什么

> **接缝（seam）**：软件设计里指"可以替换实现的那个位置"。模型接缝 = 换模型不用改其他代码的那一层。

这一层要解决的问题：**每家模型的接口、能力、怪癖都不一样，但上面的 agent 代码只想写一遍。**

12753 行——比 agent 循环本身大得多。这个比例本身就是一个结论：
**真实世界里，适配各家模型的工作量远大于"写一个 agent"。**

## 一、能力矩阵：把"这个模型能干什么"变成数据

```python
@dataclass(frozen=True)
class ModelFeatures:
    supports_reasoning_effort: bool          # 支持调节"思考力度"
    thinking_mode: Literal["adaptive", "manual", "none", "unknown"]
    supports_sampling_params: bool | None    # 支持 temperature 之类
    supports_extended_thinking: bool         # 支持扩展思考
    supports_prompt_cache: bool              # 支持提示缓存
    supports_stop_words: bool                # 支持停止词
    supports_responses_api: bool             # 支持 Responses API
    force_string_serializer: bool            # 必须用字符串序列化
    send_reasoning_content: bool
    supports_prompt_cache_retention: bool
    requires_inline_image_data: bool         # 图片必须内联 base64，不接受 URL
    supports_vision: bool                    # 能看图
```

**12 个维度。每一个都对应过至少一次"某个模型和别人不一样"的真实踩坑。**

举两个能读出故事的：

```python
# True when the model's API rejects http(s) image URLs and only accepts
# base64 ``data:`` URLs. See REQUIRES_INLINE_IMAGE_DATA_MODELS.
requires_inline_image_data: bool
```

有些模型不接受图片链接，只接受把图片编码成 base64 塞在请求里。**这不是能力差异，
是接口习惯差异**——但如果不处理，用户传个截图就直接报错。

```python
force_string_serializer: bool
```

有些模型的接口不接受结构化的内容块（文字块+图片块的列表），只接受一整个字符串。

> **可迁移的道理**：**把"各家的差异"收敛成一张显式的能力表**，而不是散落在
> 代码各处的 `if "claude" in model`。前者能测试、能一眼看全、加新模型时有清单可对照。

### 型号匹配用的是"有序规则，后面的赢"

```python
def apply_ordered_model_rules(model: str | None, rules: list[str]) -> bool:
    """Rules semantics:
    - Each entry is a substring token. '!' prefix marks an exclude rule.
    - Evaluated in order; the last matching rule wins.
    - If no rule matches, returns False.
    """
```

比如 `["claude", "!claude-3-haiku"]` 表示"所有 claude 都支持，除了 claude-3-haiku"。

**为什么需要"排除"语法？** 因为模型命名是厂商定的，同一家族里总有例外。
纯白名单会写不完，纯前缀匹配会误伤。**"先按家族收，再按型号排除"是最贴合现实的表达方式。**

### 名字要先归一化

```python
def _normalize_model_for_litellm(model: str | None) -> str | None:
    """Remove SDK/proxy routing prefixes before querying LiteLLM metadata."""
    for provider_prefix in (LITELLM_PROXY_PREFIX, OPENHANDS_PROVIDER_PREFIX):
        ...
    # Strip deployment prefixes (e.g., "prod/", "dev/", "staging/", "test/")
    for prefix in DEPLOYMENT_PREFIXES:
        ...
```

实际使用时模型名可能长这样：`litellm_proxy/prod/claude-opus-5`。前面那些是**路由用的前缀**，
查能力表时必须先剥掉，否则查不到。

> 这是所有"代理层"都会遇到的问题：**标识符在传递过程中被加了装饰，处理前要先还原。**

## 二、最硬核的部分：让不支持函数调用的模型也能用工具

### 先说什么是函数调用

> **函数调用（function calling / tool calling）**：模型接口的一个功能。你告诉模型
> "你有这些工具，参数长这样"，模型会返回一个**结构化的** JSON 说"我要调用 X，参数是 Y"。

问题：**不是所有模型都支持。** 很多开源模型、老模型只会输出纯文本。

### 解法：用提示词模拟一套协议

`fn_call_converter.py` 的文档开头：

> This will inject prompts so that models that doesn't support function calling can still
> be used with function calling agents.

**核心是定义一套 XML 格式，写进系统提示教模型用：**

```
You have access to the following functions:

{description}

If you choose to call a function ONLY reply in the following format with NO suffix:

<function=example_function_name>
<parameter=example_parameter_1>value_1</parameter>
<parameter=example_parameter_2>
This is the value for the second parameter
that can span
multiple lines
</parameter>
</function>
```

然后双向转换：

```python
convert_fncall_messages_to_non_fncall_messages(...)   # 发出去：结构化 → XML 文本
convert_non_fncall_messages_to_fncall_messages(...)   # 收回来：XML 文本 → 结构化
```

**上层的 agent 代码完全不知道这件事发生了。** 它永远看到结构化的工具调用。

### 为什么选 XML 而不是 JSON

看那个多行参数的例子就明白了：

```
<parameter=example_parameter_2>
This is the value for the second parameter
that can span
multiple lines
</parameter>
```

**JSON 里换行必须转义成 `\n`，引号必须转义。** 而代码内容里全是引号和换行——
让模型手工转义一大段代码，出错率极高。**XML 的标签边界不需要转义内容。**

> 可迁移的道理：**让模型产出的格式，要按"模型容易写对"来选，而不是按"程序容易解析"来选。**
> 解析的复杂度你可以承担，模型的出错率你承担不了。

### 提示词里的四条约束

```
- Function calls MUST follow the specified format, start with <function= and end with </function>
- Required parameters MUST be specified
- Only call one function at a time
- You may provide optional reasoning for your function call in natural language BEFORE
  the function call, but NOT after.
- If there is no function call available, answer the question like normal ... and do not
  tell the user about function calls
```

第三条：**一次只能调一个。** 模拟模式下不支持并行调用——这是能力上的妥协，写清楚了。

第四条特别值得注意：**推理文字只能在函数调用之前，不能在之后。** 为什么？
因为解析器靠 `</function>` 结尾来定位，后面还有文字会干扰。**协议约束是从解析器的
实现约束倒推出来的。**

第五条：没有可用函数时就正常回答，**而且不要跟用户提函数调用这件事**——
避免模型说出"我本来想调用工具但……"这种暴露内部机制的话。

### 停止词：一个省钱的小技巧

```python
STOP_WORDS = ["</function"]
```

> **停止词（stop word）**：告诉模型"输出到这个字符串就停"。

模型写完 `</function` 就立刻停止生成。**省掉后面所有 token 的费用和时间。**

但这个优化要看模型支持不支持：

```python
if self._model_features().supports_stop_words and not self.disable_stop_word:
    kwargs = dict(kwargs)   # 加上 stop 参数
```

**又一次能力表的用处。**

### 示例是按需加的

```python
# Skip in-context learning examples for models that understand the format
# or have limited context windows
add_iclex = not any(s in self.model for s in ("openhands-lm", "devstral", "nemotron"))
```

> **上下文学习示例（in-context learning example）**：在提示里给几个"输入→输出"的范例，
> 让模型照着学格式。

对**已经被训练过这个格式的模型**（openhands-lm 就是他们自己的模型）和
**上下文窗口小的模型**，跳过示例——前者不需要，后者付不起。

## 三、重试策略：一个很精妙的细节

```python
retry_min_wait: int = 8
retry_max_wait: int = 64
retry_multiplier: float = 2.0
wait=wait_exponential(...)
```

> **指数退避（exponential backoff）**：失败后等待时间逐次翻倍（8 秒、16、32、64）。
> 目的是给对方服务恢复的时间，而不是一直猛敲。

这部分是标准做法。**真正有意思的是这一段：**

```python
# Only adjust temperature for LLMNoResponseError
if isinstance(exc, LLMNoResponseError):
    current_temp = kwargs.get("temperature", getattr(self, "temperature", None))
    if current_temp == 0:
        kwargs["temperature"] = 1.0
        logger.warning("LLMNoResponseError with temperature=0, "
                       "setting temperature to 1.0 for next attempt.")
    else:
        logger.warning(f"LLMNoResponseError with temperature={current_temp}, "
                       "keeping original temperature")
```

**场景**：模型返回了空响应。

> **temperature（温度）**：控制随机性。0 = 每次尽量给同样的答案；越高越随机。

**关键洞察**：温度是 0 的时候，重试是**毫无意义**的——同样的输入必然得到同样的空响应。

所以重试前**把温度改成 1.0**，让它至少走一条不同的路。

而如果温度本来就不是 0，就**保持原值**——因为随机性已经存在了，重试本身就会得到不同结果，
没必要改变用户配置的参数。

> **这是"重试"这件事里最容易被忽略的一点**：
> **重试只在结果可能不同的时候才有意义。** 确定性失败必须先改变某个条件，
> 否则重试 5 次只是把同一个失败做 5 遍、等了 120 秒。
>
> 对比第五节压缩器那条"内容策略拦截是确定性的，所以要让模型换说法"——**同一个思路，
> 在两个模块各自出现了一次。**

## 四、三层降级：重试 → 换模型 → 路由

### 第二层：换一个模型试试（`fallback_strategy.py`）

```python
# Exceptions that trigger fallback to alternate LLMs (after retries exhausted).
_LLM_FALLBACK_EXCEPTIONS = (
    APIConnectionError,      # 连不上
    RateLimitError,          # 被限流
    ServiceUnavailableError, # 服务不可用
    LiteLLMTimeout,          # 超时
    InternalServerError,     # 对方 500
    LLMNoResponseError,      # 空响应
)
```

注释里 `after retries exhausted`（重试用尽之后）点明了层级关系：

```
① 同一个模型重试（指数退避，最多 5 次）
      ↓ 还是不行
② 换到备用模型（fallback）
      ↓
③ 路由（RouterLLM）—— 不是降级，是主动按需选模型
```

**注意这个清单的共同点：全是"对方的问题"，不是"我的请求有问题"。**

参数写错了、内容被过滤了**不在这个清单里**——换个模型也一样会失败，换只是浪费钱。
**只对"可能换个地方就好了"的错误做降级。**

### 第三层：路由（`router/`）

```python
class RouterLLM(LLM):
    """Base class for multiple LLM acting as a unified LLM."""
```

**它继承自 `LLM`。** 意味着"一组模型"在类型上就是"一个模型"，上层代码完全不需要知道
自己拿到的是单个还是一组。

> 这是**组合模式**的教科书用法：让"容器"和"元素"实现同一个接口。
> 第六节那个 `EnsembleSecurityAnalyzer` 也是同一个手法——**一个项目里反复出现的模式。**

配套的还有内置工具 `classify_and_switch_llm`（第三节提到过）：
**先判断任务难度，再决定用哪个模型。** 简单任务用便宜的，难任务用贵的。

## 五、用量统计：两个容易算错的地方

```python
class TokenUsage(BaseModel):
    prompt_tokens: int
    completion_tokens: int
    cache_read_tokens: int      # 从缓存读的
    cache_write_tokens: int     # 写入缓存的
    context_window: int
```

### 合并时窗口取最大值而不是相加

```python
def __add__(self, other):
    return TokenUsage(
        cache_read_tokens=self.cache_read_tokens + other.cache_read_tokens,
        ...
        context_window=max(self.context_window, other.context_window),   # ← max 不是 +
    )
```

**token 数是累加的，但"上下文窗口大小"是一个容量指标，加起来毫无意义。**
取最大值表示"这次运行里用到过的最大窗口"。

> 小细节，但这类"把容量当成流量累加"的 bug 在监控代码里极其常见。

### 缓存命中率的分母要看情况

```python
"""Fraction of accumulated input tokens served from cache (0.0-1.0).

litellm/OpenAI count cached reads inside ``prompt_tokens`` (the denominator);
ACP reports them separately, so when ``cache_read_tokens`` exceeds
``prompt_tokens`` the denominator is their sum."""

prompt, cache_read = usage.prompt_tokens, usage.cache_read_tokens
denom = prompt + cache_read if cache_read > prompt else prompt
```

**不同来源对"缓存读取算不算在输入 token 里"的口径不一样：**

- OpenAI/litellm：缓存读取**已经包含在** `prompt_tokens` 里 → 分母就是 `prompt_tokens`
- ACP：**分开报** → 分母得是两者之和

判断方法很朴素：**如果缓存读取数超过了总输入数，那显然是分开报的。**

> 可迁移的道理：**聚合多个数据源的指标时，必须搞清每个源的口径。**
> 而且这里给出的处理方式很实用——**用数据自身的矛盾来推断口径**，
> 而不是要求调用方传一个"我是哪种口径"的参数。

## 六、一个反映团队纪律的文件

`llm/utils/AGENTS.md`（给 AI 看的仓库规范）里有一段关于"支持哪些模型"的规定：

```
## Verified model lists (verified_models.py)

These lists are a curated set of models that work well, not a catalog of everything
a provider offers. Keep them short.

- For each model line, keep only the **two latest versions**
- When you add a new version, remove the oldest version in the same line
- Do not add a model just because LiteLLM or OpenRouter knows about it.
  The unverified catalog covers that.
```

**"这是一份精选清单，不是供应商目录。保持简短。"**

而且规定加新版本时**必须删掉同一系列最旧的那个**，还有测试在检查一致性
（`tests/sdk/llm/test_model_list.py`）。

> 这条规范对抗的是一种真实的熵增：**"支持的模型列表"这类清单，如果没人管，
> 会无限膨胀成一坨没人验证过的垃圾。** 明确区分"验证过的"和"知道存在的"两个清单，
> 并用测试守住规则——这是把纪律写成代码。

## 七、遥测与日志

```python
class Telemetry(BaseModel):
    def on_request(self, telemetry_ctx: dict | None) -> None: ...
    def on_response(self, ...) -> None: ...
    def on_error(self, _err: BaseException) -> None: ...

    _log_completions_callback: Callable[[str, str], None] | None = PrivateAttr(...)
```

三个钩子覆盖请求生命周期，日志落盘方式通过回调注入（可以落文件、也可以送到别处）。

回想第三节：错误信息里**只放参数名不放参数值**——因为这些信息会流到这里，
而日志是会被人和系统读到的。**安全考虑和遥测设计是配套的。**

## 八、整层的结构

```
        Agent 只看到这个
        ────────────────
        llm.generate(messages, tools, ...)
              │
   ┌──────────┴───────────────────────────────┐
   │  能力表（12 维）决定这次怎么发            │
   │   ├─ 支持函数调用？ → 原生发              │
   │   └─ 不支持？ → 转 XML 提示 + 停止词      │
   │                   收回来再转回结构化      │
   ├──────────────────────────────────────────┤
   │  重试：指数退避；空响应且温度=0 时改温度  │
   ├──────────────────────────────────────────┤
   │  降级：重试用尽 → 换备用模型              │
   │        （只对"对方的问题"降级）           │
   ├──────────────────────────────────────────┤
   │  路由：RouterLLM 继承 LLM，一组=一个      │
   ├──────────────────────────────────────────┤
   │  统计：token/花费/缓存命中率              │
   │        （窗口取 max；命中率分母看口径）   │
   └──────────────────────────────────────────┘
              │
        litellm → 各家 API
```

## 九、这一节的可迁移结论

1. **把各家的差异收敛成一张显式的能力表**，而不是散落的 `if "claude" in model`。
   能测试、能一眼看全、加新模型有清单可对照。

2. **型号匹配需要"包含+排除"的有序规则**：纯白名单写不完，纯前缀会误伤。
   "先按家族收，再按型号排除"最贴合现实。

3. **标识符在传递中会被加装饰，处理前先还原**：`litellm_proxy/prod/xxx` 要剥掉前缀
   才能查到能力。

4. **能力缺失可以用提示词补**：不支持函数调用的模型，靠一套 XML 协议 + 双向转换
   模拟出来，上层代码完全不知情。

5. **让模型产出的格式按"模型容易写对"来选**，而不是"程序容易解析"。
   选 XML 而非 JSON，就是为了避免让模型手工转义代码里的引号和换行。

6. **协议约束往往是从解析器实现倒推的**："推理只能在调用之前"这条规则，
   是因为解析器靠结尾标记定位。

7. **重试只在结果可能不同的时候才有意义**：温度为 0 的空响应，
   必须先把温度改掉再重试，否则就是把同一个失败做 5 遍。

8. **只对"对方的问题"做降级**：连不上/限流/超时/500 换模型有用；
   参数错/被过滤换模型只是浪费钱。

9. **让"容器"和"元素"实现同一接口**：`RouterLLM` 继承 `LLM`，
   于是"一组模型"在类型上就是"一个模型"。和安全模块的融合器是同一个手法。

10. **容量指标合并要取 max，不要累加**：token 数相加，上下文窗口取最大。

11. **聚合多源指标必须搞清口径，而且可以用数据自身的矛盾来推断口径**：
    缓存读取数超过总输入数，就说明是分开报的。

12. **"支持列表"这类清单需要成文的纪律 + 测试守护**，否则会膨胀成没人验证过的垃圾。
    明确区分"验证过的"和"知道存在的"。

## 下一站

`tool/tool.py`（940 行）+ `mcp/`（2085 行）—— 工具层：
工具怎么定义、本地工具和 MCP 工具怎么统一成同一个接口、
客户端工具（在前端执行）怎么做。

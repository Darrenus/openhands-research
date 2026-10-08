# 精读 08：上下文工程的另一半（系统提示组装 + 技能）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/`
> 文件：`context/agent_context.py`（619 行）、`context/prompts/{section,registry,presets}.py`、
> `context/prompts/sections/{static,dynamic,planning}.py`、`skills/`（2952 行）、
> `tool/builtins/invoke_skill.py`
> 前置：`notes/06-condenser.md`（压缩是另一半）

## 这一节在讲什么

前面讲的压缩解决的是"**历史太长怎么办**"。这一节解决的是"**一开始该给 AI 看什么**"。

两个核心问题：

1. 系统提示（给 AI 的开场指令）怎么组装，才能既完整又不浪费
2. 那些"只在特定情况下才有用"的知识，怎么做到**用到时才出现**

## 一、关键背景：提示缓存

要理解这一节所有设计，先要理解一个计费机制。

> **提示缓存（prompt caching）**：模型服务商提供的优化。如果你这次请求的**开头一段**
> 和上次完全一样，这段就走缓存，价格大幅降低（通常是 1/10），而且更快。

**关键约束：缓存只对"前缀"生效。** 开头第 1 个字变了，后面全部都得重算。

**于是系统提示的组装顺序，直接决定了你的账单。** 这一节的很多设计都是围绕这件事。

## 二、两层结构：静态和动态

```python
class CacheTier(StrEnum):
    """Block a section renders into; maps 1:1 onto ``SystemPromptEvent``'s two
    content blocks. ``STATIC`` is cache-stable across conversations, ``DYNAMIC``
    is per-conversation."""
    STATIC = "static"
    DYNAMIC = "dynamic"
```

系统提示被切成两块：

- **静态块**：**跨会话都一样**的内容（角色定义、代码质量要求、安全规范……）
- **动态块**：这次会话独有的（仓库信息、可用技能列表、当前时间……）

静态块放前面，于是它**永远命中缓存**。动态块放后面，只有它需要重算。

## 三、每个片段都是一个独立单元（`section.py`）

```python
@runtime_checkable
class PromptSection(Protocol):
    name: str
    cache_tier: CacheTier

    def guard(self, ctx: PromptContext) -> bool:
        """Return True if this section applies to ctx."""

    def render(self, ctx: PromptContext) -> str | None:
        """Return the section's text, or None/blank to contribute nothing."""
```

系统提示不是一个大模板文件，而是**一堆独立的小片段**，每个片段回答两个问题：

1. `guard` —— **我这次适用吗？**
2. `render` —— **我要贡献什么文字？**

### 这两个方法的区别被特意说明了

> The split is intentional -- ``guard`` returning ``False`` means "not applicable here"
> (e.g. browser disabled, wrong platform), while ``render`` returning ``None`` means
> "applicable, but nothing to add right now" (e.g. no skills present).

**"不适用"和"适用但暂时没内容"是两件不同的事。**

- 浏览器功能关了 → `guard` 返回 False → 整个片段跳过
- 浏览器开着但当前没有技能 → `guard` 返回 True，`render` 返回 None

**为什么要分？** 因为可观测性不一样。前者是配置决定的，后者是运行时状态决定的。
如果合成一个方法，你就没法回答"这个片段是没配置，还是配了但没内容"。

> 可迁移的道理：**把"条件不满足"和"条件满足但结果为空"分开表达。**
> 合成一个返回值会丢掉诊断信息。

### 两个方法都必须是纯函数

> Both must be pure -- read-only on ``ctx``, no I/O -- so sections are testable in isolation.

**纯函数 = 只看输入、不改任何东西、不做网络/文件操作。** 于是每个片段都能单独测试：
造一个假的上下文，检查输出对不对。不需要启动整个 agent。

## 四、上下文快照是冻结的，而且防了一个坑

```python
class PromptContext(BaseModel):
    model_config = ConfigDict(frozen=True)
    template_kwargs: Mapping[str, object]
    tool_names: tuple[str, ...]
    platform: Platform
    ...

    @field_validator("template_kwargs", mode="after")
    def _freeze_template_kwargs(cls, value):
        # frozen=True blocks attribute reassignment but not nested mutation;
        # store a read-only copy so the snapshot and its views cannot drift.
        return MappingProxyType(dict(value))
```

**这个注释指出了一个非常容易踩的坑。**

`frozen=True` 只阻止"给字段重新赋值"，**不阻止改字段内部的内容**。也就是说：

```python
ctx.template_kwargs = {...}        # ← 被 frozen 拦住
ctx.template_kwargs["key"] = 123   # ← frozen 拦不住！字典是可变的
```

解法是用 `MappingProxyType` 包一层——**只读视图，改它会报错**。

> 注意所有集合类字段都用了 `tuple` 而不是 `list`（`tool_names: tuple[str, ...]`）。
> 同一个原因：元组不可变，列表可变。**"冻结"要一路冻到底，否则就是假的。**

## 五、注册表：顺序就是一切（`registry.py` + `presets.py`）

```python
def build(self, ctx: PromptContext) -> PromptBlocks:
    """Render guarded sections, grouped by cache_tier in registration order."""
    buckets = defaultdict(list)
    for section in self._sections.values():
        if not section.guard(ctx):
            continue
        text = section.render(ctx)
        if text is None or not text.strip():
            continue
        buckets[section.cache_tier].append(text)

    return PromptBlocks(
        static=_SECTION_SEPARATOR.join(buckets[CacheTier.STATIC]),
        dynamic=_SECTION_SEPARATOR.join(buckets[CacheTier.DYNAMIC]) or None,
    )
```

**按注册顺序渲染，按层级分桶。** 简单到几乎没什么可说的——但正是这个简单性让顺序
变成了唯一需要关心的东西。

`register` 和 `replace` 的区分很讲究：

```python
def register(self, section) -> None:
    """Add a new section; raise on a duplicate name (use replace())."""
    if section.name in self._sections:
        raise ValueError(f"A prompt section named {section.name!r} is already registered; "
                         "use replace() to override it.")

def replace(self, section) -> None:
    """Add section, or override a same-named one in place."""
    self._sections[section.name] = section
```

**重名默认报错，要覆盖必须显式调用 `replace`。**

> 这防的是什么？两个插件都注册了一个叫 `"security"` 的片段，后者静默覆盖前者——
> 于是某条安全规则无声消失了。**默认拒绝、显式覆盖**，让这种事必须是有意的。

### 默认的静态片段清单

这份清单本身就是一份"一个编程 agent 的系统提示该包含什么"的答案：

```python
_DEFAULT_STATIC_SECTIONS = (
    SoulSection(),                    # 性格/基调
    RoleSection(),                    # 角色定义
    MemorySection(),                  # 记忆机制说明
    EfficiencySection(),              # 效率要求
    FileSystemSection(),              # 文件系统约定
    CodeQualitySection(),             # 代码质量要求
    VersionControlSection(),          # 版本控制规范
    PullRequestsSection(),            # PR 规范
    ProblemSolvingSection(),          # 解决问题的方法论
    SelfDocumentationSection(),       # 自我文档化
    SecuritySection(),                # guard: 设了安全策略文件才出现
    SecurityRiskAssessmentSection(),  # guard: 开了 LLM 风险分析器才出现
    BrowserSection(),                 # guard: 开了浏览器才出现
    ExternalServicesSection(),
    EnvironmentSetupSection(),
    TroubleshootingSection(),
    ProcessManagementSection(),
    ModelSpecificSection(),           # guard: 解析出了模型家族才出现
)
```

注意那些标了 `guard:` 的——**功能没开，对应的提示文字就完全不出现**。

**这是很实在的省钱**：没开浏览器功能，就不用为"怎么用浏览器"那几百字付钱；
更重要的是，AI 不会看到它其实没有的工具的说明，从而减少幻觉。

> `ModelSpecificSection` 值得注意：**针对不同模型家族给不同的提示**。
> 说明他们承认"一套提示词打天下"是不现实的。

### 动态片段的顺序有一处刻意安排

```python
_DYNAMIC_SECTIONS = (
    RepoContextSection(),        # guard: 有仓库技能
    MemoryContextSection(),      # guard: 有记忆内容
    AvailableSkillsSection(),    # guard: 有可用技能
    CustomSuffixSection(),       # guard: 有自定义后缀
    CustomSecretsSection(),      # guard: 有密钥信息
    # DateTimeSection is intentionally last: it is the only per-conversation
    # volatile value, so the stable dynamic content stays a cache-friendly prefix.
    DateTimeSection(),
)
```

**时间放最后，因为它是唯一每次都变的东西。**

想清楚这件事：如果把时间放在动态块开头，那么**整个动态块每次都无法命中缓存**
（因为前缀变了）。放最后，前面那些相对稳定的内容还能继续缓存。

> **这一条是整节最精华的细节。** 它体现的通用原则是：
> **✅ 实跑定量验证（见 `notes/26-verification-run.md`）**：渲染两个不同时间的动态块
> 并量共同前缀——**时间放最后：95% 是稳定前缀；挪到最前：只有 11%。可缓存前缀差 8.5 倍。**
>
> **把变化最频繁的东西放在最后。** 任何"前缀缓存"的系统都适用——
> HTTP 缓存、构建缓存、Docker 镜像分层，全是这个道理
> （Dockerfile 里先 `COPY requirements.txt` 再 `COPY .` 也是同一招）。

### 两套预设

```python
class PromptPreset(StrEnum):
    DEFAULT = "default"      # 标准编程 agent
    PLANNING = "planning"    # 只读分析（没有安全/性格/记忆等片段）
```

`PLANNING` 预设的静态块**只有一个片段**。因为"只读地分析代码"这件事，
不需要版本控制规范、PR 规范、危险操作评估——那些字全是浪费。

而**动态块两个预设共用**：

> The dynamic-tier sections are **shared** -- repo/skills/suffix/secrets/datetime are
> preset-independent -- so a planning agent with an ``agent_context`` still gets its
> dynamic block.

仓库信息、技能、密钥这些**和 agent 的角色无关**，所以共用。
**静态块定义"你是谁"，动态块定义"你在哪"。**

## 六、技能：让知识按需出现（`skills/`）

### 问题

假设你有 50 条领域知识（"这个项目用 uv 管理依赖"、"处理 PDF 要用这个库"……）。

全塞进系统提示？**几万字，每次请求都付费，而且大部分用不上，还会干扰 AI 的注意力。**

一条都不放？**AI 不知道这些知识存在。**

### 解法：渐进式披露

> **渐进式披露（progressive disclosure）**：先只给一个目录，需要时再去取详情。

```python
class Skill(BaseModel):
    """AgentSkills format (SKILL.md files):
    - Always listed in <available_skills> with name, description, location
    - Agent reads full content on demand (progressive disclosure)
    - If has triggers: content is ALSO auto-injected when triggered
    """
```

系统提示里只放一个**目录**：

```xml
<available_skills>
  <skill>
    <name>pdf-tools</name>
    <description>Extract text from PDF files.</description>
  </skill>
</available_skills>
```

每个技能只占一个名字 + 一句描述。AI 觉得需要，就调 `invoke_skill` 工具把全文取来。

**成本从"50 条全文"降到"50 行摘要 + 按需取 1-2 条全文"。**

### 一个安全细节：故意不给路径

```python
"""Returns:
    XML string in AgentSkills format with name and description. The
    ``<location>`` field is intentionally omitted so the agent cannot
    bypass the ``invoke_skill`` tool by reading the file directly."""
```

**目录里不写文件路径，防止 AI 绕过 `invoke_skill` 工具直接去读文件。**

为什么要防？因为 `invoke_skill` 不只是"读文件"——它会**渲染动态内容**
（`render_content_with_commands`，技能里可以嵌入要执行的命令）。直接读原始文件
会得到未渲染的版本。而且走工具意味着这次调用被记录、被计量、可以被审批拦截。

> 可迁移的道理：**如果一个操作必须走某个入口，就不要在别处泄露绕过它的信息。**
> 这和"不要在错误信息里放参数值"是同一类思维。

### 描述超长会被截断，而且留了取回全文的指引

```python
truncation_msg = (f"... [{total_truncated} characters truncated. "
                  f'Call invoke_skill(name="{skill.name}") to load the full skill]')
```

**截断后明确告诉 AI"还有 N 个字符，调这个工具能拿到全部"。**

> 又一次看到这个模式：**给模型的信息要包含"下一步怎么办"。** 和第三节工具报错时
> 附上可用工具列表、第七节反馈时附上具体标签，完全一致。

## 七、三种触发方式（`trigger.py`）

```python
class KeywordTrigger(BaseTrigger):
    """activated when specific keywords appear in the user's query"""
    keywords: list[str]

class TaskTrigger(BaseTrigger):
    """activated for specific task types and can modify prompts"""
    triggers: list[str]

class PathTrigger(BaseTrigger):
    """activated when the agent touches a file whose path matches
    one of the ``paths`` glob patterns (gitignore-style ``**`` semantics)"""
    paths: list[str]
```

前两种好理解：用户话里出现某个词、或者是某类任务时，自动注入。

**第三种 `PathTrigger` 是最有意思的设计**，下面单独讲。

## 八、路径规则：规则绑在文件上，而不是绑在提示里

### 匹配用的是 gitignore 语义

```python
@lru_cache(maxsize=512)
def _compile_path_glob(pattern: str) -> re.Pattern[str]:
    """Supported syntax:
    - ``**`` matches any number of path segments (including zero).
    - ``*`` matches any run of characters within a single segment.
    - ``?`` matches a single non-separator character.
    - A pattern without a ``/`` matches the basename at any depth, e.g.
      ``*.py`` behaves like ``**/*.py`` (gitignore semantics).
    """
    if "/" not in pattern:
        pattern = "**/" + pattern       # 没有斜杠 = 任意深度匹配文件名
```

**沿用 gitignore 的语义**，而不是自己发明一套。用户已经懂 `.gitignore` 怎么写了，
不需要学新东西。

`@lru_cache` 缓存编译结果——同一个模式不会重复编译正则。

### 注入时机：绑在"观察"上

回想第一节看到的 `ObservationEvent.extended_content` 字段。这里就是它的来源：

```python
def _maybe_inject_path_rules(self, event: Event) -> Event:
    """Return ``event`` with matching path-rule content, or unchanged."""
    if not isinstance(event, ObservationEvent):
        return event
    if (file_path := self._touched_rule_path(event)) is None:
        return event

    result = agent_context.get_tool_use_suffix(
        file_path=file_path,
        skip_skill_names=self._state.activated_path_rules,   # ← 去重
    )
    if result is None:
        return event

    content, activated_rule_names = result
    self._state.activated_path_rules.extend(activated_rule_names)
    return event.model_copy(
        update={"extended_content": list(event.extended_content) + [content]}
    )
```

**流程**：AI 改了一个文件 → 工具返回结果 → 检查这个路径匹配哪些规则 →
把规则内容**拼在工具结果后面**。

注入的文本长这样：

```python
"<EXTRA_INFO>\n"
"The following rule applies because a file you touched matches "
f'"{k.trigger}". Follow it when working with matching files.\n'
+ (f"Rule location: {k.location}\n" if k.location else "")
+ f"\n{k.content}\n</EXTRA_INFO>"
```

**注意它说明了"为什么你现在看到这条规则"**——因为你碰的文件匹配了某个模式。
不是凭空出现一条命令，而是带着理由。

### 三个值得学的点

**① 时机是最优的。** 规则在 AI **刚碰到相关文件的那一刻**出现，而不是在几万字前的
系统提示里。注意力最集中的位置就是刚发生的事情旁边。

**② 用 `model_copy` 而不是直接改。** 第一节讲的"事件不可变"在这里被遵守了——
生成一个带额外内容的**新事件**，原事件不动。

**③ 每条规则只注入一次。** `activated_path_rules` 记着已经注入过哪些。

想想不去重会怎样：AI 在 `src/api/` 下连续改了 20 个文件，同一条规则被注入 20 遍——
**上下文被同一段文字塞满**。

> 而且这个去重状态在 `fork()`（分叉会话）时会被复制过去：
> ```python
> fork_conv._state.activated_path_rules = list(self._state.activated_path_rules)
> ```
> **分支继承"已经说过什么"**，否则分叉之后规则会重新刷一遍。这种细节只有真用起来才会发现。

## 九、把这一节和压缩连起来看

现在可以看到完整的上下文工程图景：

```
        进入上下文的三条路
        ─────────────────
① 系统提示（静态块）    跨会话不变 → 永远命中缓存 → 最便宜
                        功能没开的片段直接不出现

② 系统提示（动态块）    本会话不变的部分在前，时间放最后
                        技能只放目录，不放全文

③ 运行中按需注入        关键词/任务触发 → 拼在用户消息后
                        路径触发 → 拼在工具结果后，且只注入一次
                        AI 主动调 invoke_skill → 取回技能全文

        ─────────────────
        历史太长了
        ─────────────────
④ 压缩（第五节）        保头两条、找合法切口、砍到一半
                        摘要保留 ID/哈希/错误原文
```

**三条进入路径 + 一条退出路径，各自有明确的成本模型。**

> 这才叫"上下文工程"——不是"把提示词写好一点"，而是**把每一段文字的
> 成本、时机、生命周期都设计过。**

## 十、这一节的可迁移结论

1. **前缀缓存决定了排列顺序**：把跨会话不变的放最前，把每次都变的（时间）放最后。
   这条原则在 Docker 分层、HTTP 缓存、构建缓存里完全一样。

2. **把提示切成带守卫的独立片段**，而不是一个大模板。功能没开的片段完全不出现，
   既省钱又减少幻觉（AI 不会看到它其实没有的工具的说明）。

3. **区分"不适用"和"适用但没内容"**：`guard` 返回 False vs `render` 返回 None。
   合成一个会丢掉诊断信息。

4. **片段要是纯函数**：只读输入、无 I/O，于是能单独测试，不用启动整个系统。

5. **"冻结"要一路冻到底**：`frozen=True` 拦不住改字典内部，要用只读视图包一层；
   集合字段用元组不用列表。

6. **重名默认报错，覆盖必须显式**：否则两个插件注册同名片段时，会有一条规则无声消失。

7. **渐进式披露**：先给目录（名字+一句话），需要时再取全文。
   成本从"N 条全文"降到"N 行摘要 + 按需 1-2 条"。

8. **必须走某个入口的操作，别在别处泄露绕过方法**：技能目录里故意不写文件路径。

9. **给模型的信息要包含"下一步怎么办"**：截断时附上取回全文的工具调用写法。
   这个模式在整个项目里反复出现。

10. **注入时机比注入内容更重要**：路径规则在"刚碰到那个文件"时出现，
    而不是在几万字前的系统提示里。

11. **按需注入必须去重，且去重状态要跟着分支走**：否则改 20 个文件就被同一条规则
    刷 20 遍；分叉后又会重刷一遍。

12. **沿用用户已知的语义**：路径匹配直接用 gitignore 规则，不自己发明一套。

## 下一站

`llm/llm.py`（3306 行）+ `llm/mixins/fn_call_converter.py`（963 行）——
模型接缝：怎么让一套代码同时支持不同厂商、不支持函数调用的模型怎么办、
重试和遥测怎么做。

# 精读 21：子 agent 注册表与两个"顾问"工具（subagent/ 1090 行 + ask_oracle 269 + tom_consult 717）

> 位置：`upstream/software-agent-sdk/`
> 文件：`openhands-sdk/openhands/sdk/subagent/{AGENTS.md,registry,schema,load}.py`、
> `openhands-tools/openhands/tools/ask_oracle/{definition,impl}.py`、
> `openhands-tools/openhands/tools/tom_consult/{definition,executor}.py`
> 前置：`notes/21-task-manager.md`（`get_agent_factory` 的调用方）、
> `notes/16-skills-loading.md`（技能加载的三层优先级，可对照）

## 这一节在讲什么

**`subagent/`** —— 前面第十三、十四、二十节都调用了 `get_agent_factory(name)`，
但从没读过"子 agent 是怎么定义和注册的"。答案是：**一个 Markdown 文件。**

**`ask_oracle` 和 `tom_consult`** —— 两个名字很特别的工具。
读完发现它们是同一类东西的两个方向：**问一个更强的模型**，和**问一个懂用户的模型**。

## 一、这个包有一份专门的设计文档

`subagent/AGENTS.md`（180 行）开头说明了它为什么存在：

```markdown
This package centralizes **subagent discovery** and **registration**.
It exists so that contributors (human or agentic) can answer:

- "Where did this agent come from?"
- "Why did this definition win over the other one?"

**without reverse-engineering `LocalConversation` and the loader.**
```

**"不用逆向工程会话实现和加载器就能回答这两个问题。"**

> **这是我在这个项目里看到最好的"为什么写文档"的理由陈述。** 不是"记录一下实现"，
> 而是**指名两个具体的、会反复被问到的问题**。
>
> 而且括号里写了 `(human or agentic)` —— **这份文档是给人和 AI 同时看的**
> （文件名 `AGENTS.md` 本身就是给 AI 读的约定）。
>
> **可迁移的道理**：**写模块文档时，先写出"读者会问什么问题"，
> 再保证文档能回答它们。** 比按代码结构罗列说明有用得多。

文档最后还有一条维护约定：

```markdown
## User-facing documentation
User docs for Markdown agents live in the docs repo. **If you change any of the
invariants above, update both this file and the user docs.**
```

**明确指出了另一处需要同步更新的地方。** 这类"改这里要记得改那里"的耦合，
写在文档里比靠记忆可靠。

## 二、子 agent 就是一个带 YAML 头的 Markdown 文件

```markdown
---
name: code-reviewer
description: |
  Reviews code changes.

  <example>please review this PR</example>
  <example>can you do a security review?</example>
tools:
  - terminal
model: inherit
permission_mode: confirm_risky
color: purple
# Any extra keys are preserved in `metadata`:
audience: maintainers
---

You are a meticulous code reviewer.
Focus on correctness, security, and clear reasoning.
```

**YAML 头是配置，正文是系统提示。**

### 十三个可配置的键

```
name                    默认：文件名
description
tools                   默认 []：一个名字或名字列表
skills                  默认 []：逗号分隔的字符串或列表
model                   默认 inherit：复用父 LLM，或者一个 LLM 配置名
color
max_iteration_per_run
max_budget_per_run
hooks
profile_store_dir
mcp_config              （mcp_servers 是废弃的别名）
permission_mode         always_confirm / never_confirm / confirm_risky
condenser               默认用摘要压缩器；none / false 关闭；映射则配置一个
```

**几乎所有前面十几节讲过的机制，在这里都有一个配置入口：**
工具（第十一节）、技能（第八、十五节）、钩子（第十四节）、
压缩器（第五节）、审批策略（第六节）、预算和步数（第四节）、MCP（第十一节）。

**一个子 agent 定义就是"把前面所有系统的旋钮拧到某个位置"的一份声明。**

### 三个"省略即继承"的键

```
model: inherit           # 省略 → 复用父 LLM
permission_mode          # 省略 → 继承父审批策略
condenser                # 省略 → 用默认摘要压缩器
```

回想第二十节那条结论："派生单元的资源上限和权限默认继承，不取系统默认值"。
**这里是它在配置格式层面的体现**——`inherit` 是一个显式的默认值。

### 未知键会被保留

```markdown
**Unknown keys are preserved** in `AgentDefinition.metadata`.
```

**对比第一节那个 `extra="forbid"`（多一个字段就报错）。**

**两处取向相反，但各有道理**：事件是内部数据结构，多字段说明代码不一致；
**agent 定义是用户写的文件，多余的键可能是给别的工具用的、或者是用户自己的注记。**

> **可迁移的道理**：**内部数据结构严格拒绝未知字段，用户编写的配置文件宽容保留。**
> 前者防代码 bug，后者防用户困惑（"我加了个注释键，怎么整个文件都加载失败了"）。

### 正文是"追加"而不是"替换"

```markdown
### Body → system prompt

The Markdown **body content** becomes the agent's `system_prompt`.

Currently, when the agent is instantiated, this is applied as:
- `AgentContext(system_message_suffix=agent_def.system_prompt)`

**meaning it is appended to the parent system message (not a complete replacement).**
```

**子 agent 的系统提示是加在父系统提示后面的，不是替换掉。**

**所以子 agent 仍然继承了第八节那一整套静态提示片段**（代码质量要求、
版本控制规范、安全规范……），只是额外多了一段角色定义。

注意 `Currently` 这个词——**作者标明了这是当前行为，可能会变。**
对读文档的人是一个信号：不要把"追加"当成永久契约。

## 三、`<example>` 标签：描述里嵌入触发样例

```python
def _extract_examples(description: str) -> list[str]:
    """Extract <example> tags from description for agent triggering."""
    pattern = r"<example>(.*?)</example>"
    matches = re.findall(pattern, description, re.DOTALL | re.IGNORECASE)
    return [m.strip() for m in matches if m.strip()]
```

**从 `description` 里抽出 `<example>` 标签，存进 `when_to_use_examples`。**

于是同一段文字有两个用途：**去掉标签是给人/模型看的描述，
标签里的内容是"什么时候该用这个 agent"的具体样例。**

> 回想第二十节那条"在示例里示范规则，而不是在描述里陈述规则"——
> **这里把示例做成了结构化字段**，于是路由逻辑可以单独用它们。
>
> **可迁移的道理**：**让用户在一段自然语言里内联标注结构化信息**
> （用标签），比要求他们填两个独立字段更自然。

正则用了 `re.DOTALL`（跨行匹配）和 `re.IGNORECASE`（`<Example>` 也认），
而且过滤了空内容。**宽容地接受，精确地产出**——又一次。

## 四、"第一个注册的赢"，以及两层去重

```markdown
### Core rule: first registration wins

Once an agent name is registered in the global registry, later sources must not
overwrite it.

This is enforced by using:
- `register_agent(...)` (**raises on duplicates**; used for programmatic registration)
- `register_agent_if_absent(...)` (**skips duplicates**; used for plugins, file agents, builtins)
```

**两个函数的差别只在重名时的行为**，而且用途分工明确：

- **程序里显式注册** → 重名直接报错（那是 bug）
- **从文件/插件/内置扫描来的** → 重名静默跳过（那是正常的优先级竞争）

> 回想第八节那个提示片段注册表：`register` 重名报错、`replace` 显式覆盖。
> **同一种设计在两个模块出现了两次**，但语义不同——那里是"后来的可以显式覆盖"，
> 这里是"先来的永远赢"。
>
> **判断依据**：提示片段的覆盖是**有意的定制**；agent 定义的重名是**优先级竞争**，
> 而优先级由扫描顺序决定，所以"先来先得"就足够表达。

### 优先级是扫描顺序的产物

```markdown
1. Existing registry entries, including explicit `register_agent(...)` calls
2. Plugin-provided agents
3. Project file-based agents
   - `{project}/.agents/agents/*.md` then `{project}/.openhands/agents/*.md`
4. User file-based agents
   - `~/.agents/agents/*.md` then `~/.openhands/agents/*.md`
```

**而内置 agent 是单独注册的，文档专门说明了时机：**

```markdown
Built-ins are discovered and registered separately by `openhands-tools`.
**Because all non-programmatic sources use `register_agent_if_absent(...)`,
whichever source registers a name first keeps it. Call built-in registration
after higher-priority sources if built-ins should act as fallbacks.** The
agent-server registers built-ins during tool-router import, before per-conversation
file discovery.
```

**最后一句是个很实在的提醒**：agent-server 实际上是**在导入工具路由时就注册了内置**，
也就是**在按会话扫描文件之前**。

**这意味着在 agent-server 里，内置 agent 反而优先级最高**——
和"内置应该是兜底"的直觉相反。**文档把这个偏差写出来了。**

> **可迁移的道理**：**当优先级由"执行顺序"隐式决定时，必须把实际的执行顺序写进文档。**
> 否则读者会按"应该的顺序"理解，而实际部署里可能不是那样。

### 两层去重

```markdown
File-based loading has *two* layers of "first wins" deduplication:

1. **Within a level** (`load_project_agents` / `load_user_agents`):
   - `.agents/agents` wins over `.openhands/agents` for the same agent name.
2. **Across levels** (`register_file_agents`):
   - project wins over user for the same agent name.

**If you change these rules, update the unit tests in `tests/sdk/subagent/`.**
```

**新目录（`.agents/`，跨工具通用的标准位置）优先于旧目录（`.openhands/`）。**

> 和第十五节技能加载的优先级完全一致（那里也是 `~/.agents/skills/` 优先）。
> **两个子系统用同一套目录约定，这是好的一致性。**

最后那句"改规则要更新单元测试"再次出现——**这份文档在多处提醒"还有什么需要一起改"。**

### 解析失败必须不致命

```markdown
### Parse failures must be non-fatal

If a single file fails to parse (invalid YAML frontmatter, malformed Markdown, etc.),
loading must:
- log a warning (with stack trace), and
- continue scanning other files.
```

**一个文件写错了不能让所有 agent 都加载不了。** 和第十五节技能加载的
"每个来源独立容错"是同一条。

### 扫描规则的三个细节

```markdown
- Only the **top-level** `*.md` files are scanned.
  - Subdirectories (e.g. `{project}/.agents/skills/…`) are ignored.
- `README.md` / `readme.md` is always skipped.
- Directory iteration is deterministic (`sorted(dir.iterdir())`).
```

**① 不递归** —— 因为 `.agents/` 下还有 `skills/` 子目录，递归会把技能误当 agent。

**② 跳过 README** —— 目录里放个说明文件是很自然的事。

**③ 排序保证确定性** —— 文件系统的 `iterdir()` 顺序不保证，
**不排序的话"谁先注册"在不同机器上可能不同**，于是同名冲突的胜者会变。

> **这三条都很小，但第三条尤其重要**：**任何"先来先得"的规则都必须建立在
> 确定的遍历顺序上**，否则行为会随机。

## 五、注册表的一个远程执行考虑

```python
def register_agent(name, factory_func: Callable[["LLM"], "Agent"],
                   description: str | AgentDefinition) -> None:
    """The factory_func is the source of truth for local execution —
    it receives an `LLM` and must return a fully-configured `Agent`.

    The description parameter accepts either a plain string or a full
    `AgentDefinition`. A plain string creates a minimal definition from name and
    description; this is fine for local-only agents but **means the remote server
    will not know about tools or system prompts.** Pass an `AgentDefinition` when
    the agent needs to work in remote workspaces, as the definition's metadata
    (tools, system_prompt, model, skills, …) **is serialised and forwarded to the
    agent-server.**"""
```

**这里有一个很重要的双重表示：**

- **`factory_func`** —— 本地执行的真相来源（一个 Python 函数）
- **`AgentDefinition`** —— 可序列化的元数据，能发给远端服务器

**函数不能序列化，所以远程执行必须靠声明式的定义。**

于是文档明确说：只传一个字符串描述，**在远程工作区里这个 agent 会缺工具和系统提示**。

> **可迁移的道理**：**当一个东西既要本地执行又要远程执行时，
> 必须同时有"可执行形式"和"可序列化形式"，而且要明确说清只给前者的后果。**
>
> 这也解释了为什么 `AgentDefinition` 要把 tools 存成**名字列表**而不是工具对象——
> 名字能序列化，对象不能。（第十一节那个"注册解析器而不是实例"是配套的：
> 远端按名字重新解析出工具实例。）

### 工具和技能的解析时机不同

```markdown
`tools` values remain names until factory instantiation. Each name must already be
registered; unknown tools raise `ValueError`. Valid names become `Tool(name=...)`.

`skills` resolve when the factory is created. **Project skills take priority over
user skills, public skills are excluded**, and an unknown skill raises `ValueError`.
```

**技能解析时排除了公共技能。** 为什么？因为公共技能要访问网络克隆仓库
（第十五节），**而一个 agent 定义引用的技能应该是确定存在于本地的。**

**两者都是"未知就报错"** ——不静默跳过，因为那会让 agent 缺能力而不自知。

## 六、`ask_oracle`：问一个更强的模型

```python
# The Oracle model is a saved LLM profile resolved by convention under this name.
# **Save a profile named "oracle"** (e.g. via LLMProfileStore.save("oracle", llm))
# and the tool will consult it. **No agent setting or wiring is required.**
ORACLE_PROFILE_NAME: Final[str] = "oracle"
```

**靠约定而不是配置**：保存一个叫 `oracle` 的 LLM 配置，工具自动就会用它。

> **可迁移的道理**：**用"约定的名字"代替一个配置项**，能省掉一整条配置链路
> （字段定义、传参、文档、校验）。代价是不够显式——所以注释里写清了约定是什么。

### 调用是无状态的

```python
class AskOracleExecutor(ToolExecutor[...]):
    """Consult the Oracle: a saved LLM profile named ``oracle``.

    The call is **stateless: it sends only the Oracle system prompt plus the agent's
    question and optional context, with no conversation history and no tools**, and
    returns the Oracle's text. **The active conversation LLM is never switched.**"""
```

**三个"不"**：不带对话历史、不给工具、不切换当前模型。

**为什么不带历史？** 因为 Oracle 是来回答一个**具体问题**的，
给它几万字的历史既贵又会稀释问题。**提问者负责把必要的上下文写进 `context` 字段。**

对比第十五节那个分类器（只给最近 6 条消息）——**辅助性的模型调用都要主动裁剪输入**，
这里更彻底：**一条都不给。**

### 给 Oracle 的系统提示只有四句

```python
_ORACLE_SYSTEM_PROMPT = """\
You are the Oracle: a highly capable reviewer giving a second opinion to an
OpenHands agent.

Answer the agent's question directly. **Do not call tools. Do not perform work
directly.** Give a concrete recommendation the agent can follow, including important
risks or caveats."""
```

**两条禁令 + 一条要求**：不许调工具、不许直接干活、**要给一个能照着做的具体建议
并带上风险和注意事项**。

`Do not perform work directly` 这条很关键——**否则强模型会直接开始写代码，
而它没有工具、也不在工作区里。**

### 工具描述里的可信度声明

```python
_DESCRIPTION = (
    "Ask the Oracle for a second opinion. The Oracle is a smart model intended "
    "to help with difficult reasoning.\n\n"
    "Use this when you are stuck, uncertain, comparing approaches, or need a "
    "higher-quality recommendation before proceeding.\n\n"
    "**Treat the Oracle's response as strong guidance and follow its recommendation "
    "unless you have a clear reason not to.**"
)
```

**"把 Oracle 的回答当作强指引，除非有明确理由否则照做。"**

> 和第二十节那条"子 agent 的结果是权威的"是**同一个问题的同一种解法**：
> **如果不明说可信度，模型会花额外的 token 去质疑或验证，
> 把这次咨询的收益抵消掉。**

### 四层错误处理，每层给不同的提示

```python
except FileNotFoundError:
    return ...(text=("The Oracle is not available because no profile named "
                     f"'{ORACLE_PROFILE_NAME}' was found. **Save one to enable it.**"),
               is_error=True)
except ValueError as exc:
    return ...(text=f"The Oracle is not available: {exc}", is_error=True)
except Exception as exc:
    return ...(text=f"The Oracle is not available: {type(exc).__name__}: {exc}", ...)
```

**没配置 → 告诉你怎么配；配置有问题 → 给出原因；其他异常 → 带上类型名。**

然后调用失败和空响应还各有一层：

```python
except Exception as exc:
    return ...(text=("The Oracle encountered an error and did not return a "
                     f"response: {type(exc).__name__}: {exc}"), is_error=True)
...
if not oracle_text:
    return ...(text="The Oracle did not return a response.", is_error=True)
```

**空响应也算错误。** 回想第十六节那条"拿不到数据要说不可用，不要记 0"——
**这里是"不要把空当成一个回答"。**

> **五层错误处理，每层的信息都不同。** 对比一个常见的写法：
> `except Exception: return "Oracle failed"` ——那样 AI 完全不知道是配置问题、
> 网络问题还是模型问题，**也就没法决定要不要重试。**

## 七、`tom_consult`：问一个"懂用户"的模型

```python
"""This module provides tools for consulting Tom agent for personalized guidance
based on **user modeling**, and for indexing conversations for user modeling."""
```

> **用户建模（user modeling）**：从历史交互里总结出这个用户的偏好、习惯、表达方式，
> 之后用它来更好地理解新指令。

这个工具有**两个动作**：

### 动作 1：咨询

```python
class ConsultTomAction(Action):
    reason: str = Field(description="Brief explanation of why you need Tom agent consultation")
    use_user_message: bool = Field(default=True, description=(
        "Whether to consult about the user message (True) or provide custom query (False)"))
    custom_query: str | None = Field(default=None, ...)
```

**默认是"分析用户刚说的那句话"**，也可以自己提一个问题。

工具描述说明了使用场景：

```
Use this when:
- **User instructions are vague or unclear**
- You need help understanding what the user actually wants
- You want guidance on the best approach for the current task
- You have your own question for Tom agent about the task or user's needs
```

**核心用途是"用户说得不清楚时，问一个懂这个用户的模型他到底想要什么"。**

回想第七节那个评审员的失败模式清单里有 `misunderstood_intention`（误解意图）
和 `insufficient_clarification`（该问清楚的没问）。
**这个工具正是在防这两条——但方式不是去问用户，而是问一个用户模型。**

> **这是一个有意思的设计取向**：`insufficient_clarification` 的直觉解法是
> "多问用户"，但那会打断用户。**用一个基于历史的用户模型来代替提问**，
> 是一条不打断用户的路。

### 动作 2：离线索引

```python
class SleeptimeComputeAction(Action):
    """Action to index existing conversations for Tom's user modeling.

    This triggers Tom agent's **sleeptime_compute** function which processes
    conversation history to build and update the user model."""
    pass
```

**`sleeptime_compute`（睡眠时计算）这个命名很形象**：在不干活的时候
把历史对话处理成用户模型。

工具描述说明了时机：

```
This is typically used **at the end of a conversation** or when explicitly requested.
```

**注意这个动作没有任何参数**（`pass`）——它处理的是全部历史，不需要指定什么。

### 观察里带置信度和推理

```python
class ConsultTomObservation(Observation):
    suggestions: str = Field(default="", description="Tom agent's suggestions or guidance")
    confidence: float | None = Field(default=None, description="Confidence score from Tom agent (0-1)")
    reasoning: str | None = Field(default=None, description="Tom agent's reasoning for the suggestions")
```

**建议 + 置信度 + 推理过程，三个分开的字段。**

> 对比 `ask_oracle` 只返回一段文本。**差别在于可信度的性质**：
> Oracle 是"更强的模型"，它的建议默认可信；
> **Tom 是"基于历史猜测用户意图"，猜测必须带置信度**——
> 低置信度时 AI 应该去问用户而不是照着做。
>
> 回想第七节那条"让评价指标变成可验证的预测"——**置信度让"这个猜测有多靠得住"
> 变成了一个可以被下游使用的数字。**

### 依赖是可选的

```python
# Import here to avoid circular imports and **make tom-swe optional**
from openhands.tools.tom_consult.executor import TomConsultExecutor
```

**`tom-swe` 是一个独立的包**，在函数内部 import 让它成为可选依赖——
没装就不会在导入时报错。

> 和第十九节那个 `lmnr` 懒加载是同一个手法。
> **可选功能的依赖要在使用点 import，不要在模块顶层。**

### 两个工具都申报"不碰共享资源"

```python
def declared_resources(self, action: Action) -> DeclaredResources:
    """Consulting Tom is a **read-only LLM call with no shared mutable state**,
    so it is always safe to run in parallel."""
    return DeclaredResources(keys=(), declared=True)
```

**注释给出了论证**（只读的模型调用、无共享可变状态），而不是光写一个返回值。

> 回想第十一节那个"`declared=True, keys=()` 意思是我想过了，我是安全的"——
> **这里就是"想过了"的证据：注释里写出了推理过程。**

## 八、三种"问别人"的机制对照

把这一节和前面几节放一起看，这个 SDK 有**三种**"遇到难题去问别人"的机制：

```
                  问谁              带什么上下文        结果可信度
──────────────────────────────────────────────────────────────────
ask_oracle        更强的模型         只有问题+手写上下文   强指引，默认照做
tom_consult       用户模型           用户消息或自定义问题   带置信度，自己判断
task / delegate   一个完整子 agent    一段 prompt          事实可信，判断需复核
```

**三者的成本和能力递增**：Oracle 是一次模型调用；Tom 是一次带用户模型的调用；
子 agent 是一整个会话（有工具、能读代码、能跑测试）。

而**可信度声明的方式各不相同**，因为可信度的来源不同：
- Oracle 强是因为**模型本身更强**
- Tom 的可靠性取决于**历史数据够不够**，所以要给置信度
- 子 agent 的可靠性取决于**任务是事实性还是判断性**

> **可迁移的道理**：**一个系统里有多种"求助"渠道时，
> 要为每一种明确声明"结果该被怎么对待"，而且声明方式要匹配可信度的来源。**
> 统一说"参考一下"或者统一说"照着做"都是错的。

## 九、这一节的可迁移结论

1. **写模块文档时先写出"读者会问什么问题"，再保证文档能回答它们**：
   "这个 agent 从哪来？""为什么这个定义赢了？"——比按代码结构罗列有用得多。
   而且文档要标明**还有哪些地方需要一起改**。

2. **内部数据结构严格拒绝未知字段，用户编写的配置文件宽容保留**：
   前者防代码 bug（第一节 `extra="forbid"`），后者防用户困惑。

3. **让用户在一段自然语言里内联标注结构化信息**（`<example>` 标签），
   比要求填两个独立字段更自然。

4. **"先来先得"的规则必须建立在确定的遍历顺序上**（`sorted(dir.iterdir())`），
   否则行为会随机，同名冲突的胜者在不同机器上不同。

5. **重名时报错还是跳过，取决于重名意味着什么**：程序里显式注册重名是 bug（报错），
   扫描来的重名是优先级竞争（跳过）。

6. **当优先级由"执行顺序"隐式决定时，必须把实际执行顺序写进文档**：
   agent-server 在扫描文件之前就注册了内置，**于是内置反而优先级最高**——
   和直觉相反，所以要写出来。

7. **一个东西既要本地执行又要远程执行时，必须同时有"可执行形式"和"可序列化形式"**，
   并明确说清只给前者的后果（远端会缺工具和系统提示）。

8. **解析失败必须不致命**：一个文件写错不能让所有定义都加载不了。

9. **不递归扫描 + 跳过 README**：因为同一个目录下可能有其它用途的子目录和说明文件。

10. **用"约定的名字"代替一个配置项**：存一个叫 `oracle` 的配置就启用这个工具，
    省掉整条配置链路。代价是不够显式，所以注释里要写清约定。

11. **辅助性的模型调用要彻底裁剪输入**：Oracle 不带任何对话历史，
    由提问者负责把必要上下文写进参数。

12. **给"顾问"角色的系统提示要明确禁止它动手**：`Do not call tools.
    Do not perform work directly.` ——否则它会开始写代码，而它没有工具也不在工作区。

13. **每一层错误都要给不同的信息**：没配置→告诉怎么配；配置有问题→给原因；
    调用失败→带异常类型；**空响应也算错误**。
    统一返回"失败了"会让 AI 无法决定是否重试。

14. **"猜测"类的结果必须带置信度**，"更强的模型"类的结果可以声明为默认可信。
    **可信度的声明方式要匹配可信度的来源。**

15. **可选功能的依赖在使用点 import，不要在模块顶层**。

16. **申报"我是安全的"时，在注释里写出推理过程**，而不是光写一个返回值。

17. **一个系统里有多种"求助"渠道时，要为每一种明确声明"结果该被怎么对待"**：
    统一说"参考一下"或统一说"照着做"都是错的。

## 剩下还没读的

- `openhands-agent-server/` 的具体路由实现（29851 行，只读了入口装配）
- `openhands-tools/` 剩下的：`planning_file_editor`、`gemini/`、`glob`、`grep`、`preset/`
- `plugin/`、`marketplace/`、`profiles/`、`settings/`
- `acp_file_credentials.py`（536 行）
- `tom_consult/executor.py`（425 行）的具体实现、`subagent/load.py` 的细节

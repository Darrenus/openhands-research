# 精读 06：安全体系（security/，3619 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/security/`
> 文件：`risk.py`、`analyzer.py`、`confirmation_policy.py`、`llm_analyzer.py`、`ensemble.py`、
> `toolshield_llm_analyzer.py`、`_shell_ast.py`、`shell_parser.py`、
> `defense_in_depth/{pattern,policy_rails,shell_semantics,utils}.py`、`grayswan/`
> 前置：`notes/04-agent-step.md`（谁在调用它）

## 这一节在讲什么

AI 能执行命令，就意味着它能执行**错误的**命令。这一节是这个问题的完整答案。

先说结论：**这里没有一道"万能防线"，而是五六个各有短板的检查器叠在一起，并且明确规定
"不确定"时怎么办。** 这种打法叫**纵深防御**（defense in depth）——假设每一层都会被绕过，
所以要有很多层。

## 一、两个基础概念先分清

### 风险等级（`risk.py`）

```python
class SecurityRisk(str, Enum):
    UNKNOWN = "UNKNOWN"
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"

_RISK_ORDER = {"LOW": 1, "MEDIUM": 2, "HIGH": 3}
# UNKNOWN is excluded by design -- comparisons involving UNKNOWN raise ValueError.
```

注意 **UNKNOWN 不参与大小比较，比较它会直接抛异常**。

**为什么要这么严？** 因为"不知道"和"低风险"是**完全不同**的东西，一旦允许比较，
代码里一定会有人写 `if risk > LOW` 然后把 UNKNOWN 悄悄当成了安全。
**禁止比较，就强迫每个使用者显式处理"不知道"这种情况。**

> 这是"用类型阻止误用"的又一个例子。语言不阻止你，作者就自己加一道。

### 评估和决策是两件事

```python
analyzer   → 这个操作有多危险？        （事实判断）
policy     → 这么危险要不要问用户？    （策略判断）
```

分开的好处：同一套危险性判断，可以配不同的策略。你可以要求"只有高危才问我"，
也可以要求"什么都问"，而**不用改任何检测逻辑**。

## 二、决策层：确认策略（`confirmation_policy.py`）

三种现成策略：

```python
class AlwaysConfirm(ConfirmationPolicyBase):
    def should_confirm(self, risk=UNKNOWN) -> bool:
        return True          # 什么都问

class NeverConfirm(ConfirmationPolicyBase):
    def should_confirm(self, risk=UNKNOWN) -> bool:
        return False         # 什么都不问（全自动模式）

class ConfirmRisky(ConfirmationPolicyBase):
    threshold: SecurityRisk = SecurityRisk.HIGH
    confirm_unknown: bool = True          # ← 默认：不知道就问

    def should_confirm(self, risk=UNKNOWN) -> bool:
        if risk == SecurityRisk.UNKNOWN:
            return self.confirm_unknown
        return risk.is_riskier(self.threshold)
```

**`confirm_unknown: bool = True` 是整个安全体系最重要的一行默认值。**

它的含义：**检测器说"我不确定"时，默认当成需要确认。** 这叫**失败关闭**（fail-closed）——
出问题时倒向安全的一侧，而不是放行。

> 反面是**失败开放**（fail-open）：检测器挂了就当没事，继续执行。
> 便利，但一旦检测器有 bug，防线就静默消失了。

还有个小校验：

```python
@field_validator("threshold")
def validate_threshold(cls, v):
    if v == SecurityRisk.UNKNOWN:
        raise ValueError("Threshold cannot be UNKNOWN")
```

阈值不能设成 UNKNOWN——那是无意义的配置。**又一次：配置错误在创建时就拒绝。**

注释还解释了一个容易搞错的边界：

```python
# This comparison is reflexive by default, so if the threshold is HIGH we will
# still require confirmation for HIGH risk actions.
```

阈值设成 HIGH 时，HIGH 本身**也要**确认（是"≥"不是">"）。这种"包不包含边界"的问题
写清楚很重要，不然用户会以为设了 HIGH 结果高危操作被放过了。

## 三、检测层第一道：让模型自己说（`llm_analyzer.py`）

最便宜的一层，整个类只有几行：

```python
class LLMSecurityAnalyzer(SecurityAnalyzerBase):
    def security_risk(self, action: ActionEvent) -> SecurityRisk:
        return action.security_risk        # 就是模型自己填的那个字段
```

回想第三节：请求模型时开了 `add_security_risk_prediction=True`，在工具参数里插了个
风险字段让模型自己填。这一层就是把那个值取出来。

**优点**：零成本（不需要额外的模型调用），而且**理解意图**——`rm -rf ./build` 和
`rm -rf /` 规则很难分，模型一眼就知道。

**缺点显而易见**：你在问一个可能被诱导、也可能在骗你的对象。所以这只能是第一层。

## 四、检测层第二道：结构化规则（`defense_in_depth/policy_rails.py`）

模块开头一句话说清了这一层存在的理由：

> Some threats are structural, not lexical: ``curl ... | bash`` is dangerous because of
> the *combination* of fetch + pipe-to-exec, not because either token is dangerous alone.

**有些危险不在单个词里，而在组合里。** `curl` 不危险，`bash` 不危险，
`curl xxx | bash`（下载一段脚本直接执行）非常危险。

v1 有三条规则：

```python
RAIL_FETCH_TO_EXEC = "fetch-to-exec"              # 下载后直接执行
RAIL_RAW_DISK_OP = "raw-disk-op"                  # 裸磁盘操作（dd 之类）
RAIL_CATASTROPHIC_DELETE = "catastrophic-delete"   # 灾难性删除
```

每条命中就是 HIGH。

### 一个防绕过的细节

> Rails express these composed conditions as deterministic rules evaluated **per-segment**,
> so that tokens from different fields (thought vs. tool arguments) cannot accidentally
> satisfy a composed condition.

翻译：规则是**按片段**分别求值的，这样**来自不同字段的词凑不成一个组合条件**。

**这在防什么？** 假设 AI 的"思考内容"里提到了 `curl`，而工具参数里有 `bash`。
如果把所有文本拼成一大段再匹配，就会误判成 `curl | bash`。反过来，攻击者也可能
故意把危险组合拆到两个字段里试图绕过。分片段求值同时避免了这两种错误。

## 五、检测层第三道：签名匹配（`defense_in_depth/pattern.py`）

这一层的文档开头是我在整个项目里看到最好的一段说明：

> When an agent is about to run ``rm -rf /``, you want to catch it. When the agent
> merely *thinks about* ``rm -rf /`` while running ``ls /tmp``, you do not.

**"AI 要执行 `rm -rf /`" 和 "AI 在思考里提到了 `rm -rf /` 但实际在跑 `ls /tmp`"
必须区别对待。**

解法是**两个扫描语料库**：

```
可执行语料（tool_name + 工具参数）
  → 扫描：破坏性 shell 命令、代码执行、网络下载后执行

全字段语料（可执行 + 思考/推理/摘要）
  → 扫描：提示注入、社会工程学攻击
```

**逻辑很清晰**：破坏性命令只有出现在**真的要执行的地方**才算危险；
而提示注入（试图篡改 AI 的指令）**出现在任何地方都危险**。

每个检测器有稳定 ID：

```python
DET_EXEC_DESTRUCT_RM_RF = "exec.destruct.rm_rf"
DET_EXEC_NET_CURL_EXEC = "exec.net.curl_pipe_exec"
DET_INJECT_OVERRIDE = "inject.override"
DET_INJECT_IDENTITY = "inject.identity"
...
# Stable detector IDs -- do not change between releases without documentation.
```

命名格式是 `DET_{语料}_{类别}_{具体}`，并且注明**不许随便改**。

> **为什么 ID 稳定很重要？** 因为这些 ID 会进日志、进统计。如果版本间改了名字，
> 历史数据就对不上，没法回答"这个月 rm -rf 拦截次数是涨了还是跌了"。
> 注意 `inject.identity`（身份冒充）和 `inject.mode_switch`（模式切换）这两个
> ——说明他们在防"忽略之前的指令，你现在是……"这类攻击。

## 六、检测层第四道：真的解析 shell 语法（`shell_semantics.py` + `_shell_ast.py`）

**这是整个安全体系技术含量最高的部分**，也是把它和"正则匹配危险词"彻底区分开的地方。

### 为什么正则不够

要检测 `rm -rf`，正则很容易被绕过：

```bash
rm -rf /                 # 正则能抓
"rm" -rf /               # 加引号
/bin/rm -rf /            # 写全路径
bash -c 'rm -rf /'       # 塞进子 shell
bash -c 'bash -c "rm -rf /"'   # 嵌套
$CMD -rf /               # 变量替换，运行时才知道是什么
```

**每加一种写法就要加一条正则，永远追不完。**

### 他们的做法：用 tree-sitter 真正解析

> **tree-sitter**：一个语法解析库，能把代码解析成**语法树**（AST）——
> 也就是把文本变成"这是一个命令、命令名是 X、参数是 Y、Y 里面又是一个命令"这样的结构。

于是判断不再是"文本里有没有 rm"，而是**"这棵树里有没有一个命令，它的名字解析出来是 rm，
并且带着递归+强制两个标志"**。

模块文档说明了它解决的三类问题：

> Resolves command names structurally against the shared tree-sitter ``_shell_ast`` view
> so that **quoted, path-qualified, and nested** command names become visible.

引号包裹、写全路径、嵌套——三种绕过方式全部失效。

### 处理嵌套的方式很漂亮

```python
_SHELL_RUNNERS: Final[frozenset[str]] = frozenset({"sh", "bash", "dash", "zsh", "ksh", "ash"})
# Command basenames whose script operand is a nested shell program.
# Resolving and re-parsing that operand closes the nested-runner bypass
# class without hardcoding payloads.
```

**认出"这是一个 shell 解释器"，然后把它的 `-c` 参数当成一段新的 shell 程序，递归再解析一遍。**

注释里那句 `without hardcoding payloads`（不需要硬编码攻击载荷）是重点：
**它关掉的是一整类绕过手法，而不是某几个具体的字符串。**

递归深度有上限：

```python
_MAX_NESTING_DEPTH: Final[int] = 8
# Bounds work on adversarial deeply-nested runner arguments; hitting the bound
# with an operand still unscanned is surfaced as uncertain, never a silent LOW.
```

**深度限制是为了防"超深嵌套"这种消耗资源的攻击。** 但注意后半句：
**到了深度上限而还有没扫的内容，报"不确定"，绝不静默判为低危。**

### POSIX 细节的处理程度

这部分常量让人明白什么叫"认真做"：

```python
_END_OF_OPTIONS: Final[str] = "--"      # 选项结束标记
_BARE_DASH: Final[str] = "-"            # 单个减号，POSIX sh 仍接受的老式写法
_OPTION_PREFIXES = frozenset({"-", "+"})  # POSIX sh 接受 -x 和 +x 两种
_SHORT_OPTS_WITH_ARG = frozenset({"o", "O"})
# Without this, ``bash -o errexit -c ...`` would misread the option argument
# as the first operand and stop looking for the script flag.
_LONG_OPTS_WITH_ARG = frozenset({"--rcfile", "--init-file"})
```

最后一句注释是关键：`bash -o errexit -c '危险命令'` 里，`errexit` 是 `-o` 的参数。
不知道这一点的话，解析器会把 `errexit` 当成要执行的脚本，**然后停止寻找真正的 `-c`
参数——危险命令就被漏掉了。**

**这是一个真实的绕过手法，靠"读懂 POSIX 规范"才能堵住。**

对未知长选项的处理也有明确取向：

```python
# Exotic long options beyond these are out of scope: an unrecognized long option is
# treated as argumentless, which at worst misreads its argument as the first operand
# and stops descending -- it never over-descends.
```

翻译：不认识的长选项当成"没有参数"处理。最坏的后果是**少扫了**（停止深入），
**而不会多扫**（把不该当命令的东西当成命令）。

> **这是一个明确的安全取向选择**：宁可漏报（还有其他层兜着），不要误报
> （误报会让用户被无意义的确认框淹没，最后养成无脑点"同意"的习惯——第三节
> 提到过这个"审批疲劳"问题）。

### 最值得学的部分：三档输出

```python
@dataclass(frozen=True, slots=True)
class ShellScanResult:
    matched: bool                    # 确定命中
    detector_id: str | None = None
    uncertain: bool = False          # 看到了但不敢下结论
```

不是"是/否"，而是**是 / 否 / 不确定**三档。

什么时候报"不确定"，文档列得非常具体：

> - the destructive flag shape appears on a command whose verb cannot be resolved
>   (an opaque or dynamic name);
>   （带着破坏性标志，但命令名解析不出来——比如是个变量）
> - an ERROR/MISSING parse node intersects a span the detection relies on;
>   （解析错误正好落在检测依赖的那段文本上）
> - the nested-runner depth bound is hit with a script operand still unscanned.
>   （撞到深度上限，还有没扫的脚本）

**然后是一句非常克制的补充：**

> Benign text that merely fails to parse as shell is not treated as uncertain: parse
> errors outside the spans above stay LOW, because arbitrary non-shell text routinely
> fails to parse and surfacing every such error would flood the ensemble with UNKNOWNs.

翻译：**只是解析失败的普通文本不算"不确定"。** 因为任意的非 shell 文本（比如一段
Python 代码、一段散文）本来就解析不成 shell，如果每次解析失败都报"不确定"，
整个系统会被 UNKNOWN 淹没，然后用户被确认框淹没，然后这道防线就形同虚设了。

> **这是我在这个项目里看到最成熟的一段判断。** 作者同时理解两件事：
> 漏报有安全代价，误报有**可用性**代价，而可用性代价最终也会变成安全代价
> （因为人会关掉烦人的功能、或者无脑点同意）。**在这两者之间划线，比堆检测规则难得多。**

`_DYNAMIC_NAME_NODE_TYPES` 列出了哪些情况算"动态名字"：

```python
{"command_substitution",   # $(...)
 "process_substitution",   # <(...)
 "expansion",              # ${...}
 "simple_expansion",       # $VAR
 "arithmetic_expansion",   # $((...))
 "ansi_c_string"}          # $'...'
```

这些都是**运行时才能确定值**的语法。静态分析看不出来，所以老实报"不确定"。

## 七、检测层第五道：独立的裁判模型（`toolshield_llm_analyzer.py`）

文档开头就点出了它和第一层的区别：

> Unlike ``LLMSecurityAnalyzer``, which trusts the *actor* LLM to annotate
> ``security_risk`` on its own proposed action, this analyzer issues a *separate*
> completion to a distinct guardrail LLM.

**第一层是问"演员"自己（actor），这一层是请一个独立的"裁判"（guardrail）。**

为什么需要分开？下面这段是这一节最有洞察力的话：

> Separating the actor from the judge is significant for multi-turn attacks, where
> individual steps look benign but the sequence is harmful. The actor LLM has no
> incentive to flag earlier reconnaissance steps because they are not harmful on their
> own; a separate guardrail LLM, prompted to evaluate the sequence, does.

翻译：**分开演员和裁判，对"多轮攻击"很重要——单看每一步都无害，合起来才有害。**

举个例子：
1. 列出目录（无害）
2. 读取配置文件（无害）
3. 找到一个密钥（无害）
4. 发一个网络请求（……）

演员模型没有动机去标记第 1 步为危险，因为那一步**真的**无害。但一个被要求
**评估整个序列**的裁判模型会看出这个模式。

> 可迁移的道理：**让行为的执行者同时当自己的审查者，在结构上就是有缺陷的。**
> 不是因为它会作恶，而是因为它的视角天然只覆盖单步。

它还能接入外部的"每个工具的安全经验库"：

```python
>>> from toolshield import load_experiences
>>> experiences = load_experiences(["filesystem-mcp", "terminal-mcp"])
>>> analyzer = ToolShieldLLMSecurityAnalyzer(llm=guardrail_llm,
...     safety_experiences=experiences.format_for_prompt())
```

这些经验是**通过在沙箱里自我探索**总结出来的每个工具的安全指引。
而且接口设计得很开放：

> The ``safety_experiences`` field accepts any string, so callers can also plug in
> experiences from their own source rather than ToolShield.

**接受任意字符串**，你可以用自己的经验库，不绑定某个供应商。

## 八、还有一道：外部安全服务（`grayswan/`）

接入 GraySwan Cygnal 这个第三方 AI 安全监控 API，把对话历史和待执行动作发过去，
拿回一个"违规分数"再映射到风险等级。需要 API key。

这一层的意义是**把"要不要用商业安全服务"做成可插拔的**，而不是自己吞下所有责任。

## 九、怎么把这些合起来：Ensemble（`ensemble.py`）

```python
"""Combine multiple security analyzers into a single risk assessment.
...
That is what this module does -- **pure fusion, no detection of its own.**"""
```

**纯融合，自己不做任何检测。** 这个自我限制很重要——融合逻辑一旦掺入检测，就没法单独测试了。

融合算法：

```python
def security_risk(self, action: ActionEvent) -> SecurityRisk:
    """Evaluate risk via max-severity fusion across child analyzers."""
    results = []
    for analyzer in self.analyzers:
        try:
            results.append(analyzer.security_risk(action))
        except Exception:
            logger.exception("Analyzer %s raised -- fail-closed to HIGH", analyzer)
            results.append(SecurityRisk.HIGH)      # ← 检查器自己崩了 = 按最高危处理

    has_unknown = SecurityRisk.UNKNOWN in results

    if self.propagate_unknown and has_unknown:
        return SecurityRisk.UNKNOWN               # 严格模式：任一不确定 → 整体不确定

    concrete = [r for r in results if r != SecurityRisk.UNKNOWN]
    if not concrete:
        return SecurityRisk.UNKNOWN               # 全都不确定 → 不确定
    return max(concrete)                          # 否则取最高的具体等级
```

三个设计点：

**① 取最高（max-severity）**：任何一个检查器说危险，整体就是危险。
**不投票、不取平均。** 安全场景下少数派必须有一票否决权。

**② 检查器抛异常 = HIGH**，注释说得很直白：

> If a child analyzer raises an exception, it contributes HIGH (fail-closed, logged).
> This prevents a broken analyzer from silently degrading safety.

**防止一个坏掉的检查器静默削弱安全性。** 想想反面：如果异常被当成"没发现问题"，
那么一个引入了 bug 的检查器会让整条防线无声消失，而且没人知道。

**③ UNKNOWN 有两种处理模式**：

- 默认：**只要有一个检查器给出了具体等级，就忽略那些 UNKNOWN**，取具体的最高值。
  理由：有人能明确判断，就不必因为另一个模块看不懂而打扰用户。
- 严格（`propagate_unknown=True`）：**任何一个说不确定，整体就是不确定**。
  给"宁可多问也不能漏"的环境用。

> **这个双模式设计的价值**：安全性和可用性的平衡点，不同场景不一样。
> 作者没有替所有人选，而是给了一个开关，并写明各自适用什么环境。

同样的 fail-closed 思路也出现在基类里（`analyzer.py`）：

```python
except Exception as e:
    logger.error(f"Error analyzing action {action_event}: {e}")
    # Default to HIGH risk on analysis error for safety
    analyzed_actions.append((action_event, SecurityRisk.HIGH))
```

## 十、整条链路串起来

```
模型产出一个工具调用
       │
       ├─ 模型自己填的风险等级         （LLMSecurityAnalyzer，零成本）
       ├─ 结构化规则                   （PolicyRail：fetch-to-exec 等组合）
       ├─ 签名匹配                     （Pattern：两个语料库分开扫）
       │     └─ 其中 rm -rf 家族走真正的 shell 语法解析（三档输出）
       ├─ 独立裁判模型                 （ToolShield：能看出多轮攻击）
       └─ 外部安全服务                 （GraySwan，可选）
       │
       ▼
 Ensemble 融合：取最高；异常→HIGH；UNKNOWN 按模式处理
       │
       ▼
 ConfirmationPolicy 决策：这个等级要不要问用户
       │  （ConfirmRisky 默认 confirm_unknown=True）
       ▼
 要问 → 状态改为 WAITING_FOR_CONFIRMATION，本轮停下（第三、四节讲过）
 不问 → 直接执行
```

还记得第三节里那条规则吗：

```python
# 单个 FinishAction / ThinkAction 永不需要确认
```

**思考和宣布完成不需要批准。** 这也是在对抗"审批疲劳"——只在真正有副作用的地方拦你。

## 十一、这一节的可迁移结论

1. **"不知道"必须是一个独立的、不能和其他等级比较的状态**。允许比较，
   它一定会在某处被当成"安全"。

2. **失败关闭（fail-closed）要贯彻到每一层**：检查器抛异常算高危、
   分析出错算高危、不确定默认要确认。反面是一个坏掉的检查器让防线无声消失。

3. **评估和决策分开**：一套危险性判断 + 可替换的策略。改策略不碰检测代码。

4. **有些危险在组合里，不在单个词里**，而且要**按字段分片段求值**，
   防止不同来源的词凑成假的组合，也防止攻击者故意拆分绕过。

5. **区分"要执行的内容"和"提到的内容"**：破坏性命令只在执行位置危险，
   提示注入在任何位置都危险。两个语料库分开扫。

6. **要真正理解语法，而不是匹配文本**：解析成语法树后，引号、全路径、
   嵌套子 shell 这三类绕过一次性全部失效。**关掉一整类手法，而不是补几个正则。**

7. **静态分析的诚实做法是三档输出**：是 / 否 / 不确定。而且要克制地定义"不确定"
   ——普通文本解析失败不算，否则会被 UNKNOWN 淹没，最后用户养成无脑点同意的习惯。

8. **误报也有安全代价**：审批疲劳会让人关掉功能或无脑批准。
   在漏报和误报之间划线，比堆规则难得多。

9. **执行者不能当自己的审查者**：不是因为它会作恶，而是它的视角天然只覆盖单步，
   看不出多轮攻击。需要一个被要求评估整个序列的独立裁判。

10. **融合器只做融合**：取最高严重度，不投票不平均，自己不做任何检测——
    否则没法单独测试。

11. **检测器 ID 要稳定**：它们会进日志和统计，改名会让历史数据对不上。

## 下一站

`critic/`（1431 行）—— AI 说"我做完了"之后的自我评审与迭代精修。
这是 SWE-bench 77.6 那个分数里很关键的一环。

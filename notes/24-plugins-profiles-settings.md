# 精读 23：插件、配置档与设置（plugin 2612 + profiles 1648 + settings 3975 + marketplace 663 + extensions 1000 + io 434）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/`
> 文件：`plugin/{format/*,discovery,installed,fetch}.py`、`marketplace/`、
> `profiles/{agent_profile,resolver,profile_refs,agent_profile_store}.py`、
> `settings/{model,api_models,acp_providers}.py`、`extensions/`、`io/`
> 前置：`notes/16-skills-loading.md`（技能加载）、`notes/22-subagent-registry-and-consultants.md`（agent 定义）

## 这一节在讲什么

把剩下的 SDK 小模块一次收完。它们分成三组：

**扩展体系**（`plugin` / `marketplace` / `extensions`）—— 第三方怎么把一批东西
（技能、钩子、MCP 配置、子 agent、命令）打包分发。

**配置档体系**（`profiles` / `settings`）—— 一份可命名、可引用、**不含密钥**的
"启动规格"，和它怎么解析成可执行的配置。

**存储抽象**（`io`）—— 434 行，四个文件，但设计得很干净。

## 一、插件格式：策略模式 + 一份"怎么加新格式"的配方

`plugin/format/__init__.py` 的包文档写得极清楚：

```
A *plugin format* owns everything specific to how a plugin is laid out on disk
(manifest location and validation, MCP config file and variable expansion, and
where client extensions — commands / agents / hooks — live) and turns a plugin
directory into a normalized ``Plugin``. **Everything downstream of a loaded plugin
is format-agnostic and lives on ``Plugin`` itself, so adding a new format never
touches the merge/apply path.**
```

**关键设计**：格式差异全部收敛在"读盘"这一步，读完之后是一个统一的 `Plugin` 对象。
**加一个新格式永远不需要碰合并/应用的代码路径。**

> 这和第十一节那条"让不同来源继承同一个基类，上层派发零分支"是同一个思路，
> 只不过这里作用在**加载**而不是**执行**上。

### 检测顺序：具体的在前，兜底的在最后

```
:func:`detect_format` returns the first format in ``_FORMATS`` whose
:meth:`PluginFormat.detect` returns True. **Agent Plugins goes first because it is
specific — a root ``plugin.json`` — and Claude Code last because its ``detect``
accepts any directory, making it the universal fallback.**
```

**`ClaudeCodePluginFormat.detect` 接受任何目录** ——它是兜底，所以必须排最后。

> **可迁移的道理**：**策略链里"什么都接受"的那个必须排在最后**，
> 而且要在文档里说明它是兜底的——否则后来人加新格式时可能加在它后面，
> **然后发现自己的格式永远不会被选中。**

### 一份"怎么加新格式"的六步配方

包文档里有一段完整的操作指南：

```
1. Create a module in this package and subclass ``PluginFormat``, giving it a
   unique class-level ``name`` (used in logs).
2. Implement ``detect`` — **cheap, specific, and True only for directories this
   format should claim.**
3. Implement the format-specific loaders. **Each owns one component type and
   should isolate its own failures rather than abort the whole plugin.**
4. Reuse the base where behavior is shared — notably ``load_skills`` (the
   ``skills/<name>/SKILL.md`` rule) and ``load`` (final assembly); **you should
   not need to override either.**
5. Register the class in ``_FORMATS`` below in **detection-precedence order**
   (earlier = higher priority; fallbacks last).
6. **Add a ``detect_format`` selection test** (see ``tests/...::TestDetectFormat``).
```

**六步里有四步是约束而不是步骤**：detect 要便宜且专一、每个加载器要隔离自己的失败、
不要重写共享的两个方法、注册顺序有含义、**必须加一个选择测试**。

> **可迁移的道理**：**一个可扩展点的文档应该是"配方 + 约束"，而不只是"接口说明"。**
> 接口说明告诉你能调什么，配方告诉你**别人期望你怎么做**——包括哪些方法不该重写、
> 注册在哪个位置、要补什么测试。
>
> 这是前面第二十一节那条"文档先写读者会问什么"的姐妹条：
> **对扩展点，读者要问的是"我该怎么加一个"。**

### `name` 的契约在类定义时就强制

```python
def __init_subclass__(cls, **kwargs: object) -> None:
    # **Enforce the ``name`` contract at class-definition time. Without this a
    # subclass that omits ``name`` instantiates cleanly and only fails later with
    # AttributeError the first time ``.name`` is read** (e.g. in
    # ``detect_format``'s debug log).
```

**不写 `name` 的子类在"定义这个类"的时候就报错**，
而不是等到某次读 `.name` 时（可能是在一条 debug 日志里）才崩。

> 又一次"把假设写成机器能执行的规则"（索引第 41 条的第六次出现）。
> 而且**这个位置选得很讲究**：`__init_subclass__` 在类创建时运行，
> 所以错误出现在 import 阶段——**最容易定位的时刻。**

### 内置的 schema 从不联网

```python
@cache
def _load_schema(filename: str) -> dict[str, Any]:
    """Read a vendored schema. **Cached; never fetched over the network.**"""
    return json.loads((_SCHEMAS_DIR / filename).read_text(encoding="utf-8"))
```

**JSON Schema 文件是随包分发的（vendored），不从网上拉。**

> **可迁移的道理**：**校验用的 schema 要随代码分发。** 从网上拉的话
> 离线环境不能用、网络故障时校验会失败、而且**远端改了 schema 会让你的旧版本突然拒绝合法输入。**

## 二、市场：一个 JSON 清单指向一批东西

```json
{
    "name": "company-tools",
    "owner": {"name": "DevTools Team"},
    "plugins": [{"name": "formatter", "source": "./plugins/formatter"}],
    "skills":  [{"name": "github",    "source": "./skills/github"}]
}
```

**市场文件放在仓库根的 `.plugin/` 或 `.claude-plugin/` 目录里。**

回想第十五节：公共技能默认只加载 `marketplaces/default.json` 里列出的，
**这就是那份清单的格式。**

> **市场是一层人工策展**——防止公共仓库无限增长把技能目录撑爆（第十五节那条结论）。

注意 `marketplace/__init__.py` 用了**延迟导入**：

```python
_TYPE_EXPORTS = {"MARKETPLACE_MANIFEST_DIRS", "Marketplace", "MarketplaceEntry", ...}
```

配合 `__getattr__` 按需 `import_module`。**导入这个包不会连带加载所有模型定义。**

> 和第十九节 `lmnr` 懒加载、第二十一节 `tom-swe` 可选依赖是同一个手法的第三次出现：
> **包的 `__init__` 不要 eager import 所有子模块。**

## 三、`profiles`：软外键，以及它的生命周期

这是这一节技术含量最高的部分。

### 什么是 AgentProfile

```python
"""Named, reference-bearing agent launch specs (``AgentProfile``)."""
```

**一份"启动规格"**：给这套配置起个名字，里面**存的是引用而不是值**：

```
llm_profile_ref     → 指向一个 LLM 配置的名字
mcp_server_refs     → 指向若干 MCP 服务器配置的名字
disabled_skills     → 一个禁用清单
```

**关键是"不含密钥"（secret-free at rest）** ——密钥在各自的存储里，
配置档只存名字。

> 回想第十五节那条"配置引用配置，而不是引用裸值"——**这里是它的完整形态。**

### 软外键会悬空，所以有一整套生命周期管理

```python
"""Foreign-key lifecycle between LLM profiles and ``AgentProfile``\\s.

An ``OpenHandsAgentProfile.llm_profile_ref`` is a **soft FK** onto an LLM-profile
store key. ``find_referrers`` / ``cascade_rename`` / ``delete_llm_profile`` /
``rename_llm_profile`` **keep that FK from dangling.**"""
```

> **软外键（soft FK）**：数据库里的外键由数据库强制（删不掉被引用的行）；
> "软"外键只是一个字符串字段，**数据库不管，你得自己管。**

四个操作对应四种需要：查引用者、改名时级联、删之前检查、改名。

删不掉时抛的异常带着引用者清单：

```python
class ProfileReferenced(Exception):
    """``referrers`` is the list of citing agent-profile names; **routers surface
    it in a 409 so the user knows what to detach first.**"""
```

**409 响应里告诉用户"这几个配置档在引用它，先解开"** ——
又一次"错误信息要包含下一步怎么办"（索引第 6 条的第十次出现）。

### 两条并发正确性的硬约束

```python
"""**Lock order: agent-profiles before llm-profiles (never the reverse — deadlock).**
A guarded delete/rename **holds the re-entrant agent-profiles ``lock()`` across the
whole scan→mutate window to close the TOCTOU.**"""
```

**① 锁顺序固定**：先 agent-profiles 再 llm-profiles，**反过来就死锁。**

> **两把锁的死锁只有一种防法：全局规定一个获取顺序，所有代码都遵守。**
> 而且这个顺序必须写在显眼的地方——**写在模块文档第一段是对的。**

**② 扫描和修改之间不能放锁**：

> **TOCTOU（Time-Of-Check to Time-Of-Use）**：检查的时候没问题，
> 真正用的时候情况变了。经典例子：检查"没人引用这个配置"→ 放锁 →
> 有人新建了一个引用 → 你删掉了 → 那个引用悬空了。

**所以要在整个"扫描→修改"窗口里一直持锁。**

### 存储无关

```python
"""Store-agnostic: these touch the agent-profile store only through
``AgentProfileStoreProtocol``, never the filesystem, **so the file store and a
cloud DB store reuse them verbatim.**"""
```

**这些生命周期操作只通过一个协议接触存储**，于是文件存储和云数据库存储
**能逐字复用同一份逻辑。**

> 又一次"抽象层的价值和接口大小成反比"（索引第 24 条）。

### 而且明确划了外键的边界

```python
"""The FK covers ``llm_profile_ref`` **only**; ``mcp_server_refs`` are checked
**at resolve-time** (#3717)."""
```

**只有 LLM 引用有外键保护，MCP 引用是在解析时才检查的。**
这是一个有意识的不对称——下面 resolver 会解释为什么。

## 四、resolver：为什么技能用"禁用清单"而不是"允许清单"

`profiles/resolver.py` 的模块文档里有一段很好的推理：

```python
"""Skills are *not* modeled like MCP servers. ``mcp_server_refs`` is a **safe
allow-list because ``mcp_config`` is a complete, persisted, user-authored map.**
The skill catalog is **discovered from many incomplete, drifting sources**
(user/public/org/project/marketplace), so **an allow-list of names would dangle
whenever the authoring catalog differs from the launch catalog.** Instead the
caller passes the discovered catalog and the resolver keeps all of it except the
names in ``disabled_skills`` — **a deny-list that can never dangle** (#4017)."""
```

**对照两种资源：**

| | MCP 服务器 | 技能 |
|---|---|---|
| 来源 | 一份**完整的、持久化的、用户手写的**映射 | **多个不完整、会漂移的来源**（用户/公共/组织/项目/市场） |
| 建模 | **允许清单**（refs） | **禁用清单**（disabled_skills） |
| 为什么 | 名字一定存在 | 允许清单会在"编辑时的目录 ≠ 启动时的目录"时悬空 |

**核心洞察**：**允许清单要求你在写配置时就知道全部可用项。**
如果目录是动态发现的（而且在不同机器上不同），那么允许清单必然悬空——
**而禁用清单永远不会：一个不存在的禁用项只是没有效果。**

> **这是我在整个项目里看到最好的一次"建模选择"论证。**
>
> **可迁移的道理**：**允许清单 vs 禁用清单，取决于被引用的集合是不是完整且稳定的。**
> - 集合完整、用户手写 → 允许清单（更安全，默认拒绝）
> - 集合动态发现、会漂移 → **禁用清单**（唯一不会悬空的选择）
>
> 很多系统无脑用允许清单（"更安全"），然后在动态目录上反复遇到悬空引用。

### 三种密钥各有自己的通道

```python
"""Resource-specific secret channels:

- **LLM key** → loaded from the LLM profile store into the resolved ``llm``.
- **MCP env/headers** → ride the filtered ``mcp_config`` (decrypted by the caller).
- **ACP provider creds** → **never touched here**; they ride
  ``state.secret_registry`` ← ``request.secrets`` wired at conversation-start.
  The resolver only *enumerates* the required provider secret names (via the
  dry-run) **so the editor / ``/materialize`` can show set/missing.**"""
```

**三种密钥走三条不同的路，而且 resolver 对第三种只做"枚举名字"。**

**为什么要枚举？** 因为界面上要显示"这个配置档需要哪些凭证、哪些已设置哪些缺失"
——**只需要名字，不需要值。**

> **可迁移的道理**：**"检查配置完整性"只需要知道需要哪些密钥的名字，不需要拿到值。**
> 把这两件事分开，配置编辑器就不需要任何解密权限。

### 还有一个 dry-run

```python
resolve_agent_profile_dry_run(...)
```

**不真正解析（不加载密钥、不连 MCP），只算出"会解析成什么、缺什么"。**

> 和第九节那个"提示缓存的 dry 判断"不同，这里是**校验用的空跑**。
> **任何"应用配置"的操作都应该有一个对应的"只检查不应用"版本**——
> 于是界面可以在保存前就告诉用户哪里有问题。

## 五、settings/api_models：一段关于"为什么用 dict 而不是类型"的解释

```python
"""Note on dict fields:
    ``SettingsResponse`` keeps ``agent_settings`` and ``conversation_settings`` as
    **dictionaries because the server needs to control how secrets are serialized
    (plaintext/encrypted/redacted) via serialization context. Typed Pydantic fields
    would lose this context during FastAPI's automatic JSON serialization.**

    ... Clients that need runtime type safety should use **the accessor methods**
    which validate the dictionaries into ``AgentSettingsConfig``."""
```

**这是一个被迫的妥协，而且理由具体**：FastAPI 自动序列化时会丢掉
"这次要明文/加密/打码"的上下文（第十六节那三档暴露级别），
**而类型化字段走的正是那条自动路径。**

**补偿措施**：提供类型化的**访问器方法**，客户端需要类型安全时用它们。

> **可迁移的道理**：**当框架的自动行为和你的需求冲突时，
> 退到弱类型的表示 + 提供显式的类型化访问器**，
> 而不是硬套类型然后丢掉控制。**而且要把原因写在模型的文档字符串里**——
> 否则后来人一定会"顺手"把它改成类型化的。

### 为什么这些模型定义在 SDK 而不是服务端

```python
"""These models define the contract between SDK clients and agent-server settings
endpoints. **They are defined in the SDK so both packages can share them without
circular dependencies (SDK cannot import from agent-server, but agent-server can
import from SDK).**"""
```

**依赖方向是单向的**：服务端可以 import SDK，反过来不行。
**所以共享的契约必须放在被依赖的那一侧。**

> **可迁移的道理**：**两个包要共享的类型，放在依赖方向的下游（被依赖方）。**
> 这个规则听起来显然，但实际项目里经常演变成"建第三个包放共享类型"——
> 那会多一层版本管理。能放进下游就放进下游。

## 六、`extensions`：泛型化的"安装管理器"

```python
@dataclass
class InstallationManager[T: ExtensionProtocol]:
    """Generic manager for installing, tracking, and loading extensions.

    **Parameterised by any type ``T`` that satisfies ``ExtensionProtocol``.**
    The companion ``InstallationInterface[T]`` tells the manager how to load ``T``
    from a directory on disk; **everything else (fetching, copying, metadata
    bookkeeping) is handled generically.**"""
```

**"安装一个东西"这件事被抽成了泛型**：抓取、复制、元数据记账全是通用的，
**只有"怎么从一个目录加载出 T"需要各自实现。**

于是技能和插件复用同一套安装/卸载/启用/禁用/更新的逻辑。

> 又一次"抽象层的价值和接口大小成反比"——**这次只有一个方法需要实现。**

### 来源类型的三档分类

```python
class SourceType(StrEnum):
    """LOCAL   -- a filesystem path (absolute, home-relative, or dot-relative).
    GIT     -- any git-clonable URL (HTTPS, SSH, git://, etc.).
    GITHUB  -- the ``github:owner/repo`` shorthand, expanded to an HTTPS URL."""
```

**`github:owner/repo` 是一个简写**，展开成 HTTPS URL。

**而且复用了第十八节那个带缓存的 git 克隆**：

```python
from openhands.sdk.git.cached_repo import GitHelper, try_cached_clone_or_update
from openhands.sdk.utils.redact import redact_url_credentials
```

**同一套"克隆到缓存 + 脱敏 URL 凭证"的基础设施**被技能加载（第十五节）、
插件安装、扩展安装三处复用。

> **可迁移的道理**：**"从某个来源取一份代码"应该是一个共享的基础设施**，
> 而不是每个子系统各写一遍——否则缓存策略、凭证脱敏、错误处理会不一致。

## 七、`io`：434 行的存储抽象

七个抽象方法：`write` / `read` / `list` / `delete` / `exists` /
`get_absolute_path` / **`lock`**。

三个实现：`LocalFileStore`（144 行）、`InMemoryFileStore`（99 行）、
以及一个带内存限制的 LRU 缓存（85 行）。

### `lock` 的文档里有一条很实在的警告

```python
@abstractmethod
@contextmanager
def lock(self, path: str, timeout: float = 30.0) -> Iterator[None]:
    """Acquire an exclusive lock for the given path.
    ...
    Note:
        **File-based locking (flock) does NOT work reliably on NFS mounts or
        network filesystems.**"""
```

**把"这个抽象在什么环境下不成立"写进接口文档。**

> **可迁移的道理**：**抽象的失效边界要写在抽象自己的文档里**，
> 而不是在某个实现里。因为用户看的是接口——**他需要知道"我能不能把存储放在 NFS 上"。**

### 缓存同时限制条数和内存

```python
class MemoryLRUCache(LRUCache):
    """This cache enforces **two limits**:
    1. Maximum number of entries (maxsize)
    2. **Maximum memory usage in bytes (max_memory)**

    When either limit is exceeded, the least recently used items are evicted."""
```

**为什么条数不够？** 因为文件大小差异巨大——
1000 个小文件和 1000 个 10MB 的文件占的内存差四个数量级。

而内存计算方式也解释了：

```python
def _get_size(self, value: Any) -> int:
    """For strings (the common case in FileStore), we use len() which gives accurate
    character count. **This is much more accurate than sys.getsizeof for our use
    case.** For other types, we use sys.getsizeof() as a rough approximation."""
    if isinstance(value, str):   return len(value)
    elif isinstance(value, bytes): return len(value)
    else: ... sys.getsizeof ...
```

**对字符串用 `len()` 而不是 `sys.getsizeof()`** ——注释说明了理由：
对这个用途（缓存文件内容）`len()` 更准。

> `sys.getsizeof` 算的是对象在内存里的字节数，包含 Python 的对象头开销，
> 而且对嵌套结构只算外层。**对"缓存的内容有多大"这个问题，字符数是更好的代理。**

还有一个边界处理：

```python
# Ensure minimum maxsize of 1 to avoid LRUCache issues
maxsize = max(1, max_size)
```

**`maxsize=0` 会让底层库出问题，所以下限钳到 1。**

> 和第九节那个"容量指标合并要取 max 不要累加"、
> 第二十节那个"截断标记要留位置"是同一类：**边界值的退化情况要显式处理。**

## 八、这一节的可迁移结论

1. **格式/协议的差异全部收敛在"读盘"这一步**，读完之后是统一的对象。
   **加一个新格式永远不需要碰下游的合并/应用代码。**

2. **策略链里"什么都接受"的那个必须排最后**，而且要在文档里说明它是兜底的——
   否则后来人加新格式时可能加在它后面，然后发现自己的永远不被选中。

3. **一个可扩展点的文档应该是"配方 + 约束"，而不只是"接口说明"**：
   哪些方法不该重写、注册在哪个位置、要补什么测试、detect 要多便宜。

4. **子类必须提供的类属性，用 `__init_subclass__` 在类定义时就强制**，
   否则会等到某次读取（可能在一条 debug 日志里）才崩。

5. **校验用的 schema 要随代码分发，不要从网上拉**：
   离线不能用、网络故障时校验失败、**远端改 schema 会让旧版本突然拒绝合法输入。**

6. **包的 `__init__` 不要 eager import 所有子模块**（延迟导入 + `__getattr__`）。

7. **软外键需要一整套生命周期管理**：查引用者、级联改名、删前检查。
   而且**删不掉时要在错误里列出引用者**，让用户知道先解开谁。

8. **两把锁的死锁只有一种防法：全局规定获取顺序，写在显眼处。**
   这里写在模块文档第一段。

9. **"扫描→修改"之间不能放锁**（TOCTOU）：检查完放锁再修改，
   中间可能有人新建了引用。

10. **允许清单 vs 禁用清单，取决于被引用的集合是不是完整且稳定的**：
    集合完整、用户手写 → 允许清单；**集合动态发现、会漂移 → 禁用清单
    （唯一不会悬空的选择）**。
    很多系统无脑用允许清单，然后在动态目录上反复遇到悬空引用。

11. **"检查配置完整性"只需要密钥的名字，不需要值**：
    把枚举和取值分开，配置编辑器就不需要任何解密权限。

12. **任何"应用配置"的操作都应该有一个"只检查不应用"的 dry-run 版本**。

13. **当框架的自动行为和你的需求冲突时，退到弱类型表示 + 提供显式的类型化访问器**，
    并把原因写进文档字符串——否则后来人一定会"顺手"改回去。

14. **两个包要共享的类型，放在依赖方向的下游（被依赖方）**，
    而不是新建第三个包。

15. **"从某个来源取一份代码"应该是共享的基础设施**：
    缓存策略、凭证脱敏、错误处理只写一遍。

16. **抽象的失效边界要写在抽象自己的文档里**（flock 在 NFS 上不可靠），
    因为用户看的是接口。

17. **缓存要同时限制条数和内存**：条目大小差异大时，只限条数等于没限。
    而且**内存估算要选和用途匹配的度量**（缓存文本用字符数而不是 `sys.getsizeof`）。

18. **边界值的退化情况要显式处理**：`maxsize=0` 钳到 1。

## 剩下还没读的

- `openhands-tools/` 剩下的：`glob`、`grep`、`gemini/`、`preset/`、`planning_file_editor`
- `openhands-agent-server/` 的细节：`event_service.py`、`persistence/store.py`、
  `file_router.py`、`docker/build.py`
- `acp_file_credentials.py`（536 行）
- `settings/model.py`（2477 行）的具体字段

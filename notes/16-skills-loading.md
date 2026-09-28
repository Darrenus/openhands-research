# 精读 15：技能的加载与市场机制（skills/，2952 行）

> 位置：`upstream/software-agent-sdk/openhands-sdk/openhands/sdk/skills/`
> 文件：`skill.py`（1533 行）、`utils.py`（678 行）、`installed.py`（276 行）、
> `execute.py`、`fetch.py`、`trigger.py`；另 `utils/path.py` 的 `resolves_within`
> 前置：`notes/09-context-engineering.md`（技能怎么被用、渐进式披露、路径规则）

## 这一节在讲什么

第八节讲了技能**怎么被用**（目录 + 按需取全文 + 三种触发方式）。
这一节讲技能**从哪来**。

问题比想象的复杂：技能可以来自**五个地方**（公共仓库、用户目录、项目目录、
已安装目录、第三方指令文件），同名了谁赢？远程仓库多久拉一次？
恶意技能能不能用符号链接读到仓库外的文件？

## 一、优先级：三层，晚的覆盖早的

```python
def load_available_skills(work_dir=None, *, include_user=False,
                          include_project=False, include_public=False,
                          marketplace_path=DEFAULT_MARKETPLACE_PATH) -> dict[str, Skill]:
    """Precedence (later overrides earlier via dict updates):
        public (lowest) → user → project (highest)
    """
    available: dict[str, Skill] = {}

    if include_public:
        try:
            for s in load_public_skills(marketplace_path=marketplace_path):
                available[s.name] = s
        except Exception as e:
            logger.warning(f"Failed to load public skills: {e}")

    if include_user:
        try:
            for s in load_user_skills():
                available[s.name] = s
        except Exception as e:
            logger.warning(f"Failed to load user skills: {e}")

    if include_project and work_dir:
        try:
            for s in load_project_skills(work_dir):
                available[s.name] = s
        except Exception as e:
            logger.warning(f"Failed to load project skills: {e}")

    return available
```

**优先级：项目 > 用户 > 公共。** 实现方式极其简单——**按优先级从低到高依次
往同一个字典里塞，后面的自然覆盖前面的。**

> 不需要比较优先级数字、不需要排序，**把"优先级"编码成"写入顺序"**。
> 这是最不容易写错的实现方式。

三个来源**各自 try/except**。一个来源挂了（公共仓库拉不下来）不影响另外两个。

> **可迁移的道理**：**多来源加载时，每个来源独立容错。** 否则一个不可用的可选来源
> 会让整个功能失效。

注意 `include_*` 三个开关**默认全是 `False`**。**默认不加载任何技能**——
这是个保守的默认值（尤其"公共"意味着要访问网络和克隆仓库）。

## 二、另一个合并函数，语义刚好相反

```python
def merge_skills_by_name(primary, secondary) -> list[Skill]:
    """``primary`` skills are authoritative: they take precedence on name conflicts
    and keep their order. Each ``secondary`` skill is appended only when its name
    is not already provided by ``primary``."""
    merged = list(primary)
    seen = {skill.name for skill in merged}
    for skill in secondary:
        if skill.name not in seen:
            seen.add(skill.name)
            merged.append(skill)
    return merged
```

**这里是"先来的赢"**（primary 优先，secondary 只补缺），
而上面 `load_available_skills` 是"后来的赢"。

**两个函数语义相反，但各自的文档都写清了。** 用途不同：
- `load_available_skills`：按优先级层叠，**高优先级最后写**
- `merge_skills_by_name`：已经确定 primary 是权威的，**只用 secondary 补空缺**

> 这种地方最容易出 bug。判断标准：**函数名 + 文档是否让语义无歧义。**
> `merge_skills_by_name(primary, secondary)` 的参数名本身就说明了谁赢。

## 三、公共技能：一个本地 git 克隆 + 分层缓存

```python
def load_public_skills(repo_url=PUBLIC_SKILLS_REPO, ref=PUBLIC_SKILLS_REF,
                       marketplace_path=DEFAULT_MARKETPLACE_PATH) -> list[Skill]:
    """This function maintains a local git clone of the public skills registry at
    https://github.com/OpenHands/extensions. On first run, it clones the repository
    to ~/.openhands/skills-cache/. On subsequent runs within the same process, it
    returns cached results. For branch refs it re-fetches after the cache TTL; for
    tags and commit SHAs (immutable refs) the cache never expires so no further
    network calls are made."""
```

### 最漂亮的设计：按"引用是否可变"决定缓存策略

```python
_PUBLIC_SKILLS_CACHE_TTL_SECONDS = 60.0

is_pinned = is_skills_repo_pinned(repo_path)
...
if all_skills:
    timestamp = float("inf") if is_pinned else time.monotonic()
    with _PUBLIC_SKILLS_CACHE_LOCK:
        _PUBLIC_SKILLS_CACHE[cache_key] = (timestamp, list(all_skills))
```

**`float("inf")` 当时间戳 = 永不过期。**

判断"是否固定"的方法很巧：

```python
def is_skills_repo_pinned(repo_path: Path) -> bool:
    """A pinned ref is one that cannot change over time — a tag or a specific commit
    SHA. After checking out such a ref the repository is left in detached HEAD state,
    which is the signal used here.

    Returns False on any git error so callers can safely treat the result as ``False``
    (i.e., keep polling) when the state cannot be determined."""
```

> **游离 HEAD（detached HEAD）**：git 的一种状态——你检出的是一个具体提交或标签，
> 而不是一个分支。因为不在任何分支上，HEAD 就"游离"了。

**逻辑**：
- 指向分支（`main`）→ 内容会变 → 60 秒 TTL，之后重新拉取
- 指向标签或提交 SHA → 内容永不变 → **缓存永久有效，零网络请求**

而且出错时返回 `False`（= 继续轮询）——**保守的一侧是"多拉几次"，
而不是"永久缓存一个可能过期的结果"。**

> **可迁移的道理**：**缓存策略应该由"数据能不能变"决定，而不是统一给一个 TTL。**
> 内容寻址的引用（哈希、标签、带 digest 的镜像）可以永久缓存；
> 可变引用（分支、`latest` 标签）必须定期检查。
>
> 这在很多地方适用：Docker 镜像（`:latest` vs `@sha256:...`）、
> 依赖锁文件、CDN 资源的文件名哈希。

### 只缓存非空结果

```python
# Only cache non-empty results so transient errors don't poison the cache
# for the full TTL window.
if all_skills:
```

**如果这次因为网络问题加载到 0 个技能，不要缓存这个空结果。**

否则一次瞬时故障会导致接下来 60 秒里技能完全消失，
而**真正的问题早就恢复了**。

> **可迁移的道理**：**不要缓存失败的结果**（或者给失败一个远小于成功的 TTL）。
> 这类"缓存了一个空值然后一直返回空"的 bug 排查起来非常烦人。

### 缓存键是三元组，而且有锁

```python
_PUBLIC_SKILLS_CACHE: dict[tuple[str, str, str | None], tuple[float, list["Skill"]]] = {}
_PUBLIC_SKILLS_CACHE_LOCK = threading.Lock()

cache_key = (repo_url, ref, marketplace_path)
```

**（仓库地址、引用、市场清单路径）三个都参与缓存键。** 换了任何一个都是不同的结果集。

模块级全局字典 + 一把锁——因为可能有多个会话并发加载。
读的时候还做了拷贝（`return list(cached[1])`），**防止调用方改到缓存里的列表**。

### 强制刷新时主动失效

```python
def _invalidate_public_skills_cache() -> None:
    """Called by ``sync_public_skills`` so a forced refresh re-parses immediately
    instead of waiting for the TTL."""
```

**有"立刻刷新"的入口时，必须能主动清缓存**，否则用户点了刷新还要等 60 秒。

## 四、市场清单：白名单，而不是全量

```python
def load_marketplace_skill_names(repo_path: Path, marketplace_path: str) -> set[str] | None:
    """Load the list of skill names from a marketplace manifest file."""
```

**默认只加载 `marketplaces/default.json` 里列出的技能**，而不是仓库里所有技能。

**为什么？** 因为公共仓库会不断增长。如果全量加载，那么第八节讲的那个
`<available_skills>` 目录会越来越长——**每次请求都为几百个技能的描述付钱，
而且稀释模型的注意力。**

**市场清单是一层人工策展。** 和第九节那个"验证过的模型列表不是供应商目录"
是同一种纪律。

### 两种失败的不同处理

```python
marketplace_skill_names = load_marketplace_skill_names(repo_path, marketplace_path)
if (marketplace_skill_names is None
        and marketplace_path != DEFAULT_MARKETPLACE_PATH):
    logger.warning("Configured marketplace path could not be loaded: %s", marketplace_path)
    return all_skills      # ← 用户指定的清单读不到 → 什么都不加载
```

**用户明确配了一个清单路径但读不到 → 返回空，并打警告。**
**默认清单读不到 → 降级成"加载全部"。**

> 这个区分很讲究：**用户明确指定的东西失败了，不能悄悄换成别的行为**
> ——那会让用户以为自己的配置生效了。而默认值失败可以降级。

### 查找技能文件时兼容新旧两种格式

```python
for skill_name in marketplace_skill_names:
    skill_md = skills_dir / skill_name / "SKILL.md"       # 新格式：目录/SKILL.md
    if skill_md.exists():
        all_skill_files.append(skill_md); continue

    legacy_md = skills_dir / f"{skill_name}.md"            # 旧格式：名字.md
    if legacy_md.exists():
        all_skill_files.append(legacy_md); continue

    logger.debug("Skill '%s' from marketplace '%s' not found in skills dir", ...)
```

**清单里有但仓库里找不到 → 只打 debug 日志，跳过。**

不报错的理由：清单和仓库可能不同步（清单更新了、代码还没拉到新版本）。
**一个缺失的技能不该让整个加载失败。**

### 一条重要的语义规定

```python
"""Note: When a skill directory contains a SKILL.md file (AgentSkills format), any
other markdown files in that directory or its subdirectories are treated as reference
materials for that skill, NOT as separate skills."""
```

代码实现：

```python
skill_md_files = find_skill_md_directories(skills_dir)
skill_md_dirs = {skill_md.parent for skill_md in skill_md_files}
regular_md_files = find_regular_md_files(skills_dir, skill_md_dirs)   # ← 排除上面那些目录
```

**一个技能目录里的其他 .md 文件是它的参考资料，不是独立技能。**

这解决了一个实际问题：一个复杂技能可能有 `SKILL.md` + `reference/api.md` +
`examples/basic.md`。如果全都当成技能，目录里会出现一堆碎片。

## 五、安全：符号链接不能逃出仓库

这是这一节最重要的安全设计。

```python
def find_skill_md_directories(skill_dir: Path, root: Path | None = None) -> list[Path]:
    """Args:
        root: If given, directories and SKILL.md files that resolve outside it are
            skipped, and an escaping directory is never listed."""
    for subdir in sorted(skill_dir.iterdir()):
        if root is not None and not resolves_within(subdir, root):
            continue
        if subdir.is_dir():
            skill_md = find_skill_md(subdir)
            if skill_md and (root is None or resolves_within(skill_md, root)):
                results.append(skill_md)
```

**注意检查了两次**：目录本身在不在仓库内、找到的 `SKILL.md` 在不在仓库内。

守卫函数在 `utils/path.py`：

```python
def resolves_within(path: Path, root: Path) -> bool:
    """Whether ``path`` resolves inside the resolved ``root``.

    Symlinks may point anywhere under ``root`` but not outside it. A path that
    cannot be resolved (a symlink loop raises on Python 3.12) counts as outside.
    """
    with suppress(OSError, RuntimeError):
        if path.resolve().is_relative_to(root.resolve()):
            return True
    logger.warning(f"Denying {path}: it resolves outside {root}")
    return False
```

**四个细节全部到位：**

**① `.resolve()` 两边都调用。** 只 resolve 一边是不够的——
`root` 自己也可能是个符号链接。

**② 符号链接循环会抛异常，被当作"在外面"处理。**

> Python 3.12 起，符号链接成环时 `resolve()` 会抛 `RuntimeError`（此前是静默返回）。
> 注释明确提到了这个版本行为。

**③ 异常 = 拒绝。** `with suppress(...)` 包住的是"返回 True"那一路，
**任何异常都会走到最后的 `return False`。** 这是第六节那个"失败关闭"的又一次应用。

**④ 拒绝时打警告。** 不是静默跳过——**被拒绝的路径是可观测的**。

> **这在防什么？** 公共技能仓库是从网上克隆下来的。如果里面有一个
> `evil/SKILL.md -> /Users/you/.ssh/id_rsa` 这样的符号链接，
> 那么加载技能就等于把你的私钥读进了模型上下文。
>
> **可迁移的道理**：**任何"扫描一个目录树"的代码，如果这个目录树来自外部，
> 都必须做符号链接逃逸检查。** 而且检查要在 resolve 之后做，
> 并把异常算作"不安全"。

## 六、SKILL.md 的解析：两种格式

```python
def load(cls, path, skill_base_dir=None, strict=True, skip_mcp=False) -> "Skill":
    if path.name.lower() == "skill.md":
        return cls._load_agentskills_skill(path, file_content, strict=strict, skip_mcp=skip_mcp)
    else:
        return cls._load_legacy_openhands_skill(path, file_content, skill_base_dir)
```

**按文件名分派**：`SKILL.md` 走新的 AgentSkills 标准，其他 `.md` 走旧的 OpenHands 格式。

> **前置元数据（frontmatter）**：Markdown 文件开头用 `---` 包起来的一段 YAML，
> 存放这个文件的元数据（名字、描述、触发条件等）。

### 名字的来源和校验

```python
# For SKILL.md files, use parent directory name as the skill name
directory_name = path.parent.name
...
agent_name = str(metadata_dict.get("name", directory_name))

if strict:
    name_errors = validate_skill_name(agent_name, directory_name)
    if name_errors:
        raise SkillValidationError(f"Invalid skill name '{agent_name}': {...}")
```

校验规则（`validate_skill_name`）：

```python
if len(name) > 64:
    errors.append(f"Name exceeds 64 characters: {len(name)}")
if not SKILL_NAME_PATTERN.match(name):
    errors.append("Name must be lowercase alphanumeric with single hyphens "
                  "(e.g., 'my-skill', 'pdf-tools')")
if directory_name and name != directory_name:
    errors.append(f"Name '{name}' does not match directory '{directory_name}'")
```

**第三条最有意思：前置元数据里写的名字必须和目录名一致。**

为什么要这个冗余约束？因为**两个地方都能看到名字**——
文件系统上是目录名，元数据里是 `name` 字段。如果允许不一致，
那么"这个技能叫什么"就有两个答案，引用时会混乱。
**强制一致，让文件系统结构本身就是可信的索引。**

而且校验可以关掉（`strict=False`），文档说明了用途：
`allow relaxed naming (e.g., for plugin compatibility)`。
**有明确理由的宽松模式，而不是默认宽松。**

### 两种格式的 MCP 配置来源不同

```python
# AgentSkills 格式：只从 .mcp.json 读
# Load MCP configuration from .mcp.json (agent_skills ONLY use .mcp.json)
if not skip_mcp:
    mcp_json_path = find_mcp_config(skill_root)
    ...

# 旧格式：只从前置元数据读
# Legacy skills ONLY use mcp_tools from frontmatter (not .mcp.json)
mcp_tools = metadata_dict.get("mcp_tools")
if mcp_tools is not None and not isinstance(mcp_tools, dict):
    raise SkillValidationError("mcp_tools must be a dictionary or None")
```

**两个 `ONLY` 都是大写的。** 不是"两处都看、哪个有用哪个"，而是**每种格式只有一个
合法来源**。避免了"我配了但没生效"这类问题。

`skip_mcp` 参数的用途也写了：加载插件的根技能时，MCP 配置已经在插件层加载过了，
**避免重复加载**。

### 路径规则的宽容解析

```python
@staticmethod
def _parse_paths(v: object) -> list[str] | None:
    """Parse ``paths`` frontmatter (comma-separated string or list) into a
    non-empty list of globs, or None. Mirrors Claude Code / AgentSkills."""
    if isinstance(v, str):
        v = v.split(",")
    elif not isinstance(v, list):
        raise SkillValidationError("paths must be a string or list")
    return [s for p in v if (s := str(p).strip())] or None
```

**既接受 `paths: "src/**, tests/**"`（逗号分隔的字符串）
也接受 YAML 列表。** 注释说这是**对齐 Claude Code / AgentSkills 的行为**。

`or None` 那一下：**过滤完全是空白的项之后如果空了，返回 None 而不是空列表。**
于是"配了但全是空"和"没配"表现一致。

## 七、用户和项目技能：多目录 + 遗留路径

### 用户级：三个目录，有优先级

```python
"""Searches for skills in ~/.agents/skills/, ~/.openhands/skills/, and
~/.openhands/microagents/ (legacy). Skills from all directories are merged, with
earlier entries in USER_SKILLS_DIRS taking precedence for duplicate names.

Also loads enabled installed skills from ~/.openhands/skills/installed/ (managed via
install_skill/uninstall_skill). Installed skills have lower precedence than user
skills from the directories above."""
```

**四个来源，优先级从高到低：**
1. `~/.agents/skills/`（新的标准位置，跨工具通用）
2. `~/.openhands/skills/`（OpenHands 自己的旧位置）
3. `~/.openhands/microagents/`（更老的，标了 legacy）
4. `~/.openhands/skills/installed/`（通过安装命令装的）

**注意"安装的"优先级最低。** 手写的技能可以覆盖装来的技能——
**用户自己写的东西永远赢。**

而且注意这里用的是 `seen_names` 集合 + 先到先得（**和公共技能那边的
"后到覆盖"相反**）。因为这里是**从高优先级往低优先级遍历**。

> 两种实现方式都对，但混在一个文件里读代码时要小心。
> 判断方法：**看遍历顺序是从高到低还是从低到高。**

### 项目级：还会往上找 git 仓库根

```python
"""If the working directory is inside a Git repository, this function also loads
skills from the Git repo root, so running from a subdirectory still picks up
repo-level guidance (e.g., AGENTS.md).

Skills are merged in priority order, with the *working directory* taking precedence
over the Git repo root when duplicates exist."""
```

**场景**：你在 `myrepo/backend/` 下启动 agent，但 `AGENTS.md` 在 `myrepo/` 根目录。
不往上找就读不到仓库级的规范。

找 git 根的方式很务实：

```python
def _find_git_repo_root(path: Path) -> Path | None:
    """We intentionally don't shell out to `git`, so this works even when git isn't
    installed. A directory is considered a git root if it contains a `.git` entry
    (directory *or* file, to support worktrees/submodules)."""
    for candidate in (path, *path.parents):
        if (candidate / ".git").exists():
            return candidate
    return None
```

**不调用 git 命令**，直接看有没有 `.git`。两个理由都写了：
**① git 可能没装**；**② `.git` 可能是文件而不是目录**（工作树和子模块的情况）。

> **这是很好的依赖削减**：一个只需要"找仓库根"的功能，不应该依赖 git 可执行文件存在。

### 第三方指令文件也当技能加载

```python
for root in search_roots:
    third_party_files = find_third_party_files(root, Skill.PATH_TO_THIRD_PARTY_SKILL_NAME)
```

`find_third_party_files` 的文档：

> Searches for files like .cursorrules, AGENTS.md, CLAUDE.md, etc. with
> case-insensitive matching.
>
> **Resolves symlinks so that e.g. ``CLAUDE.md -> AGENTS.md`` is detected as a
> duplicate and only the canonical (non-symlink) file is returned.**

**把其他 AI 工具的指令文件（`.cursorrules`、`CLAUDE.md`、`AGENTS.md`）
直接当技能读进来。**

而且处理了一个很现实的情况：**很多仓库会做 `CLAUDE.md -> AGENTS.md` 的符号链接**
（同一份内容给不同工具用）。**解析符号链接后发现是同一个文件，只保留规范的那个**，
避免同样的内容被加载两遍。

> 这个细节说明他们真的看过大量真实仓库的组织方式。

### 嵌套的目录级规则

```python
# Load nested third-party files (e.g. server/AGENTS.md) as directory-scoped
# path rules, keyed off work_dir (the matcher's base).
for path, rel_dir in find_nested_third_party_files(work_dir, ...):
    rule = Skill._handle_nested_third_party(path, content, rel_dir)
```

**子目录里的 `AGENTS.md` 会被转成第八节讲的那种"路径规则"**——
只在 AI 碰到该目录下的文件时才注入。

于是一个 monorepo 可以这样组织：

```
AGENTS.md              → 全局规范，一直生效
server/AGENTS.md       → 改 server/ 下的文件时才出现
frontend/AGENTS.md     → 改 frontend/ 下的文件时才出现
```

**这个映射很漂亮**：把"文件放在哪个目录"自动翻译成"什么时候注入这条规则"。
用户不需要学任何配置语法——**目录结构就是作用域声明。**

> 回想第八节那条结论："注入时机比注入内容更重要"。这里是它的自动化版本：
> **让文件系统的位置决定规则的作用范围。**

## 八、这一节的可迁移结论

1. **把"优先级"编码成"写入顺序"**：按优先级从低到高依次往同一个字典塞，
   后写的自然覆盖。不需要比较优先级数字，最不容易写错。

2. **多来源加载时每个来源独立容错**：一个不可用的可选来源不该让整个功能失效。

3. **语义相反的合并函数要靠参数名和文档消除歧义**：
   `merge_skills_by_name(primary, secondary)` 一看就知道谁赢。
   读代码时判断方法是**看遍历顺序是从高优先级到低还是反过来**。

4. **缓存策略由"数据能不能变"决定，而不是统一给一个 TTL**：
   分支引用 60 秒过期；标签和提交 SHA **永不过期**（用 `float("inf")` 当时间戳）。
   判断方式是检查 git 的游离 HEAD 状态。

5. **不要缓存失败/空的结果**：一次瞬时故障会让功能在整个 TTL 窗口内消失，
   而真正的问题早就恢复了。

6. **有"强制刷新"入口时必须能主动清缓存**，否则用户点了刷新还要等 TTL。

7. **用一层人工策展的清单挡住来源的无限增长**：默认只加载市场清单里的技能，
   而不是仓库里所有技能——否则目录会越来越长、每次请求都为它付钱。
   （和第九节"验证过的模型列表不是供应商目录"同一种纪律。）

8. **用户明确指定的配置失败了不能悄悄降级**：指定的清单读不到就返回空 + 警告；
   只有默认值失败才降级。

9. **任何扫描外部目录树的代码都必须做符号链接逃逸检查**：
   两边都 `resolve()`、符号链接成环算"在外面"、**任何异常都算不安全**、
   拒绝时要打警告。一个 `evil/SKILL.md -> ~/.ssh/id_rsa` 就能把私钥读进模型上下文。

10. **同一份信息出现在两个地方时，强制它们一致**：技能名必须和目录名相同，
    否则"这个技能叫什么"就有两个答案。宽松模式要有明确理由且非默认。

11. **每种格式只认一个配置来源**（两处 `ONLY` 都大写）：
    避免"我配了但没生效"这类问题。

12. **不要为了一个小功能依赖外部可执行文件**：找 git 仓库根只看有没有 `.git`，
    而且要考虑它可能是文件（工作树/子模块）而不是目录。

13. **解析符号链接来做去重**：`CLAUDE.md -> AGENTS.md` 这种真实存在的组织方式，
    要识别成同一个文件而不是加载两遍。

14. **让文件系统的位置决定规则的作用范围**：子目录里的 `AGENTS.md` 自动变成
    只在该目录生效的路径规则。**用户不需要学配置语法，目录结构就是作用域声明。**

## 剩下还没读的

- `agent/acp_agent.py`（4681 行，驱动 Claude Code / Codex / Gemini 等外部 agent 的协议适配层）
- `openhands-agent-server/` 的具体路由实现（29851 行，只看了入口装配）
- `openhands-tools/` 剩下的工具：`browser_use`、`apply_patch`、`task_tracker`、`workflow`
- `plugin/`、`marketplace/`、`profiles/`、`secret/`、`git/`、`observability/`

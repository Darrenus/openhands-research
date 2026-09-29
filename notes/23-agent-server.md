# 精读 22：Agent Server 的服务层（29851 行里的核心几块）

> 位置：`upstream/software-agent-sdk/openhands-agent-server/openhands/agent_server/`
> 文件：`conversation_service.py`（3031 行）、`event_service.py`（2021 行）、
> `conversation_lease.py`、`pub_sub.py`、`sockets.py`（595 行）、
> `models.py`、`persistence/store.py`（990 行）
> 前置：`notes/12-workspace-and-server.md`（入口装配）、
> `notes/05-run-loop.md`（本地会话）、`notes/21-task-manager.md`（生命周期管理）

## 这一节在讲什么

第十一节只读了服务端的**入口装配**（FastAPI 怎么挂路由、三个认证分组、延迟初始化）。
这一节读**服务层本体**：把一个本地库变成多实例、多客户端的服务，
需要解决哪些本地版本不存在的问题。

**四个核心问题：**

1. 多个服务实例可能访问同一个会话目录 —— 怎么防止两个进程同时写坏一个会话
2. 会话目录里的元数据被反复读 —— 怎么不每次都解析一个巨大的状态文件
3. 客户端断线重连 —— 怎么把断线期间的事件补上
4. 内存里挂着几百个会话 —— 什么时候可以释放

## 一、租约：用文件系统做分布式锁

```python
class ConversationLease:
    """Coordinate conversation ownership across multiple service instances.

    The lease file stores the active owner, a **monotonically increasing
    generation**, and an expiry timestamp so **stale owners can be fenced off
    after a takeover**."""
```

> **租约（lease）**：一个带过期时间的所有权声明。持有者要定期续租，
> 不续就自动失效，于是别人可以接手。分布式系统里防"两个节点都以为自己是主"的标准手法。

### 三个字段各管一件事

```python
class LeasePayload(TypedDict):
    owner_instance_id: str
    generation: int
    expires_at: float
    # Optional fields added for crash-recovery. They are absent in lease files
    # written by older versions of the agent server, so consumers must treat
    # them as optional.
    owner_host: NotRequired[str]
    owner_pid: NotRequired[int]
```

- `owner_instance_id` —— 谁持有
- `expires_at` —— 什么时候过期（默认 TTL 45 秒）
- **`generation`** —— 单调递增的"代数"

**`generation` 是防"僵尸写入"的关键**（就是注释里那个 fencing）。

**场景**：实例 A 持有租约，但它卡住了（GC 停顿、磁盘慢）。租约过期，实例 B 接手并把
generation 从 3 加到 4。**然后 A 苏醒了，以为自己还是主，要写一个文件。**

**如果只有 TTL，A 的写入会成功——数据就坏了。** 有了 generation，
A 写之前会发现自己的代数是 3 而当前是 4，**于是知道自己已经被取代了**：

```python
class ConversationOwnershipLostError(RuntimeError):
    ...
    super().__init__("conversation ownership was lost before the write completed")
```

> **这就是分布式系统里的"围栏令牌"（fencing token）。**
> **可迁移的道理**：**光有过期时间不够，必须有一个单调递增的代数**，
> 让一个苏醒的旧持有者能发现自己已经过期。
> 只靠 TTL 的租约在"持有者暂停后恢复"这个场景下是不安全的。

### 崩溃恢复：记 host 和 pid，但只在同机时用

```python
def _owner_is_dead(self, payload: LeasePayload) -> bool:
    """Return True if the lease's recorded owner process is gone.

    **Only considered when the recorded ``owner_host`` matches this host:
    liveness checks for PIDs on other hosts are meaningless.** Lease files
    written by older agent-server versions don't include host/pid, so this
    returns False (preserving the legacy TTL-only behavior) for them."""
```

**如果租约的持有者进程已经死了，不用等 45 秒就能接手。**

但判断"死了没"只在**同一台机器上**才有意义——`pid 12345` 在另一台机器上是别的进程。
所以先比对 host。

而 liveness 检查本身极其保守：

```python
def _is_pid_alive(pid: int) -> bool:
    """Uses ``os.kill(pid, 0)`` which is portable across POSIX platforms and
    available on Windows since Python 3.2. **When liveness cannot be determined
    (permission errors, unsupported platforms, etc.) we conservatively report
    the process as alive so we never steal a lease that might still be in use.**"""
    if pid <= 0:
        return False
    try:
        os.kill(pid, 0)
    except ProcessLookupError:
        return False              # 确定死了
    except PermissionError:
        # Process exists but is owned by another user.
        return True               # 活着（只是不是我的）
    except OSError:
        # Unknown error - be conservative and assume the process is alive.
        return True               # 不知道 → 当活着
    return True
```

> `os.kill(pid, 0)` 是 POSIX 的惯用法：**信号 0 不发送任何信号，只做权限和存在性检查。**

**三种异常三种判断，而且"不确定"一律算活着。**

> **这是第六节那条"失败关闭"在分布式场景的应用**：
> 不确定对方是否还活着时，**宁可多等 45 秒，也不要抢一个可能还在用的租约。**
> 抢错了的后果是两个进程同时写——比多等一会儿严重得多。

### 老版本兼容

`owner_host` / `owner_pid` 是**后加的可选字段**，注释明确说：
老版本写的租约文件里没有，所以要**退化成"只看 TTL"的旧行为**。

> 回想第十一节那个 MCP 版本兼容、第二十节那个废弃参数——
> **这是第三次看到"存档/协议的向后兼容"被明确处理。**
> 分布式系统里这尤其重要：**滚动升级期间新旧版本必然同时在跑。**

## 二、缓存会话信息：一个双重签名的失效机制

```python
@dataclass
class _ConversationRecord:
    stored: StoredConversation
    execution_status: ConversationExecutionStatus
    # Signature when execution_status was last read. None = unverified, re-read.
    state_signature: tuple[int, int] | None = None
    # Memoised by _base_state_path.
    base_state_path: str | None = None
    # Full ConversationInfo composed from the persisted state at
    # ``state_signature``. **Sidebar polling repeatedly asks for the same rows;
    # keep the validated object until base_state.json changes rather than
    # reparsing a large nested ConversationState on every request.**
    cached_info: ConversationInfo | None = None
    # Fingerprint of ``stored`` metadata (title, metrics, …) as of the last
    # composition. ``cached_info`` is keyed by ``state_signature`` alone, so
    # **metadata-only updates (``meta.json``, e.g. auto-title) must also
    # invalidate it.**
    stored_signature: int | None = None
```

**问题**：界面的侧边栏在轮询会话列表。每次都要返回每个会话的状态、标题、花费……
**而这些信息在一个可能很大的嵌套 JSON（`base_state.json`）里。**
每次请求都重新解析一遍非常贵。

**解法**：缓存解析结果，用**两个签名**判断是否失效。

### 为什么需要两个签名

`state_signature` 是 `tuple[int, int]` —— **大概是文件的 (mtime, size)**，
用来判断 `base_state.json` 变了没。

**但注释指出了一个坑**：`cached_info` 只按 `state_signature` 做键，
**而标题、统计这些元数据存在另一个文件 `meta.json` 里**。

**于是自动生成标题这种"只改元数据不改状态"的操作，不会改变 `state_signature`,
缓存就不会失效——界面上标题永远不更新。**

**所以要第二个签名** `stored_signature` 来跟踪元数据的变化。

> **可迁移的道理**：**缓存的失效条件必须覆盖所有影响缓存内容的数据源。**
> 这个 bug 的形态很典型：缓存的内容来自 A 和 B 两个来源，
> 但失效只检查了 A。**而且很难发现——因为大部分时候 A 和 B 会一起变。**
>
> 判断方法：**列出缓存内容的每一个字段，问它从哪来，确保每个来源都在失效条件里。**

### `None` 是第三种状态

```python
# Signature when execution_status was last read. **None = unverified, re-read.**
state_signature: tuple[int, int] | None = None
```

**`None` 表示"还没验证过，必须重读"** ——不是"没有签名"，
而是一个明确的"缓存不可信"信号。

> 又一次"未知必须是一个独立状态"（这是**第八次**了）。

## 三、启动只读元数据，用到才加载

```python
@dataclass
class ConversationService:
    """Manage persisted conversations and their live runtimes.

    **Startup loads only lightweight metadata. An ``EventService`` and its event
    history are hydrated when a conversation needs a live runtime.**"""
```

```python
async def __aenter__(self):
    self.conversations_dir.mkdir(parents=True, exist_ok=True)
    self._run_executor = ThreadPoolExecutor(
        max_workers=self.max_concurrent_runs,
        thread_name_prefix="conversation-run",
    )
    self._event_services = {}
    self._conversation_records = await asyncio.to_thread(self._load_catalog_sync)
```

**启动时只扫 `meta.json`（小文件），不读 `base_state.json`（可能几 MB）。**

```python
def _load_catalog_sync(self, conversation_id: UUID | None = None):
    for conversation_dir in directories:
        meta_file = conversation_dir / "meta.json"
        if not meta_file.exists():
            continue                    # ← 没有元数据文件就跳过，不报错
```

**没有 `meta.json` 的目录直接跳过** ——可能是残留的临时目录、或者正在创建中的会话。
**启动不能因为一个坏目录就失败。**

> 回想第十五节技能加载、第二十一节 agent 定义加载——**"扫描目录时单个失败不致命"
> 这是第三次出现。**

注意扫描是 `await asyncio.to_thread(...)` —— **文件 I/O 放到线程里，不阻塞事件循环。**
这个模式在这个文件里到处都是（下面还会看到）。

### 线程池的命名

```python
thread_name_prefix="conversation-run",
```

> 和第十三节委派工具那个 `name=f"Task-{agent_id}"` 同一个道理：
> **给线程起有意义的名字，调试时能在堆栈里认出来。**

`max_concurrent_runs: int = 10` —— **线程池大小就是并发上限。**
用线程池的容量来做限流，而不是另外写一个信号量。

## 四、四层锁：并发控制的分层

```python
_lifecycle_lock: asyncio.Lock                              # 生命周期操作的互斥
_lifecycle_condition: asyncio.Condition                    # 等待条件
_active_lifecycle_operations: int                          # 正在进行的操作计数
_exclusive_lifecycle_pending: bool                         # 有独占操作在等
_conversation_locks: WeakValueDictionary[UUID, asyncio.Lock]  # 每会话一把锁
```

**前四个字段合起来实现的是一个"读写锁"**：

- `_active_lifecycle_operations` 数着有多少个普通操作在跑
- `_exclusive_lifecycle_pending` 标记有独占操作（比如关服务）在等
- 新的普通操作看到这个标记就不再进入，等已有的排空

**第五个是每会话一把锁**，用 `WeakValueDictionary`：

> **弱值字典（WeakValueDictionary）**：值不被这个字典"持有"——
> 没有别的地方引用那把锁时，它会被自动回收，字典里的条目也自动消失。

**为什么用弱引用？** 因为会话可能有成千上万个。**如果用普通字典，
每个访问过的会话都会永久留下一把锁对象**——内存慢慢泄漏。

**弱引用让"当前没人在用的锁"自动消失。**

> **可迁移的道理**：**按 id 分配的锁（或任何按键缓存的辅助对象）要用弱引用容器**，
> 否则键的基数无界时会内存泄漏。

## 五、租约续期：一个循环管所有会话

```python
async def _renew_all_leases_loop(self) -> None:
    """**Single background task that renews leases for all active conversations.
    Replaces N per-conversation renewal tasks with one centralized loop,
    reducing asyncio task overhead.** Each renewal involves synchronous file I/O
    (FileLock + read + write), so individual calls are **offloaded via
    ``asyncio.to_thread`` to avoid blocking the event loop.**"""
    while True:
        await asyncio.sleep(LEASE_RENEW_INTERVAL_SECONDS)
        ...
        await asyncio.to_thread(event_service.renew_lease)
```

**从"每个会话一个续期任务"改成"一个循环续所有会话的租约"。**

**为什么要改？** asyncio 任务不是免费的——每个任务有栈、有调度开销。
**几百个会话就是几百个几乎什么都不做的任务。**

而每次续期是**同步文件 I/O**（文件锁 + 读 + 写），所以要 `to_thread`
——**否则一次慢磁盘操作会卡住整个事件循环**（进而卡住所有 HTTP 请求）。

> **可迁移的道理**：**"每个对象一个定时任务"在对象数量大时是反模式**，
> 改成"一个定时任务遍历所有对象"。而循环体里的阻塞操作要挪到线程里。

还有一个外部续期的开关：

```python
# _renew_all_leases_loop task on ConversationService.
event_service._external_lease_renewal = True
```

**有些场景下租约由别人续**（比如一个外部编排器），这时要关掉自己的续期，
**否则两边都在续，generation 会乱。**

## 六、发布订阅：谁在监听事件

```python
class Subscriber[T](ABC):
    # **Deltas arrive at token rate; consumers opt in rather than inherit them.**
    receives_streaming_deltas: ClassVar[bool] = False

    @abstractmethod
    async def __call__(self, event: T): ...
    async def close(self): ...
```

**这个类变量的注释一句话说清了一个重要设计**：
流式增量事件（每个 token 一个）**默认不发给订阅者**，要显式 opt in。

**为什么？** 一次对话可能有几十万个 token 事件。
**如果默认全发，那些只关心"任务完成了吗"的订阅者（比如 webhook）会被淹没**,
而且每个事件都要序列化一遍。

> **可迁移的道理**：**高频事件默认不投递，让订阅者显式声明要不要。**
> 反面（默认全发 + 让订阅者自己过滤）会让每个订阅者都付出序列化和调度的代价。

### 订阅者数量有上限

```python
class MaxSubscribersError(Exception):
    """Raised when a PubSub instance has reached its subscriber limit."""

def subscribe(self, subscriber: Subscriber[T]) -> UUID:
    if (self.max_subscribers is not None
            and len(self._subscribers) >= self.max_subscribers):
        raise MaxSubscribersError(f"Subscriber limit reached ({self.max_subscribers})")
```

**防的是：一个 bug 或者恶意客户端不断建 WebSocket 连接，
每个都订阅同一个会话——内存和 CPU 都会被吃掉。**

> **任何"外部可以无限增加"的资源都需要一个上限。**

## 七、WebSocket：三种认证，而且推荐的那种有理由

```python
"""These endpoints are separate from the main API routes to handle
WebSocket-specific authentication. Three auth methods are supported
(highest to lowest precedence):

1. **First-message auth** (recommended): The client sends
   ``{"type": "auth", "session_api_key": "..."}`` as the very first WebSocket
   frame after the connection opens. **This keeps tokens out of URLs and
   therefore out of reverse-proxy / load-balancer access logs.**
2. Query parameter ``session_api_key`` — **deprecated**, kept for backwards compat.
3. ``X-Session-API-Key`` header — for non-browser clients.
"""
```

**推荐的做法是"连上之后第一帧发认证信息"**，理由写得很具体：
**令牌放在 URL 里会进反向代理和负载均衡器的访问日志。**

> 回想第十六节那条"判断一个值是不是秘密，看拿到它能做什么"，
> 以及那里列的泄露渠道（日志、追踪导出、PR 附件）。
> **这里又多了一个：URL 会被中间设施记录。**
>
> **可迁移的道理**：**WebSocket 的认证不要放在 URL 查询参数里。**
> 浏览器的 WebSocket API 不支持自定义请求头，所以"第一帧认证"是正确的替代方案。
>
> 而且**旧的查询参数方式保留但标记为 deprecated** —— 不破坏现有客户端。

## 八、断线重连：三种补发模式

```python
resend_mode: Literal["all", "since"] | None
after_timestamp: datetime | None
# Deprecated parameter - kept for backward compatibility
resend_all: bool = False
```

```
- 'all': Resend all existing events
- 'since': Resend events after 'after_timestamp' (requires after_timestamp)
- None: Don't resend, just subscribe to new events
```

**三种模式对应三种客户端场景：**
- **首次打开一个历史会话** → `all`（要看全部）
- **断线重连** → `since`（只要断线期间的）
- **只关心新事件** → `None`

`since` 模式的文档说明了它的价值：

```
**Enables efficient bi-directional loading where REST fetches historical events
and WebSocket handles events after a specific point.**
```

**REST 拉历史（可以分页、可以缓存），WebSocket 只负责某个时间点之后的。**
两者在时间轴上对接。

### 时区处理写得很细

```
Timestamps are interpreted in server local time. **Timezone-aware datetimes are
converted to server timezone. Naive datetimes assumed in server timezone.**
```

**两种输入两种处理，都写明了。**

> 时区是分布式系统里的经典 bug 源。**"带时区的转换过来，不带时区的假定为服务器时区"**
> 是一个明确的契约——而不是让调用方猜。
>
> 但注意这个设计本身有个隐含风险：**以服务器本地时区为基准**，
> 如果服务器时区变了（换机房、改配置），历史时间戳的解释就变了。
> 用 UTC 会更稳。这是一个值得留意的取舍。

**废弃参数的处理又一次出现**：

```python
resend_all: Annotated[bool, Query(include_in_schema=False, deprecated=True)] = False
```

```
resend_all: DEPRECATED. Use resend_mode='all' instead. Kept for backward
compatibility - **if True and resend_mode is None, behaves as resend_mode='all'**.
```

`include_in_schema=False` —— **从 API 文档里隐藏，但仍然接受。**
和第二十节那个 `SkipJsonSchema` 是同一个手法，**这是第二次出现。**

而且明确了**共存时的优先级**：新参数优先，老参数只在新参数没给时生效。

> **可迁移的道理**：**废弃一个参数时要明确"新旧都传了怎么办"。**
> 不写清楚的话不同实现会不一致。

## 九、Webhook：把事件推给外部系统

```python
@dataclass
class ConversationWebhookSubscriber:
    """Webhook subscriber for conversation lifecycle events (start, pause, stop)."""
    spec: WebhookSpec
    session_api_key: str | None = None
    # **Per-instance sleep seam; see WebhookSubscriber._sleep.**
    _sleep: Callable[[float], Awaitable[None]] = field(
        default_factory=lambda: asyncio.sleep, init=False)
```

**`_sleep` 被做成了一个可替换的字段**，注释叫它 "seam"（接缝）。

**为什么？** 因为重试逻辑里有 `await asyncio.sleep(backoff)`。
**测试的时候不想真的等几秒**——注入一个假的 sleep 就能瞬间跑完。

> **可迁移的道理**：**让"时间"成为一个可注入的依赖。**
> 任何含退避/超时/定时的代码，如果直接调 `asyncio.sleep`，
> 它的测试就只能真的等。**把 sleep 做成一个字段，测试成本降一个数量级。**
>
> 第九节那条"模型接缝"也叫 seam——**同一个词在两个地方表达同一个概念：
> 可替换的位置。**

而且这个 webhook 是**不批量的**：

```python
async def post_conversation_info(self, conversation_info: BaseModel):
    """Post conversation info to the webhook **immediately (no batching)**."""
```

**生命周期事件（开始、暂停、结束）要立刻发。** 对比事件流的 webhook
（另一个类）会批量——**因为生命周期事件稀少且重要，事件流密集且单条不重要。**

> **可迁移的道理**：**批量策略要按"事件的频率和重要性"分开定。**
> 统一批量会延迟重要通知，统一即时会把高频事件变成 DoS。

重试也有：`for attempt in range(self.spec.num_retries + 1)`。

## 十、把前面 21 节串起来：服务层是怎么复用它们的

读完这一层，可以看到前面所有子系统在服务端的落点：

```
第一、二节 事件树/视图      → EventService 负责持久化和分发事件
第四、五节 会话循环/压缩     → LocalConversation 被服务端包一层
第六节   安全与审批         → WebSocket 上报"等待确认"，客户端批准后再 run
第九节   模型接缝           → llm_router / provider_connections_router
第八、十五节 提示与技能      → skills_router / skills_service（768 行）
第十一节 工具层             → tool_router；客户端工具走 WebSocket 回传
第十四节 钩子              → hooks_router
第十六节 ACP              → conversation_service 里对 ACPAgent 的特殊处理
第十九节 可观测性           → telemetry/ 整个子包
第二十、二十一节 子 agent    → sub_agents_router / agent_profiles_router
```

**服务层自己新增的东西只有四类：**
1. **多实例协调**（租约 + 围栏）
2. **缓存与懒加载**（双签名失效、启动只读元数据）
3. **推送与重连**（PubSub、WebSocket 三种补发模式）
4. **资源上限与回收**（线程池限流、订阅者上限、弱引用锁、空闲驱逐）

> **这个划分很干净**：SDK 管"一个会话怎么跑"，服务层管"很多会话、很多客户端、
> 很多实例怎么共存"。
>
> **可迁移的道理**：**把库变成服务时，新增的复杂度应该集中在"多"上**——
> 多实例、多客户端、多并发。如果业务逻辑也在服务层重新实现了一遍，
> 说明库的抽象有问题。

## 十一、这一节的可迁移结论

1. **光有过期时间的租约不安全，必须有单调递增的代数（围栏令牌）**：
   一个暂停后恢复的旧持有者能靠代数发现自己已被取代，从而拒绝写入。

2. **判断"对方进程是否还活着"要先比对主机名**：
   pid 在另一台机器上是别的进程。而且**不确定时一律当作活着**——
   抢错租约的后果比多等 45 秒严重得多。

3. **分布式系统里的字段演进要明确处理旧格式**：
   滚动升级期间新旧版本必然同时在跑，老租约文件没有新字段就退化成旧行为。

4. **缓存的失效条件必须覆盖所有影响缓存内容的数据源**：
   内容来自 `base_state.json` 和 `meta.json` 两处，
   **只检查前者会让自动标题永远不更新**。
   **判断方法：列出缓存内容的每个字段，问它从哪来。**

5. **启动只加载轻量元数据，重状态按需加载**，而且**扫描目录时单个坏条目要跳过而不是失败**。

6. **文件 I/O 要 `to_thread`**，否则一次慢磁盘操作会卡住整个事件循环和所有 HTTP 请求。

7. **按 id 分配的锁要用弱引用容器**：否则键的基数无界时会内存泄漏。

8. **"每个对象一个定时任务"在对象数量大时是反模式**：
   改成一个循环遍历所有对象，循环体里的阻塞操作挪到线程里。

9. **外部接管某个职责时要有开关关掉自己的那份**：
   否则两边都在续租，代数会乱。

10. **高频事件默认不投递，让订阅者显式声明要不要**：
    否则每个订阅者都要为几十万个 token 事件付序列化和调度的代价。

11. **任何"外部可以无限增加"的资源都需要上限**：订阅者数量、并发运行数。

12. **WebSocket 认证不要放在 URL 查询参数里**：**URL 会进反向代理和负载均衡的访问日志。**
    浏览器不支持自定义 WebSocket 头，所以"第一帧发认证"是正确替代方案。

13. **断线重连需要"从某个时间点之后"的补发模式**，和 REST 分页拉历史在时间轴上对接。

14. **时区契约要写明**：带时区的怎么转、不带时区的假定成什么。
    （但以服务器本地时区为基准本身有风险——服务器时区变了历史解释就变了。）

15. **废弃参数要说明"新旧都传了怎么办"**，并用 `include_in_schema=False`
    从文档隐藏但继续接受。

16. **让"时间"成为可注入的依赖**：把 `sleep` 做成一个字段（seam），
    含退避/超时的代码的测试成本降一个数量级。

17. **批量策略要按"事件的频率和重要性"分开定**：
    生命周期事件立刻发，事件流批量发。统一批量会延迟重要通知，
    统一即时会把高频事件变成 DoS。

18. **把库变成服务时，新增的复杂度应该集中在"多"上**（多实例、多客户端、多并发）。
    **如果业务逻辑在服务层重新实现了一遍，说明库的抽象有问题。**

## 剩下还没读的

- `event_service.py`（2021 行）的细节、`persistence/store.py`（990 行）
- `file_router.py`（1083 行）、`mcp_router.py`（782 行）、`openai/service.py`（747 行）
- `docker/build.py`（1297 行，构建 agent-server 镜像）、`docker_runtime/`
- `openhands-tools/` 剩下的：`planning_file_editor`、`gemini/`、`glob`、`grep`、`preset/`
- `plugin/`、`marketplace/`、`profiles/`、`settings/`
- `acp_file_credentials.py`（536 行）

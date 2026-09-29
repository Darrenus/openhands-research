# openhands-research

阅读 [OpenHands](https://github.com/OpenHands) 开源 coding agent 实现的学习笔记。

这个仓库放的是笔记。代码本身通过 git submodule 指向 OpenHands 官方仓库，不复制、不修改，
所有权和版权归原作者（MIT）。

## 研究对象

| 子模块 | 内容 | commit 数 |
|---|---|---|
| [`upstream/software-agent-sdk`](https://github.com/OpenHands/software-agent-sdk) | agent 循环、事件流、上下文管理、工具层、agent-server | 2408 |
| [`upstream/extensions`](https://github.com/OpenHands/extensions) | skills、automations、MCP 集成 | 359 |
| [`upstream/automation`](https://github.com/OpenHands/automation) | 调度、webhook、运行历史与派发 | 254 |
| [`upstream/OpenHands-CLI`](https://github.com/OpenHands/OpenHands-CLI) | 基于 SDK 构建的命令行前端 | 354 |

选这四个的理由：它们覆盖了从单次 agent 决策到长期自动化调度的完整链路，且全部是 MIT。
OpenHands 组织下星数最高的 `OpenHands` 仓库（89k★）现在只是 React 前端控制台，
agent 架构已全部迁出到 `software-agent-sdk`。

## 笔记

精读顺序即阅读顺序，每篇自带"可迁移结论"小节。

| 笔记 | 主题 |
|---|---|
| [01-sdk-architecture](notes/01-sdk-architecture.md) | 整体架构拆解 + 精读路线 |
| [02-event-model](notes/02-event-model.md) | 事件模型：不可变账本、事件树、并行工具调用的还原 |
| [03-view-and-properties](notes/03-view-and-properties.md) | 上下文视图：可操作位置与四条结构规则 |
| [04-agent-step](notes/04-agent-step.md) | 单步决策：四道门、四种错误恢复、并行执行与资源锁 |
| [05-run-loop](notes/05-run-loop.md) | 外层循环：预算、暂停、卡死检测、并发消息不丢 |
| [06-condenser](notes/06-condenser.md) | 压缩器：硬压软压、砍到一半、四级降级 |
| [07-security](notes/07-security.md) | 安全体系：五层检测、shell 语法解析、失败关闭 |
| [08-critic](notes/08-critic.md) | 评审员：可验证的预测、失败模式图谱、迭代精修 |
| [09-context-engineering](notes/09-context-engineering.md) | 提示装配与技能：前缀缓存、渐进式披露、路径规则 |
| [10-llm-seam](notes/10-llm-seam.md) | 模型接缝：能力表、非原生函数调用、三层降级 |
| [11-tool-layer](notes/11-tool-layer.md) | 工具层：三种来源一个接口、唯一出口脱敏 |
| [12-workspace-and-server](notes/12-workspace-and-server.md) | 执行环境与服务端：五方法抽象、认证分组、日志纪律 |
| [13-tool-implementations](notes/13-tool-implementations.md) | 具体工具：tmux 保状态、原子写入、子 agent 委派 |
| [14-hooks](notes/14-hooks.md) | 钩子系统：退出码协议、提示注入隔离、异步进程管理 |
| [15-routing](notes/15-routing.md) | 模型路由：确定性路由器 vs 元配置分类路由、账外开销 |
| [16-skills-loading](notes/16-skills-loading.md) | 技能加载：三层优先级、按可变性分层缓存、符号链接逃逸防护 |
| [17-acp-agent](notes/17-acp-agent.md) | ACP 适配层：驱动外部 agent、空闲超时 vs 硬超时、厂商差异表 |
| [18-browser-and-patch](notes/18-browser-and-patch.md) | 浏览器与补丁工具：fuzz 分数量化不确定性、注入 JS、分级错误策略 |
| [19-secrets-and-git](notes/19-secrets-and-git.md) | 密钥与 git：引用而非持有、流式脱敏算法、两种 diff 基准 |
| [20-observability-and-orchestration](notes/20-observability-and-orchestration.md) | 可观测性与编排：零成本追踪、显式上下文传递、AI 写编排代码 |

## 拉取

```bash
git clone --recurse-submodules https://github.com/Darrenus/openhands-research.git
```

已经 clone 过的话：

```bash
git submodule update --init --recursive
```

更新到上游最新：

```bash
git submodule update --remote
```

## 基准参考

software-agent-sdk 在 SWE-bench Verified 上的成绩是 77.6，技术报告见
[arXiv:2511.03690](https://arxiv.org/abs/2511.03690)。

## 许可

本仓库的笔记采用 MIT。`upstream/` 下的各子模块遵循其各自仓库的许可证。

子模块锁定版本记录于 git 索引；当前 SDK 指向 `e21d77673`。

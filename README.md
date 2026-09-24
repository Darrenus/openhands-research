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

- [`notes/01-sdk-architecture.md`](notes/01-sdk-architecture.md) — SDK 整体架构拆解：
  事件树、上下文视图与安全切点、两层主循环、错误恢复策略、安全纵深防御、
  Critic 迭代精修、多 agent 委派。附精读顺序。

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

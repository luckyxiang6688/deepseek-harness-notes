# DeepSeek Harness 架构拆解

个人整理的 **DeepSeek Harness** 与 **Super-agent** 架构笔记，视角是产品经理而非框架开发者——关心的是「哪些能力该由运行时承担、哪些该交给提示词」，以及 Agent 的权限边界与失败模式如何设计。

> 说明：以下内容基于公开资料与个人理解整理，非官方文档，细节以官方实现为准。

## 文档

| 文件 | 内容 |
|---|---|
| [DeepSeekHarness-架构整理.md](DeepSeekHarness-架构整理.md) | 总览、Seam 与 Provider 家族、Cordis 插件内核、事件系统、插件/服务/Bundle、Profile 与 Patch、`ctx` 服务容器、Agent 运行时、Scope 与 Fiber 生命周期、工具注册与实例隔离 |
| [Superagent-学习整理.md](Superagent-学习整理.md) | Super-agent 的 Router 调度中枢、Worker 执行单元、Thread 订阅关系、Mandate 授权契约、Policy 权限策略、与 Harness 的连接机制 |
| [DeepSeek-Harness-通俗解读.html](DeepSeek-Harness-通俗解读.html) | 面向非工程背景的通俗版本 |
| [deepseek-harness-深度分析.html](deepseek-harness-深度分析.html) | 带图示的深度分析 |
| [08-superagent-design.html](08-superagent-design.html) | Super-agent 设计稿整理 |

## 我关注的四个问题

1. **能力归属**：什么该做成运行时能力（可复用、可治理），什么该留在提示词里（灵活、成本低）
2. **隔离与生命周期**：多 Agent 场景下，作用域、实例与状态如何隔离，失败如何收敛
3. **授权模型**：Agent 能做什么、由谁授权、授权契约如何不被绕过
4. **可观测性**：调度链路与决策过程如何被记录和复盘

## 相关

- [skills](https://github.com/luckyxiang6688/skills) —— 可复用的 Claude Code Skill 合集

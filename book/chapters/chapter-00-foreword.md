# 0. 前言

2026 年 8 月 13 日，DeepSeek 开源了两件东西：新一代模型 DeepSeek-V4-Pro，以及一个名为 **DeepSeek Harness（`dsh`）** 的项目。前者是模型，后者是"模型之外的一切"——工具系统、上下文工程、会话管理、安全沙箱、Agent 主循环，全部以一种激进的方式组织起来：**一切皆插件（Everything is a Plugin）**。撑起这套架构的底层框架，是一个名为 **Cordis** 的元框架，其设计论文有一个颇有野心的标题——《A Programming Paradigm for Spatiotemporal Composability》（一种面向时空可组合性的编程范式）。

同一年，"Harness"成为了 Agent 工程领域最重要的词。OpenAI 用 5 个月、零人工代码、约 100 万行产出的实验证明了 "Humans steer, agents execute" 不只是一句口号；Anthropic 用一系列工程博客证明"Harness 比模型更重要"；Composio 的基准测试则用数字证明：同一个模型，换一副 Harness，通过率可以相差 20 个百分点，成本可以相差 4 倍。

**这本书回答三个问题：**

1. **Cordis 解决了什么问题，核心原理是什么？** 为什么传统的插件系统和依赖注入框架无法满足 Agent Harness 的需求？"时间可组合性"与"空间可组合性"这两个听起来学术的概念，如何落地成 `ctx.effect()` 和 `inject` 这样具体的代码？
2. **DeepSeek Harness 的架构与实现是怎样的？** 从 Profile/Bundle/Patch 三层组合，到 Session 事件溯源日志、Turn/Step 主循环、工具执行管线、模型接入层、上下文压缩、沙箱与审批——我们将逐行阅读源码。
3. **如何从 0 到 1 复刻全部能力？** 从安装使用、插件开发，到最后一章手写一个迷你 Agent Harness。

**本书结构**

- **概念篇（第 1–2 章）**：Agent Harness 的由来与 DeepSeek 的入局。
- **Cordis 原理篇（第 3–4 章）**：动态组合问题与时空可组合性原理。
- **Cordis 源码篇（第 5–6 章）**：Context、Fiber、服务解析、Loader 与框架对比。
- **dsh 架构篇（第 7–11 章）**：总体架构、会话与循环、工具系统、模型与上下文、安全双层。
- **实战篇（第 12–14 章）**：安装使用、插件开发、从零复刻。
- **升华篇（第 15 章）**：设计动机与技术权衡——为什么 Cordis 与 dsh 是这样造出来的。
- **附录**：Cordis API、dsh 命令与环境变量、内置插件清单速查、术语表与参考资料。
- **附录导航**：正文涉及的 API、命令、插件名与术语均可在附录 A–E 中回查；附录可作为案头速查手册独立使用。

**阅读须知**

- DeepSeek Harness 处于 Developer Preview（v0.1），官方明确声明会有破坏兼容性的变更。本书所有源码引用基于 2026 年 8 月的 master 快照（v0.1.0-rc.5），细节请以官方仓库最新代码为准。
- 书中所有代码片段均标注了其在仓库中的文件路径，建议配合 https://github.com/deepseek-ai/deepseek-harness 与 https://github.com/cordiverse/cordis 对照阅读。
- 阅读本书需要基本的 TypeScript 知识；不需要任何 Agent 框架经验——我们从零讲起。

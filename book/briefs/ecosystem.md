# Agent Harness 概念生态与 DeepSeek Harness 的定位（背景调研 Brief）

## 一、Agent Harness：概念起源与定义

### 1.1 命名与公式化定义

"Harness"（马具/驾驭系统）作为 AI Agent 工程术语，公认由 HashiCorp 联合创始人 Mitchell Hashimoto 于 2026 年 2 月正式命名。核心隐喻："不帮野马跑更快，但能拽住方向、控制节奏、防止闯祸"。

最流行的公式化定义来自 LangChain 团队：**Agent = Model + Harness**——"如果你不是模型，你就是 Harness"。Harness 是除模型本身之外的每一块代码、配置与执行逻辑；原始模型不是 Agent，当 Harness 赋予它状态、工具执行、反馈循环和可执行约束时，它才成为 Agent。

区分两个层次：
- **Agent Harness** 是技术实体——管理 Agent 如何运行的控制系统（工具调用管理、记忆注入与清理、重试降级、人工审批、上下文注入、子 Agent 调度）；
- **Harness Engineering** 是方法论——如何设计、构建和维护 Harness 的工程学科。

### 1.2 OpenAI：Harness Engineering 博客（2026-02-11）

OpenAI 技术人员 Ryan Lopopolo 发表《Harness Engineering: Harnessing Codex in an Agent-First World》（https://openai.com/index/harness-engineering/）。关键事实与观点：
- 团队 5 个月实验：从零构建并交付内部 Beta 产品，**没有一行人工编写的代码**；约 100 万行代码、约 1500 个 PR，团队从 3 人扩到 7 人，平均每个工程师每天产出 3.5 个 PR，整体耗时约为纯手写的 1/10。
- 核心口号：**"Humans steer. Agents execute."（人类掌舵，智能体执行）**。
- 关键论点："Agent 在运行上下文中看不到的东西等于不存在"——仓库知识要成为 system of record，用结构化 docs/ 目录取代臃肿单文件 AGENTS.md（渐进式披露），并用 linter/CI 机械强制文档新鲜度。
- 让日志、指标、trace 成为 Agent 可调用的运行时能力；给 Agent 可验证目标。
- 早期教训：把所有规范塞进一个巨大的 AGENT.md 导致 Agent 注意力涣散，后改为主文件只作目录索引、详细内容按需加载。

### 1.3 Anthropic：更早的系统性 Harness 论述

Anthropic 最先系统提出 Harness 设计理念，分别于 2025-11-26 和 2026-03 发布《Effective Harnesses for Long-Running Agents》与《Harness Design for Long-Running Apps》。

**《Effective Harnesses for Long-Running Agents》**（2025-11-26，anthropic.com/engineering）：核心问题是 Agent 在离散会话中工作、每个新会话都没有之前的记忆（类比"换班工程师"）。解决方案是**双 Agent 结构**：initializer agent 首次运行搭好环境，coding agent 每次会话做增量推进并留下清晰工件供下一班接续。长任务两大失控根源：上下文一致性下降，以及"上下文焦虑"（context anxiety）——模型接近自以为的上下文上限时过早收尾。

**工具设计原则**（《Writing Effective Tools for Agents》）：
- 核心洞察：**"Agents are non-deterministic users of deterministic tools."**
- 少而精、策略性选择工具；清晰命名空间；返回有意义的上下文；token 高效；把工具描述当作 prompt engineering。
- 评估驱动开发：原型→评测→分析失败→迭代。
- 实证：Claude Sonnet 3.5 仅靠精细化工具描述即在 SWE-bench Verified 达到 SOTA。

**渐进式披露（Progressive Disclosure）**：不要把全部工具定义一次性载入上下文；Skills 只在上下文中放名称+短描述，按需加载完整内容。官方案例：从 150,000 token 降到 2,000 token（约 98.7% 降幅）。

**"Harness 比模型更重要"的论据链**：
- LangChain 在 TerminalBench 2.0 上仅改变 Harness（模型不变），排名从 30 名开外升到第 5。
- 仅更换文件编辑接口的调用方式，编码基准分数从 6.7% 跳到 68.3%。
- Cursor 团队 2024 年发现：同一 Claude 模型、同一基准，不同 harness 设计下得分 46% vs 80%。
- Databricks 内部基准：同一模型经不同 harness，任务成本相差 2 倍以上而输出质量相同（高效 harness 每轮少发约 3 倍上下文）。
- Anthropic 2026-04 事故复盘：系统提示、缓存、reasoning effort 三处 harness 层改动叠加导致 Claude "变笨"，模型权重并无变化。

## 二、DeepSeek 组建官方 Agent Harness 团队

### 2.1 立项与确认（2026-05）

- 2026-05-18，DeepSeek 官网上线 Agent Harness 产品经理与研发工程师岗位（北京）。职位描述写明 **"Model + Harness = Agent"**，"除模型本身以外的所有工作，都属于 Harness 的范畴"（来源：科创板日报，2026-05-20）。
- 产品经理岗位要求深度使用过 Claude Code、Codex、Cursor、OpenCode、GitHub Copilot、Manus 等产品。
- 2026-05-20，DeepSeek 资深研究员**陈德里**（Deli Chen）在 X 发帖证实："来 DeepSeek 从零做 Code Harness"，"简单来说就是对标 Claude Code"。
- 36kr 解读：API 价格越低，卖 token 越不赚钱；"DeepSeek 需要从卖模型调用，转向卖工作流结果。"

### 2.2 团队负责人与全球招募（2026 年 7–8 月）

团队负责人**崔添翼**（Cui Tianyi）：浙大计算机系毕业，Jane Street 任职九年，2026-03 加入 DeepSeek 组建 Harness 团队（来源：东方财富，2026-08-13）。选量化背景的逻辑：量化交易的本质是把"模型"翻译成可审计、可回退、无隐性行为的执行系统，与 Agent Harness 工程挑战高度同构。崔添翼同时也是 Cordis 上游生态（Koishi/cordiverse，ID Shigma）的作者。

时间线：
- 2026-07-06/07："DeepSeek Harness"微信公众号注册并通过企业认证；
- 2026-07-31：V4-Flash 正式版 API 公测，官方文档首次出现"DeepSeek Harness 极简模式（即将发布）"；
- 2026-08-01/02：崔添翼在 X 全球公开招募内测用户，优先有 Agent Harness 开源项目经验者；
- 招募成为全球开源 Agent 生态大摸底：截至 8 月 3 日，769 位开发者报名、712 个去重仓库、累计 120 万+ Stars、覆盖 18 个赛道；
- 2026-08-11：官方公众号（黑鲸 Logo）正式上线；
- 2026-08-13：DeepSeek Harness v0.1 与 DeepSeek-V4-Pro 同日开源（MIT），GitHub 发布，几天内社区产出约 300 个插件。

### 2.3 社区反响与战略解读

- 背景：2025 年全球 AI 编程工具市场 295.7 亿美元、预计 2030 年达 646.8 亿美元（Research and Markets）。
- 2026-06 Claude Code 被曝"隐写回传"事件、工信部 NVDB 发布安全后门风险提示，阿里、腾讯、美团、京东等排查并转向自研 Harness——为国产 Harness 提供市场契机。
- 成本叙事：第三方测试显示用 DeepSeek V4 实现类似 Claude Code 体验的成本"仅为几十分之一"——"更便宜的模型 + 更好的 Harness 优化"。

## 三、同类 Harness 对比

### 3.1 设计理念谱系

- **Claude Code（Anthropic）**：终端级工程代理。源码层面是庞大状态机：7 种 Continue reason + 10 种 Terminal reason 枚举化；**5 层上下文压缩漏斗**；应用层 AST 解析做权限管控；27 种 Hook + Skills 懒加载；prompt cache 字段字节级对齐作为头号成本优化。哲学：**harness 替模型决策**。
- **Codex（OpenAI）**：开源 Rust workspace，朴素 `loop {}` + 4 种退出原因，安全甩给 OS（Landlock/Seatbelt/Seccomp），高风险操作人工审批；扩展全部走 MCP；上下文按需读取。哲学：**给模型递工具的薄壳**。两家共识："Harness 承担不变量，模型承担决策"，"prompt 不是产品力，harness 才是产品力"。
- **OpenCode**：开源多模型路线；benchmark 显示其 harness 层存在"重排版 bug"（三个模型在 OpenCode 中都试图重排既有代码），证明问题出在 harness 而非模型。
- **Pi Agent（Mario Zechner）**：极简主义——核心只有 read/write/edit/bash 四个工具，刻意不做 MCP、子 Agent、plan 模式、权限弹窗，一切交给扩展；"there are many agent harnesses, but this one is yours"。Databricks 基准中 Pi 以最高通过率且成本显著更低（每轮少发约 3 倍上下文）。
- **Aider**：git 原生（每次修改自动 commit）、仓库 map（tree-sitter 摘要）注入上下文、多种"编辑格式"适配不同模型能力。

### 3.2 Harness 评测：Composio 基准（2026-08-11）

《Finding the Best Harness for DeepSeek V4 Flash》：同一 DeepSeek V4-Flash 模型、30 工作流 × 8 harness、240 次运行，总通过率仅 53.8%。结果：**Pi Agent 66.7%（$0.028/成功）＞ Prime Agent 62.5% ＞ OMP 56.7% ＞ Claude Code / Codex / DeepAgents 53.3% ＞ Hermes 50% ＞ OpenCode 46.7%**。Claude Code 中位完成时间最短（122.7s）但单次成功成本最高（$0.195，因 prompt cache 命中率仅 1.5%）。结论："No harness had the best result for every measure."

姊妹篇：同一 Kimi K3 模型在 8 个 harness 下通过率从 68% 到 88%——20 个百分点差距完全来自 harness。

## 四、为什么"模型之外的脚手架"成为竞争焦点

1. **模型同质化、Harness 成差异化壁垒**：同一模型换 harness，基准差距可达 20–34 个百分点、成本差 2–4 倍。
2. **Context Engineering 是核心战场**：Agent 看不到的东西等于不存在（OpenAI）；上下文一致性下降与"上下文焦虑"是长任务失控主因（Anthropic）；压缩策略直接决定成本与稳定性。
3. **缓存命中是真金白银**：prompt cache 对齐、渐进式披露（15 万 token → 2 千 token）。
4. **工具设计是杠杆最大的低成本改进**：仅改工具描述即获 SWE-bench SOTA。
5. **商业逻辑：从卖 token 到卖工作流结果**。
6. **可控性与安全成为准入门槛**：Claude Code 后门事件使企业意识到 harness 层决定数据出境、权限边界与可审计性。
7. **协同进化原则**：Harness 会随模型变强而变薄，但"永远不会消失"。

## 附：不确定性
- 人名以一手来源为准：陈德里（Deli Chen）、崔添翼（Cui Tianyi）。
- Anthropic 第二篇《Harness Design for Long-Running Apps》（2026-03）仅见于转述。

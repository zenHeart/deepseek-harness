# 《从 Cordis 到 DeepSeek Harness》章节素材 Brief：设计动机与技术权衡

> 调研时间基准：2026-08-13（DeepSeek Harness v0.1 开发者预览版发布日）。所有英文原文附中文翻译；每条论断标注来源 URL。

---

## 一、Cordis / Koishi 的设计演进史：从聊天机器人框架到"时空可组合性"元框架

### 1.1 关键时间线（一手来源锚点）

- **2020-01**：Shigma（崔添翼）发布 Koishi 首个正式版。官方 FAQ："Koishi 是一个由 Shigma 创立的开源项目。它于 2020 年 1 月发布了第一个正式版本。"（https://koishi.chat/zh-CN/about/faq.html）
- **2022-05（Koishi 4.7.1）**：Cordis 从 Koishi 核心中被抽象为独立包。Release notes 原文：「**infra:** 发布了新的核心包 cordis，它作为 Koishi 的底层框架提供了上下文、插件系统、事件模型等核心功能」（https://github.com/koishijs/koishi/discussions/691）。这是"为什么从 Koishi 中抽象出 Cordis"的官方首次表述：把与聊天领域无关的框架层独立出来。
- **2024-03（Koishi 4.17.2）**：Cordis 继续向"资源安全"演进，「**cordis:** 实验性地引入了 `ctx.set()`，能够资源安全地设置一个服务」（https://github.com/koishijs/koishi/discussions/1378）。
- **2026-08-13**：伴随 dsh 发布，cordiverse/paper 仓库公开预印本 **《A Programming Paradigm for Spatiotemporal Composability》**（草稿日期 2026-08-13），Cordis 仓库（https://github.com/cordiverse/cordis，约 997 stars）tagline 改为 "Meta-Framework of Spatiotemporal Composability"，并直接链接 dsh 的 cordis-primer 文档。

### 1.2 Shigma 自己的设计陈述：《可逆的插件系统》（koishi.chat 官方 cookbook，一手长文）

来源：https://koishi.chat/zh-CN/cookbook/design/disposable.html —— 这是章节最重要的一手文献，可直接引用。

**（a）Cordis 的自我定位：**

> "Koishi 的一切都从 Cordis 开始。但我想大部分 Koishi 的开发者都不知道 Cordis 是什么。如果让我来定义的话，Cordis 是一个**元框架 (Meta Framework)**，即一个用于构建框架的框架。Cordis 的名字来源于拉丁语的心。我希望它能成为未来软件（至少是我开发的软件）的核心。"

> "作为一个元框架，Cordis 并不耦合任何具体的领域或场景。它所提供的能力是大多数框架都不足为奇的——插件系统，但在这个系统背后却是大多数框架都没有达成的目标：**可逆性**。"

**（b）动机："软件文明在退步"论——现代框架丢失了资源回收能力：**

> "在过去，我们用 C 这样的语言编写程序时，我知道 `open()` 会返回一个 `fd`……而在如今，当我使用 Koa 时，我可以使用 `app.use()` 来注册一个中间件……但很遗憾的是，Koa 不会告诉你如何取消这个中间件，Vue 不会告诉你如何卸载这个组件，Node.js 甚至会永久占用这个 `.node` 文件。……或许是人们认为重启过于方便了，因此一些框架的开发者们已经完全不考虑回收资源的需求了。"

**（c）可逆性带来的三个好处（设计目标）：**

> "**可组合性 (Composibility)**。……现实中的模块也应当是可拆卸的，现实中的插件也应当是可以拔出的。不可逆的软件即便进行了模块化，也只会随着时间推移而变得更加臃肿。"
> "**可靠性 (Reliability)**。当软件的规模增加时，可逆性可以确保软件所使用的内存和其他资源都在可控范围内……可逆性也意味着可追踪性，即便某个模块出现了资源泄露，我们也可以快速定位错误的来源。"
> "**可访问性 (Availability)**。……如果其中的每个组件都是可逆的，我们就可以在保证其他功能持续运行的情况下替换掉任何一个组件，甚至可以滚动更新整个程序自身。"

**（d）可逆 Koishi 的形式化定义（"路径无关"）：**

> "可逆的 Koishi 是指，对于任何一个 Koishi 实例，任意进行加载和卸载插件操作后，最终行为仅与最终启用的插件相关；与中间是否重复加载过插件、插件之间的加载或卸载顺序都无关。你也可以简单理解为「路径无关」。"

由此得到的工程红利原文："**热重载**……显著降低了用户的开发和更新成本，并大幅提高了 Koishi 应用的 SLA。""如今，Koishi 已经有超过 3000 个插件，其中的依赖错综复杂。而即使是在这个规模下，Koishi 仍然能够妥善处理所有插件的加载、卸载和更新。这一切都得益于 Cordis 的可逆性。"

**（e）effect 思想的数学来源**：文中把副作用变换空间 F=C→C 构造为幺半群，再要求逆元成群，引入 effect 函子并证明其为同态；核心类比是 C++ 智能指针 / Java GC / Rust 所有权："这些封装不仅不会导致内存泄露，反而通过提高易用性减少了开发者的心智负担。" 落点是 **ctx（上下文）对象**："上下文对象是一个插件中唯一的可变部分，它同时担任了 C 参数和返回值的角色……可以将上下文比作一个副作用的插座。"——这就是 dsh primer 中 "Registrations are reversible effects" 的源头。

**（f）"时空可组合性"一词的原始出处**（同一篇文末）：

> "很多人谈论可组合性，主要说的是解耦……但其实我们编写的代码并不是静态的，可组合性可以在更多的维度上定义：
> - 逻辑可组合性：代码自身的解耦能力（常见的理解方式）。
> - **时间可组合性**：代码可以被同时加载、可以被回收副作用的能力（本文主要介绍的部分）。
> - **空间可组合性**：代码之间能够有效声明和隔离依赖关系的能力。"

文末还坦承局限（可用于章节的"权衡"小节）："很遗憾，目前并不能（在语言层面确保资源安全）。开发者只需设置几个全局变量……就可以绕过 Cordis 的保护机制。……这种渐进性是 Cordis 的一大优势。"

### 1.3 论文摘要（effect/coeffect 的形式化来源，一手）

来源：https://github.com/cordiverse/paper （README 摘要，Draft of August 13, 2026）

> "Modern software—from plugin systems to **self-evolving agent harnesses**—increasingly requires *dynamic composition*, yet its formal foundations remain underdeveloped. We identify two orthogonal dimensions of the problem: *temporal composability*, the ability to completely revert a component's side effects upon removal, and *spatial composability*, the ability to declare and reactively manage inter-component dependencies. We address the two dimensions by **lifting classical effect and coeffect concepts to runtime mechanisms**. In particular, we formalize *revertible effects*, in which every context transformation carries an inverse that the runtime tracks. We formalize *reactive coeffects*, in which each change of the context notifies a component against its coeffect specification. We unify the effect context and the coeffect context into a single *context type*…"
>
> 译：现代软件——从插件系统到**自我演化的 Agent Harness**——越来越需要*动态组合*，但其形式化基础仍不完善。我们识别出两个正交维度：*时间可组合性*（移除组件时能完全回收其副作用）与*空间可组合性*（声明并响应式管理组件间依赖）。我们通过**把经典的 effect 与 coeffect 概念提升为运行时机制**来解决这两个维度：形式化"可逆 effects"（每次上下文变换都携带由运行时跟踪的逆操作）与"响应式 coeffects"（上下文的每次变更都会按照组件的 coeffect 规约通知它），并把 effect 上下文与 coeffect 上下文统一为单一的*上下文类型*。

注意摘要直接把 "self-evolving agent harnesses" 写进动机句——论文与 dsh 发布同日（2026-08-13），可见 Cordis 理论化与 dsh 工程化是同一作者的闭环。dsh README 亦明确：dsh "is powered by Cordis, whose design is described in *A Programming Paradigm for Spatiotemporal Composability*"（https://github.com/deepseek-ai/deepseek-harness/blob/master/README.md）。

---

## 二、dsh 官方文档中的设计动机（一手原文逐条摘录）

基础 URL 前缀：`https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/`

### 2.1 为什么"一切皆插件 / 没有特权核心"（docs/architecture.md）

> "Every part of the product is a plugin, including the model adapter, the tool registry, the session log, and the agent loop itself, so every part is replaceable from configuration. **There is no privileged core to patch**: you extend dsh by mounting a plugin beside the others, and registrations are effects that unwind when their plugin unloads."
>
> 译：产品的每一部分都是插件——包括模型适配器、工具注册表、会话日志乃至 agent 循环本身——因此每一部分都可从配置层替换。**没有可供打补丁的特权核心**：扩展 dsh 的方式是把一个插件挂到其他插件旁边，而所有注册都是插件卸载时会回卷的 effects。

### 2.2 为什么 vendoring（vendor/README.md 首段）

> "They are copied into this monorepo instead of being depended on via npm, so that **the harness fully owns its framework layer (auditable, patchable, pinned)**."
>
> 译：这些包被复制进本 monorepo 而非经由 npm 依赖，从而让 harness **完全拥有自己的框架层（可审计、可打补丁、版本钉死）**。

配套细节（同样是一手动机）：vendored 包重命名进 `@deepseek-ai` scope 的理由是"publishing the harness publishes this framework layer too, and a publication under the upstream names would squat them on the registry"（发布 harness 会连带发布框架层，用上游名字发布会抢注 registry 上的包名）；目录名与版本号刻意不变，"so the manifest below still reads as an upstream snapshot"（清单仍可读作上游快照）。本地修改日志共 18 条，每条要求 "every divergence from upstream must be listed"（与上游的每一处分歧必须列出），其中第 6 条 "fiber.ts lifecycle hardening" 本地修补了三个重入处置缺口、第 11 条修复了上游 patch 算法中"inserted rows silently unpatchable"的问题——都是"可审计+可回退"方法论的直接体现。

### 2.3 为什么 Model-visible ⟺ logged（docs/architecture.md §Session log；AGENTS.md）

architecture.md 原文：

> "The session log is the source of the context the model sees. `deriveMessages()` projects model history from it, and raw `assistant/chunk` events preserve replay and UI fidelity. **Model-visible means logged.** Anything that reaches a model request must be reconstructable from the log, and a runtime invariant asserts it. This is why a new model-visible input requires a new session event: extend `SessionEventMap` and render from the log."
>
> 译：会话日志是模型所见上下文的来源……**模型可见即已落账**。任何到达模型请求的内容必须能从日志重建，且有一个运行时不变量断言这一点。这就是为什么任何新的模型可见输入都需要一个新的会话事件。

AGENTS.md 把它写成双向约束："**Model-visible ⟺ logged**: anything that reaches a model request must be reconstructable from the session log; a new model-visible input requires a session event."（https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/AGENTS.md）

### 2.4 为什么 Plugins not loop changes（AGENTS.md）

> "**Plugins, not loop changes**: new behavior goes on documented extension points; changing `agent-loop` requires updating docs/architecture.md."
>
> 译：**用插件，而非改循环**：新行为挂到文档化的扩展点上；改动 `agent-loop` 必须同步更新架构文档（即把 loop 的变更显式化、留痕化）。

配套条款："New behavior attaches to a documented extension point. Changing the loop itself updates this map."（architecture.md "Where new behavior goes" 表格，含 20 行"目标→机制"映射）。

### 2.5 为什么 profile / bundle / patch 三层（docs/architecture.md §Profiles and bundles）

> "A running `dsh` is a plugin tree composed at boot from ordered layers." / "A **profile** is a named composition stored in the Harness home… A **bundle** is a distribution format for Cordis config rows and the code they mount, so whatever it inserts stays patchable by the layers above it."
>
> 译：一个运行中的 dsh 是在启动时由有序分层组合出的插件树。profile 是命名的组合；bundle 是 Cordis 配置行及其所挂代码的分发格式，**它插入的任何内容都仍可被其上层的层所 patch**。

分层顺序："each bundle in the profile's listed order, then the profile's `cordis.patch.yml`, then the home-level one, then any `--patch` overlay."（先 bundle、再 profile 补丁、再 home 级补丁、再命令行 overlay）——这是一个"声明式合并、越靠后优先级越高"的部署分层模型，动机在 bundle/base/README.md 中亦有说明："Later bundle layers … and the user's profile `cordis.patch.yml` override these rows by id"。

### 2.6 为什么 session 用事件溯源（docs/subsystems/session.md）

> "A `Session` is an **append-only log** of typed `SessionEvent`s — the single source of truth for an agent's whole interaction history. The LLM message history is *derived* from the log, never stored separately; **replay is re-derivation from the same events**."
>
> 译：Session 是类型化 SessionEvent 的**仅追加日志**——agent 全部交互历史的唯一事实来源。LLM 消息历史是从日志*派生*的，从不单独存储；**回放就是从同一批事件重新派生**。

以及架构文档的衍生红利句："Fork, resume, transcripts, telemetry, and persistence all derive from this stream."（分叉、恢复、转录、遥测、持久化全部派生自同一条事件流。）

### 2.7 为什么工具必须声明 output schema（docs/subsystems/tools.md + docs/cookbook/adding-a-tool.md）

`ToolDefinition` 的定义即动机：工具 = 模型可见 schema + "**a mandatory canonical output declaration**"（强制性的规范输出声明）。"The registry's `schemas()` builds the model-facing `ToolSchema[]` by an explicit allowlist — `output`/`execute`/`finalizeContent`/`timeoutMs`/`isConcurrencySafe`/`presentCall`/`presentResult` **must never leak into a model request**."（注册表通过显式白名单构造模型可见的 schema——output、execute 等字段绝不泄漏进模型请求。）

cookbook 的规则表述（adding-a-tool.md）：

> "**Declare and return one canonical JSON value.** … Do not return content blocks from the body or make callers parse prose for ids and fields."（**声明并返回唯一的规范 JSON 值**……不要在函数体里返回内容块，也不要让调用方从散文里解析 id 和字段。）
> "Throwing or returning an invalid value means `isError`. The registry catches throws and contains schema, renderer, metadata-projector, and lossless-JSON failures…"（抛异常或返回非法值即 `isError`；注册表兜住所有 schema/渲染器/元数据/无损 JSON 失败。）

### 2.8 为什么审批 fail-closed（docs/subsystems/approval.md）

> "`ApprovalOutcome` is closed and fail-closed. `allowed-once` grants only the asked-about action; callers deny on `rejected`, `cancelled`, and `unavailable`. **A missing, non-owning, throwing, or non-conforming answerer becomes `unavailable` rather than opening the gate.**"
>
> 译：审批结果是封闭类型且 fail-closed。`allowed-once` 只授予被询问的那个动作；调用方在 rejected/cancelled/unavailable 下一律拒绝。**缺失的、非属主的、抛异常的或不合规的应答者都会变成 `unavailable`，而不是打开闸门。**

审计对设计："Every request receives a fresh `ApprovalRequestId`. The brand pairs the `approval/asked` and `approval/decided` audit events…"（每个请求获得全新的 ApprovalRequestId，用品牌化类型把 `approval/asked` 与 `approval/decided` 两条审计事件配对——即"审计对"。）`never` 策略的动机注释："The strict headless stance (CI, unattended runs) and the policy whose outcome is knowable without asking."（严格的 headless 立场——CI、无人值守运行——其结果无需询问即可预知。）

### 2.9 为什么 MCP 默认关闭（packages/mcp/README.md + base bundle 配置事实）

观察事实：base bundle（`packages/bundle/base/cordis.patch.yml`）不挂载任何 `dsh-mcp-client` 行；`packages/mcp/README.md` 定位仅为"bridge the harness to the MCP ecosystem"的可选桥接。mcp-client README 给出的使用方式是"**One plugin instance per MCP server in `cordis.yml`**"（每个 MCP server 一个插件实例，写在用户自己的 cordis.yml 里）——即 MCP 是 opt-in 配置而非默认能力。安全相关的设计细节可作为动机佐证：`env` 字段是"Extra env vars merged on top of **scrubbed ambient env**"（在清洗过的环境变量之上合并）；命名规则保证"Names are pure functions of `(serverName, rawName)` — connection order, re-syncs, and other servers never rename a tool"（工具名是纯函数，连接顺序、重新同步、其他 server 都不会导致改名）——前者防环境泄漏，后者保证模型可见契约稳定。

### 2.10 为什么 web_fetch 默认禁用（packages/bundle/base/cordis.patch.yml 内联注释，一手）

> "# Every mode enables the stable model-facing web_search tool. … **Fetch stays disabled and no fetch provider is mounted: that provider defers SSRF protection and the model would choose the request target.**"
>
> 译：每种模式都启用稳定的模型可见 web_search 工具……**Fetch 保持禁用且不挂载任何 fetch provider：该 provider 把 SSRF 防护留待后续实现，而请求目标是由模型选择的**——在 SSRF 防护就绪之前，默认不放行"模型自选 URL 的抓取"。

配套稳定注册原则（tool-web README）："Tool registration follows product **enablement**, not backend availability… This keeps the model schema stable without making plugin load order, credential state, or HMR timing part of the model-facing contract."（工具注册跟随产品启用配置，不随后端可用性波动——保持模型 schema 稳定，不让插件加载顺序、凭证状态、HMR 时机进入模型可见契约。）

### 2.11 其他高价值 AGENTS.md 条款（体现同一套方法论，可作为章节侧栏）

- "**Misconfiguration fails loud** at load when self-contained…; never silently skip a missing referent."（配置错误要响亮地失败，绝不静默跳过缺失引用。）
- "**Explicit > implicit at package boundaries**: defaulting is an explicit `resolve(request): Spec` step…, never a hidden `?? default` inside `run()`."（包边界显式优于隐式；默认值走显式 resolve，绝不藏在 run() 里的 `?? default`。）
- "**No hardcoded tunables in plugins**… a `DEFAULT_*` constant or test hook is not configurability."（插件里不允许硬编码可调参数；DEFAULT_ 常量不算可配置性。）
- "**Pre-release stance: foundation over blast radius**… prefer the correct foundation over compatibility shims."（发布前立场：正确的地基优先于兼容垫片。）

---

## 三、官方发布文《DeepSeek Harness 开发者预览版：一切皆插件》（2026-08-13）要点

原文署名"DeepSeek Harness团队"，华尔街见闻全文转载：https://wallstreetcn.com/articles/3779385 （澎湃转载确认系公众号原文：https://www.thepaper.cn/newsDetail_forward_33777963）。

关键表述（可直接引用）：

1. **总纲**："DeepSeek Harness 采取'一切皆插件'的设计思路。我们采用插件式开放架构来构建 Agent Harness：模型、工具、技能、会话、沙箱、存储、循环、调度、UI 等所有 Agent 能力均由插件组合而成，可自由替换、灵活重组。"
2. **Cordis 定位**："DeepSeek Harness 基于具有时空可组合性的 Cordis 插件系统构建。**Cordis 元框架只负责插件的加载与卸载以及依赖关系**，Agent Harness 的所有具体组件都是不同的 Cordis 插件。插件通过 Cordis 服务与事件彼此协作，并可以在配置层自由组合。开发者无需改动 DeepSeek Harness 的源码本身，就能以插件的方式独立选择、替换或扩展其中的任一能力。"
3. **四种运行模式**（每种默认加载不同插件集合）：
   - 标准模式：提供完整的工具组合；
   - PTC 模式：程序化工具调用（Programmatic Tool Calling），由模型生成的一段代码来组合多轮工具调用；
   - 极简模式：仅保留一个 shell 工具与一个文件编辑工具，用于最小环境下的模型基准测试；
   - 创造模式：可以检查当前运行时、在内存中试验 Cordis 插件，并据此组合和创作新的模式。
   （发布文截图中的模式选择器补充：PTC"通过 Code Mode SDK 呈现工具，让模型用一个 TypeScript 程序组合多步操作"；极简模式"仅提供持久 bash 与 str_replace_editor 的双工具编码 Agent"。）
4. **每一次运行都有迹可循**："模型看到的一切，都会写入仅追加（append-only）设计的会话日志，包括系统提示词、思维链、工具调用与结果、子 Agent 调度，以及每一次上下文注入。在 Trajectory 视图中，你可以按来源查看这些信息。恢复、分叉、检索与回放也都共享同一份事件流。"
5. **132 个插件**：发布文中的设置页截图显示"插件列表 **132**"（include、timer、hmr、llm、session、typert-registry、agent、user-interaction、settings-local、credentials-local 等均可逐条启停）——"一切皆插件"不是口号而是 UI 可数的运行事实。
6. **结尾定位**："DeepSeek Harness 目前的 v0.1 版本只是一个起点，我们期待与全球开发者一起，在**开源、开放、可复用、可组合**的基础设施之上，共同探索智能上限。诚邀全球 Harness 开发者共建 DSH 插件生态。"（文末配图："DeepSeek Harness 内测用户制作的部分插件"。）
7. 发布前验证：V4-Flash 正式版 API 文档已写明 Code Agent 基准"使用 DeepSeek Harness 极简模式（即将发布）作为框架进行测试"（IT之家/36氪报道，https://developer.cloud.tencent.com/news/4358005 ）——极简模式是官方基准的可复现底座。

---

## 四、同类对比中的取舍：Cordis 路线 vs Claude Code / Codex 路线

### 4.1 三条路线的结构性差异（社区一手分析）

- **Claude Code**：TypeScript 单体 + hooks + MCP（仅客户端）。"它的很多精华都发生在'这轮怎么接下一轮'这个问题上……它的 harness 首先是活着的，先得连续活下去，然后才谈如何把规则拆得更漂亮。"（harness-books 对比卷，https://harness-books.agentway.dev/book2-comparing/exported/book2-comparing.pdf ）
- **Codex CLI**：Rust 单体（core/src/lib.rs），"把线程、rollout、state bridge、instructions、skills、hooks、sandboxing、exec policy、tools 拆成模块……让控制层显式地长成可组合、可导入、可序列化、可策略化的器官"（同上）；扩展面主要是 MCP。
- **dsh**：不以 hooks/MCP 为扩展终点，而是把整个产品（含 agent loop 本身）做成运行时反应式插件系统上的插件。DoNews/搜狐引官方文档："DSH 建立在 Cordis 之上，包括模型适配器、工具注册、Session Log，甚至 Agent Loop 本身都被设计成可以替换的插件……因此，它并不只是'DeepSeek 版 Claude Code'或者'DeepSeek 版 Codex'。"（https://www.donews.com/news/detail/1/6670452.html ）

### 4.2 dsh 对 Claude Code / Codex 的"兼容而非依赖"姿态

dsh 把对手的扩展机制实现为**自己插件树里的普通插件**：`packages/hooks/`（AGENTS.md：「Claude Code/Codex hook bridges + wire-protocol library」）、`packages/mcp/mcp-client`（MCP 工具以 `mcp__<server>__<name>` 注册进 `ctx.tools`，README 明言这是"the same server-qualified shape Claude Code and Codex use"）、`packages/acp/`（Agent Client Protocol server）。取舍含义：MCP/hooks 是生态入口，不是架构中心——中心是 Cordis 的服务+事件+可逆 effect。

### 4.3 社区评价与质疑

- **正面**：第一财经转引开发者："'一切皆插件'让模型仅仅成为代理技术栈中一个可替换的部分，同时采用 MIT 这一相对宽松的开源协议，也是一个激进的举措，这'非常酷也非常早期'。"有用户称"这是他'向 Claude Code 说再见的那一天'"；"DeepSeek 将一切开源，这样我们就不必生活在 Anthropic 的统治之下了。"（https://finance.sina.com.cn/stock/wbstock/2026-08-13/doc-inineuqi3337598.shtml ）
- **生态标准卡位论**：Codex 生态合作伙伴宋斐："Harness 对外开放，将意味着第三方进入同一环境后，官方分数与外部复现之间的落差就可以直接核对……如果 Harness 真的开源，行业统一脚手架这个位置，它就有机会先占住。"（南方财经，http://www.sfccn.com/2026/8-11/wOMDE0MDdfMjIwNjAwOQ.html ）
- **上手热度**：社区解读文《DeepSeek Harness 发布了，居然没CLI》介绍 Cordis"五个比较关键的概念：插件、上下文、服务依赖、类型化事件和可撤销的副作用"（https://www.eet-china.com/mp/a516864.html ）。
- **质疑/冷静面**（章节应保持平衡）：
  - InfoQ 评价："有创新但编排范式未突破"（经 https://www.aireadinghub.com/article/14996 转引）。
  - 崔添翼 5 月公开招聘被报道时的分析指出迁移风险："从量化交易系统迁移到面向开发者的 Agent 产品，目标用户、反馈循环与发布节奏完全不同，过往经验的可迁移性需要在实际产品中验证。""Jane Street 风格的工程文化在国内招聘市场并不容易复制。"（https://www.chooseai.net/news/3901/ ）
  - 官方自己也压低了预期：README 与发布文均强调 developer preview、"THERE WILL BE COMPATIBILITY-BREAKING CHANGES"。
  - 参照系：同期爆火的 Pi（约 8.6 万 stars，36氪/量子位 https://www.36kr.com/p/3934404658642055 ）走"核心极简 + 扩展点克制"路线，社区评论"Claude Code 给你 27 种钩子，Pi 就给你 before/after 两个"——这与 dsh"132 插件全量插件化"形成鲜明对照，是章节讨论"插件化程度光谱"的现成素材。

---

## 五、量化背景的方法论迁移：Jane Street 经验如何长进 dsh 的设计里

### 5.1 履历事实（一手/权威报道）

- 浙大计算机系（NOI 铜牌保送、6 枚 ACM 亚洲区域赛金牌，《背包九讲》作者），2013 年入 Jane Street（香港/纽约，软件开发与研究员，覆盖股票与固收），任职 9 年；2022 年联合创立 TSY Capital（用 ML 生成信号、Rust 自研低延迟执行系统）；2026 年 3 月加入 DeepSeek 组建 Harness 团队。（36氪《在做Harness这件事上，DeepSeek更信搞量化的》https://m.36kr.com/p/3828532121837831 ；钛媒体"隆中对" https://www.tmtpost.com/7999833.html ）
- 陈德里（DeepSeek 研究员）2026-05-20 在 X 官宣组建 Harness 团队，"对标 Claude Code"。

### 5.2 "量化思维 = Harness 思维"的同构表述（七牛云新闻，最直接的迁移论述）

来源：https://news.qiniu.com/archives/1786527719433

> "量化交易的核心工作之一，是把对市场的判断（'模型'）翻译成能够在真实市场中执行的交易系统（'Harness'）：信号必须在毫秒内转换为订单，**执行失败需要回退，所有状态必须可审计，系统不能有隐性行为**。这和 AI 编程 Agent 的工程挑战高度同构——在不确定的模型输出和确定的工程执行之间，需要极其严格的控制层。"

### 5.3 逐条映射（设计 ↔ 量化工程纪律）

| dsh 设计 | 原文/出处 | 量化对应物 |
|---|---|---|
| Model-visible ⟺ logged，运行时不变量断言 | architecture.md §Session log | 全状态可审计、无隐性行为 |
| append-only 事件溯源；fork/resume 皆派生自同一事件流 | session.md | 交易日志/行情回放 |
| 审批 fail-closed；`unavailable` 不开闸 | approval.md | 风控默认拒绝 |
| `approval/asked` + `approval/decided` 审计对，ApprovalRequestId 品牌配对 | approval.md | 审计对/对账 |
| vendoring：auditable, patchable, pinned；18 条 divergence 全记录 | vendor/README.md | 供应链钉版 + 变更留痕 |
| "Misconfiguration fails loud"、"never silently skip a missing referent" | AGENTS.md | 配置即风险敞口，必须显式 |
| "Explicit > implicit at package boundaries"（显式 resolve，拒绝隐藏的 `?? default`） | AGENTS.md | 无隐性行为 |
| Loader/Include 事务化配置重 reconcil­iation，失败回滚（本地修改 #8） | vendor/README.md | 执行失败需要回退 |
| 极简模式 = 官方基准可复现底座 | 发布文/V4-Flash 文档 | 回测环境与现实对齐 |

媒体侧的同构判断可引：「高频量化交易系统的核心壁垒从来不是策略创新，而是极端复杂环境下的稳定执行、异常兜底、全程日志追溯与风险精准可控」（clawpk，https://clawpk.net/articles/409-deepseek-harness-agent-framework.html ）；「在量化里，不能被稳定执行的策略价值就是 0。在 AI 里，不能安全操作文件、命令、代码的模型，也只是一个聊天框罢了。」（36氪）。

---

## 六、素材使用提示（给作者的备注）

1. **最强一手引用优先级**：koishi.chat《可逆的插件系统》全文 > cordiverse/paper 摘要 > dsh 仓库 architecture.md / AGENTS.md / vendor/README.md / approval.md / base cordis.patch.yml 内联注释 > 华尔街见闻发布文全文。
2. 发布日同时出现 paper、Cordis tagline 更新与 dsh 开源，三者互为印证"理论—框架—产品"闭环，可作为本章的叙事主线。
3. 132 插件截图、"插件列表 UI"细节来自发布文内嵌图片（wallstreetcn 转载版可见），引用时注明系发布文截图。
4. 负面/质疑声（InfoQ"编排范式未突破"、chooseai 的迁移风险四条）建议在章节末尾单设"未决问题"小节，保持全书可信度。
5. 注意区分事实与推断：Cordis 的 effect/coeffect 与函数式编程 Algebraic Effects 的关系，Shigma 本人在 disposable.html 文末明确讨论过（"少数函数式编程语言实现了 Algebric Effects……Cordis 在设计上能够与主流的 OOP 语言完美结合"），可据此展开而不必自行揣测。

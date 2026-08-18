# 16. 附录

> 本附录所有内容以 DeepSeek Harness v0.1.0-rc.7 与 Cordis 2026-08 master 快照为准。项目处于 Developer Preview，细节可能随版本变化。

## 16.1 附录 A：Cordis API 速查

> 本附录与 6.5 节的 API 对照表同源——6.5 节是阅读源码时的对照视角，本附录是完整速查版。

**Context**（`@deepseek-ai/cordis`，上游 `cordiverse/cordis` packages/core）：

| API | 说明 |
|---|---|
| `ctx.plugin(plugin, config?)` | 加载插件，创建 Fiber 实例；返回 `Fiber & PromiseLike<Fiber>`。同一插件可多次实例化 |
| `ctx.inject(deps, callback)` | 依赖就绪后执行回调；依赖消失自动卸载、重载自动重启 |
| `ctx.effect(execute, label?)` | 登记一个可逆 effect；`execute` 可返回 disposer、`Promise<disposer>` 或 (Async)Iterable of disposers；卸载时 LIFO 逆序执行 |
| `ctx.provide(name, value?)` | 提供服务实现；本身是一个 effect，返回 disposer |
| `ctx.get(name)` / `ctx.set(name, value)` | 服务/属性存取；未声明 inject 即访问会抛错 |
| `ctx.mixin()` / `ctx.accessor()` | 把子服务的方法/属性以 accessor 投射到 Context 上 |
| `ctx.isolate(name)` | 为服务名生成独立 symbol 键，实现同名服务的命名空间隔离 |
| `ctx.intercept(name, config)` | 沿 Context 原型链分层覆盖服务配置 |
| `ctx.extend(meta?)` | 原型链派生子上下文 |
| `ctx.on(event, fn)` / `ctx.once` | 挂事件监听；返回注销函数（注册即 effect） |
| `ctx.emit(event, ...args)` | 广播观察，不 await |
| `ctx.parallel(event, ...)` | 并行扇出，错误聚合为 AggregateError |
| `ctx.serial(event, ...)` | 按序执行，返回非空即短路（语义同 bail 的异步版） |
| `ctx.bail(event, ...)` | 同步短路 |
| `ctx.waterfall(event, ...args, final)` | 环绕中间件：监听器收 `(...args, next)`，**不调 `next()` 即短路整条链** |

**Fiber**（插件的一次实例化，生命周期状态机）：

| API | 说明 |
|---|---|
| `fiber.dispose()` | 卸载：执行全部 disposer |
| `fiber.restart()` | 卸载并重新执行插件体 |
| `fiber.update(config)` | 热更新配置（走 `internal/update` 瀑布链） |
| `fiber.await()` | 等待 Fiber 达到稳定态 |
| `fiber.getEffects()` | 查看已登记的 effect 清单 |
| `FiberState` | `PENDING / LOADING / ACTIVE / FAILED / DISPOSED / UNLOADING` |

**插件三形态与元数据**：

```ts
type Plugin = Plugin.Function | Plugin.Constructor | Plugin.Object
interface Base {
  name?: string
  Config?: StandardSchemaV1        // 配置校验（schemastery / zod 均可）
  inject?: Inject                  // 依赖声明：数组或带配置的对象
  provide?: string | string[]      // 声明提供的服务名
}
```

**Service 基类**：`class Foo extends Service` 构造时自动 `ctx.reflect.provide(name, this)`；可配 `@Inject()` 装饰器（类级或方法级）。

## 16.2 附录 B：dsh 常用命令与环境变量速查

### 16.2.1 命令

| 命令 | 作用 |
|---|---|
| `npx @deepseek-ai/dsh web` | 启动 Web UI（默认 `http://127.0.0.1:3080`），`dsh --profile web` 的别名 |
| `dsh --profile <name>` | 启动 `$DSH_HOME/profiles/<name>` 下的 profile |
| `dsh --profile headless "任务文本"` | 一次性无头模式：`completed` 退出码 0，否则 1；不开端口 |
| `dsh web --port 8080` | `--port` 属于 web app（启动器只解析自己的 flag，其余透传） |
| `dsh web --trusted-host <主机名>` | 远程部署加受信主机（**不支持 `--host 0.0.0.0`**） |
| `dsh plugin --profile <name> add <spec>` | 安装插件：npm 包 / `github:owner/repo` / `./路径` / `file:` / `link:` |
| `dsh plugin --profile <name> remove/why/update ...` | 直接转发给 profile 目录下的 pnpm |
| `dsh --profile web --dump-config` | 查看组合后的插件树（每行注释标明来源文件），不启动 |
| `dsh --profile web --dump-default-config` | 查看默认组合 |
| `pnpm dsh web --patch ./x/cordis.yml` | 源码运行时叠加 patch 覆盖层 |

源码开发日常：`pnpm run typecheck` / `check:all` / `doc-sync` / `hygiene` / `build`；演示：`pnpm run demo:cordis`（自指 demo）、`pnpm run demo:acp`（ACP server）。

### 16.2.2 配置文件

| 文件 | 用途 |
|---|---|
| `$DSH_HOME/settings.yaml` | 主设置（provider、模型目录等，热重载） |
| `$DSH_HOME/.credentials.yaml` | 密钥存储（只写，Web UI 写入） |
| `$DSH_HOME/profiles/<name>/package.json` + `cordis.patch.yml` | profile 插件依赖与用户补丁层 |
| `$DSH_HOME/cordis.patch.yml` | 全局补丁层（优先级高于单 profile 层） |

凭据解析顺序：环境变量 → `.credentials.yaml` → 调用目录 `.env` → `$DSH_HOME/.env`。

### 16.2.3 环境变量

| 变量 | 作用 |
|---|---|
| `DEEPSEEK_API_KEY` / `DEEPSEEK_BASE_URL` | DeepSeek 凭据与可选端点 |
| `DEEPSEEK_SEARCH_BASE_URL` | web_search 端点 |
| `DSH_PERMISSION_MODE` | 进程级权限预设：`read-only` / `workspace-write` / `danger-full-access` |
| `DSH_TOOLS_MODE` | 工具暴露方式：`native` / `code` / `both` |
| `DSH_TELEMETRY_MODE` | `FULL`（OTLP/HTTP）/ `FEEDBACK_ONLY`；`DSH_TELEMETRY_DISABLED` 硬性关闭（默认全本地、DISABLED） |
| `NODE_USE_ENV_PROXY=1` | 源码运行时走 HTTP 代理 |

关机：SIGINT/SIGTERM 触发最多 5 秒优雅释放，第二次信号强制退出。

## 16.3 附录 C：内置插件清单速查（dsh-base bundle）

`packages/bundle/base/cordis.patch.yml` 是每个 profile 的第一层，约 70 个插件行。任何一行都可被用户 patch 替换（`--dump-config` 查看）。

| 分组 | 插件 |
|---|---|
| 框架 | `cordis-plugin-timer`、`cordis-plugin-hmr` |
| 核心脊柱 | `dsh-llm`、`dsh-session`、`dsh-typert-*`、`dsh-agent`、`dsh-agent-loop`、`dsh-tools`、`dsh-system-prompt`、`dsh-agent-default-model`（默认 `deepseek-official / deepseek-v4-flash`） |
| 模型接入 | `dsh-llm-deepseek`、`dsh-llm-pi-ai`（默认休眠，设置热加载）、`dsh-llm-retry`、`dsh-token-meter` |
| 设置与凭据 | `dsh-settings-file`、`dsh-credentials-local` |
| 持久化 | `dsh-session-persistence-jsonl`、`dsh-session-query-sqlite`（FTS5，`openAt: never`）、`dsh-attachment-local`（图片内容寻址，不入日志） |
| 沙箱与审批 | `dsh-sandbox-local`、`dsh-sandbox-policy`、`dsh-bash-sandbox` / `dsh-pwsh-sandbox`（互斥平台）、`dsh-user-approval`、`dsh-permission-presets`（三档预设） |
| 工具 | `tool-bash` / `tool-pwsh`、`tool-fs`（edit/read/read_image/write）、`tool-fs-search`（glob/grep，内置 ripgrep）、`tool-str-replace-editor`、`tool-todo`、`tool-goal`、`tool-skill`、`tool-subagent(-control/-report)`、`tool-workflow`、`tool-ralph`、`tool-web`（默认禁用，SSRF 考量）、`tool-jobs`、`dsh-plan-mode` |
| 上下文工程 | `dsh-agent-instructions`（AGENTS.md/CLAUDE.md，64KB 上限）、`dsh-compaction-basic`、`dsh-compaction-tool-result-pruner`、`dsh-spill-local` / `dsh-spill-policy`（超大结果外溢）、`dsh-repeat-tool-reminder` |
| 其他 | `dsh-session-telemetry-otel`（默认 DISABLED）、`dsh-session-title(-llm)`、`dsh-skill*`、`dsh-commands`、`dsh-user-questions`、`dsh-session-checkpoint-policy`、`dsh-tool-call-timeout-policy` |

`dsh-web-app` bundle 叠加浏览器应用层（~40 个 React 客户端 UI 插件）；`dsh-headless` 叠加一次性运行器。

## 16.4 附录 D：术语表

| 术语 | 定义 |
|---|---|
| **Harness** | Agent 的"躯体"：模型之外的一切——工具、循环、会话、沙箱、UI。Model + Harness = Agent |
| **Profile** | `$DSH_HOME/profiles/<name>/` 下的具名插件组合：有序 bundle 补丁层的叠加 |
| **Bundle** | Cordis 配置行 + 所挂载代码的分发格式；`package.json` 的 `dsh.profile` / `dsh.bundle` 字段声明身份 |
| **Patch** | 以行为单位（按 `id`）整体替换或插入插件配置行的覆盖层；分层顺序：bundle → profile patch → home patch → `--patch` |
| **Fiber** | Cordis 中插件的一次实例化：有生命周期状态机与副作用清单（DisposableList） |
| **Effect（可撤销效果）** | 每次对上下文的变换都携带逆操作；落地为 `ctx.effect()`，卸载时 LIFO 回滚——"注册即 effect" |
| **Coeffect（反应式余效果）** | 依赖声明驱动运行时反应：`inject` + 服务上下线自动 notify，依赖就绪自启、消失自卸、重载级联重启 |
| **Seam（能力缝）** | 三角色结构：Service Definition（接口）/ Provider（实现）/ Consumer（消费方）；换 provider 可整体迁移 Bash/PTY/LSP 到远程沙箱 |
| **Turn** | 一次用户输入到 agent 停下的完整回合 = 0..n 个 Step |
| **Step** | 一次模型请求 + 其产生的全部工具调用 |
| **Waterfall** | 环绕中间件事件分发：监听器收 `(...args, next)`，调 `next()` 委派下游（可改写参数），不调即短路 |
| **Session 日志** | append-only 的 `SessionEvent` 序列，唯一事实源；模型历史由 `deriveMessages()` 投影 |
| **投影（Projection）** | 从 append-only 日志派生模型可见消息历史的纯函数（dsh 的 `deriveMessages()`）；fork、resume、压缩、审计都是同一日志上的不同投影或投影区间改写 |
| **epoch 机制** | Cordis 内部区分注册代际的递增计数：服务重载或上下文派生时 epoch 递增，旧代际的监听与注册随之失效，保证热重载（HMR）后没有残留副作用 |
| **Model-visible ⟺ logged** | 不变量：凡进入模型请求的内容必须能从会话日志重建，有运行时断言 |
| **Approval（审批）** | 一次性权限询问 seam；fail-closed（`unavailable` = 拒绝）；策略 `ask` / `never` |
| **Sandbox（沙箱）** | `read-only` / `workspace-write` / `danger-full-access` 三档；本地后端 bwrap/Landlock/Seatbelt/Windows ACL，功能探测 fail-closed |
| **Code Mode** | 保留工具 `run_code`：模型写代码，其中子工具调用作为绑定重新进入完整受守护管线 |
| **Provider** | 模型接入适配器（如 `llm-deepseek`、`llm-pi-ai`）；也泛指能力 seam 的实现方 |
| **Typert** | dsh 自研的类型图生成 / RPC 反射系统 |
| **ACP** | Agent Client Protocol，JSON-RPC stdio，连接 Zed 等外部客户端 |

## 16.5 附录 E：参考资料链接汇总

**核心仓库**

- DeepSeek Harness：https://github.com/deepseek-ai/deepseek-harness
- npm 包：https://www.npmjs.com/package/@deepseek-ai/dsh
- Cordis：https://github.com/cordiverse/cordis
- Cordis 论文（preprint）：https://github.com/cordiverse/paper —— *A Programming Paradigm for Spatiotemporal Composability*（未上 arXiv，2026-08 草稿）

**官方发布与一手动态**

- DeepSeek Harness 官网：https://deepseek.com/harness/
- 官方发布文（微信公众号）《DeepSeek Harness 开发者预览版：一切皆插件》：https://mp.weixin.qq.com/s/mANdGRI4fO_sEbC1ECEoZQ
- 崔添翼（tianyi）MIT 开源发布原推：https://x.com/tianyi/status/2087888089759015218
- Jiayuan Zhang 推文（乐高汽车隐喻、自进化软件雏形、函数式风格）：https://x.com/jiayuan_jy/status/2087911060154314963

**官方文档**（仓库内路径，文档站 https://deepseek-harness.github.io/deepseek-harness）

- 架构：`docs/architecture.md`；Cordis 入门：`docs/cordis-primer.md`
- 用户指南：Web UI `docs/user/guide/index.md`、providers `docs/user/guide/providers.md`
- CLI 参考：`apps/cli/reference/README.md`
- 插件教程：`docs/user/develop/basic/index.md`、`tool.md`；扩展烹饪书：`docs/cookbook/`
- 子系统：`docs/subsystems/`（tools / approval / sandbox / compaction）；工具执行管线 `docs/tool-execution-pipeline.md`
- 开发指南与工程宪章：`docs/development.md`、`AGENTS.md`
- 生成目录（权威 schema 来源）：`docs/tool-catalog.md`、`docs/config-catalog.md`、`docs/module-graph.md`

**生态与社区**

- 插件发现：https://github.com/topics/dsh-plugin （截至 2026-08-18 已超 7000 个公开仓库，含 Awesome DSH Plugins 每日兼容性追踪）
- TUI 示例：`dsh-tianshu-tui`；第三方 profile 示例：`github:deepseek-harness/turtle-ui`
- 支持：GitHub Discussions；Discord（英文）；企业微信群 + 微信公众号（中文）

**媒体报道**

- IT之家、网易、财联社等（2026-08-13）：dsh v0.1 与 DeepSeek-V4-Pro 同日开源
- 界面新闻（2026-08-12）："DeepSeek Harness 团队"公众号完成注册认证 —— https://m.jiemian.com/article/14910886.html
- 智东西（2026-08-04，腾讯新闻转载）：内测招募 769 人报名 / 712 个去重仓库 / 120 万+ Stars / 18 个赛道 —— https://view.inews.qq.com/a/20260804A0BP5100
- 量子位（2026-06-22）：崔添翼 5 月起公开招聘 Harness 研究员/工程师/产品经理 —— https://qbitai.com/2026/06/437249.html
- 新浪（2026-08-13）：内测期间社区已产出约 300 个插件 —— https://cj.sina.cn/articles/view/7880069125/1d5b0500506801tym4

**同名混淆提醒**

- PyPI 的 `deepseek-harness` / `deepseek-harness-cli`（0.2.0）是第三方协议探针工具，与官方 DSH **无关**；官方分发渠道只有 npm `@deepseek-ai/dsh` 与 GitHub 仓库。

**相关上游**

- Koishi 机器人框架生态（Cordis 源起，作者 Shigma 团队）；上游依赖 `cosmokit`、`schemastery`、`@cordisjs/*`（loader / hmr / group / timer / include）

## 16.6 附录 F：调研手记与证据快照

本书建立在五篇并行调研报告之上（2026-08-13/14 完成），全文收录于仓库 `book/briefs/`，供修订时溯源核对：

| 手记 | 范围 | 关键快照 |
|---|---|---|
| `book/briefs/ecosystem.md` | Agent Harness 概念生态 | Hashimoto 命名（2026-02）、OpenAI 百万行实验、Composio/Writer 基准数据 |
| `book/briefs/cordis.md` | Cordis 论文与上游源码 | `packages/core/src` 共 1848 行（context.ts 78 行 / fiber.ts 486 行）逐文件剖析 |
| `book/briefs/dsh-source.md` | dsh 仓库源码（rc.5 快照） | README/CONTRIBUTING/AGENTS 要点、monorepo 全景、profile/bundle/patch 机制 |
| `book/briefs/dsh-usage.md` | 安装、使用与插件开发 | npm/源码两条路径、Web UI 四步上手、profile 体系与凭据解析顺序 |
| `book/briefs/design-motivation.md` | 设计动机与技术权衡素材 | Koishi → Cordis 编年史、一手引文与来源 URL 清单 |

**勘误说明**：调研手记写作时将 Cordis 作者 Shigma 与 Harness 团队负责人崔添翼标注为同一人；经复核（量子位、36氪、论文署名等一手来源），两人为不同个体——Shigma 本名史一凡，是 Koishi/Cordis 作者、论文一作；崔添翼为量化背景的团队负责人、论文三作。正文 2.2 节与 15.6 节已按此修正，手记原文保留原貌以存证据链。

**勘误说明（v1.2.0，引用审计）**：初版有两处事实偏差已更正——(1) "约 300 个社区插件"的时间归属误写为"开源后几天内"，经新浪等多源核实应为内测期间即已产出，开源后呈爆发式增长（GitHub topic `dsh-plugin` 2026-08-18 实时计数超 7000）；(2) 第 1 章论据三"Cursor 46% vs 80%（2024）"未找到可回查的原始出处，已替换为可核验的 Endor Labs 交叉基准数据（同一 GPT-5.5 跨 Codex/Cursor harness 61.5% vs 87.2%）。

**勘误说明（v1.3.0，全事实溯源审计）**：(1) 更正 v1.2.0 的一处误改——公众号注册时间初版记为 07-06 实为**正确**（21 世纪经济报道查证公众号信息：2026-07-06 注册、07-07 完成认证），v1.2.0 误将"媒体集中报道时间"（08-11/12）当作注册时间，本版恢复并补强来源；(2) 第 1 章 OpenAI 文章标题误作《Harnessing Codex...》，正名为《Leveraging Codex in an Agent-First World》（Ryan Lopopolo，2026-02-11）；(3) SWE-bench SOTA 实证日期由 2025-11 更正为 2025-01（Erik Schluntz《Raising the Bar on SWE-bench Verified with Claude 3.5 Sonnet》），表述由"仅靠工具描述"修正为"工具描述与脚手架设计"；(4) 论文二作单位"北京大学张玮组"更正为"北京大学张伟（Wei Zhang）组"（36 氪核实）；(5) Pi Agent 的 README 引语"this one is yours"未能在官方仓库核实，已替换为 README 可核验原文（四个工具、No MCP、No sub-agents、No permission popups）；(6) Koishi 起点由"2020 年 1 月"修正为"仓库创建于 2019 年 12 月、2020 年初发布"（GitHub 仓库元数据）；(7) Pi 的 stars 数更新为 9 万+（2026-08-18 实时计数）。此外，TSY Capital"Rust 自研低延迟交易系统"（钛媒体 2026-05-23）、chooseai 迁移风险分析（chooseai.net 2026-05-21）、宋斐"统一脚手架"论述（21 世纪经济报道 2026-08-11）、第一财经开发者评价（2026-08-13）等引语均已逐字回溯到原始报道并补充链接。

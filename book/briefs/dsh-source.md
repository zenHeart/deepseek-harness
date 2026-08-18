# DeepSeek Harness（dsh）源码级调研报告

调研对象：<https://github.com/deepseek-ai/deepseek-harness>（`master`，版本 `0.1.0-rc.5`，MIT 协议）。所有论断均来自直接克隆仓库后的源码与文档阅读，引用格式为「文件路径（仓库相对路径）」；对应 URL 为 `https://github.com/deepseek-ai/deepseek-harness/blob/master/<路径>`。

## 一、项目总览（README / CONTRIBUTING / AGENTS 要点）

**README.md** 要点：
- 定位：`dsh` 是 DeepSeek AI 开源的 agent harness，架构上 **"everything is a plugin"**，由 **Cordis** 框架驱动（Cordis 的设计见论文 *A Programming Paradigm for Spatiotemporal Composability*）。
- 状态：developer preview，**明确声明会有兼容性破坏变更**。
- 运行方式：`npx @deepseek-ai/dsh web`（Web UI 默认 `http://127.0.0.1:3080`）；源码运行需 `pnpm install && pnpm run build && pnpm dsh web`。
- 插件生态：给插件仓库打 `dsh-plugin` topic 以便发现。

**CONTRIBUTING.md** 的特殊立场：暂不接受外部 PR（团队很小），鼓励以「写插件、写博客、答问题」的方式参与生态——"You may consider this repository an idea, an official showcase, and a source of inspiration, but not a mandate."

**AGENTS.md**（149 行）是工程宪章，核心约定：
- 每个 npm 包命名 `@deepseek-ai/dsh-<name>`；vendored 包重新 scoped 且 `private: true`；`@deepseek-ai/cordis` 是每个 harness 包的 peerDependency。
- 全 ESM（`"type": "module"`）；CLI 源码启动走 `node --import tsx/esm`。
- **"Registrations are effects"**：一切注册经 `ctx.effect()` / `ctx.on()`，注册函数返回 disposer。
- 类型化事件用 TypeScript declaration merging 扩展；`SessionEventMap` 成员默认 required-on-read。
- **"Model-visible ⟺ logged"**：凡进入模型请求的内容必须能从 session 日志重建；新增模型可见输入必须新增 session 事件。
- **"Plugins, not loop changes"**：新行为挂到文档化的扩展点上；改动 `agent-loop` 必须同步更新 `docs/architecture.md`。
- Waterfall 监听器必须调用 `next()` 委派，否则短路整个链。
- 发布前立场：优先正确地基而非兼容垫片，`SESSION_FORMAT_VERSION` 保持 `0`，SQLite 用单调 `SCHEMA_VERSION`。

## 二、Monorepo 结构

根 `package.json`（`@deepseek-ai/dsh-root@0.1.0-rc.5`）：pnpm 11.7 workspace，Node 引擎 `^22.19.0 || >=24.0.0`，包管理器 `pnpm@11.7.0`。workspace 成员（`pnpm-workspace.yaml`）：

```yaml
packages:
  - vendor/*                  # vendored Cordis 框架层
  - packages/*/*              # 全部 harness 包（<group>/<pkg> 两层结构）
  - native/landlock-run       # Linux Landlock 原生启动器
  - apps/*                    # apps/cli 拥有 dsh bin；apps/web 为 Web 入口
  - website                   # VitePress 文档站
  - examples                  # 仅依赖解析用途的可运行 cordis.yml 叶子
  - python/sdk-runtime        # 单 exe 构建的闭包根
```

`AGENTS.md` 给出的 `packages/` 分组即**职责地图**：

| 分组 | 职责 |
|---|---|
| `core/` | 产品 API 脊柱：`session`（事件溯源日志）、`system-prompt`（提示词装配）、`tools`（工具注册表+执行管线）、`agent`（Agent 接口/注册表）、`agent-loop`（默认驱动）、`scope`（agent 作用域注册原语） |
| `llm/` | 模型接入层：`llm`（消息/流词汇 + adapter seam，`ctx.llm`）、`llm-deepseek`（DeepSeek 官方适配器）、`llm-pi-ai`（多 provider 适配器）、`llm-retry`、`token-meter` |
| `shell/`、`subprocess/`、`terminal/` | bash/pwsh 能力 seam + 本地/pwsh provider + 持久 PTY 会话 |
| `fs/` | 文件系统 seam + 策略（`tool-fs`、`tool-fs-search`、`fs-observation-policy`） |
| `sandbox/` | 进程沙箱 seam + 本地后端（bwrap/Landlock/Seatbelt/Windows ACL） |
| `compaction/` | 上下文压缩 seam + `compaction-basic` + `tool-result-pruner` + `/compact` 命令 |
| `subagent/`、`workflow/`、`plan/`、`preset/`、`goal/`、`todo/` | 子智能体、持久工作流、计划模式、agent 预设、目标管理、todo 工具 |
| `session/` | 持久化（JSONL/SQLite）、projection、标题、telemetry（OTLP） |
| `interaction/` | 审批（`user-approval`）、权限预设、命令系统、ask-user |
| `web/`、`client/`、`host/`、`api/` | Web 服务能力 seam、~40 个 React 客户端 UI 插件、BFF/API 网关 |
| `boot/`、`bundle/`、`apps/` | 启动胶水与 profile 分层 bundle；`apps/cli` 提供 `dsh` bin |
| `e2b/`、`lsp/`、`mcp/`、`hooks/`、`acp/`、`sdk/` | E2B 沙箱 POC、语言服务器、MCP、Claude Code/Codex hook 桥、ACP 自动化协议、TS/Python SDK |
| `typert/` | 自研的类型图生成/RPC 反射系统（"Typert"） |

每个包的 `package.json` 描述精炼地说明了职责，例如：
- `@deepseek-ai/dsh-agent-loop`：*"The concrete agent loop plugin for the DeepSeek Harness"*
- `@deepseek-ai/dsh-session`：*"Event-sourced session store"*
- `@deepseek-ai/dsh-tools`：*"Tool registry and execution pipeline"*
- `@deepseek-ai/dsh-llm-deepseek`：DeepSeek provider 适配器，默认路由 `deepseek-official`，模型目录默认 `deepseek-v4-flash` / `deepseek-v4-pro`（`packages/llm/llm-deepseek/src/index.ts`）。

## 三、插件系统源码级分析

### 3.1 Cordis 是什么、如何用

Cordis 是**源码 vendored 进仓库**的插件框架（`vendor/cordis`，上游 `cordiverse/cordis@4.0.0-rc.7`，重新 scoped 为 `@deepseek-ai/cordis`）。`vendor/README.md` 解释 vendoring 动机："so that the harness fully owns its framework layer (auditable, patchable, pinned)"；连 `cosmokit`、`schemastery`（Zod 式校验库）、`loader`、`hmr`、`group`、`timer`、`include`、`logger-console` 也一并 vendored。

`docs/cordis-primer.md` 总结 Cordis 五要素：
1. **插件即 Service 对象**——一个带可选 `inject` 与 `apply(ctx)` 的函数，或 `Service` 子类；
2. **Context 即服务仓库**——服务占据稳定的 `ctx.<key>`（`ctx.tools`、`ctx.llm`、`ctx.sessions`…），按 key 而非具体实现相互发现；
3. **`inject` 声明依赖**——加载顺序由服务可用性驱动，而非手工 boot 排序（base bundle 注释原文："Row order carries no load semantics (activation is service-availability driven)"）；
4. **类型化事件**——declaration merging 声明事件名，以 `emit` / `waterfall` / `parallel` / `serial` 四种模式分发（`waterfall` 是 around-middleware：监听器收 `(...args, next)`，不调 `next()` 即短路）；
5. **注册即可逆 effect**——卸载插件时自动回滚。

### 3.2 组合机制：Profile / Bundle / Patch 三层

`docs/architecture.md`「Profiles and bundles」：一次运行的 `dsh` 是 boot 时从**有序分层**组合出的插件树。
- **Profile**：存在 Harness home 中的具名组合，列出所叠 bundle、树外插件、用户自己的 `cordis.patch.yml`；官方提供 `web` 与 `headless` 模板。
- **Bundle**：Cordis 配置行 + 所挂载代码的分发格式；`package.json` 的 `dsh.profile`/`dsh.bundle` 字段声明身份。
- 分层顺序：profile 列出的各 bundle → profile 的 `cordis.patch.yml` → home 级 patch → `--patch` overlay；patch 按行 `id` 整体替换 config 或插入新行。
- 可用 `dsh --profile web --dump-config` 查看实际启动树，**任何一行都可被用户 patch 替换**。

Loader 配置支持 `!!js` 表达式插值（`cordis-primer.md`「Loader Configuration」），bundle 里大量使用，例如按平台禁用 bash/pwsh：

```yaml
# packages/bundle/base/cordis.patch.yml
- id: tool-bash
  name: '@deepseek-ai/dsh-tool-bash'
  disabled: !!js process.platform === 'win32'
- id: sandbox-policy
  name: '@deepseek-ai/dsh-sandbox-policy'
  config:
    mode: !!js process.env.DSH_PERMISSION_MODE ?? 'workspace-write'
    workspaceRoot: !!js process.cwd()
- id: approval
  name: '@deepseek-ai/dsh-user-approval'
  config:
    policy: !!js "(process.env.DSH_PERMISSION_MODE ?? 'workspace-write') === 'danger-full-access' ? 'never' : 'ask'"
```

### 3.3 内置插件清单（dsh-base bundle）

`packages/bundle/base/cordis.patch.yml` 是每个 profile 的第一层，约 **70 个插件行**，按职能分组（均为实际文件内容）：

- **框架**：`cordis-plugin-timer`、`cordis-plugin-hmr`
- **核心脊柱**：`dsh-llm`、`dsh-session`、`dsh-typert-*`、`dsh-agent`、`dsh-agent-loop`、`dsh-tools`、`dsh-system-prompt`、`dsh-agent-default-model`（默认 `deepseek-official / deepseek-v4-flash`）
- **模型接入**：`dsh-llm-deepseek`、`dsh-llm-pi-ai`（默认休眠，设置文件热加载 provider profile）、`dsh-llm-retry`、`dsh-token-meter`
- **设置与凭据**：`dsh-settings-file`（`$DSH_HOME/settings.yaml` 热重载）、`dsh-credentials-local`（环境变量 + `$DSH_HOME/.credentials.yaml` + `.env`）
- **持久化**：`dsh-session-persistence-jsonl`、`dsh-session-query-sqlite`（FTS5 全文搜索默认 `openAt: never`）、`dsh-attachment-local`（图片字节存内容寻址引用，不入日志）
- **沙箱与审批**：`dsh-sandbox-local`、`dsh-sandbox-policy`、`dsh-bash-sandbox`/`dsh-pwsh-sandbox`（互斥平台）、`dsh-user-approval`、`dsh-permission-presets`（`read-only` / `workspace-write` / `danger-full-access` 三档预设）
- **工具**：`tool-bash`/`tool-pwsh`、`tool-fs`（edit/read/read_image/write）、`tool-fs-search`（glob/grep，内置 ripgrep）、`tool-str-replace-editor`、`tool-todo`、`tool-goal`、`tool-skill`、`tool-subagent(-control/-report)`、`tool-workflow`、`tool-ralph`、`tool-web`（默认禁用，SSRF 考量）、`tool-jobs`、`dsh-plan-mode`（`exit_plan_mode`）
- **上下文工程**：`dsh-agent-instructions`（加载 AGENTS.md/CLAUDE.md，64KB 上限）、`dsh-compaction-basic`、`dsh-compaction-tool-result-pruner`、`dsh-spill-local`/`dsh-spill-policy`（超大结果外溢存储）、`dsh-repeat-tool-reminder`
- **其他**：`dsh-session-telemetry-otel`（OTLP 遥测，默认 DISABLED）、`dsh-session-title(-llm)`、`dsh-skill*`、`dsh-commands`、`dsh-user-questions`、`dsh-session-checkpoint-policy`、`dsh-tool-call-timeout-policy`

`dsh-web-app` bundle（`packages/bundle/web-app/cordis.patch.yml`）再叠加浏览器应用层；`dsh-headless` 叠加无服务器一次性运行器。

### 3.4 工具（tools）的定义方式

工具 = `ToolDefinition`（`docs/subsystems/tools.md` + `packages/core/tools/src/index.ts`）：模型可见的 `ToolSchema`（name/description/parameters）+ **强制**的 canonical 输出声明 `output: { schema, render, presentationMeta? }` + `execute(args, exec)` + 可选 `finalizeContent`（最后一道同步内容不变量）、`timeoutMs`、`isConcurrencySafe`（并行分类器）、UI 呈现器 `presentCall/presentResult`。注册表 `schemas()` 用显式白名单构造模型可见 schema，宿主侧字段绝不泄漏进模型请求。

第一方工具用 `defineTool` DSL 编写并注册（真实代码，`packages/fs/tool-fs/src/edit.ts:83`）：

```ts
ctx.tools.register(defineTool({
  name: 'edit',
  description: 'Edit an existing UTF-8 text file by replacing literal text.',
  parameters: {
    file_path:  { type: 'string', required: true, description: 'Path to edit, resolved by the filesystem backend.' },
    old_string: { type: 'string', required: true, description: 'Literal text to replace. Must match exactly.' },
    new_string: { type: 'string', required: true, description: 'Literal replacement text...' },
    replace_all:{ type: 'boolean', description: 'Replace all matches. Defaults to false; ...' },
    ...sandbox.escalationModes.length > 0 ? sandbox.schemaFields() : {},
  },
  output: {
    schema: { type: 'object', additionalProperties: false, properties: {
      path: { type: 'string', required: true },
      before: { type: 'string', required: true },
      after: { type: 'string', required: true } } },
    render: (args, value) => [{ type: 'text', text: formatEditOutput(...) }],
    presentationMeta: (args, value) => ({ diffs: computeHunkDiffs(...) }),
  },
  async execute(args: EditToolArgs, exec) {
    const sandboxPolicy = await sandbox.resolvePolicy('edit', args, exec)
    const target = await ctx.fs.resolve(input.filePath, ...)
    const intent = await ctx.waterfall('fs/edit-intent', target, exec, () => undefined)
    outcome = await ctx.fs.editText(target, {...}, intent, exec.signal, sandboxPolicy)
    ctx.emit('fs/observed', target, { kind: 'present', version: outcome.version }, exec)
    return { path: target.displayPath, before: outcome.before, after: outcome.after }
  },
}))
```

`register()` 支持**作用域注册**（scoped tools 遮蔽全局；保留名 `run_code` 永不可注册，因为它是 Code Mode 传输层）与 `restrict()`（agent 作用域的 allow/deny 过滤），均返回 disposer（`packages/core/tools/src/index.ts:1037`）。

**Code Mode**：`dsh-tools` 自带保留工具 `run_code`——模型写一段代码（TS/Python SDK），其中对子工具的调用作为绑定重新进入完整受守护工具管线，子调用记录 `tool/code-dispatch` 事件（`docs/tool-catalog.md`、`packages/core/tools/src/code-mode.ts`）。

### 3.5 权限与审批机制

双层结构（`docs/subsystems/approval.md`、`docs/subsystems/sandbox.md`）：

1. **审批 seam（`ctx.approval`，`dsh-user-approval`）**：一次性权限询问。`ApprovalOutcome` 封闭且 fail-closed：`'allowed-once' | 'rejected' | 'cancelled' | 'unavailable'`；应答者缺失/抛错 → `unavailable` → 拒绝。会话级策略 `'ask' | 'never'`（never 在服务内部强制执行，连 prepend 应答者也绕不过），每次询问记 `approval/asked` + `approval/decided` 审计对（log-only，不进模型 transcript）。工具管线中 `tools/pre-execute` waterfall 返回 `ask` 时由 `ctx.approval` 弹出一次性询问。
2. **沙箱 seam（`ctx.sandbox`）**：`SandboxMode = 'read-only' | 'workspace-write' | 'danger-full-access'`；每次调用携带完整 `SandboxExecutionPolicy`（mode + canonicalized workspaceRoot + sessionId），支持并发会话不同边界、审批后一次性提权重试。后端报告 `SandboxEnforcement: 'full' | 'partial'`（旧 Landlock ABI、Windows ACL 为 partial，消费者必须区分）。本地后端：Linux bwrap / Landlock（自研 `native/landlock-run` 原生启动器，三包 npm 家族按平台 optional dep 分发）、macOS Seatbelt、Windows ACL restricted-token——**功能探测、fail-closed**。此外还有单调 guard（deny 或弃权）与 `fs/write-intent` / `fs/edit-intent` 事件门（read-before-write 策略由 `dsh-fs-observation-policy` 插件实现）。

## 四、关键代码片段

### 4.1 Agent 主循环（`packages/core/agent-loop/src/agent.ts`）

`ReactLoopAgent` 是默认驱动：inbox 收件箱 + turn/step 相位机。核心循环（`agent.ts:246-330`，节选）：

```ts
private async turn(): Promise<boolean> {
  ...
  this.session.append('turn/start', { turn })
  while (true) {
    signal.throwIfAborted()
    const decision = await this.preStep(target, { turn, step })   // agent/pre-step waterfall
    if (decision.kind === 'reject') { turnEnds = { kind: 'blocked' }; return false }
    this.session.append('step/start', { turn, step })
    try {
      for (const message of decision.messages) {
        this.session.append('user/message', message, { surfaceOp: 'append' })
      }
      const stepEnd = await this.step(decision.assembly)
      ...
    } finally { this.session.append('step/end', { turn, step }) }
    if (turnEnds && this.inbox.nextStep.length === 0) {
      await this.dispatch.serial('agent/turn-stopping', { turn, signal })
    }
    if (turnEnds && this.inbox.nextStep.length === 0) break
    target = 'next-step'
  }
  ...finally { this.session.append('turn/end', { turn, reason: turnEnds! }) }
}
```

单步 = 一次模型请求 + 其工具调用（`agent.ts:332-401`）：

```ts
private async step(assembly: PromptAssembly): Promise<StepEndReason | null> {
  const system = renderPrompt(assembly)
  while (true) {
    const { request, preparedCall } = await this.buildRequest(
      turn, step, assembly.tools, system, this.session.deriveMessages(), signal)
    const assembler = new BlockAssembler()
    const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
    for await (const chunk of stream) {
      chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
      assembler.push(chunk)
    }
    // 失败走 agent/request-error waterfall，可返回 retry action
    ...
    this.session.append('assistant/message', { turn, step, message, usage },
      { surfaceOp: 'append', sourceEventSeqs: chunkSeqs })
    if (finish.kind === 'max-tokens') return { kind: 'max-tokens' }
    const toolCalls = message.content.filter(b => b.type === 'tool-call')
    if (toolCalls.length === 0) return { kind: 'completed' }
    const { concluded } = await executeToolCalls(this.loopCtx, turn, step, toolCalls, signal,
      context => this.inbox.splice('next-step', this.inbox.nextStep.length, 0, [context]))
    return concluded ? { kind: 'completed' } : null
  }
}
```

输入路径：`followup()` → next-turn 唤醒；`steer()` → next-step 唤醒；`inject()` → next-step 不唤醒（等下一条消息带入）。`buildRequest` 把 `agent/request` waterfall 提出的 config 交给 `ctx.llm.prepareCall()` 绑定适配器，并把 `request/header`、`request/context` 记入日志——**每次模型请求都可从日志重建**。

### 4.2 工具调用调度（`packages/core/agent-loop/src/tool-calls.ts`）

并行工具调用用「有界滚动池 + 屏障」：exclusive 调用形成屏障，parallel 调用在 `maxParallelToolCalls` 上限内重叠 dispatch，但**结果永远按模型顺序提交**；每组开始前重读 `ctx.tools.executionMode()`（注册表变更可即时制造屏障）；abort 时为未启动调用补写合成错误结果（`tool call aborted before dispatch`）保证 replay 合法：

```ts
const fillPool = async (): Promise<void> => {
  while (!aborted && nextToStart < group.length && inFlight.size < maxParallelToolCalls) {
    if (nextToStart > 0 && mode === 'parallel'
      && ctx.tools.executionMode(nextCall.exec).kind !== 'parallel') break
    await startCall(nextToStart++)
    await commitReady()   // 只跨连续的模型序 slot 前进
  }
}
```

### 4.3 工具执行管线（`docs/tool-execution-pipeline.md`）

顺序：`tool/call`（执行前先落日志）→ `tools/pre-execute` waterfall（hooks、权限、沙箱）→ 单调 guards → `ctx.approval` 一次性询问（ask 时）→ `tools/execute` around-waterfall（超时、重试、指标包装）→ 工具体（其中 fs 变更再过 `fs/write-intent`/`fs/edit-intent`）→ `tools/post-execute` waterfall（可 accept/block/replace/追加 context）→ 注册表外层归一化（快照失败转 isError）→ `finalizeContent` → `tools/result` 同步通知 → `tool/result` 落日志 → 追加 context FIFO 注入。三层 waterfall 均可改写调用；denied 也走 post。

### 4.4 模型接入层（`packages/llm/llm/src/index.ts` + `packages/llm/llm-deepseek/`）

`ctx.llm` 是 adapter seam：`registerAdapter(providers, adapter)` 注册 provider 路由并返回 disposer；`prepareCall(config, signal)` 把请求 config 绑定到具体适配器并解析 exact-model 默认值（contextWindow、adapterDefaults）。DeepSeek 适配器（`llm-deepseek/src/index.ts`）特点：
- **连接事实按请求解析**而非加载时冻结：`llm-deepseek` 设置段（settings.yaml，Web Models 页可写）热覆盖 base URL/目录/key，改动下一请求即生效，进行中的流保留其起始快照；
- API key 经 `ctx.credentials` 按请求解析（默认 `DEEPSEEK_API_KEY` 环境变量），缺 key 时请求报 `MISSING_CREDENTIAL` 而非插件加载失败；
- 默认目录 `deepseek-v4-flash` / `deepseek-v4-pro`，默认 contextWindow 1,000,000、maxTokens 256,000、reasoningEffort `high`，支持 `streamIdleTimeoutMs`（默认 5 分钟）与 provider 自有 retryPolicy；SSE 流解析在 `sse.ts`，消息翻译在 `translate.ts`。

### 4.5 上下文压缩（`docs/subsystems/compaction.md` + `packages/compaction/compaction-basic/`）

压缩是**可选能力 seam**而非 loop 脊柱：Service Definition（`dsh-compaction`，`ctx.compaction`）+ Provider（`compaction-basic`：token-meter 驱动的压力策略 + LLM 摘要后端）+ 人类消费者（`/compact` 命令）。机制：
- 三个 log-only 事件 `compaction/start | summary | end` 构成可检测崩溃的锁（孤儿 start 可识别）；
- 摘要本身复用 `user/message` 并以 `surfaceOp: { op: 'replace', start, end }` 做**唯一一次 surface 替换**——模型历史是日志的投影，压缩即替换投影区间；
- 自动压力压缩挂在 serial `agent/pre-step`（请求派生前）；canonical context-overflow 走 `agent/request-error` 恢复路径，仅当 surface 替换代数前进时才返回 retry；
- 摘要前先跑可选的 `toolResultPruner`（无模型、replay-safe 的头/中/尾裁剪），重新计量后可能无需摘要即推进；
- 区间边界保持 tool-call/result 配对（`toolPairingBalancedBefore/After`），允许一个超限 turn 的早期已闭 step 先行压缩。

## 五、底层工具链（以源码为准）

| 层 | 依赖/机制 | 证据 |
|---|---|---|
| 文件搜索 | **`@vscode/ripgrep`** 打包的 rg 二进制，经 `ctx.subprocess` 前台调用，无宿主安装依赖、不过 shell 层 | `packages/fs/tool-fs-search/package.json`、`src/glob.ts`、`src/grep.ts` |
| 终端 PTY | **`node-pty`** 持久 PTY 会话 seam（`ctx.terminals`），六个 `terminal_*` 工具 | `packages/terminal/*`（`terminal-bash/src/sanitize.ts` 消费 node-pty 数据块） |
| 沙箱 | Linux **bwrap / Landlock**（自研 native addon `native/landlock-run`，Rust/原生启动器，三平台 npm 包）、macOS **Seatbelt**、Windows **ACL restricted-token**（`sandbox-windows-acl`） | `docs/subsystems/sandbox.md`、`native/README.md` |
| diff | `diff@^9`（npm jsdiff）：`tool-fs` 的 edit 结果 hunk 计算与 UI diff 卡片 | `packages/fs/tool-fs/package.json`、`packages/client/ui-trajectory/package.json` |
| 存储 | SQLite（FTS5 全文检索，单调 `SCHEMA_VERSION`）：`session-query-sqlite`、`session-persistence-sqlite`、`storage-sqlite`；JSONL 为默认 session 持久化 | `packages/session-query/session-query-sqlite` 等 |
| Web 栈 | React + Vite（`apps/web`）、`ws`（WebSocket 双工：HTTP 上/WebSocket 下）、`@tanstack/react-virtual`、`shiki` 语法高亮、VitePress 文档站 | `packages/client/*`、`apps/web` |
| 遥测 | OpenTelemetry JS（OTLP/HTTP logs），默认 DISABLED，env 显式开启 | `packages/session/session-telemetry-otel`（bundle 配置注释详述） |
| 远程沙箱 | **E2B**（`e2b` SDK）：fs/subprocess 的 E2B provider | `packages/e2b/*` |
| 校验/Schema | vendored **schemastery**（`z.object` 风格）用于所有插件 `Config` 运行时校验 | 各包 `import z from '@deepseek-ai/schemastery'` |
| 构建/测试 | TypeScript project references 双聚合（host/client）+ **tsdown** 打包、tsx ESM 启动、**vitest**（unit/e2e/snapshot/web/perf/stress 六套配置）、oxlint、knip、jscpd、lefthook、playwright | 根 `package.json`、`docs/development.md` |
| 子代理互操作 | `@agentclientprotocol/sdk`（ACP）、`@anthropic-ai/claude-agent-sdk`、`@openai/codex`（hook 桥/委托） | 根 devDependencies、`packages/hooks`、`packages/acp` |
| 自研基础设施 | **Typert**（类型图生成 + RPC 反射 + 设置表单 schema 层）、`code-runtime`（worker-thread 代码执行 seam，Code Mode 后端） | `packages/typert`、`packages/code-runtime` |

值得注意：**没有 tree-sitter/AST 工具进入模型侧**；代码理解走 LSP seam（`packages/lsp`）。也没有自研 diff 应用算法——edit 是字面量替换，`str_replace_editor` 是独立 view/create/replace/insert 工具。

## 六、架构精要（可直接入书的描述）

1. **Model + Harness = Agent 的工程实现**：loop 本身（`agent-loop`）也只是注册在 `ctx.agentLoop` 的一个可替换插件；替换模型适配器、工具注册表、session 日志乃至循环，都只是改一行配置。
2. **Session 日志是唯一事实源**：`Session` 是 append-only 的 `SessionEvent` 日志（`turn/*`、`step/*`、`user/message`、`assistant/chunk|message`、`tool/call|result`、`request/header`、merge 扩展事件…），模型历史由 `deriveMessages()` 投影；fork/resume/transcript/telemetry/压缩全部从同一流派生。"Model-visible means logged" 由运行时不变量断言。
3. **三个事件域**：durable session 事件（重放事实）、live `agent/*` 事件（协调/拦截在飞工作）、capability 事件（`fs/*`、`tools/*` 挂策略）。`agent/pre-step`、`agent/request`、`llm/stream`、`tools/pre|execute|post-execute` 是 waterfall 拦截点；`agent/turn-stopping` 是 serial 终点检查点。
4. **Capability seam 三角色**：Service Definition（接口）/ Provider（实现）/ Consumer（通常是模型侧工具）。fs 与 subprocess 共享一个"执行世界"，把 provider 指向远程沙箱即可整体搬走 Bash/PTY/LSP——无需 fork provider。
5. **安全姿态**：审批 fail-closed（unavailable=拒绝）、沙箱后端功能探测 fail-closed、enforcement `partial` 必须显式上报、`never` 策略在服务内部不可绕过、guard 单调（deny-or-abstain）、read-before-write 经 `fs/*-intent` 事件门。

## 附：不确定性说明
- 调研基于撰写时 `master` 浅克隆（v0.1.0-rc.5）；项目自述快速迭代、将有破坏性变更，细节（默认模型名、bundle 行集）可能随版本变动。
- `docs/tool-catalog.md`、`docs/config-catalog.md`、`docs/module-graph.md` 等为生成文件（`scripts/gen-*.ts` 生成并 CI 校验新鲜度），是引用工具 schema/配置字段的权威来源。
- 全部事实来自源码/文档直接阅读（观察事实）。

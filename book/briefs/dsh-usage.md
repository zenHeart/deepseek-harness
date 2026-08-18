# DeepSeek Harness（dsh）使用与插件开发调研 Brief

> 项目处于 **Developer Preview（v0.1）**，官方明确警告：**THERE WILL BE COMPATIBILITY-BREAKING CHANGES**。所有 API、配置字段、目录结构均可能随版本变化。

## 0. 项目定位

DeepSeek Harness（`dsh`）是 DeepSeek AI 开源（MIT）的 Agent Harness，核心理念 **"Everything is a Plugin"**，底层由插件框架 [Cordis](https://github.com/cordiverse/cordis) 驱动。模型适配器、工具注册表、会话日志、Agent 循环本身全部是插件，均可从配置层替换。产品定位对标 OpenAI Codex / Anthropic Claude Code。
来源：https://github.com/deepseek-ai/deepseek-harness

## 1. 安装与运行

### 1.1 通过 npm 运行

前置条件：Node.js（仓库开发要求 Node 22.19+ / 24+）。

```sh
npx @deepseek-ai/dsh web
```

启动 Web UI，默认 `http://127.0.0.1:3080`。`dsh web` 是 `dsh --profile web` 的硬编码别名。

### 1.2 从源码运行

```sh
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install        # 仓库固定 pnpm@11.7.0，建议先 corepack enable
pnpm run build      # 生产运行必须先构建（含前端产物）
pnpm dsh web        # 以 tsx 运行 TypeScript 入口并透传所有参数
```

注意：`pnpm dsh` 用 `node --import tsx/esm` 直接跑源码，**不检查产物新旧**——前端 bundle 过期时会继续跑旧代码，需要重新 `pnpm run build`。需要 HTTP 代理时设 `NODE_USE_ENV_PROXY=1`。
来源：apps/cli/reference/README.md

### 1.3 CLI 入口模式（profile 体系）

`dsh` 是"profile 启动器"：一个 profile 是 `$DSH_HOME/profiles/<name>/` 下的一组有序插件 bundle 补丁层。

| 命令 | 作用 |
|---|---|
| `dsh --profile <name>` | 启动 `$DSH_HOME/profiles/<name>` 下的 profile |
| `dsh --profile headless "任务文本"` | 一次性无头模式：新建持久会话、执行任务、打印最终答案并退出（`completed` 退出码 0，否则 1；不开端口、不写 stderr） |
| `dsh web` | `--profile web` 的别名；支持 `--host`、`--port`、可重复的 `--trusted-host` |
| `dsh plugin --profile <name> <pnpm 参数>` | 管理 profile 插件，直接转发给 profile 目录下的 pnpm（add/remove/why/update 等） |

其他要点：
- `web` 和 `headless` profile 首次使用会从自带模板自动初始化；其他名字必须先 `dsh plugin --profile <name> add ...` 创建。
- 调用目录即默认工作区根目录；启动时加载 `AGENTS.md`/`CLAUDE.md`（渲染预算 65,536 字节）。
- 启动器只解析自己的 flag，第一个不认识的 token 起全部交给 app：`dsh --profile web --port 8080`（`--port` 属于 web app）。
- **当前不支持 `--host 0.0.0.0`**；远程部署用 `--trusted-host` 加受信主机名。
- `--dump-default-config` / `--dump-config` 可在不启动的情况下查看组合后的插件树，每行注释标明来源文件。
- 官方自带 bundle：`@deepseek-ai/dsh-base`、`@deepseek-ai/dsh-web-app`、`@deepseek-ai/dsh-headless`。第三方 profile 示例：`dsh --profile tui --resume <id>`（社区已有 `dsh-tianshu-tui` 等 TUI 插件）。
- 关机：SIGINT/SIGTERM 触发最多 5 秒优雅释放，第二次信号强制退出。

### 1.4 Web UI 使用流程（http://127.0.0.1:3080）

1. **配置模型**：Settings → Models，填入 DeepSeek API key 保存，路由立即生效，**无需重启**。
2. **选择工作区**：点 "Choose workspace"，添加并选中启动 `dsh` 的项目目录；未选工作区前会话输入框不可用。
3. **运行任务**：新建会话发送如 "Summarize this repository and identify its main packages." Agent 可读写工作区文件、执行命令、委派子任务、维护计划；触及权限策略需审批的操作时，Web UI 弹确认。
来源：docs/user/guide/index.md

## 2. 配置（Provider / Key / 文件 / 权限）

### 2.1 模型 Provider

- **DeepSeek**：Settings → Models 页只有 DeepSeek 卡片，一个 API-key 输入框。密钥**只写**（保存后页面只拿到脱敏描述符），实际存于 `$DSH_HOME/.credentials.yaml`。
- **目录内置 provider**：点 "Add provider"，选 Anthropic、OpenAI 等，填 key 即可。**原生鉴权的 provider 例外**：Bedrock 需 AWS 凭据+region、Vertex 需 ADC project、Azure 需 api-version、Codex 需 OAuth。
- **自定义 provider**（公司网关/自托管）：选 "Add a custom provider"，填小写 Provider ID（**永久不可改**）、base URL、API 协议（如 `openai-completions`）、凭据和至少一个模型。可用 "Fetch available models" 通过 `GET /models` 拉取模型清单。

### 2.2 配置文件位置与格式

- `$DSH_HOME/settings.yaml`：主设置文件。
- `$DSH_HOME/.credentials.yaml`：密钥存储（由 Web UI 写入）。
- `$DSH_HOME/profiles/<name>/package.json` + `cordis.patch.yml`：profile 的插件依赖与用户补丁层；`$DSH_HOME/cordis.patch.yml` 为全局层（优先级高于单 profile 层）。
- 凭据解析顺序：继承的环境变量 → `$DSH_HOME/.credentials.yaml` → 调用目录的 `.env` → `$DSH_HOME/.env`。源码开发时：

```sh
DEEPSEEK_API_KEY=sk-...
DEEPSEEK_BASE_URL=https://...   # 可选
```

- 自定义 provider 的 `settings.yaml` 示例（含视觉模型声明）：

```yaml
llm-pi-ai:
  providers:
    my-gateway:
      apiKeyEnv: GATEWAY_API_KEY
      api: openai-completions
      baseURL: https://gateway.example/v1
      models:
        - id: legacy-chat
        - id: vision-preview
          input: [text, image]     # 手工录入的模型默认视为纯文本
```

  `defaultInput: [text, image]` 可设在路由级作兜底（默认 `[text]`）；catalog provider 用 `modelOverrides` 按模型 id 收窄。图片发送前被拒的排错：`MISSING_CREDENTIAL`、`UNKNOWN_MODEL`、401、图片模态未声明。
来源：docs/user/guide/providers.md

### 2.3 权限 / 审批

- 新会话默认 `workspace-write` 权限预设：Bash 和文件写操作被限制在会话工作区与系统临时目录内；读取、网络访问、进程可见性不受限。
- `DSH_PERMISSION_MODE` 环境变量改进程级默认；Web UI General 设置中的权限只影响**之后**新建的会话。
- `DSH_TOOLS_MODE=native|code|both` 选工具暴露方式；自带 `minimal`（极简模式）agent 预设：固定系统提示词 "You are a helpful software engineer assistant."，只挂持久 `bash` + `str_replace_editor` 两个工具。
- 遥测默认全本地：`DSH_TELEMETRY_MODE=FULL`（OTLP/HTTP）、`FEEDBACK_ONLY`、`DSH_TELEMETRY_DISABLED`（硬性关闭）。
来源：apps/cli/reference/README.md

## 3. 插件开发

### 3.1 插件的本质（Cordis）

插件是一个导出 `apply` 函数的 TypeScript 模块，框架加载时传入共享 `ctx`：

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello-plugin'

export function apply(ctx: Context) {
  console.log('[hello-plugin] plugin loaded!')
}
```

三种形态：**函数式**（最常用）、**对象式**（`export default { name, inject, apply }`）、**类式**（`extends Service`）。依赖通过 `export const inject = ['tools']` 声明，框架等依赖就绪后再调 `apply`。所有通过 `ctx` 注册的监听器/工具/定时器在插件卸载时**自动清理**；需要显式释放的资源用 `ctx.effect()` 返回 disposer。
来源：docs/user/develop/basic/index.md

### 3.2 加载本地插件（最小复刻路径）

```sh
mkdir -p scratch-plugin/src
# 写 scratch-plugin/src/my-plugin.ts（上面的代码）
# 写 scratch-plugin/cordis.yml（路径必须绝对）：
```

```yaml
- insert:
    - id: hello
      name: '/absolute/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

```sh
pnpm dsh web --patch ./scratch-plugin/cordis.yml
# 打开 http://127.0.0.1:3080，启动日志出现 [hello-plugin] plugin loaded!
```

### 3.3 编写一个工具插件

```ts
import type { Context } from '@deepseek-ai/cordis'
import { defineTool } from '@deepseek-ai/dsh-tools'

export const name = 'greet-tool'
export const inject = ['tools']

export function apply(ctx: Context) {
  ctx.tools.register(defineTool({
    name: 'greet',
    description: 'Greet someone by name.',
    parameters: {
      name: { type: 'string', required: true, description: 'The name to greet' },
    },
    output: {
      schema: { type: 'string' },
      render: (_args, value) => [{ type: 'text', text: value }],
    },
    async execute(args) {
      return `Hello, ${args.name}!`
    },
  }))
}
```

然后在 Web UI 里问 "Use the greet tool to greet Ada." 即可验证。进阶参考：插件配置（config.md）、工具编写参考（cookbook/adding-a-tool.md，含嵌套 schema、后台任务、策略钩子、Code Mode、UI 卡片）。
来源：docs/user/develop/basic/tool.md

### 3.4 官方插件生态与扩展点地图

架构文档给出"新功能挂哪里"的对照表：加模型 provider → 在 `ctx.llm` 注册适配器；加模型可见能力 → `ctx.tools`；加人类命令 → `ctx.commands`；加后台任务 → `ctx.jobs`；文件系统 → `ctx.fs` provider 或 `fs/*` 事件；沙箱 → `ctx.sandbox`；拦截请求/工具/回合 → `agent/*`、`tools/*` waterfall 事件（监听器必须调 `next()`）；注入模型可见上下文 → `agent.inject()`；分叉会话 → `ctx.sessions.fork(...)`。扩展烹饪书：extension-cookbook.md、adding-a-package.md、adding-a-tool.md、adding-an-llm-adapter.md、adding-a-conversation-node.md。
来源：docs/architecture.md

### 3.5 发布与发现

- **发现**：给插件仓库加 GitHub topic [`dsh-plugin`](https://github.com/topics/dsh-plugin)。该 topic 下已有 228+ 公开仓库（2026-08），生态包括 TUI（dsh-tianshu-tui）、Web UI 皮肤与面板、视觉/OCR 工具插件、agent 搜索、`@file` 提及、VS Code 集成、"Awesome DSH Plugins" 目录（带每日兼容性追踪——侧面印证 breaking change 频繁）。
- **安装第三方插件**：`dsh plugin --profile <name> add <spec>`（支持 npm 包、`github:owner/repo`、本地路径 `./`、`file:`/`link:`）。包只要在 `package.json` 声明 `"dsh": { "bundle": { "patch": "./cordis.patch.yml" } }` 即作为 bundle 加入层叠。Git 源码包通过 `prepare` 脚本在 install 时构建——pnpm ≥10 默认拦截，首次 add 会失败并提示把打印的 key 拷进 profile 的 `pnpm-workspace.yaml` 的 allowBuilds 后重跑。示例：

```sh
dsh plugin --profile tui add github:deepseek-harness/turtle-ui
dsh --profile tui
```

## 4. 开发文档要点（development / architecture / 贡献）

**前置**：Node 22.19+/24+、Corepack 启用的 pnpm 11.7.0、Git ≥2.26；可选 DeepSeek API key（真机 e2e 无 key 自动跳过）。`pnpm install` 顺带装 Lefthook hooks 和翻译配对 merge driver；克隆后跑 `pnpm run typecheck` 通过即就绪。

**构建顺序**（双聚合 TypeScript 工程，Host/Client 分离以避免 Cordis `Context` 声明合并冲突）：

```sh
tsc -b tsconfig.host.json
tsdown --env.DSH_BUILD_FACE host
tsc -b tsconfig.client.json
tsdown --env.DSH_BUILD_FACE client
pnpm run build:web
```

**日常命令**：`pnpm run typecheck`、`pnpm run check:all`、`pnpm run doc-sync`、`pnpm run hygiene`。CI 为无密钥 workflow，另有真机 API e2e（`pnpm run test:e2e`）。

**演示**：`pnpm dsh --profile headless "summarize this workspace"`（需 `DEEPSEEK_API_KEY`）、`pnpm run demo:cordis`（可检查/修改自己运行时的自指 demo）、`pnpm run demo:acp`（JSON-RPC stdio 的 ACP server）。

**架构核心概念**（docs/architecture.md）：
- 无特权核心：一切都是插件，注册即 effect，卸载即回滚。
- profile/bundle 层叠顺序：profile manifest `dsh.profile.bundles` 顺序 → profile 的 `cordis.patch.yml` → 全局 `$DSH_HOME/cordis.patch.yml` → `--patch` 覆盖层；后层整行替换（非深合并）。
- 核心包：`core/session`（仅追加 SessionEvent 日志，`ctx.sessions`）、`core/system-prompt`、`core/tools`（`ctx.tools`）、`core/agent` + `core/agent-loop`、`llm/llm`（`ctx.llm`）。
- **Turn/Step 模型**：Step = 一次模型请求+其工具调用；Turn = 0..n 个 Step。事件流 `turn/start → step/start → agent/request → llm/stream → assistant/chunk* → tool/call → tools/pre-execute → execute → post-execute → step/end → turn/end`。waterfall 监听器必须 `next()`。
- **Model-visible ⟺ logged**：凡进入模型请求的内容必须能从会话日志重建，有运行时不变量断言；新增模型可见输入必须扩展 `SessionEventMap`。
- 能力 seam 三角色：Service Definition / Service Provider / Consumer；换一个 provider 可整体迁移 Bash/PTY/LSP 到远程沙箱。

**AGENTS.md 关键约定**：包一律 `@deepseek-ai/dsh-<name>`、全 ESM、`@deepseek-ai/cordis` 为 peerDependency；注册即 effect；waterfall 必须 `next()`；判别联合以 `assertNever` 收尾；新行为走文档化扩展点，改 agent-loop 必须同步更新 docs/architecture.md。

## 5. 社区、常见问题与限制

1. **Developer Preview**：README 明示"会有破坏兼容性的变更"；社区出现"每日兼容性追踪"目录即佐证。媒体报道（IT之家、网易、财联社等，2026-08-13）确认 v0.1 与 DeepSeek-V4-Pro 同日开源，几天内社区已产出约 300 个插件。
2. **无官方 TUI 自带**：终端界面需装社区插件（如 dsh-tianshu-tui）。官方界面以 Web UI 为主，headless 用于一次性任务。
3. **不支持 `--host 0.0.0.0`**；对外暴露需 `--trusted-host`。
4. **MCP 默认关闭**：CLI 附带 `@deepseek-ai/dsh-mcp-client` 供补丁层使用，但默认不启用任何 MCP server——每个 server 命令都是 agent 沙箱外的可信可执行代码。
5. **web_fetch 默认禁用**，需补丁层插入 provider 并启用；`web_search` 用 `DEEPSEEK_API_KEY`（可用 `DEEPSEEK_SEARCH_BASE_URL` 改端点）。
6. **模型模态坑**：手工录入的模型默认纯文本；DeepSeek 官方 chat-completions 路由纯文本不可配置。
7. **凭据/环境**：`MISSING_CREDENTIAL`、`UNKNOWN_MODEL` 最常见；密钥只写、存 `$DSH_HOME/.credentials.yaml`，切勿写进 cordis.yml 或提交。
8. **Git 源码插件安装**：pnpm ≥10 allowBuilds 拦截导致首次 `dsh plugin add` 失败是预期行为，按提示配置后重跑。
9. **支持渠道**：GitHub Discussions；Discord（英文）/ 企业微信群+微信公众号（中文）。
10. **同名混淆提醒**：PyPI 上的 `deepseek-harness` / `deepseek-harness-cli`（0.2.0，第三方协议探针工具）与官方 DSH **不是同一项目**；官方分发渠道只有 npm `@deepseek-ai/dsh` 与 GitHub 仓库。

## 主要来源清单

- README（中/英）：https://github.com/deepseek-ai/deepseek-harness
- npm：https://www.npmjs.com/package/@deepseek-ai/dsh
- Web UI 指南 / providers：docs/user/guide/index.md、providers.md
- CLI 参考：apps/cli/reference/README.md
- 插件教程：docs/user/develop/basic/index.md、tool.md
- 开发指南 / 架构 / AGENTS.md：docs/development.md、architecture.md、AGENTS.md
- 插件生态：https://github.com/topics/dsh-plugin

# 书籍变更日志

《从 Cordis 到 DeepSeek Harness》的版本沿革。版本号以 `book/metadata.yaml` 的 `version` 字段为准；每次发版由 `book` 分支推送触发 CI 构建 EPUB 并发布到 gh-pages 落地页。

格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，日期为 ISO 格式。

## [1.5.0] - 2026-08-24

追踪 master 至 dsh `0.1.1-rc.2`（基点 `0.1.0-rc.7` → 当前，约 13k 提交差距，经 Agent Notes 逐篇溯源与 `git merge-base` 版本归属核验）。

### 新增

- 第 11 章新增 **11.7 凭据面：从环境变量名到授权流**——`CredentialKey` 第二键空间（`<scope>/<id>`，`api-key`/`grant` 联合）、`dsh-authorization` 授权流 seam（流拥有写入、交互随请求而行）、`.credentials.yaml` 版本化与就地升级（小结与参考资料顺延为 11.8/11.9）
- 第 8 章 8.1 增**会话日志版本纪律**：单调 `SESSION_FORMAT_VERSION`、方向感知读规则、逐事件 `ignorable` 标记（rc.8 落地）
- 第 8 章 8.4 增**跨会话引用**通道：`dsh-session:` 规范提及、冻结快照、64KB 独立上限、`\u003c` 转义与不可信背景警告
- 第 10 章 10.2 增 llm-deepseek 两处传输韧性：**reasoning 逐轮回传**（rc.8，网关可哈希重建思维链）与 **Files 失败的内联回退**（0.1.1-rc.2，独立 `filesApiTimeoutMs` 与 20MiB 水位，绝不混用两种图片传输）
- 第 15 章新增 **15.4.9 会话日志版本机制**权衡：可见的过度拒绝 vs 静默的上下文掏空
- 第 7 章表 7-1 增 `experimental/` 分组（Agent Teams 私有孵化，机械隔离于发布集）；新增 rc.7 → 0.1.1-rc.2 演进段
- 第 2 章时间线增开源后 rc 日更节奏（08-13 开源 → 08-21 已至 0.1.1-rc.2）
- 第 5 章章首增 vendored 副本（`vendor/cordis/src/`）阅读映射指引
- 附录 D 术语表增 `SESSION_FORMAT_VERSION`、`CredentialRef / CredentialKey` 两条

### 变更

- 第 8 章 8.3.2 `step()` 源码摘录更新为**中断流前缀固化**版本（`interrupted: true` 的 `assistant/message`，行号校正为 `agent.ts:332-420`），逐段解读顺延为 7 点并补四个被拒替代方案
- 第 11 章 11.3 Linux 后端增 **bwrap 私有 PID 命名空间**（0.1.1-rc.1 起）：封堵 procfs magic link 逃逸，探测与真实包裹同一 profile 构造器
- 第 10 章 10.3 更新 OAuth 现状：`dsh-authorization` 落地、`openai-codex` 重回目录，登录界面入口尚未提供（如实标注）
- 第 12 章与附录基线版本升至 `0.1.1-rc.2`；`.credentials.yaml` 文档化为 `refs:`/`records:` 两段、启动自动升级旧布局
- base bundle 插件行数表述统一为实际值 78（原"约 70"）

### 基建

- `book` 分支 rebase 到最新 master（9/9 无冲突）；`node book/validate.js` 17 章通过

## [1.4.0] - 2026-08-19

### 变更

- 读者视角全稿重写：剔除审核者视角冗余表述，生态事实更新至 dsh `0.1.0-rc.7`（PTC 模式更名、设置卡片注册、Codex/Claude Code 子代理、附件内容寻址、`low` 推理档）

### 修复（CI，2026-08-20/21）

- 根治 EPUB 构建 apt stall：按需安装（功能检测跳过）、apt 分段硬超时、清索引重试；构建超时由 45 分钟收敛至 20 分钟，对齐 v1.4.0 元数据

## [1.3.0] - 2026-08-18

### 变更

- 全量事实溯源审计：人物（崔添翼/史一凡/陈德里）、事件（隐写回传、内测招募）、时间节点（公众号注册 vs 报道、开源日）逐项核对一手来源

## [1.2.0] - 2026-08-18

### 变更

- 引用审计：全书引用真实性核验与修正，参考资料链接逐条复核

## [1.1.0] - 2026-08-18

### 新增

- 附录 F：调研底稿与延伸阅读（`book/briefs/` 五篇底稿入库），附人物与事实辨析（Shigma≠崔添翼、张伟署名、公众号两个时间点）

### 修复

- 深度评审：事实性错误修正、章节源码引用以原始 markdown 源重建

## [1.0.0] - 2026-08-18

### 新增

- 书籍脚手架与全部 17 章：8 个部分，从 Model+Harness=Agent 认知、Cordis 原理与源码、dsh 架构五章，到实战两章、迷你复刻与设计权衡
- EPUB 构建链：`validate.js`（章节清单/标题编号/源码链接）、`postprocess.js`、`validate-render.js`、`build-epub.mjs`、Mermaid 渲染
- GitHub Actions 发布链：`book` 分支推送构建 EPUB 并发布 gh-pages 落地页，Release 事件自动附件

[1.5.0]: https://github.com/zenHeart/deepseek-harness/releases/tag/book-v1.5.0
[1.4.0]: https://github.com/zenHeart/deepseek-harness/commit/fb111c1d60
[1.3.0]: https://github.com/zenHeart/deepseek-harness/commit/28ef4c239a
[1.2.0]: https://github.com/zenHeart/deepseek-harness/commit/72485c011b
[1.1.0]: https://github.com/zenHeart/deepseek-harness/commit/e4f1a75c2b
[1.0.0]: https://github.com/zenHeart/deepseek-harness/commit/4abafd578c

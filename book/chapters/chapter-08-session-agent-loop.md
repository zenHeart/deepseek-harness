# 8. Session 事件溯源与 Agent 主循环

上一章我们看到，dsh 里连 agent 主循环都只是一个插件。但这个插件跳动的方式决定了整个系统的气质：dsh 选择了**事件溯源（event sourcing）**作为会话的根本模型——Session 是一条 append-only 的 `SessionEvent` 日志，是系统里唯一的事实源；模型看到的历史、UI 渲染的轨迹、fork/resume/压缩，全部是从这条日志派生出的投影。本章先讲日志与投影，再剖析 `ReactLoopAgent` 的 `turn()` 与 `step()` 两个核心方法（逐段读真实源码），最后梳理输入路径、并行工具调度与三个事件域的拦截点清单。

## 8.1 Session：唯一事实源

`packages/core/session`（`@deepseek-ai/dsh-session`，自述 *"Event-sourced session store"*）把一次会话建模为一条**只追加**的事件日志。事件的词表覆盖了一次智能体会话的全部事实：`turn/start | turn/end`、`step/start | step/end`、`user/message`、`assistant/chunk | assistant/message`、`tool/call | tool/result`、`request/header | request/context`，以及各插件通过 declaration merging 扩展的合并事件（如 `approval/asked`、`fs/observed`、`compaction/*`）。

这套设计的支柱是上一章提到的不变量：**Model-visible ⟺ logged**。凡进入模型请求的内容，必须能从 session 日志重建；运行时在每个请求派生时断言这条不变量。其推论非常强大：

- **模型历史是投影，不是存储。** `session.deriveMessages()` 从日志计算出送给模型的消息序列。存的是“发生过什么”，而不是“模型看到什么”——后者永远可以由前者重算。
- **fork / resume / transcript / telemetry / 压缩全部从同一流派生。** fork 是复制一段日志继续追加；resume 是重放日志恢复相位机；UI 轨迹、OTLP 遥测只是日志的不同消费者。
- **压缩 = 替换投影区间。** 摘要本身复用 `user/message` 事件，附带 `surfaceOp: { op: 'replace', start, end }` 做唯一一次 surface 替换——模型历史是日志的投影，压缩只是换了一段投影（详见 10.4.2 节）。
- **请求可重建。** `buildRequest` 会把 `request/header`、`request/context` 记入日志，因此每一次模型请求都可以事后从日志完整重放——这对调试“模型为什么这样回答”是无价之宝。

![事件溯源：日志为事实源，投影为视图](../images/fig-c3-event-sourcing.png)

*图 8-1 事件溯源：append-only 日志记录一切，投影从中派生*

日志还要能被**未来的（和过去的）运行时**正确读取，这需要日志自己的版本纪律（rc.8 落地）。`SESSION_FORMAT_VERSION` 是一个单调整数，刻意不做 major/minor 之分——"某一步能否自动升级"是那一步的升级器是否存在的属性，而不是编号方案该预先承诺的东西（与 SQLite 后端的 `SCHEMA_VERSION` 同一先例）。读规则按方向区分：版本相等，正常读；**读到比自己新的日志，方向感知地拒绝**（报 `SessionFormatUnsupportedError`，明示"由更新的 harness 写入——请升级"，并指向原始日志文件让用户仍能看到文本）——首发运行的运行时是所有后续决策的地板，它缺少的拒绝行为永远补不进用户已经在跑的拷贝；读到更旧的，经 n→n+1 升级链在内存中转换查看，只有真正继续会话时才原子化持久化转换结果（原文件留作备份）。词表增长则由逐事件 `ignorable` 标记兜底，普通事件新增永不触发版本 bump：读取器遇到不认识的 event type 即拒绝解释该日志，**除非**事件信封携带 `ignorable: true`。默认严格而非默认宽容，是因为忘记标记的代价应该是一次"看得见的过度拒绝"（不便），而不是一次"静默恢复出一个被掏空上下文的会话"（安全事故）——这条不对称与全书 fail-closed 的元原则同构。什么情况下必须 bump？只有当旧运行时无法再以完全正确的语义处理新日志时：header 形状、事件信封、核心事件语义、surface 机制——"能解析不报错"不是标准，**静默跳过影响重建的内容就是误读**。

## 8.2 Turn / Step 相位模型

dsh 把会话时间划分为两级相位：

- **Step = 一次模型请求 + 其工具调用。** 一次 step 以 `step/start` 开始、以 `step/end` 结束，中间包含流式接收、工具执行。
- **Turn = 0..n 个 Step。** 一个 turn 从一次用户输入（或 followup）的兑现开始，直到模型给出不含工具调用的回答、且没有 next-step 输入插队为止。

典型事件流为：

```text
turn/start → step/start → agent/request → llm/stream → assistant/chunk*
→ tool/call → tools/pre-execute → execute → tools/post-execute → tool/result
→ step/end → (step/start …)* → turn/end
```

![Turn/Step 事件流](../images/mm-turn-step.png)

*图 8-2 Turn/Step 事件流*

图 8-1 展示了这条流水线。注意 `step/end` 在 `finally` 中追加——即使 step 中途失败，日志仍然闭合，replay 永远合法。

## 8.3 ReactLoopAgent 源码剖析

默认驱动是 `ReactLoopAgent`（`packages/core/agent-loop/src/agent.ts`），内部维护一个 **inbox 收件箱** 和一台 turn/step 相位机。收件箱按唤醒语义区分：followup 进入 next-turn（唤醒循环开启新 turn）；steer 进入 next-step（唤醒循环，插队进下一 step）；inject 也写入 next-step，但**不唤醒**——等下一条消息把它顺带带入。下面逐段精读。

### 8.3.1 `turn()`：相位机的外环（`agent.ts:246-330`，节选）

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

逐段解读：

1. **落 `turn/start`。** 每个 turn 以日志事件开幕，`turn` 序号贯穿之后所有事件的归属字段。
2. **`preStep`：第一个拦截闸。** `agent/pre-step` 是一个 waterfall——所有插件可以在此审查、改写甚至**否决**这一步。返回 `reject` 时整个 turn 以 `{ kind: 'blocked' }` 收场。自动上下文压缩就挂在这里（serial 模式，请求派生前执行）。
3. **先落 `step/start`，再落输入消息。** `decision.messages`（本步要带入的用户/插队消息）以 `surfaceOp: 'append'` 追加为 `user/message` 事件——先落日志，再进请求，Model-visible ⟺ logged 的微观体现。
4. **`step/end` 在 `finally` 中。** 无论 `step()` 成功、抛错还是被 abort，step 事件对永远闭合。
5. **`agent/turn-stopping`：serial 终点检查点。** 当循环判断 turn 即将结束（模型已给出终结回答且 inbox 无 next-step 插队）时，以 serial 模式派发一次“turn 停止”事件，给插件最后一次挽留/善后机会（例如 goal 插件检查目标未达成则追加一条 next-step 消息让 turn 继续）。
6. **`turn/end` 携带结束原因。** `reason` 是 `completed | blocked | max-tokens` 等封闭判别联合，同样是日志事实。

### 8.3.2 `step()`：一次请求的生命周期（`agent.ts:332-420`，节选）

```ts
private async step(assembly: PromptAssembly): Promise<StepEndReason | null> {
  const system = renderPrompt(assembly)
  while (true) {
    const { request, preparedCall } = await this.buildRequest(
      turn, step, assembly.tools, system, this.session.deriveMessages(), signal)
    const assembler = new BlockAssembler()
    try {
      const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
      for await (const chunk of stream) {
        signal.throwIfAborted()
        chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
        assembler.push(chunk)
      }
    } catch (error) {
      if (signal.aborted) {                       // 取消：固化已送达前缀（rc.8 新增）
        const content = assembler.interruptedBlocks()
        if (content.length > 0) {
          this.session.append('assistant/message', { turn, step, interrupted: true, ... },
            { surfaceOp: 'append', sourceEventSeqs: chunkSeqs })
        }
      }
      throw error
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

逐段解读：

1. **构建请求。** `buildRequest` 做三件事：用 `this.session.deriveMessages()` 投影出模型历史（再次注意：历史是算出来的）；经 `agent/request` waterfall 让插件改写请求 config；调用 `ctx.llm.prepareCall()` 把 config 绑定到具体适配器。同时把 `request/header`、`request/context` 落日志——请求从此可重建。
2. **流式接收 + BlockAssembler。** `assistant/chunk` 事件**逐 chunk** 落日志，同时喂给 `BlockAssembler` 把碎片组装成完整消息块（文本 / reasoning / tool-call）。chunk 事件的序号被收集到 `chunkSeqs`。
3. **取消时固化已送达前缀（rc.8 新增）。** 整个流消费包在 try/catch 里：若捕获时 `signal.aborted`，`assembler.interruptedBlocks()` 取出已送达的文本与 reasoning 块（不含 tool-call——中断发生在派发前，不存在真实结果），落一条 `interrupted: true` 的 `assistant/message`，`sourceEventSeqs` 精确指向已落日志的 chunk 序列，然后才重新抛出。这条消息赶在 `step/end` 与 aborted `turn/end` 之前落日志，于是"取消后继续追问"与"在取消处 fork"都能把用户已经读到的内容带进模型历史。设计上有四个被拒绝的替代方案值得记住：直接丢弃前缀（用户读到的文本在后续请求里凭空消失）；投影时从 chunk 临时组装（`deriveMessages()` 与每个客户端投影都要各写一套中断规则，且日志里没有权威消息）；保留完整 tool-call 并补合成 abort 结果（这些调用从未派发，合成结果是在谎称一次执行）；追加一条模型可见的 `[interrupted by user]`（需要新的 source 类型与投影规则，事实已由 aborted `turn/end` 承载）。注意不对称性：**provider 错误终止的流仍然丢弃前缀**——错误 turn 没有用户的取消决策，需要自己的保留策略。
4. **失败恢复。** 流出错走 `agent/request-error` waterfall：插件可返回 retry action 让循环原地重试。协议层的 canonical context-overflow 错误也走这条路——仅当压缩令 surface 替换代数前进时才返回 retry，避免无限循环。
5. **组装完成，落 `assistant/message`。** 注意 `sourceEventSeqs: chunkSeqs`：这条汇总事件显式声明自己由哪些 chunk 事件派生，日志内部自带血缘关系。
6. **终结判定。** `max-tokens` 直接以同名单元的 `StepEndReason` 返回；消息里没有 tool-call 则 `{ kind: 'completed' }`，step（以及可能的 turn）就此结束。
7. **有 tool-call 则执行。** `executeToolCalls` 负责并行调度（见 8.5）；它的最后一个参数是 context 追加回调——工具结果附带的 context 通过 `inbox.splice('next-step', ...)` 进入收件箱，等下一条 step 带入。若调度器报告 `concluded`（例如 steer 消息已经回答了问题），step 直接判完成。

返回 `null` 表示“本 step 的工具已执行完，继续同 step 内的下一轮模型请求”——`step()` 里的 `while (true)` 正是 ReAct 循环的体现：一次 step 内部可以进行多轮“请求 → 工具 → 再请求”。

## 8.4 输入路径：followup / steer / inject

人在 loop 运行中说话，有三种语义，分别写入 inbox 的不同槽位、携带不同的唤醒语义：

| 路径 | 槽位 | 语义 |
|---|---|---|
| `followup()` | next-turn | 追加到下一 turn，唤醒循环开始新 turn |
| `steer()` | next-step | 立即插队进下一 step，唤醒循环（打断当前方向） |
| `inject()` | next-step | 写入但不唤醒——等下一条消息把它顺带带入 |

三者最终都变成 `user/message` 事件落日志（Model-visible ⟺ logged），区别只在相位机的唤醒时机。这让“用户随时插话”成为一等公民，而不是 UI 层的补丁。

0.1.0-rc 系列后期，输入路径旁又多了一条**跨会话引用**通道（`dsh-session-reference`）：Web 端 `@` 提及另一个会话时，宿主生成 `@[label](dsh-session:<base64url(JSON)>)` 规范提及；`agent/pre-step` 上的监听器解析被接受的用户消息，把目标会话的当前 surface 折叠成一份**冻结快照**——最多 3 条引用、每条默认 64KB 独立上限、JSON 聚合序列化且每个数据 `<` 都转义为 `\u003c`（源文本无法拼出包裹用的类 XML 标签而逃数据区），并置于固定的"不可信背景"警告之下（模型被告知：除非当前用户复述，否则不执行被引用会话里的指令）。快照紧随原消息以 sourced `user/message` 落日志——Model-visible ⟺ logged 因此不需要新事件类型就成立。这个机制是投影模型的又一次复用：引用内容是源会话日志的一次只读投影（只保留直接用户消息、已完成的助手文本与最新压缩检查点谱系），且作为 injected context 被排除在目标会话自身的投影之外——快照永不递归传播。任何读取、校验或预算失败都让整个准备失败、终止当前 turn，而不是给模型看半截上下文。

## 8.5 并行工具调度：有界滚动池 + 屏障

模型一次返回多个 tool-call 时，`packages/core/agent-loop/src/tool-calls.ts` 用“有界滚动池 + 屏障”调度：exclusive 调用形成屏障（其前其后的 parallel 组不得跨越它），parallel 调用在 `maxParallelToolCalls` 上限内重叠 dispatch，但**结果永远按模型给出的顺序提交**——模型看到的工具结果序列与其请求顺序一致。核心代码：

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

三个细节值得注意：

1. **每组开始前重读 `ctx.tools.executionMode()`。** 如果某个插件在两次调用之间改了注册表（例如用 `restrict()` 收紧了某工具的并发分类），调度器立刻感知，可能即时制造一道屏障。注册表的动态性被尊重到每一次 dispatch。
2. **`commitReady()` 只跨越连续的模型序 slot 前进。** 即便第 3 个调用先完成，只要第 2 个还没好，结果就不能提交——顺序一致性优先于吞吐。
3. **abort 时补写合成错误结果。** 循环被中断时，未启动的调用会被补写一条 `tool call aborted before dispatch` 的合成 `tool/result`——日志里每个 `tool/call` 都有配对的 `tool/result`，replay 永远合法。

## 8.6 三个事件域与关键拦截点

dsh 的事件分三个域，职责清晰：

1. **durable session 事件**（重放事实）：`turn/*`、`step/*`、`user/message`、`assistant/*`、`tool/*`、`request/*`——落日志，是唯一事实源。
2. **live `agent/*` 事件**（协调/拦截在飞工作）：不落日志，作用于“正在进行”的循环。
3. **capability 事件**（`fs/*`、`tools/*`、`llm/*`）：挂策略的地方，如 read-before-write 由 `fs/write-intent` / `fs/edit-intent` 事件门实现。

关键 waterfall 拦截点清单（写插件时最常用）：

| 拦截点 | 模式 | 用途 |
|---|---|---|
| `agent/pre-step` | waterfall/serial | 否决或改写一步；自动压缩挂载点 |
| `agent/request` | waterfall | 改写模型请求 config（路由、温度、工具集） |
| `agent/request-error` | waterfall | 请求失败恢复，返回 retry action |
| `llm/stream` | waterfall | 流层面包装（重试、计量） |
| `tools/pre-execute` | waterfall | 工具执行前：hooks、权限、沙箱 |
| `tools/execute` | around-waterfall | 包超时、重试、指标 |
| `tools/post-execute` | waterfall | accept/block/replace 结果、追加 context |
| `agent/turn-stopping` | serial | turn 终点检查，可挽留 |

记住宪章：waterfall 监听器必须调用 `next()`，否则短路整条链。

## 8.7 本章小结

- Session 是 append-only 的 `SessionEvent` 日志，唯一事实源；模型历史由 `deriveMessages()` 投影，fork/resume/压缩/telemetry 全部从同一流派生；`Model-visible ⟺ logged` 由运行时断言。日志自身有版本纪律：单调 `SESSION_FORMAT_VERSION` + 逐事件 `ignorable` 标记，新日志方向感知地拒绝、旧日志经升级链内存转换。
- 相位模型：Step = 一次模型请求 + 其工具调用；Turn = 0..n 个 Step；事件对在 `finally` 中闭合。
- `ReactLoopAgent` 的 `turn()` 是外环相位机（preStep 闸、turn-stopping 检查点），`step()` 是 ReAct 内环（chunk 落日志、BlockAssembler 组装、request-error 恢复、工具调度）；被取消的流会把已送达前缀固化为 `interrupted: true` 的 `assistant/message`，取消后的追问与 fork 不再丢失用户已读内容。
- 输入三路径 followup/steer/inject 语义分明，跨会话引用以冻结快照形式复用投影模型；并行调度用有界滚动池 + 屏障，结果按模型序提交，abort 补写合成结果保证 replay 合法。
- 三个事件域（durable / live / capability）+ 八个关键 waterfall 拦截点，是插件介入循环的全部合法入口。

下一章聚焦循环里最大的扩展面：工具系统与执行管线。

## 8.8 本章参考资料

- [packages/core/agent-loop/src/agent.ts](https://github.com/zenHeart/deepseek-harness/blob/master/packages/core/agent-loop/src/agent.ts) — `ReactLoopAgent` 的完整源码：inbox 收件箱、turn/step 相位机、主循环全在这一个文件里。8.3 节的每段摘录都标了行号，建议打开原文对照精读。
- [packages/core/agent-loop/src/tool-calls.ts](https://github.com/zenHeart/deepseek-harness/blob/master/packages/core/agent-loop/src/tool-calls.ts) — 并行工具调度的全部实现，百余行。"有界滚动池 + 屏障"与按模型序提交结果的原文，8.5 节的三个细节都可在此复核。
- [docs/subsystems/session.md](https://github.com/zenHeart/deepseek-harness/blob/master/docs/subsystems/session.md) — session 子系统官方文档：append-only 事件日志、"回放即重新派生"投影模型的权威说明，想自己实现事件溯源会话时的范本。
- [docs/architecture.md](https://github.com/zenHeart/deepseek-harness/blob/master/docs/architecture.md) — 官方架构文档：Turn/Step 事件流、"Model-visible ⟺ logged" 不变量与三个事件域划分的出处，本章概念地图的官方版本。
- [AGENTS.md](https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/AGENTS.md) — dsh 工程宪章原文，waterfall 监听器必须调用 `next()` 等循环侧约定在此；准备在 `agent/*` 事件上写拦截插件前，先读它。

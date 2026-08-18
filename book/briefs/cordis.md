# Cordis 源码级调研报告：DeepSeek Harness 的底层元框架

## 0. 关键定位与来源

- **GitHub 仓库**：https://github.com/cordiverse/cordis （标题：*"Meta-Framework of Spatiotemporal Composability"*，MIT 协议，TypeScript，monorepo 约 550 commits）。注意：**不在 deepseek-ai 组织下**，而是在 `cordiverse` 组织（该项目源自 Koishi 机器人框架生态，作者 Shigma/崔添翼团队，DeepSeek Harness 以 vendor 方式引入）。
- **DeepSeek Harness 仓库**：https://github.com/deepseek-ai/deepseek-harness （README 原文："It uses an architecture where **everything is a plugin**, and is powered by Cordis"）。
- **论文**：《A Programming Paradigm for Spatiotemporal Composability》，**未上 arXiv**，托管于 https://github.com/cordiverse/paper （2026-08-13 草稿 preprint）。
- **官方入门文档**：https://deepseek-harness.github.io/deepseek-harness/reference/cordis-primer （"Cordis 入门"，中文）。

源码证据来自对 `cordiverse/cordis` master 分支的完整克隆（`packages/core/src/*.ts` 共 1848 行核心代码）。

## 1. Cordis 解决什么问题

论文摘要（github.com/cordiverse/paper README）明确指出：

> Modern software—from plugin systems to self-evolving agent harnesses—increasingly requires *dynamic composition*, yet its formal foundations remain underdeveloped.

即：**现代软件（插件系统、自进化 Agent Harness）越来越需要"动态组合"，但缺乏形式化基础**。论文把问题分解为两个正交维度：

1. **时间可组合性（temporal composability）**：组件被移除时，其产生的**一切副作用都能被完整撤销**（revert）。传统插件系统（如 VS Code 的 deactivate、npm 动态 require/unrequire）都依赖插件作者自觉清理，框架无法保证。
2. **空间可组合性（spatial composability）**：组件间的**依赖可以被声明并由运行时反应式地管理**——依赖就绪才启动，依赖消失就自动卸载，依赖变更就自动重启。传统 DI 容器（Inversify、Spring）的依赖图在启动时解析一次后基本静止，无法应对"服务随插件上下线而动态出现/消失"的 Agent 场景。

为什么现有 agent 框架不够：DeepSeek Harness 的设计是"一切皆插件"——模型、工具、技能、会话、沙箱、存储、循环、调度、UI 全部由插件组合。其"创造模式"甚至允许 Agent **在运行时检查自己的插件树、动态挂载/卸载模型自己编写的临时插件**（"给高速行驶的汽车换发动机"）。这要求底层框架能**保证任意插件的注册副作用有明确清理路径**——这正是 Cordis 的 Context + Effect 机制提供的。

## 2. 核心原理：时空可组合性

论文的核心手法是**把编程语言理论中的 effect（效果）与 coeffect（余效果）概念提升为运行时机制**：

- **Revertible Effects（可撤销效果）**：每一次对上下文的变换都携带一个"逆操作"，运行时负责追踪。→ 落地为 `ctx.effect()`：执行体返回一个 disposer，框架登记；卸载时逆序调用。
- **Reactive Coeffects（反应式余效果）**：上下文每次变化都会按组件的 coeffect 规格（即依赖声明）通知组件。→ 落地为 `inject` 声明 + 服务注册/注销时的自动 `notify` → 相关 Fiber 自动 `_refresh()` → 卸载/重载。
- 论文将 effect context 与 coeffect context 统一为单一 **Context 类型**，并在其上定义 **component（组件）** 与**动态组合演算**，其元理论把单组件的时空可组合性推广到交错组合的整个系统。

工程实现的对应关系（官方 primer 总结的五个核心概念）：

| 论文概念 | 代码实体 | 文件 |
|---|---|---|
| Component / 组件 | **Fiber**（插件的一次实例化，有状态机与副作用清单） | `packages/core/src/fiber.ts` |
| Effect context | **Context**（Proxy 化的服务容器 + 副作用挂载点） | `packages/core/src/context.ts` |
| Revertible effect | `ctx.effect(execute, label)` 返回 `Disposable` | `packages/core/src/fiber.ts` |
| Reactive coeffect | `inject` 声明 + `ReflectService.notify()` | `packages/core/src/registry.ts`, `reflect.ts` |
| 服务 | `Service` 基类 / `ctx.provide()` | `packages/core/src/service.ts` |
| 插件 | `Plugin = Function \| Constructor \| { apply }` | `packages/core/src/registry.ts` |
| 事件通信 | `EventsService`（emit/waterfall/parallel/serial/bail） | `packages/core/src/events.ts` |

## 3. 源码剖析

### 3.1 目录结构（monorepo，yarn workspaces + yakumo 构建）

```
packages/
├── core/            # cordis 本体：Context/Fiber/Registry/Reflect/Events/Service/Logger
│   └── src/{context,fiber,registry,reflect,events,service,logger,utils,index}.ts
├── loader/          # @cordisjs/plugin-loader：声明式配置加载器（Entry/EntryGroup/EntryTree）
│   └── src/config/{entry,group,isolate,tree,utils}.ts
├── hmr/             # @cordisjs/plugin-hmr：基于 chokidar 的热模块替换
├── include/         # @cordisjs/plugin-include：YAML/JSON 配置文件读写（含 !!js 表达式）
├── group/           # 插件分组
├── timer/           # setTimeout/setInterval 的 effect 化封装
├── logger-console/  # 控制台日志服务
├── utils/           # List 等 effect 化工具容器
└── create/          # 脚手架
```

### 3.2 Context：Proxy 化的效果/余效果统一上下文

`packages/core/src/context.ts`（全文 78 行）——Context 构造时把自身包成 **Proxy**（`ReflectService.handler`），所有属性读写都被拦截，从而实现"未声明 inject 就访问服务会报错"：

```ts
export class Context {
  constructor() {
    this[symbols.isolate] = Object.create(null)
    this[symbols.intercept] = Object.create(null)
    const self = new Proxy<this>(this, ReflectService.handler)
    this.root = self
    this.fiber = new Fiber(self, {}, Object.create(null), null, () => []) // 根 Fiber
    this.reflect = new ReflectService(self)
    this.registry = new RegistryService(self)
    this.events = new EventsService(self)
    this.logger = new LoggerService(self)
    return self
  }
  extend(meta = {}): this { /* 原型链派生子上下文 */ }
  isolate(name: string, label?: symbol) { /* 服务隔离域 */ }
  intercept(name, config) { /* 服务配置拦截/分层覆盖 */ }
}
```

`context.ts` 中的 Proxy `get` 处理器（`packages/core/src/reflect.ts`）是"反应式余效果"的关键：访问 `ctx.foo` 时会沿 Fiber 链向上查找服务实现，若该服务在 inject 中但未激活，则抛出带溯源信息的错误：

```ts
// packages/core/src/reflect.ts — ReflectService.handler.get
const error = new Error(`cannot get property "${prop}" without inject`)
...
return ctx.events.waterfall('internal/get', ctx, prop, error, () => {
  const key = target[symbols.isolate][prop]
  let fiber = (ctx[symbols.shadow] as Context ?? ctx).fiber
  while (true) {
    const impl = fiber.store?.[prop]
    if (impl) return getTraceable(ctx, impl.value)
    if (prop in fiber.inject) {
      error.message = `cannot get required service "${prop}" in inactive context`
      throw error
    }
    if (!fiber.runtime) throw error
    if (fiber.parent[symbols.isolate][prop] !== key) throw error  // 隔离域边界
    fiber = fiber.parent.fiber
  }
})
```

### 3.3 Fiber：组件运行时与生命周期状态机

`packages/core/src/fiber.ts`（486 行）。每个 `ctx.plugin(p, config)` 调用创建一个 Fiber。状态机：

```ts
export const enum FiberState {
  PENDING, LOADING, ACTIVE, FAILED, DISPOSED, UNLOADING,
}
```

**Effect 类型**——执行体可以同步返回 disposer、异步返回、甚至是（异步）生成器逐个 yield 多个 disposer：

```ts
export type Effect<T = any> = SyncEffect<T> | AsyncEffect<T>
type SyncEffect<T = any> = Disposable<T> | Iterable<Disposable<T>, void, void>
type AsyncEffect<T = any> = Promise<Disposable<T>> | AsyncIterable<Disposable<T>, void, void>
```

**`ctx.effect()` 核心实现**（fiber.ts）：所有 disposer 登记入 `DisposableList`，卸载时**逆序**执行（LIFO，类似 defer 栈）：

```ts
const dispose = () => {
  let task!: void | Promise<void>
  for (const dispose of disposables.splice(0).reverse()) {
    if (task) { task = task.then(dispose) }
    else { const result = dispose(); if (isObject(result) && 'then' in result) task = result as any }
  }
  return task
}
```

**反应式重启**：Fiber 的"epoch"字符串由所有 inject 依赖的提供方 uid 拼成；任何依赖变化 → `_refresh()` 重算 epoch → 不同则触发 `_unload()`（执行全部 disposer）→ `_reload()`（重新执行插件体）：

```ts
// packages/core/src/fiber.ts
_refresh() {
  let epoch: string | boolean = false; epoch = ''
  for (const name of Object.keys(this.inject)) {
    const impl = this._store[name]
    if (!impl) { epoch = INACTIVE; break }        // 依赖缺失 → 进入休眠
    epoch += ':' + impl.fiber.uid
  }
  this._setEpoch(epoch)                            // epoch 变化 → 自动卸载/重载
}
```

这即是论文"reactive coeffects"的直接实现：**插件不需要手写启动顺序，依赖就绪自动启动、依赖消失自动卸载、依赖重载自动级联重启**。

### 3.4 Plugin 模型与 Registry

`packages/core/src/registry.ts`：插件的三种形态与元数据：

```ts
export type Plugin<T = any> = Plugin.Function<T> | Plugin.Constructor<T> | Plugin.Object<T>
export namespace Plugin {
  export interface Base<T = any> {
    name?: string
    Config?: StandardSchemaV1<any, T>   // 标准 schema（zod/schemastery 均可）做配置校验
    inject?: Inject                      // 依赖声明（数组或带配置的对象）
    provide?: string | string[]          // 声明提供的服务名
    intercept?: Dict<boolean>
  }
  export interface Function<T = any> extends Base<T> { (ctx: Context, config: T): any }
  export interface Object<T = any>  extends Base<T> { apply(ctx: Context, config: T): any }
}
```

`ctx.plugin()` 创建 Runtime（同一插件可多次实例化，`runtime.fibers` 列表）并 new Fiber；配置经 `resolveConfig()` 用 Standard Schema 校验（`fiber.ts` 顶部），不合法抛 `ValidationError`。

**`@Inject` 装饰器**（registry.ts）：类或方法级声明依赖，方法级注入会在初始化钩子中自动 `ctx.inject(inject, cb)` 包装。

### 3.5 Service 与服务解析（空间组合）

`packages/core/src/service.ts`：`Service` 子类构造时自动 `ctx.reflect.provide(name, this)`，把自身注册进上下文。`provide()` 本身是一个 effect——**服务注销 = 撤销该 effect**，并 `notify()` 唤醒所有依赖方（`reflect.ts`）：

```ts
// packages/core/src/reflect.ts
provide(name: string, value?: any, check?: () => boolean) {
  return this.ctx.fiber.effect(() => {
    this.props[name] = { type: 'service' }
    ...
    this.store[key] = impl                      // 服务实现注册表（按 isolate symbol 索引）
    this.ctx.fiber.store![name] = impl
    if (this.ctx.fiber.state === FiberState.ACTIVE) this.notify([name])
    return async () => {                        // 撤销路径：时间可组合性
      delete this.store[key]
      const fibers = this.notify([name])
      await Promise.allSettled(fibers.map(fiber => fiber.await()))
      delete this.ctx.fiber.store![name]
    }
  }, `ctx.provide(${JSON.stringify(name)})`)
}
```

辅助机制：
- **`ctx.isolate(name)`**：为服务名生成独立 symbol 键，实现同名服务的命名空间隔离（同一 Context 树中可存在多份互不可见的实现）。
- **`ctx.intercept(name, config)`**：沿 Context 原型链分层覆盖服务配置（`Service[symbols.resolveConfig]` 自根向下合并）。
- **`ctx.mixin()` / `ctx.accessor()`**：把子服务的方法/属性以 accessor 形式投射到 Context 上（ReflectService 构造时即把 `registry.inject/plugin`、`events.on/emit/...` mixin 到 ctx）。
- **`trace/bind`**：Proxy 追踪值与回调的上下文归属，跨上下文传递时保持身份。

### 3.6 事件模型

`packages/core/src/events.ts`：五种分发模式（类型化，通过 `interface Events` 声明合并扩展）：

| 模式 | await | 语义 |
|---|---|---|
| `emit` | 否 | 广播观察 |
| `parallel` | 是 | 并行扇出，错误聚合为 AggregateError |
| `serial` | 是 | 按序执行，可 bail（返回非空即短路） |
| `bail` | 否 | 同步短路 |
| `waterfall` | 否 | **环绕中间件**：监听器收 `(...args, next)`，调 `next()` 委托下游，不调则短路 |

`ctx.on()` 返回注销函数，其内部同样是 `fiber.effect()` 注册（`register()`），所以**事件监听也自动可撤销**。内置一组 `internal/*` 元事件驱动框架自身：`internal/service`（服务上下线）、`internal/update`（配置热更新瀑布链）、`internal/get|set`（属性访问拦截）、`internal/plugin|status`（Fiber 生命周期）。

### 3.7 Loader / HMR：声明式动态组合

`packages/loader/src/` 把插件系统外化为**配置即组件树**：`Entry`（一个插件条目：id/name/config/inject/disabled/group）→ `EntryGroup` → `EntryTree`。`Loader` 监听 `internal/update`，把 Fiber 的配置变更**回写配置文件**（配置协调 reconciliation）：

```ts
// packages/loader/src/index.ts
ctx.on('internal/update', function (config, noSave, next) {
  if (!this.entry || noSave || this.parent.fiber?.entry === this.entry) return next()
  const unparse = this.runtime?.Config?.['simplify']
  this.entry.options.config = unparse ? unparse(config) : config
  this.entry.parent.tree.write()          // 配置持久化
  return next()
}, { global: true, prepend: true })
```

`packages/hmr/src/index.ts` 用 chokidar 监听文件变化，沿 `ModuleJob` 依赖图找出受影响插件并触发重载（`'hmr/reload'` 事件）——即论文所说的 "declarative component loader with configuration reconciliation and hot module replacement"。

## 4. 与 VS Code 插件体系 / Inversify / Spring 的对比

| 维度 | Cordis | VS Code Extensions | Inversify(JS) | Spring (IoC) |
|---|---|---|---|---|
| 组合单元 | Plugin → Fiber 实例，可同插件多实例 | Extension（单实例 activate） | 类型绑定 (bind→to) | Bean |
| 依赖表达 | `inject` 声明 + **运行时反应式解析**，缺失则休眠、出现则自启 | 静态 `contributes`/`activationEvents` | 构造器注入，容器启动时一次性解析 | `@Autowired`，启动期织入 |
| 依赖变化 | **自动级联卸载/重启**（epoch 机制） | 不支持（reload 整个窗口） | 不支持 | 有限（@RefreshScope 等外挂） |
| 副作用撤销 | **框架强制**：每个 effect 携带 disposer，LIFO 自动执行 | 靠作者把资源 push 进 `context.subscriptions`，仅覆盖部分 API | 无 | `DisposableBean.destroy`，容器关闭才触发 |
| 上下文结构 | Context 为 **Proxy + 原型链派生**（isolate/intercept 分层） | 全局 ExtensionContext，无层级 | Container 可层级但静态 | ApplicationContext 父子层级 |
| 事件模型 | 5 种类型化分发模式，waterfall 环绕中间件 | EventEmitter 单一模式 | 无内置 | ApplicationEvent 单一模式 |
| 配置 | Standard Schema 校验 + 声明式 YAML 树 + 热写回 | settings.json 静态贡献点 | 无 | PropertySource/@ConfigurationProperties |
| 形式化基础 | 有论文：effect/coeffect 提升为运行时，组合演算 | 无 | 无 | 无 |

**相似点**：`inject` 声明式依赖 ≈ Inversify 的 `@inject`/Spring 的 `@Autowired`；Context 层级 ≈ Spring 父子 ApplicationContext；disposer 列表 ≈ VS Code `subscriptions`。**本质差异**：Cordis 把"注册"和"依赖"都建模为**可逆的、反应式的一等运行时对象**（时间 + 空间两个维度），而上述框架要么撤销靠自觉、要么依赖图静态。这正是 Agent Harness"一切皆插件、运行时可自重组"（DeepSeek Harness 创造模式允许 Agent 动态挂载自写插件）所需要的。

## 5. 关键 API 速查（可直接引用于书籍）

`Context`（packages/core）：
- `ctx.plugin(plugin, config?) → Fiber & PromiseLike<Fiber>`
- `ctx.inject(deps, callback)` — 依赖就绪后执行
- `ctx.effect(execute, label?) → Disposable`（execute 可返回 disposer / Promise<disposer> / (Async)Iterable<disposer>）
- `ctx.provide(name, value?) → disposer`、`ctx.get/set(name)`、`ctx.mixin`、`ctx.accessor`
- `ctx.isolate(name)`、`ctx.intercept(name, config)`、`ctx.extend(meta)`
- 事件：`ctx.on/once/emit/parallel/serial/bail/waterfall`
- Fiber：`fiber.dispose()`、`fiber.restart()`、`fiber.update(config)`、`fiber.await()`、`fiber.getEffects()`、`FiberState` 枚举
- `Service` 基类 + `@Inject()` 装饰器；`Plugin.Config`（Standard Schema）

## 6. 不确定性与遗留

- 论文 PDF 全文（github.com/cordiverse/paper）只读到摘要级内容；形式化细节（组合演算规则、metatheory 证明）未逐节核对。
- 论文未上 arXiv；引用应以 GitHub 最新草稿为准（作者自述 "under active revision"）。
- Cordis 标注 "API is not yet stable"；上述源码对应 2026-08 master 快照。
- DeepSeek Harness 以 vendor 方式内嵌 cordis（见 cordis-primer），其 `vendor/README.md` 记录了同步流程。

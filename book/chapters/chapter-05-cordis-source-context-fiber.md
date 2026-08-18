# 5. Cordis 源码剖析：Context、Fiber 与服务解析

上一章我们讨论了 Cordis 提出的”时空可组合性”：时间上，每个组件的副作用必须可被完整撤销；空间上，组件间的依赖由运行时反应式管理。本章不再停留在概念层，而是直接打开 `cordiverse/cordis` 仓库的核心源码（`packages/core/src/`，全核心仅约 1848 行 TypeScript），逐文件、逐段地看清这套机制是如何落地的。读完本章，你应该能在不看文档的情况下，推断出任意一段 Cordis 插件代码在运行时的行为。

## 5.1 Monorepo 目录结构总览

Cordis 采用 yarn workspaces + yakumo 构建的 monorepo，各包职责如下：

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

值得注意的架构决策：**`core` 之外的一切都是插件包**。日志控制台、定时器、加载器、HMR 都不是框架硬编码的一部分，而是构建在同一套 Context/Fiber 原语之上的可选组件。这种”自举式”设计本身就是对框架表达力的检验——如果核心原语足以支撑 loader 和 HMR 这种元级功能，那它大概率也能支撑 Agent Harness 里的模型、工具、沙箱等业务插件。

论文概念与代码实体的对应关系，是阅读源码的地图：

| 论文概念 | 代码实体 | 文件 |
|----|----|----|
| Component / 组件 | **Fiber**（插件的一次实例化，有状态机与副作用清单） | `packages/core/src/fiber.ts` |
| Effect context | **Context**（Proxy 化的服务容器 + 副作用挂载点） | `packages/core/src/context.ts` |
| Revertible effect | `ctx.effect(execute, label)` 返回 `Disposable` | `packages/core/src/fiber.ts` |
| Reactive coeffect | `inject` 声明 + `ReflectService.notify()` | `packages/core/src/registry.ts`、`reflect.ts` |
| 服务 | `Service` 基类 / `ctx.provide()` | `packages/core/src/service.ts` |
| 插件 | `Plugin = Function \| Constructor \| { apply }` | `packages/core/src/registry.ts` |
| 事件通信 | `EventsService`（emit/waterfall/parallel/serial/bail） | `packages/core/src/events.ts` |

![Cordis 核心概念关系](../images/file4.png)

*图 5-2 Cordis 核心概念关系（同图 4-1）*

## 5.2 context.ts：78 行构造一个 Proxy 化上下文

`packages/core/src/context.ts` 全文只有 78 行，却是整个框架的入口。核心代码如下：

``` ts
// packages/core/src/context.ts
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

逐行拆解：

1.  **两个 symbol 属性袋**。`symbols.isolate` 和 `symbols.intercept` 都是以 symbol 为键的私有映射表（`Object.create(null)` 避免原型污染）。isolate 表记录”服务名 → 隔离键”的映射，是实现同名服务命名空间隔离的基础；intercept 表记录服务配置的分层覆盖规则。

2.  **构造函数返回 Proxy**。`const self = new Proxy<this>(this, ReflectService.handler)` 是全局最关键的一行：构造函数 `return self`，意味着 `new Context()` 拿到的不是 Context 实例本身，而是包了一层的 Proxy。**此后所有对 `ctx.xxx` 的属性读写都会先经过 `ReflectService.handler` 的拦截**。这正是”未声明 inject 就访问服务会报错”这一约束能够强制执行的机制基础——它不靠约定，不靠 lint，而是靠 JS Proxy 在运行时物理拦截。

3.  **根 Fiber**。`this.fiber = new Fiber(self, {}, ...)` 为上下文创建一个没有插件体的”根 Fiber”。Cordis 中一切副作用都挂在某个 Fiber 上，根 Fiber 就是整棵组件树的根节点；它的 disposer 列表为空（`() => []`），生命周期与应用同寿。

4.  **内建服务自举**。reflect、registry、events、logger 四个服务在构造期就实例化并挂载——注意它们传入的都是 `self`（Proxy 之后的上下文），保证服务内部对 `ctx` 的访问同样走拦截逻辑。

`extend()`、`isolate()`、`intercept()` 三个方法构成 Context 的派生体系：`extend` 用原型链派生子上下文（子上下文可访问父上下文的服务，且拥有自己的 Fiber 子树）；`isolate` 为服务名生成独立 symbol 键；`intercept` 沿原型链分层覆盖服务配置。三者将在 6.4 节展开。

## 5.3 reflect.ts 的 Proxy get 处理器：反应式余效果的关口

既然 `ctx` 是 Proxy，那么”访问服务”这个动作的全部语义就集中在 `ReflectService.handler.get` 里（`packages/core/src/reflect.ts`）。这是”反应式余效果”最关键的一段代码：

``` ts
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

这段代码的逻辑按执行顺序讲解：

**第 0 步：预埋错误对象**。处理器一开始就构造 `cannot get property "${prop}" without inject` 错误。注意这里只是构造，不是抛出——错误对象会作为参数进入 waterfall 链，允许 `internal/get` 的监听器改写它（例如附上更详细的溯源信息）。

**第 1 步：走 waterfall 元事件**。整个查找过程被包在 `ctx.events.waterfall('internal/get', ...)` 里。这是 Cordis 的一个标志性设计：**属性访问本身也是可拦截的事件**。框架内部机制（如 trace、访问计数）和第三方插件都可以监听 `internal/get` 来增强属性解析行为。waterfall 的 `next` 回调里才是真正的查找逻辑。

**第 2 步：沿 Fiber 链向上查找**。循环从当前上下文所属的 Fiber 开始，逐层向父 Fiber 爬升：

- `const impl = fiber.store?.[prop]`：每个 Fiber 有一个 `store`，记录以本 Fiber 为作用域提供的服务实现。找到即返回，且返回值经过 `getTraceable` 包装——返回的不是裸实现，而是一个携带上下文归属信息的可追踪代理，保证服务实例跨上下文传递时身份不丢失。
- `if (prop in fiber.inject)`：当前 Fiber 声明了对该服务的 inject 依赖，但此刻 store 里没有实现——说明依赖尚未激活（Fiber 处于休眠/未就绪状态）。抛出 `cannot get required service "${prop}" in inactive context`。这条错误路径非常重要：它把”依赖缺失”从”运行到一半 undefined 报错”提前为”访问瞬间带明确原因的异常”。
- `if (!fiber.runtime) throw error`：爬到根 Fiber 仍未找到，抛最初预埋的 “without inject” 错误——即你访问了一个既没有提供、也没有声明 inject 的属性。
- `if (fiber.parent[symbols.isolate][prop] !== key) throw error`：**隔离域边界检查**。如果父级上下文中该服务名对应的隔离键与当前层级不同，说明跨越了 `ctx.isolate()` 划定的命名空间边界，同名服务互不可见，查找终止。这一行就是服务隔离的运行时防线。

整段代码仅十余行，却把”服务解析 + 依赖校验 + 隔离边界 + 可扩展拦截”四件事统一在一条向上爬升的循环里。理解了它，就理解了 Cordis 空间可组合性的运行时内核。

## 5.4 fiber.ts：组件运行时与生命周期状态机

`packages/core/src/fiber.ts` 共 486 行，是 core 包中体量最大的文件，也是”时间可组合性”的主战场。每个 `ctx.plugin(p, config)` 调用都会创建一个 Fiber——注意是”每次调用创建一个”，同一插件可以多次实例化，各 Fiber 独立管理自己的状态与副作用。

### 5.4.1 状态机

``` ts
// packages/core/src/fiber.ts
export const enum FiberState {
  PENDING, LOADING, ACTIVE, FAILED, DISPOSED, UNLOADING,
}
```

![Fiber 生命周期状态机](../images/file5.png)

*图 5-1 Fiber 生命周期状态机（同图 4-2）*

六个状态的流转语义：

- **PENDING**：Fiber 已创建，但 inject 依赖尚未全部就绪，处于休眠。这是 Cordis 区别于传统 DI 容器的关键状态——依赖缺失不是错误，而是一种合法的等待。
- **LOADING**：依赖就绪，插件体正在执行（可能是异步的）。
- **ACTIVE**：插件体执行完毕，所有 effect 已登记，Fiber 对外提供服务。
- **FAILED**：插件体或其 effect 抛出异常。失败的 Fiber 不会拖垮整棵树，其已登记的 disposer 仍会被执行。
- **UNLOADING**：正在执行 disposer 列表（可能因为依赖消失、配置更新或显式 dispose）。
- **DISPOSED**：终态，所有副作用已撤销。

状态之间的转换几乎都由运行时自动驱动：`internal/status` 元事件会向外部广播每次变迁，loader、HMR、调试工具都靠订阅它来感知组件树的变化。

### 5.4.2 Effect 类型：同步、异步与生成器 disposer

时间可组合性的原子单位是 effect。fiber.ts 顶部的类型定义揭示了它的灵活度：

``` ts
// packages/core/src/fiber.ts
export type Effect<T = any> = SyncEffect<T> | AsyncEffect<T>
type SyncEffect<T = any> = Disposable<T> | Iterable<Disposable<T>, void, void>
type AsyncEffect<T = any> = Promise<Disposable<T>> | AsyncIterable<Disposable<T>, void, void>
```

一个 effect 执行体可以返回：

1.  **同步 disposer**：最常见的形式，`() => { server.close() }`；
2.  **Promise\<disposer\>**：异步初始化完成后才给出撤销路径（比如先 await 建立连接，再返回关闭连接的方法）；
3.  **（异步）生成器**：`function*` 逐个 yield 多个 disposer。这种形式适合”多阶段初始化”——每完成一个阶段就 yield 该阶段的撤销函数，若后续阶段失败，已 yield 的 disposer 仍然完整覆盖已发生的副作用。

这一设计把”初始化进行到哪一步，清理就覆盖到哪一步”变成了类型系统层面可表达的结构，而不是靠开发者手写 try/catch 累加清理逻辑。

### 5.4.3 DisposableList：LIFO 逆序撤销

`ctx.effect()` 登记的所有 disposer 进入 `DisposableList`，卸载时的执行方式如下：

``` ts
// packages/core/src/fiber.ts
const dispose = () => {
  let task!: void | Promise<void>
  for (const dispose of disposables.splice(0).reverse()) {
    if (task) { task = task.then(dispose) }
    else { const result = dispose(); if (isObject(result) && 'then' in result) task = result as any }
  }
  return task
}
```

这段实现有两个精心处理的细节：

- **`splice(0).reverse()`**：先清空列表再逆序，实现 LIFO（后进先出）语义——与 Go 的 `defer` 栈同理，后登记的副作用（通常是更”外层”的资源，如依赖其他资源的连接）先撤销。逆序是正确性的要求：若先建立的资源先释放，后建立的资源在清理时就会引用已失效的对象。
- **同步/异步混合编排**：`task` 变量把同步 disposer 和返回 Promise 的异步 disposer 串成一条链。遇到第一个异步 disposer 之前全部同步执行；之后每个 disposer 都 `.then` 在前一个的 Promise 上，保证即使混杂异步清理也严格保序。整个 dispose 过程最终返回一个 Promise，使 `fiber.dispose()` 可以被 await。

### 5.4.4 epoch 机制与 `_refresh()`：反应式重启

这是论文”reactive coeffects”最直接的代码化身：

``` ts
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

机制拆解：

- 每个 Fiber 的 **epoch** 是一个字符串，由它所有 inject 依赖的**提供方 Fiber 的 uid** 拼接而成（`':' + impl.fiber.uid`）。
- 每当有服务注册或注销，`ReflectService.notify()` 会唤醒所有声明了相关 inject 的 Fiber，触发各自的 `_refresh()`。
- `_refresh` 重算 epoch：若某个依赖没有实现（`!impl`），epoch 置为 `INACTIVE`，Fiber 进入 PENDING 休眠；否则与旧 epoch 比较。
- **epoch 不变**：依赖的提供方没换，什么都不做。**epoch 变了**：`_setEpoch` 内部触发 `_unload()`（执行全部 disposer，LIFO 撤销所有旧副作用）然后 `_reload()`（用同一插件体和配置重新实例化）。

于是，“插件不需要手写启动顺序，依赖就绪自动启动、依赖消失自动卸载、依赖重载自动级联重启”这三句话，在源码层面就是上面这十行。用提供方 uid 而非服务名拼 epoch 是精髓所在：同名服务被**替换实现**时 uid 变化，依赖方会感知到并级联重启——这正是”给高速行驶的汽车换发动机”的最小实现。

## 5.5 registry.ts：Plugin 三形态与元数据

`packages/core/src/registry.ts` 定义了插件的类型与注册流程：

``` ts
// packages/core/src/registry.ts
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

三种插件形态：

- **函数插件**：`(ctx, config) => any`，最轻量，适合无状态逻辑；
- **构造器插件**：可被 `new`，类形态便于组织有状态服务（配合 `Service` 基类）；
- **对象插件**：带 `apply` 方法的对象，便于以声明式风格携带元数据。

元数据字段的设计意图：

- **`Config`**：采用 Standard Schema（`StandardSchemaV1`）这一跨库标准接口，zod、schemastery 等任何实现了该标准的校验库均可直接使用。`ctx.plugin()` 时配置经 `resolveConfig()` 校验，不合法抛 `ValidationError`——配置错误在插件体执行之前就被拦下。
- **`inject`**：可以是字符串数组（`['database', 'http']`），也可以是带配置的对象形式（支持可选依赖等细粒度声明）。它是 5.4.4 节 epoch 机制的输入。
- **`provide`**：声明本插件提供的服务名，供运行时和工具链做静态分析。
- **`intercept`**：声明本插件要拦截哪些服务的配置。

此外，`ctx.plugin()` 会创建 **Runtime**（同一插件的多次实例化共享一个 Runtime，`runtime.fibers` 记录所有 Fiber），并 new 出本次的 Fiber。

**`@Inject` 装饰器**（同在 registry.ts）提供面向类开发的声明风格：类级 `@Inject(['database'])` 等价于静态 `inject` 属性；**方法级**装饰则更巧妙——它会在初始化钩子中自动用 `ctx.inject(inject, cb)` 把该方法包装起来，使方法体只在依赖就绪后才执行。这把反应式依赖管理下沉到了方法粒度。

## 5.6 service.ts 与 provide()：为什么”服务注销 = 撤销 effect”

`packages/core/src/service.ts` 中的 `Service` 基类做了一件小事：子类构造时自动调用 `ctx.reflect.provide(name, this)` 把自身注册进上下文。真正的机制在 `packages/core/src/reflect.ts` 的 `provide()`：

``` ts
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

这段代码是时空可组合性两条线索的交汇点，逐段讲解：

**注册段（effect 执行体）**：

- `this.props[name] = { type: 'service' }`：登记服务元信息；
- `this.store[key] = impl`：把服务实现写入注册表。`key` 是按 isolate symbol 索引的——这正是 5.3 节 get 处理器中隔离域检查所依赖的数据结构；
- `this.ctx.fiber.store![name] = impl`：同时写入当前 Fiber 的 store，使沿 Fiber 链的查找能命中；
- `if (... === FiberState.ACTIVE) this.notify([name])`：如果当前 Fiber 已激活，立刻广播服务上线，唤醒所有等待该服务的休眠 Fiber。

**撤销段（返回的 async disposer）**：`provide()` 本身是一个 effect——**服务注销不需要任何单独的 API，它就是撤销这个 effect**。撤销路径按严格的顺序做三件事：

1.  `delete this.store[key]`：先从注册表摘除，此后新的 `ctx[name]` 访问不再命中；
2.  `notify([name])` 并 `await Promise.allSettled(fibers.map(fiber => fiber.await()))`：通知所有依赖方，并**等待它们全部完成卸载/重启**。这一步保证了”服务下线时，依赖它的组件已经全部安全停机”——不会发生服务已销毁而依赖方仍在调用的悬挂状态。`allSettled` 而非 `all`，保证个别 Fiber 卸载失败不会阻塞整体流程；
3.  `delete this.ctx.fiber.store![name]`：最后清理 Fiber store。

effect 的 label（`` `ctx.provide(${JSON.stringify(name)})` ``）也不是装饰——Fiber 的 `getEffects()` 会返回所有登记 effect 的标签，调试工具和 Agent 自检能力（如 DeepSeek Harness 创造模式中 Agent 检视自己的插件树）都依赖它。

## 5.7 events.ts：五种分发模式

`packages/core/src/events.ts` 实现类型化事件系统（通过 `interface Events` 的声明合并，插件可以扩展事件签名获得完整类型检查）。五种分发模式的语义对比：

| 模式 | await | 语义 |
|----|----|----|
| `emit` | 否 | 广播观察 |
| `parallel` | 是 | 并行扇出，错误聚合为 AggregateError |
| `serial` | 是 | 按序执行，可 bail（返回非空即短路） |
| `bail` | 否 | 同步短路 |
| `waterfall` | 否 | **环绕中间件**：监听器收 `(...args, next)`，调 `next()` 委托下游，不调则短路 |

`waterfall` 是最有表现力的一种：监听器签名是 `(...args, next)`，它可以在调用 `next()` 前后插入自己的逻辑（环绕，around-advice），也可以改写传给下游的参数，或干脆不调 `next()` 实现短路。5.3 节属性解析包在 `internal/get` 的 waterfall 里、第 6 章将看到的配置回写包在 `internal/update` 的 waterfall 里，都是同一模式的运用——**框架把自己的关键路径全部做成了可环绕的中间件链**。

另一个一致性设计：`ctx.on()` 返回注销函数，其内部同样通过 `fiber.effect()` 注册——**事件监听也自动可撤销**。监听器所属的 Fiber 卸载时，监听自动摘除，不存在传统 EventEmitter 中忘调 `removeListener` 导致的内存泄漏与幽灵回调。

最后，框架内置一组 `internal/*` 元事件驱动自身：`internal/service`（服务上下线）、`internal/update`（配置热更新瀑布链）、`internal/get|set`（属性访问拦截）、`internal/plugin|status`（Fiber 生命周期）。下一章的 Loader 与 HMR 几乎完全建立在这组元事件之上。

## 5.8 本章小结

本章沿源码走完了 Cordis 核心的四条主线：

1.  **Context 是 Proxy**（context.ts，78 行）：属性访问被物理拦截，“未声明 inject 不得访问”是运行时强制而非约定；
2.  **服务解析沿 Fiber 链爬升**（reflect.ts 的 get 处理器）：命中即返回可追踪代理，依赖未激活抛带溯源的错误，isolate 边界终止查找，全程可走 `internal/get` 中间件扩展；
3.  **Fiber 是组件的运行时化身**（fiber.ts，486 行）：六态状态机、三种形态的 disposer 返回值、LIFO 逆序撤销的 DisposableList、以及由依赖提供方 uid 拼成的 epoch——十行 `_refresh()` 实现了”依赖就绪自启、消失自卸、变更级联重启”；
4.  **注册即 effect**（reflect.ts 的 provide()）：服务注销没有独立 API，撤销 effect 即注销，且撤销路径会等待所有依赖方安全停机。

下一章我们离开 core 包，看这些原语如何支撑起声明式加载、热重载与配置回写，并把 Cordis 放到与 VS Code Extensions、Inversify、Spring 的坐标系中比较。

## 5.9 本章参考资料

- [Cordis 仓库](https://github.com/cordiverse/cordis) — 本章逐行剖析的上游源码仓库（master 快照，MIT 协议）。
- [fiber.ts（Cordis 源码）](https://github.com/cordiverse/cordis/blob/master/packages/core/src/fiber.ts) — 支撑本章 Fiber 生命周期状态机、disposer LIFO 逆序执行与 epoch 反应式重启的实现细节。
- [reflect.ts（Cordis 源码）](https://github.com/cordiverse/cordis/blob/master/packages/core/src/reflect.ts) — 支撑本章 Proxy get 处理器、服务 `provide()` 撤销路径与 `notify()` 唤醒依赖方的源码依据。
- [registry.ts（Cordis 源码）](https://github.com/cordiverse/cordis/blob/master/packages/core/src/registry.ts) — 支撑本章插件三形态、`inject`/`provide` 元数据与 Standard Schema 配置校验。
- [events.ts（Cordis 源码）](https://github.com/cordiverse/cordis/blob/master/packages/core/src/events.ts) — 支撑本章 emit/parallel/serial/bail/waterfall 五种类型化事件分发模式。
- [service.ts（Cordis 源码）](https://github.com/cordiverse/cordis/blob/master/packages/core/src/service.ts) — 支撑本章 Service 基类自动注册进上下文的实现。
- [Cordis 入门（dsh 官方文档）](https://deepseek-harness.github.io/deepseek-harness/reference/cordis-primer) — 官方中文入门，对照本章源码剖析的概念地图。

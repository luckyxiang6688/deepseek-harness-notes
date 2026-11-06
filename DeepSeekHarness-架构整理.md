# DeepSeek Harness 架构整理（总分结构）

> 本文整理此前讨论的核心内容，按“先总览、再分层展开”的结构组织，覆盖 Provider、Seam、Cordis 内核、事件系统、装配机制、插件与服务、`ctx`、Agent 运行时、Scope/Fiber、工具隔离等主题。

---

## 一、总览：DeepSeek Harness 是什么

DeepSeek Harness 可以理解为一套 **“造 Agent 的底盘”**，其设计哲学是 **“一切皆插件”（Everything is a plugin）**。它不是一个单体 Agent 工具，而是一个可自由重组、可替换、可热插拔的 Agent 运行时骨架。

整体可以分成几层理解：

1. **Cordis 插件内核层**  
   提供插件运行、上下文容器、依赖注入、类型化事件、可逆副作用、作用域派生、生命周期管理等通用元能力。

2. **Seam / Provider 能力层**  
   用“服务定义 + 服务提供者 + 消费方”的三段式结构，把能力抽象成可替换的 Seam；Provider 是其中真正“干活”的实现角色。

3. **Agent 领域服务层**  
   在 Cordis 之上构建会话、工具、模型、Agent 循环、子代理、工作流、沙箱、持久化等 Agent 领域能力，以 `ctx.<key>` 形式暴露。

4. **装配与分发层**  
   通过 Profile、Bundle、Patch 把插件和配置分层叠加，形成最终可启动的 Harness 实例。

5. **执行与隔离层**  
   Agent Loop 驱动“思考—行动”循环；Session 以事件溯源记录唯一事实；Scope/Fiber 负责按 Agent 隔离注册与生命周期清理；工具在可见性和执行资源上实现隔离。

一句话总结：

> **Cordis 提供运行规则，Seam/Provider 提供可替换能力，Agent Loop 驱动执行，Session 记录事实，Scope/Fiber 保证隔离与清理，Profile/Bundle/Patch 完成装配。**

---

## 二、Seam 与 Provider 家族

### 2.1 Seam 是什么

**Seam（接缝）是一个完整的、可替换的能力包**，不是单个角色。一个 Seam 必须同时包含三个角色：

| 角色 | 作用 |
|---|---|
| **Service Definition（服务定义）** | 定义能力契约，例如 `ctx.shell` 的抽象类，不包含实现 |
| **Service Provider（服务提供者）** | 契约的具体实现，真正执行逻辑 |
| **Consumer（消费方）** | 使用能力的工具或插件，只依赖契约，不知道背后 Provider 是谁 |

官方强调：**Seam 是完整能力，绝不是其中一个角色**。单独一个 Provider 或 Consumer 都不能称为 Seam。

### 2.2 Seam 的核心价值

- **实现可替换**：替换 Provider 即可改变能力实现，Consumer 不改。
- **彻底解耦**：Provider 与 Consumer 互不依赖，都只依赖 Service Definition。
- **统一扩展骨架**：子代理、工作流、上下文压缩、沙箱、Web 访问等都可基于三段式模型构建。

### 2.3 Provider 家族是什么

Provider 是 Seam 中的“具体实现者”。在 DeepSeek Harness 中，Provider 家族是“一切皆插件”的落地核心，决定“这个能力具体由谁、以何种方式实现”。

主要类型包括：

1. **LLM Provider（模型适配器）**  
   把 Harness 内部统一请求翻译成具体模型 API 调用。例如：
   - `dsh-llm-deepseek`：直连 DeepSeek 官方 API。
   - `dsh-llm-pi-ai`：承载 Anthropic、OpenAI 等大量提供方接入。

2. **Subagent Provider（子代理提供方）**  
   决定子任务“在哪里、以什么方式”运行。Subagent 能力允许**多个命名 Provider 在同一上下文并存**，例如：
   - 进程内 Provider：`spawn`、`fork`，支持可续接。
   - 进程外 Provider：`codex`、`claude-code`，通常不支持续接。

3. **路由与生态 Provider**  
   例如 `dsh-openrouter-providers`，根据 OpenRouter 目录控制请求最终路由到哪个下游提供商。

### 2.4 Provider 的单例与多例约束

- **单例约束**：多数核心能力（如 `ctx.fs`、`ctx.shell`）在一个上下文上只能存在一个 Provider，加载第二个会报错。
- **多例并存**：Subagent 等少数能力允许同时注册多个具名 Provider，由上层策略选择。

---

## 三、Cordis 内核：插件化运行的基石

Cordis 是 DeepSeek Harness 底层的插件框架，定义系统最基础的运行规则。其核心概念包括：

### 3.1 插件（Plugin）

插件是**一切能力的载体**。模型适配器、工具注册表、会话日志、Agent Loop、UI 都可以是插件。

一个插件通常由以下部分组成：

- **实现体**：带有 `apply(ctx)` 的函数，或 `Service` 子类。
- **依赖声明（可选）**：通过 `inject` 声明依赖的服务。
- **生命周期**：安装到 `Context` 后获得 `PluginHandle`，其创建的资源是可逆副作用，卸载时自动清理。

示例结构：

```typescript
export const name = 'my-tool'
export const inject = ['tools']

export function apply(ctx) {
  ctx.tools.register('my-tool', { /* schema 与 handler */ })
}
```

### 3.2 上下文（Context）

`ctx` 是**服务容器**，每个服务占据稳定 key，如 `ctx.tools`、`ctx.llm`。插件通过 key 查找和使用服务，而不是直接导入实现，实现彻底解耦。

### 3.3 服务依赖注入（Inject）

插件通过 `inject` 声明需要哪些服务，Cordis 等待依赖就绪后才启动插件。**加载顺序由依赖关系自动决定**，无需手动编排。

### 3.4 类型化事件（Typed Event）

插件间通信的核心机制。服务通过 TypeScript 声明事件，并以 `emit`、`waterfall`、`parallel` 等模式分发，用于策略拦截、行为观察等协作。

### 3.5 可逆副作用（Reversible Effect）

任何注册（提示词片段、工具 Schema、事件监听等）都被视为可撤销副作用。在 `reload` 或 `teardown` 时自动清理，不留残留状态。这是热重载和干净卸载的关键。

---

## 四、事件系统：双重核心角色

### 4.1 事件的核心作用

事件在 DeepSeek Harness 中有双重角色：

1. **微观插件协作层面的“神经系统”**  
   作为插件间通信总线，实现松耦合扩展点。一个插件可以宣告“某件事发生了”，而不知道谁在监听。

2. **宏观状态与行为层面的“唯一真源”**  
   Session 是 append-only 的类型化 `SessionEvent` 日志，是 Agent 完整交互历史的唯一真源。聊天历史是从事件日志派生出来的视图。

此外，在 Super-agent 架构中，外部群聊消息、GitHub 评论、CI 结果等被抽象为 **Trigger（触发器）**，事件驱动 Agent 从被动响应转为主动推进工作。

### 4.2 事件的五种分发模式

| 模式 | 核心语义 | 是否等待异步 | 返回值作用 | 典型场景 |
|---|---|---|---|---|
| **`emit`** | 单向通知 | 否 | 忽略 | 日志、遥测 |
| **`bail`** | 竞争性短路 | 否 | 第一个有效值胜出 | 多级缓存、候选查询 |
| **`serial`** | 顺序执行并等待 | 是 | 第一个有效值终止 | 有序初始化、设置阶段 |
| **`parallel`** | 并行扇出并等待 | 是 | 忽略 | 并行通知、独立处理 |
| **`waterfall`** | 环绕拦截/包装 | 否（可自行 await） | 可修改、替换或短路 | 权限门禁、请求改写、后处理 |

### 4.3 各模式例子

- **`emit`**：会话创建后，日志插件、指标插件、缓存插件同时监听 `session/created`，各自处理，互不影响。
- **`bail`**：命令执行前安全检查，黑名单检查若发现高危指令直接返回原因，后续权限检查不再执行。
- **`serial`**：系统启动自检，先异步检查数据库，成功后再执行数据迁移；任一步返回错误即终止。
- **`parallel`**：插件卸载时并行清理临时文件、关闭连接、刷新日志，等待全部完成。
- **`waterfall`**：路径守卫监听 `tools/pre-execute`，路径不合法直接返回 `deny` 且不调用 `next()`，工具不执行；合法则 `next()` 放行。

### 4.4 事件如何定义与交互

事件通过 TypeScript 声明合并定义。例如：

```typescript
declare module '@deepseek-ai/cordis' {
  interface Events {
    'tools/pre-execute': (
      exec: ToolExecution,
      next: () => Promise<PreToolDecision>
    ) => Promise<PreToolDecision>
  }
}
```

插件通过 `ctx.on('tools/pre-execute', async (exec, next) => { ... })` 监听。`waterfall` 中不调用 `next()` 即短路，是权限门禁的核心机制。

---

## 五、插件、服务与 Bundle

### 5.1 插件与 Bundle 的关系

- **插件**：最小可插换功能单元，是运行时真正承载能力的实体。
- **Bundle**：插件的**分发格式**，是一个 npm 包，通过 `package.json` 中的 `dsh.bundle` 指向补丁文件（如 `cordis.patch.yml`），声明如何安装和配置插件。

关系链：

> **Bundle（npm 包）→ 补丁文件（配置行）→ 插件（功能实体）**

一个 Bundle 可以包含多个插件；一个插件也可以独立分发，通过 `--patch` 或 Profile 配置加载。

### 5.2 插件与服务的关系

- **插件**：可装卸的功能单元。
- **服务**：插件挂在共享 `ctx` 上的**命名能力接口**，如 `ctx.tools`、`ctx.llm`。

协同机制是依赖注入：

- 消费方通过 `inject: ['metrics']` 声明需要服务。
- 提供方通过继承 `Service` 并 `super(ctx, 'metrics')` 生产服务。
- 框架保证依赖就绪后才执行消费方 `apply`。
- 服务提供者卸载时，依赖它的插件自动 `dispose`；服务重新出现时自动重新加载。

### 5.3 `metrics` 服务例子

`MetricsService` 提供的核心能力是一个 `record(event, value)` 方法，像“记账员”记录事件和数值。

消费方插件：

```typescript
export const inject = ['metrics']

export function apply(ctx) {
  ctx.metrics.record('tool_call', 1)
}
```

消费方不知道 `MetricsService` 类存在，也不知道内部实现，只消费“调用 `record` 方法的能力”。

### 5.4 服务像系统内部能力吗？

像，例如日志服务、指标服务、文件系统服务等。但有两个区别：

1. **服务是插件生产出来的能力**，不是天然存在的系统设施；插件卸载，服务消失。
2. **“总线”更对应事件系统，而不是 Service**。Service 是命名能力接口，Event 是插件间通信总线。

---

## 六、Profile、Bundle 与 Patch 的协同

### 6.1 各自角色

| 概念 | 角色 |
|---|---|
| **Bundle** | 可分发的“预制件”，包含一组插件配置行 |
| **Profile** | 装配档案，按顺序列出要叠加哪些 Bundle |
| **Patch** | 覆盖指令，按 `id` 定位配置行并整行替换其 `config` |

### 6.2 叠加链

启动时按严格顺序叠加：

> 空配置树 → 按 `dsh.profile.bundles` 顺序应用每个 Bundle 的 Patch → Profile 自己的 `cordis.patch.yml` → 用户主目录级 Patch → 命令行 `--patch` 临时覆盖

最终合并成一棵生效配置树，再启动 Harness。`--dump-config` 可预览最终结果。

### 6.3 例子：把默认模型换成 DeepSeek

Profile 的 `package.json` 列出：

```json
"dsh": {
  "profile": {
    "bundles": ["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-web-app"]
  }
}
```

`dsh-base` 先提供 `llm-deepseek` 默认行；Profile 的 `cordis.patch.yml` 再按 `id: llm-deepseek` 覆盖：

```yaml
- id: llm-deepseek
  config:
    apiKey: 'my-personal-key'
    baseURL: 'https://api.deepseek.com/v1'
    model: 'deepseek-chat'
```

Patch 是**整行替换**，不是深度合并，因此需要重新声明希望保留的所有字段。命令行 `--patch` 可临时再覆盖，仅本次生效。

---

## 七、`ctx`：服务容器与作用域实例

### 7.1 `ctx` 不是会话上下文实例

`ctx` 是 Cordis 的核心对象，是**通用服务容器和能力访问入口**。一个特定会话上下文，是 `ctx` 通过 `extend()` 或 `isolate()` 派生出的**子作用域**。

### 7.2 作用域实例怎么理解

“作用域实例”可以理解为：从全局 `ctx` 这棵“主干”上生长出的、拥有自己“局部环境”的“分支”。它共享主干基础设施，但可以覆盖或隔离特定部分，且不污染主干。

不是 `new Parent()` 的类实例化，而更像：

```javascript
const sessionCtx = Object.create(rootCtx)
```

即从已有实例派生，原型继承父实例当前状态。

派生方式：

- **`ctx.extend(meta)`**：附加元数据标签，子上下文原型继承父级属性，`meta` 遮蔽同名属性。
- **`ctx.isolate(name, label?)`**：让特定服务在子上下文中拥有独立解析作用域，可提供完全不同的实现，父级不受影响。
- **`ctx.intercept(name, config)`**：不替换实现，而是为该服务合并特定配置。

### 7.3 `ctx` 的基础设施

分两层：

**第一层：Cordis 元基础设施**

- 服务容器与依赖注入
- 类型化事件总线
- 可逆副作用所有者
- 生命周期作用域
- 作用域派生机制

**第二层：Harness 领域服务**

核心运行时：

| ctx 键 | 职责 |
|---|---|
| `ctx.sessions` | 事件溯源会话，append-only 日志唯一真源 |
| `ctx.systemPrompt` | 提示词片段、工具 Schema、变量组装 |
| `ctx.tools` | 工具注册表和守卫执行流水线 |
| `ctx.agents` | 活跃 Agent 协调、委托创建、`agent/*` 事件 |
| `ctx.agentLoop` | 实体 Agent 循环驱动器 |

模型与执行环境：

| ctx 键 | 职责 |
|---|---|
| `ctx.llm` | 模型适配器注册表和流式调用接口 |
| `ctx.fs` / `ctx.bash` / `ctx.subprocess` / `ctx.pty` | 文件系统、Shell、子进程、终端等执行原语 |
| `ctx.sandbox` / `ctx.sandboxPolicy` | 沙箱约束和共享策略 |

能力与治理：

| ctx 键 | 职责 |
|---|---|
| `ctx.skills` / `ctx.subagents` / `ctx.workflows` | 技能、子代理委托、多 Agent 编排 |
| `ctx.compact` / `ctx.tokenMeter` | 上下文压缩与 Token 压力感知 |
| `ctx.settings` / `ctx.credentials` | 分层配置与凭据引用 |
| `ctx.sessionPersistence` / `ctx.sessionQuery` | 会话持久化与检索 |

### 7.4 可逆副作用所有者

“可逆副作用所有者”指：

- **副作用**：插件对环境所做的改变，如注册工具、监听事件、打开文件、启动定时器。
- **可逆**：系统记录“如何撤销它”，插件卸载或热重载时自动完整回滚。
- **所有者**：每个副作用归属于产生它的 `ctx` 作用域；该 `ctx` 销毁时，名下副作用按后进先出自动清理。

比喻：像一个“事务型内存管理器”，插件每做一笔修改，系统记下回滚日志；插件卸载时逆序回滚，让环境回到插件从未加载的状态。

---

## 八、Agent 运行时：定义、`ctx.agents` 与 `ctx.agentLoop`

### 8.1 系统里 Agent 怎么定义

在 Super-agent 架构中，Agent 不再是一次性会话，而是**常驻的、连续的实体**：跨会话保持上下文，并行推进多项任务，环境有事件时自主响应，进程崩溃后可恢复。

架构上采用 **Router-Worker 双层模型**：

- **Router**：负责任务分解、分发、协调、汇总，不直接执行具体工作。
- **Worker**：专注执行具体任务，上下文隔离，向 Router 汇报。

每个活跃 Agent 拥有自己的作用域，其注册的工具、事件监听等贡献限定在作用域内。

### 8.2 `ctx.agents` 与 `ctx.agentLoop` 的区别

| 概念 | 定位 |
|---|---|
| **`ctx.agents`** | Agent 的接口层与注册中心，定义公共契约、活跃 Agent 注册表、`agent/*` 事件词汇、发起方作用域 |
| **`ctx.agentLoop`** | Agent 接口的唯一具体实现，即驱动器，真正驱动推理—行动循环 |

协同方式：`ctx.agentLoop` 通过 `ctx.agents.setFactory()` 将创建逻辑注册到 `ctx.agents`。扩展插件应依赖 `ctx.agents` 接口，而不是直接依赖 `ctx.agentLoop` 实现，从而保证主循环可替换。

### 8.3 `ctx.agentLoop` 的能力

1. **事务性的 Agent 创建与恢复**  
   创建/恢复是受回滚保护的整体事务：构造私有会话、实体 Agent、作用域上下文，进入注册表，发出 `session/created`、`agent/created`、`agent/session-start`，全部成功后启动驱动器。创建者与提供者共同拥有 Agent，销毁汇聚到同一静默边界。

2. **Turn/Step 生命周期驱动**  
   一个 Turn 是完整执行过程，可包含零个或多个 Step；一个 Step 是一次模型请求及其工具调用。标准流程：认领提示词 → 开启轮次 → 组装请求 → 调用模型 → 分发工具 → 结果写回日志。

3. **生命周期事件与并发仲裁**  
   通过 `agent/*` 事件宣告生命周期；通过 `enter()` 仲裁同一 `SessionId` 的并发创建，保证只有一个成功发布。

### 8.4 `ctx.agentLoop` 的交互对象

- **与 `ctx.agents`**：实现 `AgentFactory` 契约并注册自身。
- **与 `ctx.sessions`**：在事件溯源日志上开启轮次，追加模型可见事实，历史从日志派生。
- **与 `ctx.systemPrompt`**：组装请求前缀。
- **与 `ctx.llm`**：流式获取模型响应。
- **与 `ctx.tools`**：分发工具调用，触发受保护执行流水线。
- **与 Scope**：在 `ctx.agents.withInitiator()` 内运行，确保注册限定在 Agent 作用域。

### 8.5 `ctx.sessions` 是不是用户开启的会话

不完全是。用户看到的会话是 UI 表现；`ctx.sessions` 本质是**事件溯源的 Agent 交互日志**，是 append-only 的 `SessionEvent` 真源。LLM 消息历史是从事件日志派生的视图。回放、Fork、恢复、轨迹追踪因“一切都是事件”而天然支持。

### 8.6 `ctx.workflows` 的核心作用与应用场景

`ctx.workflows` 是**脚本驱动的多 Agent 编排**能力：允许模型一次性编写 JavaScript 编排脚本，由 Harness 执行，大规模扇出子 Agent，并控制并发上限、分阶段执行、收集结构化结果。

适用场景：

- 跨多个文件的审计
- 批量处理与多维度评测
- 一次性迁移
- 任务数量多、结构固定、需要精确控制顺序和并发的复杂编排

---

## 九、Scope 与 Fiber：隔离与生命周期

### 9.1 Scope 的核心作用

Scope 是为每个 Agent 构建的**隔离注册单元**，解决“共享底层基础设施，同时让每个 Agent 拥有独立工具、提示词、策略、监听器，并保证不泄漏、随生命周期清理”的矛盾。

没有 Scope 会陷入两种极端：

- 每个 Agent 一套独立服务图：共享基础设施重复。
- 全局一份注册图：Agent 私有贡献泄漏。

Scope 通过“带作用域的注册层”解决：每个活跃 Agent 拥有一个扁平注册层（`agent.ctx`）。操作看到的贡献 = **部署级全局注册 + 该操作所属 Agent 的匹配层**。

### 9.2 Scope 的能力

- **`createScope(ctx, key)`**：铸造带 `key` 标签的作用域。
- **`scopeOf(ctx)`**：读取上下文携带的作用域标签；`undefined` 表示全局作用域。
- **`scopeTarget(base, key)`**：构建作用域过滤事件分发载体，只送达全局监听器和标签匹配的作用域监听器。
- **`Scope.dispose()`**：幂等、共享的静默边界，统一等待并拆除该作用域所有注册。
- **`Scope.rawDispose`**：暴露底层精确拆除函数，便于嵌套清理。
- **`ScopeLayer` / `ScopedLayers<L>`**：管理注册表完整贡献，急切全局层 + 懒加载精确作用域层，`merge()` 按插入顺序合并。

### 9.3 Scope 的实现机制

- **标签与扁平结构**：每个作用域由不透明 `ScopeKey` 标识；解析不遍历父级或兄弟，只合并全局层和当前作用域层，保证确定性隔离。
- **标签继承**：通过上下文原型链自然继承；嵌套时最近标签生效。
- **生命周期归属**：通过 `Scope.ctx` 的每个注册都绑定到该作用域背后的 Cordis Fiber；销毁时自动完整撤销。

### 9.4 Scope 例子：两个 Agent 的工具间

Agent A（代码审查）和 Agent B（内容创作）同时运行。

- Agent A 启动时 `createScope(ctx, 'agent-A')`，在作用域内注册 `scan_vulnerability`。
- Agent B 使用标签 `'agent-B'` 的 Scope 解析工具，只看到全局工具和 `'agent-B'` 层工具。
- Agent A 的 `scan_vulnerability` 对 B 不可见。

### 9.5 Fiber 是什么

Fiber 是**一个已加载插件的运行时实例**，记录插件状态、配置和所有副作用。它是 Scope 的底层支撑。

核心作用：

- **生命周期状态管理**：`PENDING → LOADING → ACTIVE → FAILED → UNLOADING → DISPOSED`。
- **副作用收集**：通过 `ctx.effect()`、`ctx.on()`、`ctx.tools.register()` 等注册的清理逻辑由 Fiber 统一追踪。
- **自动清理**：卸载时按注册反顺序自动执行清理函数，等待异步清理完成，确保无残留。

### 9.6 Fiber 例子

Agent A 插件加载时拥有 Fiber：

- 执行 `ctx.effect(() => { const conn = createConnection(); return () => conn.close() })`，Fiber 收集清理函数。
- `ctx.on` 注册监听器、`ctx.tools.register` 注册工具也作为副作用被追踪。
- Agent A 卸载时，Fiber 进入 `UNLOADING`，逆序关闭连接、移除监听器、注销工具。

### 9.7 Scope 与 Fiber 的关系

- **Scope 是逻辑视图**：通过标签定义“什么应该被隔离”。
- **Fiber 是物理载体**：通过生命周期和副作用栈管理“如何实现隔离与清理”。

`createScope` 创建的 Scope，其底层就是一个 Fiber；Scope 的注册和销毁最终通过 Fiber 执行和保证。

---

## 十、工具注册与两个 Agent 的实例隔离

### 10.1 工具可以在运行时注册

工具并非全部预先存在、运行时只使用。注册位置取决于调用 `ctx.tools.register()` 时所在的上下文作用域：

- **全局注册**：普通插件上下文中注册，进入全局层，对所有 Agent 可见。
- **作用域注册**：在 Agent 专属上下文（`agent.ctx`）中注册，只为该 Agent 注册，并在其作用域内遮蔽同名全局工具。

Agent 运行时可以在 `setup(agentCtx)` 阶段动态挂载专属能力。

### 10.2 两个 Agent 是否各有一份工具实例

对于“改写文件”工具：

- **工具定义通常共享**：`ToolDefinition`（schema 和执行器）可以是共享引用。
- **注册层隔离**：每个 Agent 拥有独立扁平注册层，解析时只看到全局层 + 自己的层。Agent A 注册或覆盖的工具，Agent B 看不到。
- **执行资源隔离**：真正的隔离发生在执行阶段。文件路径会在 `tools/execute` 环节被重写到该子 Agent 专属私有挂载点。

总结表：

| 层面 | 是否隔离 | 说明 |
|---|---|---|
| 工具定义 | 否（通常共享） | schema 和执行器逻辑可共用 |
| 注册层 | 是 | 每个 Agent 有独立作用域层，可添加或遮蔽工具 |
| 执行资源 | 是 | 文件路径等执行时重写到 Agent 私有挂载点 |

所以两个 Agent 不是各自持有一份工具代码副本，而是各自拥有独立的“工具可见性列表”和“文件系统资源视图”。

---

## 十一、插件使用示例：`greet` 插件

### 11.1 方式一：安装社区插件

```bash
npx @deepseek-ai/dsh@0.2.0-rc.2 plugin --profile web add dsh-plugin-greet@0.3.2
npx @deepseek-ai/dsh@0.2.0-rc.2 web
```

在 Web UI 输入：

> Use the greet tool to greet Ada.

支持参数如 `language`（`en`/`zh`）、`style`（`friendly`/`formal`）。

### 11.2 方式二：通过 `--patch` 加载本地插件

准备 patch：

```yaml
- name: '@deepseek-ai/dsh-system-prompt'
- name: '@deepseek-ai/dsh-tools'
- name: './scratch-plugin/greet-tool.ts'
```

启动：

```bash
pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

### 11.3 编程式调用

```typescript
const result = await ctx.tools.execute({
  calls: [{
    type: 'tool-call',
    id: brandString<ToolCallId>('demo-1'),
    name: 'greet',
    arguments: JSON.stringify({ name: 'Cordis' }),
  }],
  signal: new AbortController().signal,
})
```

核心不变：最终 `ctx.tools` 注册表中多了一个名为 `greet` 的工具。

---

## 十二、总结：关键关系一览

| 主题 | 核心结论 |
|---|---|
| **Seam** | 完整可替换能力，包含服务定义、Provider、Consumer 三角色 |
| **Provider** | Seam 中的具体实现角色，决定能力“由谁、如何”实现 |
| **插件** | 最小可装卸功能单元 |
| **Bundle** | 插件的分发格式，声明如何安装配置插件 |
| **服务** | 插件生产出的命名能力接口，供其他插件注入消费 |
| **事件** | 既是插件通信总线，也是 Session 事件溯源的事实来源 |
| **Profile/Bundle/Patch** | 分层声明、按序叠加的装配系统 |
| **`ctx`** | Cordis 服务容器，不是会话实例；会话是其派生作用域 |
| **`ctx.agents`** | Agent 接口层与注册中心 |
| **`ctx.agentLoop`** | Agent 接口的唯一具体驱动器，负责 Turn/Step 生命周期 |
| **`ctx.sessions`** | 事件溯源交互日志，LLM 历史从日志派生 |
| **`ctx.workflows`** | 脚本驱动的多 Agent 编排 |
| **Scope** | 按 Agent 隔离的扁平注册层，基于标签与生命周期绑定 |
| **Fiber** | 插件运行时实例，管理生命周期与可逆副作用清理 |
| **工具隔离** | 定义通常共享，注册层隔离，执行资源隔离 |

> **总体一句话：DeepSeek Harness 通过 Cordis 内核提供可插拔运行时，通过 Seam/Provider 提供可替换能力，通过事件与 Session 保证可观察和可回放，通过 Scope/Fiber 保证按 Agent 隔离与自动清理，最终由 Profile/Bundle/Patch 组装成可运行的 Agent 底盘。**
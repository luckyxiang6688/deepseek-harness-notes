# DeepSeek Harness Super-agent 架构整理（总分结构）

> 本文整理此前讨论中关于 Super-agent 的全部核心内容，按“先总览、再分层展开”的结构组织，覆盖其定位、与 Harness 的关系、Router/Worker 双层架构、核心领域模型、权限策略、事件机制、注册内容与安装方式。

---

## 一、总览：Super-agent 是什么

Super-agent 是构建在 DeepSeek Harness 之上的**上层调度架构**，以 **Bundle/插件** 形式安装到 Harness 运行时中。它的目标不是“让一个 Agent 更聪明”，而是**让一组 Agent 像一个高效团队一样协同工作**，解决当任务从“一个”变成“多个”时的调度、隔离、权限与持续性问题。

### 1.1 核心定位

| 维度 | 说明 |
|---|---|
| **架构层级** | Harness 的上层应用，以 Cordis 插件身份入驻 |
| **核心模式** | Router / Worker 双层架构 |
| **设计目标** | 多任务并行、常驻协作、安全可控、可审计可重放 |
| **与 Harness 的关系** | Super-agent 是插件，Harness 是宿主运行时 |

### 1.2 一句话总结

> **Harness 提供“造 Agent 的底盘”，Super-agent 利用这个底盘构建了一个“Agent 团队管理系统”：Router 负责派活，Worker 负责干活，Mandate 划定边界，Policy 控制准入，Preset 定义角色，任务协议记录生命周期。**

### 1.3 与 `ctx.agents`、`ctx.agentLoop` 的层级区别

| 概念 | 架构层级 | 核心职责 | 类比 |
|---|---|---|---|
| **`ctx.agentLoop`** | 单个 Agent 内部 | 驱动一个 Agent 完成“思考-行动”的核心循环（Turn/Step） | 一个员工的个人工作流 |
| **`ctx.agents`** | 单个 Agent 的管理 | 管理 Agent 的身份、注册表和生命周期事件 | 公司的人事部 |
| **Super-agent** | 多个 Agent 的系统 | 通过 Router/Worker 双层架构，协调多个 Agent 并行工作，解决跨会话的任务管理问题 | 公司的项目管理办公室（PMO） |

**关键关系**：Super-agent 使用 `ctx.agents` 来管理众多的 Worker Agent，而每个 Worker Agent 内部则由 `ctx.agentLoop` 驱动其工作循环。Super-agent 解决的是“如何管理一个 Agent 团队”，`ctx.agents` 和 `ctx.agentLoop` 解决的是“如何构成一个 Agent 个体”。

---

## 二、Router：全局调度中枢

### 2.1 核心定位

Router 是 Super-agent 架构中的**常驻全局协调者**，设计原则是 **“只动口，不动手”**。它不深入任何具体任务的执行细节，而是专注于协调、决策与分派。

### 2.2 核心能力

1. **统一入口与全局上下文维护**  
   所有人类消息或外部事件先汇集于 Router。它维护全局层面的交互上下文，理解不同任务之间的关联、整体进展以及需要向用户反馈的信息。

2. **任务归属判断与分派**  
   当新消息或事件到达时，Router 的核心决策是判断“这件事属于哪个任务”，然后将其分派给对应的 Worker。

3. **跨任务的协调与汇总**  
   Router 同时管理多个并行任务的生命周期，不等待某个 Worker 完成，而是持续接收新消息、协调其他工作，并汇总 Worker 反馈的进展。

4. **决策与指挥，而非执行**  
   Router 的产出是“决策”（把什么任务、在何时、交给哪个 Worker），而不是代码、测试或文件修改。

### 2.3 场景举例：GitHub 工作流

- **场景**：系统同时收到一个 GitHub Issue（要求修复 Bug）和一个 Pull Request（需要进行代码审查）。
- **Router 的调度**：
  1. 判断与分派：Router 识别出两个独立任务，将“修复 Bug”分派给 Worker A，将“审查 PR”分派给 Worker B。
  2. 并行的 Worker：Worker A 和 Worker B 在各自完全隔离的 Session 中并行工作，上下文和中间结果互不干扰。
  3. Router 的持续协调：Worker 工作期间，Router 继续接收新消息（如 Issue 新评论），判断归属并路由给对应 Worker，同时接收 Worker 反馈的进展。

**关键点**：任务分派是**架构层面的职责约束**，而不是依赖某个主 Agent“记得”去委派，避免了主 Agent 因自行处理耗时任务而阻塞整个系统。

### 2.4 Router 如何处理消息积压

Router 面对“消息快于 Worker 处理”的压力时，核心策略是**分层卸载**，而非让 Router 变快。

#### 📥 直通 Worker（Subscription Direct-pass）

当 Router 处理了某个 Thread 的**第一条消息**、并为其创建（或分配）了 Worker 后，**该 Thread 后续的所有消息不再经过 Router**，而是由程序直接投递给已订阅的 Worker。这从源头消除了绝大部分可能的积压。

#### 📦 批处理（Batching）

对于不得不经过 Router 的消息，在一个 Router 推理轮次进行期间，新到达的 Trigger 被**暂存并合并**，一起注入到**下一次**模型调用中。这利用了 LLM 推理的 Prefill 阶段并行计算特性，额外耗时几乎可忽略。

#### 🚦 并发上限与排队

Super-agent 设置了**最大 Worker 并发上限**。超过阈值的任务**进入排队**，而不是无限创建新的会话和 Worker，保护系统稳定性。

#### 🧹 入口过滤与异步化

消息到达 Router 之前，先经过**程序化的快慢双通道过滤**：按来源、群组、白名单等规则过滤噪音，只有候选事件才交给 Router 的模型判断。Worker 侧采用**异步 ReAct 循环**，耗时的工具调用不阻塞 Worker 循环。

---

## 三、Worker：执行单元

### 3.1 核心定位

Worker（Executor）是 Super-agent 中的**具体执行单元**。每个 Worker 在**独立的 Session** 中运行，拥有隔离的上下文和完整的工具环境，专注于推进一项具体任务。

### 3.2 驱动方式

**Worker 由 DeepSeek Harness 的 `AgentLoop` 驱动**。每一个 Worker 本质上都是一个在 Harness 上运行的独立 Agent 会话，其“思考-行动”循环由 `ctx.agentLoop` 管理。Super-agent 的调度逻辑是在 `AgentLoop` 这个执行引擎**之上**的一层编排。

### 3.3 生命周期与上下文复用

- **阶段性完成后不销毁**：当 Worker 完成一个阶段的工作后，通常不会被立即销毁。
- **可复用 Worker**：对于连续性的任务，可以通过传入 `reuse_task_id` 参数，让新任务**复用某个已完成任务的 Worker 上下文**。
- **上下文生命周期**：一个 `Ticket` 可以比一次具体的 `dispatch` 存活更久，而 `Worker Session` 承载实际执行过程。只要 `Ticket` 还在，相关的 Worker 上下文就**没有被销毁的必要**。

---

## 四、Thread 与 Worker 的多对多订阅关系

### 4.1 关系本质：订阅，而非绑定

Thread 和 Worker 之间不是固定的一对一绑定，而是**订阅关系（Subscription）**：

- **一个 Worker 可以订阅多条 Thread**。
- **一条 Thread 也能被多个 Worker 订阅**（例如两个 Worker 同时关注同一个 PR，通过 PR 本身协调）。

### 4.2 消息如何精确投递

当一条消息到达一个已被多个 Worker 订阅的 Thread 时，系统并不会“选择”其中一个，而是**广播给该 Thread 上所有活跃的订阅者**。

**核心机制**：Thread 持有**一个订阅列表（Subscriptions）**，这是一个 Thread 与多个 Ticket（工作项）之间的多对多关系。当一个 Trigger 被归入某个 Thread 后，系统执行 **“Subscription fan-out”**：将该 Trigger 作为上下文，投递给**每一个**订阅了此 Thread 的活跃 Ticket。

### 4.3 Worker 收到消息后的处理

每个 Worker 收到消息后，依据自己所持的 **Mandate（授权范围）** 来决定行动：

- **在自己的范围内**：消息涉及的内容属于该 Worker 被授权处理的范围，则在自己的 Session 中继续处理。
- **超出范围则“交还”**：如果消息内容超出该 Worker 的 Mandate，它会将消息**“交还（Hand back）”** 给系统，重新进入 Eval 流程，由 Router 判断是否需要创建新的 Ticket。

### 4.4 关键安全设计

**通过订阅直接送达的消息，只传递上下文，不传递任何新的权限**。这条消息可以 bypass 启动新任务所需的“门控”，但它**不能**扩大 Worker 已有的权限边界。Worker 能做什么，始终严格受限于它最初被分配的那个 Mandate。

---

## 五、核心领域模型

Super-agent 注册了一套**轻量的任务协议**，用于记录任务从诞生到交付的完整生命周期，使其可重放、可审计。

### 5.1 领域对象一览

| 对象 | 核心作用 |
|---|---|
| **Trigger** | 记录“从哪里来了一条什么事件”，是任务的**原始输入**，不可变 |
| **Thread** | 按时间顺序**组织相关 Trigger** 的逻辑容器，是事件聚合的单元 |
| **Ticket** | 代表一个持续跟进的工作项，是任务在系统中的**持久化“身份证”** |
| **Assignment** | 记录 Ticket 与具体执行者（Worker）之间的**绑定关系** |
| **Subscription** | Thread 与 Ticket 之间的**多对多关系**，决定后续事件投递给哪些工作项 |
| **Mandate** | 随 Ticket 一同签发的**不可变授权契约**，界定 Worker 的权限边界 |
| **Eval** | 新任务的**“准入门控”**，判断新 Trigger 是否构成需要启动新工作的请求 |
| **Channel** | 定义 Ticket 的处理结果应写回到哪个外部地址（如 GitHub Issue、PR） |
| **Source** | 外部系统的**连接器**，从 GitHub、飞书等读取事件流并转化为 Trigger |

### 5.2 流水线

> **Source 产生 Trigger → Trigger 被归入 Thread → 新 Trigger 经 Eval 门控创建 Ticket（并附带 Mandate）→ Ticket 通过 Assignment 交给 Worker 执行 → 已有的 Thread 上的后续消息，通过 Subscription 直接扇出给已订阅的 Worker**

### 5.3 持久化

这些对象以 JSONL 或 SQLite 账本（Ledger）等形式持久化，使任务**可重放、可审计**，也使得“同一条事件重复接收时不会创建两个 Ticket”这类去重逻辑成为可能。

---

## 六、Mandate：不可变的授权契约

### 6.1 核心机制

Mandate 是**在任务被接受（Eval）时一次性签发的、不可变的“授权契约”**。它回答的是“**这项任务被允许在什么边界内、以什么强度执行**”，而不是“具体每一步该怎么做”。

### 6.2 权限授予的组成要素

| 要素 | 说明 |
|---|---|
| **Policy 版本** | 授权所依据的规则集版本，用于判断旧授权是否仍然有效 |
| **Scope（范围）** | 明确界定 Worker 被允许操作的对象边界 |
| **Strength（强度）** | 授权的“力度”等级，决定 Worker 执行的自主程度 |
| **匹配的规则** | 具体是哪一条准入规则让请求被接受，记录授权来源依据 |
| **发送者身份** | 源系统上发起请求的人，是权限的**原始来源** |

### 6.3 不可变性

一旦签发，Mandate 的内容**永远不会被修改**。如果需要扩大工作范围，**不能**修改原有 Mandate，而必须**开启一个新的 Ticket 并签发一份全新的 Mandate**。

### 6.4 权限与内容的彻底隔离

消息、评论和引用的文档，只是决策的“输入”，**它们本身无法改变已经授予的权限**。即使一条 PR 评论里包含提示注入（Prompt Injection），这条消息在 Mandate 的授权模型里**根本没有“修改权限”这个能力位**。

### 6.5 撤销与重验

1. **显式撤销**：**只有人**（而不是 Worker 自己）可以发出 `stop` 指令来显式撤销一个 Mandate。
2. **策略重验（Revalidation）**：任务中断后恢复执行之前，系统会强制用**当前的 Policy** 重新校验 Mandate 是否依然成立。如果授权所依赖的条件已失效，Mandate 被判定失效，任务无法继续。

### 6.6 总结

> Mandate 的授权方式可以概括为 **“一次签发，边界固化”**。它在任务开始时，就静态地、不可变地划定了权限的**最大范围（Scope）和最高强度（Strength）**。后续所有消息内容，都只能在这个围栏内影响“具体怎么做”，而无法扩张围栏本身。

---

## 七、Policy：Super-agent 的权限策略

### 7.1 核心定位

Super-agent 自带的 Policy 是一套**上层业务策略**，专门回答“**谁**可以发起工作，以及这项工作**允许做到什么程度**”。与 Harness 底层通用的审批机制不同，它是策略层而非执行层。

### 7.2 核心机制

**1. 严格的“准入名单”（Allowed Actors）**  
系统默认**只接受**来自 `allowedActors` 白名单中成员的**显式请求**。不在名单上的人员或机器人产生的“推断性候选”，默认都被拒绝。

**2. 分级的“请求强度”（Explicit vs. Inferred）**  
- **Explicit（显式）**：白名单人员直接对 Agent 提出的请求，任何模式下都有效。
- **Inferred（推断）**：来自机器人或准入表格的间接线索，**永远不会**被当作显式请求处理。这从源头防止了通过伪造消息提升权限。

**3. 可渐进式的“信任模式”（Off → Observe → Auto）**  
- **Observe（观察模式）**：推断性候选请求**只记录、不执行**，供人类审查。
- **Auto（自动模式）**：积累足够信任后，符合条件的推断请求可自动执行。但即便在自动模式下，接受非显式候选**仍然强制需要经过评估环节**。

**4. 内置的“操作审批门控”（Approval Gates）**  
通过 `requireApprovalFor` 列表，强制要求某些高风险类别的操作（如 `write`、`network`、`publish`）在执行前必须获得批准。通过 `maxSteps` 限制单个执行器的最大步数，实现“有界自治”。

**5. 版本化与可审计的规则集**  
Policy 是**版本化**的，每次修改生成不可变编号快照并记录修改来源，支持回滚。Policy 变更时，系统将其投递给所有活跃 Ticket，但**不会撤销已签发的 Mandate**。

### 7.3 与 Harness 底层权限机制的关系

- **Super-agent 的 Policy（策略层）**：定义业务规则，回答“这个任务该不该被创建？”以及“它被允许在什么范围内活动？”。它决定了工作流的“入口”和“边界”。
- **Harness 的底层机制（执行层）**：提供安全执行。当 Worker 在 Mandate 范围内尝试执行具体高风险操作时，Harness 的**审批门禁**和**沙箱**进行实时校验和拦截。它回答“当前这个具体动作能否被执行？”。

> 两者构成纵深防御：Policy 负责“把不该做的工作挡在门外”，Harness 审批负责“把允许做的工作安全地执行”。

---

## 八、事件与授权：Super-agent 与 Harness 的关系

### 8.1 事件系统：策略过滤 vs. 通信总线

- **Harness 的事件系统是底层的“通信总线”**：基于 Cordis 框架，提供 `emit`、`waterfall` 等多种分发模式，负责在插件间可靠地传递和分发类型化事件。
- **Super-agent 的事件逻辑是上层的“策略过滤器”**：不取代 Harness 的事件总线，而是在其之上构建**两级过滤**机制：
  1. **程序层（Fast Path）**：用纯代码规则（来源、白名单）快速过滤噪音。
  2. **语义层（Slow Path）**：通过初筛的事件（被抽象为 Trigger）进入 Router，由 LLM 进行语义研判。

> Harness 的事件总线负责“如何送达”，Super-agent 的事件逻辑定义了“是否接收以及如何处理”。

### 8.2 授权委托：宏观凭证 vs. 微观审批

- **Super-agent 的 Mandate 是宏观的“任务授权凭证”**：定义 Worker 在整个任务生命周期内的权限边界，不可变。回答“这个任务被允许做什么”。
- **Harness 的审批系统是微观的“单次操作门禁”**：通过 `ctx.approval` 对每一个具体的高风险操作进行实时审批，默认 fail-closed。回答“当前这个动作能否执行”。

> Super-agent 的 Mandate 划定了 Worker 的活动范围，Harness 的审批系统在这个范围内对每一次越界或高风险的具体操作进行实时校验和拦截。

---

## 九、Super-agent 与 Harness 的连接机制

### 9.1 连接本质：以 Cordis 插件身份“入驻”

Super-agent 并非运行在 Harness 之外，而是**作为插件（Bundle）安装在 Harness 运行时之上**。可以把它理解为一个安装在操作系统（Harness）上的高级应用程序（Super-agent）。

- **Harness 是宿主（Host）**：提供 Cordis 插件内核、服务注册表（`ctx`）、事件总线、会话日志等底层机制。
- **Super-agent 是插件（Plugin/Bundle）**：通过 Harness 的插件机制（如 `cordis.patch.yml`）安装到系统中。

连接发生在 Harness 加载 Profile 时。Super-agent 的 Bundle 中包含 `cordis.patch.yml` 和 `src/index.ts` 入口，Cordis 运行时执行其 `apply(ctx)` 函数，Super-agent 获得与 Harness 内核交互的入口 `ctx`，并开始**自注册（Self-registration）**。

### 9.2 Super-agent 注册的核心内容

#### 🧩 核心服务与入口

| 注册内容 | 核心作用 |
|---|---|
| **`super-agent` 服务** | Super-agent 在 Harness 中暴露的**主入口**，封装任务调度、路由决策等上层逻辑。UI 或其他插件通过 `ctx.superAgent` 访问其能力 |
| **Preset 目录** | 注册 Router 和 Worker 两种角色预设，实现**角色分离**。Router 预设配置轻量模型、不挂载重型执行工具；Worker 预设配置强模型和完整工具集 |

#### 📜 任务协议与领域模型

注册一套**轻量的任务协议**，用于记录任务从诞生到交付的完整生命周期。核心领域对象包括 Trigger、Thread、Ticket、Assignment、Subscription、Mandate、Eval、Channel、Source（详见第五节）。

#### 🚪 事件源与数据存储

| 注册内容 | 核心作用 |
|---|---|
| **Sources（事件源）** | 如 `GitHub`、`inbox` 等 Source 插件，从外部系统轮询或监听事件，转化为内部 Trigger，是 Super-agent 感知外部世界的“感官” |
| **Store（状态存储）** | 运行时锁定状态目录并打开 Store，用于持久化 Trigger、Ticket、Decision 等核心对象（通常为 SQLite 账本），保证任务状态在进程重启后依然可恢复和审计 |

#### 🖥️ 控制面板与可观测性

| 注册内容 | 核心作用 |
|---|---|
| **面板路由（Panel Routes）** | 注册 Web 面板路由，提供原生界面展示任务活动、人工处理事项，复用 DSH 官方会话界面 |
| **UI 投影（Projections）** | 如 `superAgentUsage` 投影，展示 Token 消耗等运行指标，服务于可观测性和可配置性 |

### 9.3 连接关系总结

> Super-agent 通过 **Cordis 插件机制**实现自注册。它向 Harness 的 `ctx` 注册的是一整套**任务生命周期管理组件**：以 `super-agent` 服务为入口，通过 **Preset 目录**定义 Router/Worker 角色，以**任务协议**作为状态载体，由**事件源**驱动、**控制面板**呈现。这些注册内容共同将 Harness 从一个单 Agent 运行时，扩展为一个能够安全、可审计地调度多任务、管理常驻 Agent 团队的**上层调度系统**。

---

## 十、Preset 目录：Agent 的“配置模板”

### 10.1 本质

Preset 目录是一个**按会话（per-session）组装 Agent 的配置单元**。一个 Preset 就是一个目录，里面包含一个 **`agent.cordis.yml`** 文件，定义了该预设会加载哪些插件（即挂载哪些工具、提示词片段、技能等）。

### 10.2 核心作用

**不同的 Preset 让同一个 Harness 进程可以同时运行多个“配置不同”的 Agent**。一个会话如果指定了某个 Preset，它就会获得该 Preset 定义的工具和技能，而其他会话保持自己的配置，互不干扰。

### 10.3 在 Super-agent 中的应用

Super-agent 的 Router 和 Worker 职责分离，通过**为它们指定不同的 Preset**来实现：

- **`dsh-superagent-router`**：用于主会话，承担路由职责。配置轻量模型，可以不挂载执行工具，专注于沟通和派活。
- **`dsh-superagent-worker`**：用于执行会话，承担具体工作。配置更强的模型，挂载完整的工具集，在隔离的上下文中执行任务。

> Preset 目录的存在，使得 Router 和 Worker 的角色分离不是硬编码，而是通过**可选择的配置组合**来实现的。

---

## 十一、安装方式

Super-agent 通常**不是通过图形化“插件市场”点击安装**，而是通过 DeepSeek Harness 提供的**命令行插件管理工具 `dsh plugin`** 完成。核心命令格式：

```bash
dsh plugin --profile <profile名> add <来源>
```

### 11.1 方式一：从 npm 包安装

```bash
dsh plugin --profile web add dsh-super-agent
```

> 注意：npm 上的 `dsh-super-agent` 包可能已被标记为废弃，并提示迁移至 `dsh-super-code`。安装前建议确认最新状态。

### 11.2 方式二：从 GitHub 仓库安装

```bash
npx -y @deepseek-ai/dsh plugin --profile web add github:ArtlexYoung/dsh-super-agent#c7b77d34bc4321599e006314c96489ef4f32cb64
```

优点是**精确锁定到某个 Commit 或分支**，适合需要特定版本或进行二次开发的场景。

### 11.3 方式三：从本地源码安装

```bash
dsh plugin --profile web add /abs/path/to/dsh-super-agent
```

这是开发者最常用的方式，允许本地修改代码后直接安装测试。

### 11.4 安装后验证

1. **重启 DSH 服务**：运行 `dsh web` 重新启动 Harness 的 Web 界面。
2. **强制刷新浏览器页面**：清除旧的前端缓存。
3. **检查侧边栏**：DSH 左侧栏应出现 **“Super-agent”** 入口。首次打开时系统会自动为其准备独立工作区。

---

## 十二、总结：关键关系一览

| 主题 | 核心结论 |
|---|---|
| **Super-agent 定位** | Harness 的上层调度架构，以 Cordis 插件身份入驻，构建多 Agent 团队管理系统 |
| **与 `ctx.agents`/`ctx.agentLoop`** | Super-agent 使用 `ctx.agents` 管理 Worker，每个 Worker 由 `ctx.agentLoop` 驱动 |
| **Router** | 常驻全局协调者，“只动口不动手”，负责判断、分派、协调、汇总 |
| **Worker** | 具体执行单元，在独立 Session 中运行，由 `AgentLoop` 驱动，可复用上下文 |
| **Thread/Worker 关系** | 多对多订阅关系，消息通过 Subscription fan-out 广播给所有订阅者 |
| **消息积压处理** | 直通 Worker、批处理、并发上限、入口过滤四层卸载 |
| **Mandate** | 不可变授权契约，一次签发、边界固化，消息内容无法改变权限 |
| **Policy** | 上层业务策略，默认保守准入，Explicit/Inferred 分级，Observe→Auto 信任模式 |
| **事件与授权关系** | Super-agent 是策略层（过滤、授权），Harness 是机制层（总线、审批、沙箱） |
| **注册内容** | `super-agent` 服务、Preset 目录、任务协议、Sources、Store、面板路由、UI 投影 |
| **Preset** | 按会话组装 Agent 的配置模板，Router/Worker 角色分离的实现方式 |
| **安装方式** | `dsh plugin --profile <name> add <npm包/GitHub仓库/本地路径>` |

> **总体一句话：Super-agent 是 Harness 上的“Agent 团队管理系统”：Router 负责派活，Worker 负责干活，Mandate 划定边界，Policy 控制准入，Preset 定义角色，任务协议记录生命周期，事件源驱动运转，一切以 Cordis 插件身份注册到 Harness 的 `ctx` 之上。**
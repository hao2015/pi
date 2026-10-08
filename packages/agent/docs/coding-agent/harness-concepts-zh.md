# Harness 概念层次

AgentSession、AgentHarness、AgentLane、Session、Branch、SessionRepo、invocation Context 各自包含什么、相互关系。

```text
┌─────────────────────────── 编排层（pi 特有）─────────────────────────────┐
│  AgentSession  core/agent-session.ts                                    │
│  prompt/steer/followUp 队列 · compact · 工具注册表 · 扩展绑定 · 重试      │
│  持有 Agent + AgentHarness，订阅事件，把用户操作翻译成 harness 调用        │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   ▼
┌──────────────────── packages/agent：durable harness ─────────────────────┐
│  AgentHarness（门面，agent-harness.ts）                                  │
│  lane 管理 · 全局配置读写（tools/models/retry/compaction/queue modes）    │
│  watchSession · hooks · events · close                                  │
└───────────┬─────────────────────────────────────────────────────────────┘
            │ lane(name) — 每个 lane 绑定一个 Branch
            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  AgentLane（一条对话路径上的执行单元，runtime/lane.ts）                    │
│  accept() + drive()（真正的 agent 行为）· prompt/steer/followUp/abort    │
│  compact · navigateTree · resume · per-lane 模型/工具配置 · watch        │
└───────────┬─────────────────────────────────────────────────────────────┘
            │ 读写 entry、切换历史
            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Session（一个会话的持久化记录，session/types.ts）                        │
│  entry 存储 · Branch 操作 · values/lists（typed 地址寻址）               │
│  单写者 mutation 线：beginMutation()（独占，0/1 次 commit）              │
└───────────┬─────────────────────────────────────────────────────────────┘
            │ 持有底层 Storage（commit + scan）
            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  SessionRepo（session 生命周期工厂）                                     │
│  create / open / list / delete / fork（按 branch 或整树复制）             │
│  实现：MemorySessionRepo（测试）· JsonlSessionRepo（JSONL 主文件+sidecar）│
└─────────────────────────────────────────────────────────────────────────┘

  Branch：session 内 entry 树（按 parentId）的一条命名路径；
          tip 决定下一次模型请求的历史；能力面最窄：
          getTipId / findEntries / appendMessage / appendCustomEntry

  Context（invocation context，来自 chord）：每一层每个方法的尾随参数
  { abortSignal, telemetryContext, value(key) }
  不可变，父链接 + copy-on-write：withAbortSignal / withTelemetryContext /
  withContextValue。不用 AsyncLocalStorage，不保留 caller context。
```

## 各概念包含的内容

**AgentSession**（`packages/coding-agent/src/core/agent-session.ts`，约 4000 行）
pi 特有的编排层。持有 `Agent`（loop 在 `packages/agent/src/agent-loop.ts`）和 `AgentHarness`；管理 prompt/steering/followUp 队列、compact、重试、扩展绑定、会话树导航、per-lane 模型/工具配置的翻译。

**AgentHarness**（`packages/agent/src/harness/agent-harness.ts:587`）
durable harness 的门面，不持有运行状态。管理 lane 的获取/创建（`lane(name)`）、全局配置（tools/models/streamOptions/retryPolicy/compactionSettings/queueModes）、`watchSession()` 全局快照、`hooks`/`events`。构造 `AgentHarness.create(options, context)` 返回 harness + `OpenOperation[]`——附着到一个已打开的 session。

**AgentLane**（`agent-harness.ts:538`）
一条对话路径上的执行单元，绑定一个 Branch。`accept(request)` + `drive(options)` 接受并驱动操作；`prompt/steer/followUp/abort/compact/navigateTree/resume` 是用户级操作；`getModel/setModel/setThinkingLevel/setActiveTools` 是 per-lane 配置。一个 harness 可有多个 lane（子 agent、后台任务），对应 "subagents = child lanes"。

**Session**（`packages/agent/src/harness/session/types.ts:530`）
一个会话的持久化记录。entry 存储（`getEntry`/`findEntries`/`scanBranch`）、Branch 操作（`branch(name)`/`createBranch`）、values/lists（`setValue`/`appendList`，typed 地址寻址）、单写者 mutation 线（`beginMutation()` 返回独占 `SessionMutation`，恰好 0/1 次 commit，二次拒绝；`mutate(callback)` 是回调域内的独占变更能力）。

**Branch**（`session/types.ts:521`）
session 内 entry 树的命名路径。entry 按 `parentId` 构成树，从根到叶的每条路径是一个 branch；tip 决定下一次模型请求的历史。`AgentSession.navigateTree()` 切换的就是 branch tip。能力面最窄：`getTipId`/`findEntries`/`appendMessage`/`appendCustomEntry`。

**SessionRepo**（`session/types.ts:592`）
session 的生命周期工厂：`create/open/list/delete/fork`。`fork` 支持按 branch 或整棵树复制（复制配置，lane 状态重置为 idle）。实现：`MemorySessionRepo`（测试，无持久化）、`JsonlSessionRepo`（JSONL 主文件 + sidecar）。

**Invocation Context**（chord `types.ts:15`；规格 `packages/agent/docs/telemetry.md`）
每个公开方法尾随的 `Context` 参数，刻意设计（同一接收方可能同时服务本地调用者和 RPC 客户端，不能自己保留 caller context，也不能用 `AsyncLocalStorage`）。包含 `abortSignal`（RPC adapter 映射请求取消）、`telemetryContext`（span 父子关系）、`value(key)`（调用域内的值：权限、租户 ID 等）。

## 相互关系

- `createAgentSession()` 是所有入口（TUI/print/RPC/SDK）的唯一构造通道，内部创建 Session、AgentHarness、Agent 并组装成 AgentSession。
- AgentSession 调 AgentHarness；AgentHarness 通过 `lane(name)` 派发 AgentLane；AgentLane 绑定一个 Branch 并在一条对话路径上驱动执行。
- AgentLane 读写 Session（entry 存储、values、mutation 线）；Session 持有底层 Storage；SessionRepo 在最外层管理 session 的生命周期（创建/打开/分叉/删除）。
- Context 是横切的：从最外层调用到最内层实现，每一层的每个方法都尾随 `Context`，取消和遥测沿 ctx 传播。

## 相关文档

- `packages/agent/docs/rpc.md`：service transport 语义（RPC 模式下 strict-JSON 边界、context 剥离与重建、requestId 取消映射）
- `packages/agent/docs/telemetry.md`：telemetry 模型（Context 的 telemetryContext、span 层级 caller → rpc.client → rpc.server → implementation）
- `packages/agent/docs/pico-v3.md` / `pico2.md`：下一代 harness 重设计（pico3 是已实现但未接线的实验）
- `architecture-zh.md`：AgentSession / agent loop / compaction / 工具执行的整体架构图

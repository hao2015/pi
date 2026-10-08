# Pi Agent Harness — 新成员上手指南 (Onboarding)

> 由 `/understand` 知识图谱生成，对应 commit `bf8e4b953`。数据源：`.ua/knowledge-graph.json`。

## 项目概览

| | |
|---|---|
| **名称** | pi-monorepo (Pi Agent Harness) |
| **语言** | TypeScript 为主，辅以 markdown、javascript、yaml、shell、dockerfile、sql 等 |
| **框架** | Docker、GitHub Actions、Vitest |
| **说明** | npm workspaces monorepo，包含可自我扩展的编码 agent CLI、agent 运行时、统一多供应商 LLM API，以及配套的 TUI、持久化、遥测等包 |

## 架构分层 (9 层)

依赖方向：`coding-agent → ai / tui / agent / chord`，`agent → ai / chord`，`durable → chord`。

1. **CLI 应用层** (513 nodes) — `packages/coding-agent`：CLI 入口、交互模式、扩展系统、配置与迁移逻辑；顶层消费者。
2. **Agent 运行时层** (208) — `packages/agent`：agent 事件循环、harness 会话上下文与类型基础。
3. **AI 提供商 API 层** (208) — `packages/ai`：anthropic/openai/google/bedrock 等适配器、模型目录与 OAuth 认证。
4. **终端 UI 层** (78) — `packages/tui`：终端 UI 组件库与差分渲染。
5. **共享基础层** (112) — chord、durable、protocol、telemetry。
6. **客户端/服务端层** (37) — `packages/server` + `packages/client`：WebSocket 远程会话通道。
7. **评测层** (28) — `packages/evals`：模型评测套件 + Docker 沙箱。
8. **会话后端层** (34) — `packages/session-backends`：SQLite 持久化与迁移。
9. **构建与项目基础设施** (75) — 构建脚本、test.sh、CI 工作流、husky 钩子、根配置。

## 核心概念

- **三层组装**：`cli.ts`（薄入口）→ `main.ts`（分发）→ `sdk.ts`（`createAgentSession` 一站式组装资源加载、模型解析、认证）→ `agent-session.ts`（大脑）。
- **类型先行**：各包都有核心类型文件（`packages/agent/src/types.ts`、`packages/ai/src/types.ts`），是跨包契约。
- **统一资源路径**：`coding-agent/src/config.ts` 是唯一资源路径来源（兼容源码/npm/二进制三种安装方式）——不要用 `__dirname`。
- **扩展系统**：`core/extensions/types.ts` 定义扩展作者的公共 API 契约。
- **erased-only TS**：`packages/*/src` 只用可擦除语法（strip-only 模式），不能有 enum/namespace/参数属性。

## 学习路线 (15 步)

1. **项目总览** — `README.md` + `packages/coding-agent/docs/how-pi-works.md`
2. **CLI 入口链** — `cli.ts` → `cli/setup.ts` → `main.ts`
3. **核心配置** — `coding-agent/src/config.ts`
4. **Agent 类型基础** — `agent/src/types.ts` + `docs/harness.md`
5. **Harness 运行时核心** — `agent-harness.ts`、`runtime/lane.ts`、`runtime/drive.ts`
6. **AI 多提供商 API 层** — `ai/src/types.ts`、`models.ts`、`providers/all.ts`
7. **提供商适配器** — `api/anthropic-messages.ts`、`api/openai-completions.ts`
8. **终端 UI 组件库** — `tui/src/index.ts`
9. **内置工具系统** — `core/tools/edit.ts`、`read.ts`、`bash.ts`
10. **扩展系统** — `core/extensions/types.ts`
11. **交互模式编排** — `interactive-mode.ts` + `theme/theme.ts`
12. **AgentSession 大脑** — `core/agent-session.ts` + `sdk.ts`
13. **会话持久化 Schema** — sqlite-node `001_initial.sql`（entries / scalar_values / branch_entries 表）
14. **远程会话架构** — `protocol` → `server` → `client` → `chord`
15. **评测、CI 与发布** — `evals/docker/Dockerfile`、`.github/workflows/ci.yml`、`test.sh`

## 关键文件地图

| 文件 | 作用 |
|---|---|
| `coding-agent/src/cli.ts` | CLI 薄入口，全局初始化后交给 main.ts |
| `coding-agent/src/config.ts` | 核心配置：常量、安装方式检测、全部资源路径 helper |
| `coding-agent/src/core/sdk.ts` | SDK 公共入口：`createAgentSession` 一站式组装 |
| `coding-agent/src/core/agent-session.ts` | ~4000 行大脑：agent 循环、消息队列、compaction、重试 |
| `coding-agent/src/core/extensions/types.ts` | 扩展系统类型契约（扩展作者 API） |
| `coding-agent/src/modes/interactive/interactive-mode.ts` | ~6900 行交互模式总编排 |
| `agent/src/types.ts` | 消息、工具调用、流事件等类型基础 |
| `agent/src/harness/agent-harness.ts` | AgentHarness 公共接口与选项 |
| `ai/src/models.ts` | Models 运行时：provider 注册、认证、流式对话 |
| `ai/src/types.ts` | ~1100 行 AI 类型契约 |
| `tui/src/index.ts` | TUI 公共 API barrel |
| `protocol/src/index.ts` | 握手/信封协议消息 |
| `server/src/index.ts` | Server、SessionRouter、传输层 |

## 复杂度热点 (新手谨慎)

**最大热点**（阅读顺序建议：先接口后实现）：

- `coding-agent/src/modes/interactive/interactive-mode.ts` — 交互模式总编排，单文件最大
- `coding-agent/src/core/agent-session.ts` — 会话大脑，约 4000 行
- `coding-agent/src/core/sdk.ts` / `ai/src/types.ts` / `agent/src/types.ts` — 类型密集的契约文件
- `coding-agent/src/config.ts` / `core/extensions/types.ts` + `loader.ts` + `runner.ts` — 扩展系统三件套
- `ai/src/api/*` — 每个供应商一个适配器（anthropic、openai-responses、google、bedrock、mistral…），改一处要理解 `transform-messages.ts` 的公共转换层
- `ai/src/auth/oauth/*` — OAuth 流程各供应商独立实现
- `coding-agent/src/core/tools/*` — bash/edit-diff/grep/find 工具实现

全仓 complex 级文件 305 个，集中在 `coding-agent/src/core` 与 `ai/src/api`。

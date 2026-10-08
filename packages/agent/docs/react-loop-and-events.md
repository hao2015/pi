# Agent Architecture: ReAct Loop, Turn, and Event Mechanism

本文档总结了项目（`pi-mono`）中核心 **ReAct 循环**、**Turn（轮次）** 以及 **Event（事件机制）** 的代码实现与职责划分。

## 1. ReAct 循环 (Reasoning + Acting Loop)

核心 ReAct 循环位于 [`packages/agent/src/agent-loop.ts`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent-loop.ts) 中的 [`runLoop`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent-loop.ts#L155) 函数。

### 循环结构

- **外层循环 (`while (true)`)**：
  位于 [`packages/agent/src/agent-loop.ts:170`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent-loop.ts#L170)，负责监控并注入 Steering Queue（运行中插入的指导消息）和 Follow-up Queue（排队等待的后续消息）。
- **内层循环 (`while (hasMoreToolCalls || pendingMessages.length > 0)`)**：
  位于 [`packages/agent/src/agent-loop.ts:174`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent-loop.ts#L174)，实现了标准的 ReAct 交互逻辑：
  1. **模型请求与流式响应**：调用 `streamAssistantResponse` 向 LLM 求解并获取流式输出。
  2. **Tool Calls 解析**：从模型响应中提取 `toolCall` 内容块。
  3. **工具执行**：通过 `executeToolCalls` 执行该批次工具，收集 `toolResults` 并存入上下文。
  4. **状态判断**：更新 `hasMoreToolCalls` 状态并检查终止条件。

### Agent 封装类

[`packages/agent/src/agent.ts`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent.ts) 中的 [`Agent`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent.ts) 类封装了 `runLoop`，管理全局对话状态（`AgentState`）、消息队列以及启动 ReAct 循环的 `prompt()` / `continue()` 方法。

---

## 2. Turn（轮次）管理

**Turn** 指从模型生成单次回答（助手消息）到该回答触发的所有工具调用执行结束的全过程。

### 底层 Turn 钩子与生命周期

- **事件发射**：进入新一轮时触发 [`turn_start`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent-loop.ts#L176)，完成工具批处理后触发 [`turn_end`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent-loop.ts#L224)。
- **动态更新机制**：每轮 Turn 结束时调用 `prepareNextTurn` 允许修改下一轮的模型、thinking level 或上下文，并通过 `shouldStopAfterTurn` 检查是否需要安全退出。

### Session 层扩展

[`packages/coding-agent/src/core/agent-session.ts`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/coding-agent/src/core/agent-session.ts) 中的 [`AgentSession`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/coding-agent/src/core/agent-session.ts) 在 Agent 基础上增加了高层 Turn 控制：
- 对话上下文自动压缩（Auto-Compaction）；
- 模型回退（Model Fallback）；
- Turn 级别的重试与打断逻辑。

---

## 3. Event（事件派发与订阅分发）

项目采用基于事件驱动的解耦设计，用于更新 UI、日志记录和 RPC 协议传输。

### 事件类型定义

在 [`packages/agent/src/types.ts:428`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/types.ts#L428) 中定义了 [`AgentEvent`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/types.ts#L428) 联合类型：
- **Agent 生命周期**：`agent_start`, `agent_end`
- **Turn 生命周期**：`turn_start`, `turn_end`
- **Message 生命周期**：`message_start`, `message_update` (流式更新), `message_end`
- **Tool 执行生命周期**：`tool_execution_start`, `tool_execution_update`, `tool_execution_end`

### 事件分发链路

1. **底层发射**：`agentLoop` 通过 `AgentEventSink` 发射事件至流对象 [`EventStream`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/agent/src/agent-loop.ts#L37)。
2. **Session 转发**：[`AgentSession`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/coding-agent/src/core/agent-session.ts#L395) 订阅 Agent 事件并通过 `_handleAgentEvent` ([`agent-session.ts:609`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/coding-agent/src/core/agent-session.ts#L609)) 转发为上层 `SessionEvent`。
3. **前端/消费端订阅**：
   - **TUI 界面**：[`interactive-mode.ts:3060`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/coding-agent/src/modes/interactive/interactive-mode.ts#L3060) 通过 `session.subscribe()` 刷新终端组件。
   - **RPC 服务**：[`rpc-mode.ts:355`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/coding-agent/src/modes/rpc/rpc-mode.ts#L355) 订阅并透传事件给外部客户端。
   - **CLI 模式**：[`print-mode.ts:108`](file:///Users/hzhang/di-code/agent-harness/pi-mono/packages/coding-agent/src/modes/print-mode.ts#L108) 订阅并输出格式化日志到标准输出。

# Pi 架构概览

AgentSession、Agent loop、compaction、工具执行之间的关系。

```text
┌───────────────────────── 接口层（都走同一个 AgentSession）─────────────────────────┐
│  TUI (interactive)   print / JSON mode   RPC mode (stdin/stdout JSONL)   SDK      │
│                                                    createAgentSession() ────┐    │
└─────────────────────────────────────────────────────────────────────────────│────┘
                                                                              ▼
┌──────────────────────────── 编排层：core/agent-session.ts ─────────────────────────┐
│  AgentSession                                                                     │
│  • prompt()/steer()/followUp() 队列    • compact() → core/compaction              │
│  • abort()/waitForIdle()               • 工具注册表 (active/excluded tools)        │
│  • setModel()/setThinkingLevel()       • bindExtensions() → extensions/           │
│  • navigateTree() 会话树分支            • retry / cacheWarmer                      │
└───────┬───────────────────────────────┬───────────────────────────────┬───────────┘
        │ 持有并订阅事件                  │ 写入/读取                      │ 请求构建
        ▼                               ▼                               ▼
┌────────────────── packages/agent ───────────────┐   ┌──── core/session-manager ────┐
│  Agent + agent-loop.ts（真正的循环）              │   │  SessionManager              │
│  build request → stream → execute tools →       │   │  JSONL 会话树（entry+parent）  │
│  record results → 下一 turn / 结束 run           │   │  active branch → 历史         │
│         │                    │                   │   └──────────────────────────────┘
│         ▼                    ▼                   │
│  harness（工具执行）    ModelRuntime/provider     │   ┌──── core/compaction ────────┐
│  core/tools：bash/edit/read/…  模型流式响应        │   │  压缩逻辑 → 摘要 entry 写回树  │
└─────────────────────────────────────────────────┘   └──────────────────────────────┘
```

关键关系：

- 接口层（TUI / print / RPC / SDK）都通过 `createAgentSession()` 走同一个 `AgentSession`。
- `AgentSession`（`core/agent-session.ts`）是编排层：持有 `Agent` 并订阅其事件，管理 prompt/steering 队列、压缩、重试、扩展绑定、会话树导航。
- Agent loop 本体在 `packages/agent/src/agent-loop.ts`：build request → stream → execute tools → record results，循环直到没有新的模型请求。
- 工具定义在 `core/tools/`，由 loop 通过 `packages/agent` 的 harness 执行。
- `SessionManager` 负责 JSONL 会话树持久化，active branch 供 loop 组装请求历史。
- compact 由 `AgentSession` 触发，逻辑在 `core/compaction`，结果作为摘要 entry 写回会话树。

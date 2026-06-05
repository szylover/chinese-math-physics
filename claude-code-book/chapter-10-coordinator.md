# 第十章：多智能体协调 (Coordinator Mode)

## 10.1 概述

协调器模式（Coordinator Mode）是 Claude Code 最先进的功能之一。它将 Claude 从单一代理转变为一个**代理编排器**，能够生成多个工作代理（Worker）并行处理任务。

## 10.2 架构设计

```
┌─────────────────────────────────────┐
│       协调器 (Coordinator)           │
│   ┌─────────────────────────────┐   │
│   │ 角色：任务分解、分派、综合    │   │
│   │ 工具：AgentTool,             │   │
│   │       SendMessageTool,       │   │
│   │       TaskStopTool           │   │
│   └──────────┬──────────────────┘   │
│              │                       │
│   ┌──────────┴──────────────────┐   │
│   │                              │   │
│   ▼              ▼               ▼   │
│ ┌──────┐   ┌──────┐    ┌──────┐    │
│ │Worker│   │Worker│    │Worker│    │
│ │  A   │   │  B   │    │  C   │    │
│ └──────┘   └──────┘    └──────┘    │
│  Bash        FileEdit     Grep      │
│  FileRead    FileWrite    LSP       │
│  GrepTool    GlobTool     ...       │
└─────────────────────────────────────┘
```

## 10.3 协调器模式的启用

```typescript
// coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

需要两个条件同时满足：
1. `COORDINATOR_MODE` Feature Flag 启用
2. `CLAUDE_CODE_COORDINATOR_MODE` 环境变量为 truthy

## 10.4 协调器的系统提示

协调器有一个专用的系统提示（约 400 行），定义了其角色和行为规范：

```typescript
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates 
  software engineering tasks across multiple workers.

## 1. Your Role
You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible

## 2. Your Tools
- **AgentTool** - Spawn a new worker
- **SendMessageTool** - Continue an existing worker
- **TaskStopTool** - Stop a running worker

## 4. Task Workflow
| Phase          | Who       | Purpose                     |
|----------------|-----------|------------------------------|
| Research        | Workers   | Investigate codebase         |
| Synthesis       | **You**   | Craft implementation specs   |
| Implementation  | Workers   | Make targeted changes        |
| Verification    | Workers   | Test changes work            |
...`
}
```

### 核心原则

1. **永远综合，不要委派理解**："Never write 'based on your findings'. You never hand off understanding to another worker."
2. **并行是超能力**："Parallelism is your superpower. Launch independent workers concurrently."
3. **写操作串行**："Read-only tasks — parallel freely. Write-heavy tasks — one at a time per set of files."
4. **自包含提示**："Workers can't see your conversation. Every prompt must be self-contained."

## 10.5 Worker 工具可用性

```typescript
export function getCoordinatorUserContext(
  mcpClients: ReadonlyArray<{ name: string }>,
  scratchpadDir?: string,
): { [k: string]: string } {
  if (!isCoordinatorMode()) return {}

  const workerTools = isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)
    ? [BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
        .sort().join(', ')
    : Array.from(ASYNC_AGENT_ALLOWED_TOOLS)
        .filter(name => !INTERNAL_WORKER_TOOLS.has(name))
        .sort().join(', ')

  let content = `Workers have access to these tools: ${workerTools}`

  // MCP 工具
  if (mcpClients.length > 0) {
    const serverNames = mcpClients.map(c => c.name).join(', ')
    content += `\nWorkers also have access to MCP tools from: ${serverNames}`
  }

  // Scratchpad 目录
  if (scratchpadDir && isScratchpadGateEnabled()) {
    content += `\nScratchpad directory: ${scratchpadDir}`
    content += `\nWorkers can read and write here without permission prompts.`
  }

  return { workerToolsContext: content }
}
```

### 内部工具隔离

```typescript
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,      // Worker 不能创建团队
  TEAM_DELETE_TOOL_NAME,      // Worker 不能删除团队
  SEND_MESSAGE_TOOL_NAME,     // Worker 不能发消息给其他 Worker
  SYNTHETIC_OUTPUT_TOOL_NAME, // 内部使用
])
```

Worker 被阻止使用协调相关的工具，避免层级混乱。

## 10.6 Worker 通知机制

Worker 完成时，结果以 XML 格式作为用户消息传递给协调器：

```xml
<task-notification>
  <task-id>{agentId}</task-id>
  <status>completed|failed|killed</status>
  <summary>{human-readable status summary}</summary>
  <result>{agent's final text response}</result>
  <usage>
    <total_tokens>N</total_tokens>
    <tool_uses>N</tool_uses>
    <duration_ms>N</duration_ms>
  </usage>
</task-notification>
```

## 10.7 会话模式切换

```typescript
export function matchSessionMode(
  sessionMode: 'coordinator' | 'normal' | undefined,
): string | undefined {
  if (!sessionMode) return undefined

  const currentIsCoordinator = isCoordinatorMode()
  const sessionIsCoordinator = sessionMode === 'coordinator'

  if (currentIsCoordinator === sessionIsCoordinator) return undefined

  // 动态切换环境变量
  if (sessionIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
  } else {
    delete process.env.CLAUDE_CODE_COORDINATOR_MODE
  }

  return sessionIsCoordinator
    ? 'Entered coordinator mode to match resumed session.'
    : 'Exited coordinator mode to match resumed session.'
}
```

恢复会话时自动匹配保存时的协调器模式。

## 10.8 团队代理 (Agent Swarms)

```typescript
// tools.ts
...(isAgentSwarmsEnabled()
  ? [getTeamCreateTool(), getTeamDeleteTool()]
  : []),
```

团队代理是协调器模式的扩展——允许创建持久化的代理团队，成员之间可以通过 `SendMessageTool` 通信。

## 10.9 设计启示

### 提示即架构

协调器的行为完全由其系统提示定义。这意味着：
- 修改协调策略不需要改代码
- 可以 A/B 测试不同的协调策略
- LLM 的能力提升直接改善协调质量

### 关注点分离

协调器只做高层决策（分解、分派、综合），不直接操作文件系统。Worker 只执行具体任务，不做全局决策。

---

**下一章**：[MCP 协议与外部集成](./chapter-11-mcp.md)

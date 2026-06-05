# 第十一章：MCP 协议与外部集成

## 11.1 什么是 MCP

MCP（Model Context Protocol，模型上下文协议）是 Anthropic 推出的开放标准，用于在 LLM 应用和外部工具/数据源之间建立标准化的连接。Claude Code 是 MCP 的深度实践者。

## 11.2 MCP 在 Claude Code 中的集成

```
Claude Code
    │
    ├── 内置工具 (BashTool, FileReadTool, ...)
    │
    └── MCP 工具
         │
         ├── MCP Server A (如 GitHub)
         │    ├── mcp__github__create_issue
         │    ├── mcp__github__search_code
         │    └── mcp__github__get_file_contents
         │
         ├── MCP Server B (如数据库)
         │    ├── mcp__db__query
         │    └── mcp__db__schema
         │
         └── MCP Server C (自定义)
              └── mcp__custom__my_tool
```

## 11.3 MCP 服务层

```
src/services/mcp/
├── client.ts          # MCP 客户端管理
├── config.ts          # 配置解析
├── types.ts           # 类型定义
├── claudeai.ts        # Claude AI MCP 配置
├── officialRegistry.ts # 官方 MCP 注册表
├── utils.ts           # 工具函数
└── xaaIdpLogin.ts     # XAA IdP 登录
```

### 11.3.1 MCP 服务器连接

```typescript
type MCPServerConnection = {
  name: string
  // 连接状态、工具列表、资源等
}
```

### 11.3.2 MCP 配置

```typescript
import {
  areMcpConfigsAllowedWithEnterpriseMcpConfig,
  dedupClaudeAiMcpServers,
  doesEnterpriseMcpConfigExist,
  filterMcpServersByPolicy,
  getClaudeCodeMcpConfigs,
  getMcpServerSignature,
  parseMcpConfig,
  parseMcpConfigFromFilePath,
} from 'src/services/mcp/config.js'
```

MCP 配置支持多种来源：
- 项目级别配置（`.claude/mcp.json`）
- 用户级别配置
- 企业级别配置
- Claude AI 远程配置

### 11.3.3 企业 MCP 管控

```typescript
doesEnterpriseMcpConfigExist()       // 是否存在企业配置
filterMcpServersByPolicy()           // 按策略过滤服务器
areMcpConfigsAllowedWithEnterpriseMcpConfig()  // 企业策略允许性检查
```

## 11.4 MCP 工具与内置工具的融合

```typescript
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)

  // 内置工具优先，按名称去重
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

**冲突解决**：当 MCP 工具与内置工具同名时，内置工具优先。

## 11.5 MCP 资源

除了工具，MCP 还支持"资源"（Resources）——代表可读取的数据源。

```typescript
import { ListMcpResourcesTool } from './tools/ListMcpResourcesTool/ListMcpResourcesTool.js'
import { ReadMcpResourceTool } from './tools/ReadMcpResourceTool/ReadMcpResourceTool.js'
```

| 工具 | 功能 |
|------|------|
| `ListMcpResourcesTool` | 列出所有可用的 MCP 资源 |
| `ReadMcpResourceTool` | 读取特定 MCP 资源的内容 |

## 11.6 MCP 命令

通过 `/mcp` 斜杠命令管理 MCP 服务器：

```typescript
import mcp from './commands/mcp/index.js'
import { registerMcpAddCommand } from 'src/commands/mcp/addCommand.js'
import { registerMcpXaaIdpCommand } from 'src/commands/mcp/xaaIdpCommand.js'
```

## 11.7 Elicitation 处理

MCP 工具可以通过 `-32042` 错误码触发 URL 引出（Elicitation）：

```typescript
handleElicitation?: (
  serverName: string,
  params: ElicitRequestURLParams,
  signal: AbortSignal,
) => Promise<ElicitResult>
```

在 SDK 模式下，通过 `structuredIO.handleElicitation` 处理。在 REPL 模式下，使用基于队列的 UI 路径。

## 11.8 服务器排除

```typescript
import {
  excludeCommandsByServer,
  excludeResourcesByServer,
} from 'src/services/mcp/utils.js'
```

可以按服务器排除特定的命令和资源。

## 11.9 设计启示

### 开放协议的价值

MCP 将"工具"的概念从硬编码扩展为可插拔。任何实现 MCP 协议的服务器都可以被 Claude Code 使用，无需修改 Claude Code 本身。

### 权限统一

MCP 工具与内置工具共享同一套权限系统。`filterToolsByDenyRules` 对两者一视同仁，确保安全策略的一致性。

---

**下一章**：[终端 UI：React + Ink](./chapter-12-terminal-ui.md)

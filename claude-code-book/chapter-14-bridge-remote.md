# 第十四章：IDE 桥接与远程会话

## 14.1 桥接系统概述

Claude Code 通过桥接系统（Bridge System）实现与 IDE 的双向通信。支持 VS Code 和 JetBrains 等主流 IDE。

```
┌─────────────┐         ┌──────────────────┐
│   VS Code   │◄──JWT──►│   Claude Code    │
│  Extension  │  Bridge │   CLI Process    │
└─────────────┘         └──────────────────┘
       │                        │
       │    JSON-RPC 消息        │
       │◄──────────────────────►│
       │                        │
```

## 14.2 桥接架构

```
src/bridge/
├── bridgeMain.ts                  # 桥主循环
├── bridgeMessaging.ts             # 消息协议
├── bridgePermissionCallbacks.ts   # 权限回调
├── replBridge.ts                  # REPL 会话桥
├── jwtUtils.ts                    # JWT 认证
└── sessionRunner.ts               # 会话执行管理
```

### 14.2.1 认证

桥接使用 **JWT（JSON Web Token）** 进行身份验证，确保只有授权的 IDE 扩展可以连接到 CLI 进程。

### 14.2.2 消息协议

基于 JSON-RPC 的双向消息协议，支持：
- IDE → CLI：用户输入、命令请求
- CLI → IDE：流式响应、工具调用通知、权限请求

### 14.2.3 权限回调

```typescript
// bridgePermissionCallbacks.ts
// IDE 接管权限对话框的显示
// CLI 发送权限请求 → IDE 显示对话框 → 用户决策 → 回传给 CLI
```

## 14.3 远程会话

```
src/remote/
├── RemoteSessionManager.ts    # 远程会话管理器
└── ...
```

Claude Code 支持远程会话——在一台机器上运行 CLI，从另一台机器（或移动设备）访问：

```typescript
import { createRemoteSessionConfig } from './remote/RemoteSessionManager.js'
import { getRemoteSessionUrl } from './constants/product.js'
```

### 14.3.1 Teleport

```typescript
import { teleportToRemoteWithErrorHandling } from './utils/teleport.js'
```

"Teleport" 功能允许将本地会话传送到远程环境：
- 验证 Git 状态
- 处理分支切换
- 恢复远程会话
- 处理仓库不匹配

### 14.3.2 跨平台传递

```typescript
import desktop from './commands/desktop/index.js'    // → 桌面应用
import mobile from './commands/mobile/index.js'      // → 移动应用
```

## 14.4 服务器模式

```
src/server/
├── createDirectConnectSession.ts
└── ...
```

```typescript
import { createDirectConnectSession, DirectConnectError } from './server/createDirectConnectSession.js'
```

Claude Code 可以作为服务器运行，接受来自其他客户端的直接连接。

## 14.5 LSP 集成

```typescript
import { initializeLspServerManager } from './services/lsp/manager.js'
```

Claude Code 集成了 **LSP（Language Server Protocol）**，提供代码智能功能：
- 符号跳转
- 类型信息
- 代码补全建议
- 诊断信息

LSP 通过 `LSPTool` 暴露给 LLM，使其能够利用语言服务器的能力进行更精确的代码分析。

## 14.6 设计启示

### CLI 优先，IDE 其次

Claude Code 的核心是 CLI 工具，IDE 集成通过桥接层实现。这种架构确保了：
- CLI 可以独立使用
- IDE 扩展是可选的增强
- 不同 IDE 共享相同的后端逻辑

---

**下一章**：[成本追踪与遥测](./chapter-15-cost-telemetry.md)

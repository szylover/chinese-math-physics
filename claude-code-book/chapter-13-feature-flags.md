# 第十三章：Feature Flag 与条件编译

## 13.1 双层 Feature Flag 系统

Claude Code 使用两层 Feature Flag 系统，分别在编译时和运行时控制功能：

| 层次 | 机制 | 作用时机 | 管理方式 |
|------|------|----------|----------|
| **编译时** | `bun:bundle` | Build/Run | 代码中静态定义 |
| **运行时** | GrowthBook | 应用启动后 | 远程控制面板 |

## 13.2 编译时 Feature Flag (`bun:bundle`)

### 13.2.1 工作原理

```typescript
import { feature } from 'bun:bundle'

// 编译时：如果 VOICE_MODE 未启用，整个分支被移除
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Bun 的 bundler 在编译时将 `feature()` 调用替换为 `true` 或 `false` 常量，然后通过死代码消除（Tree Shaking）移除不可达的代码分支。

### 13.2.2 开发环境 Polyfill

```typescript
// plugins/bunBundleDev.ts
plugin({
  name: 'bun-bundle-dev',
  setup(build) {
    build.module('bun:bundle', () => ({
      exports: {
        feature(flag: string): boolean {
          return enabledFlags.has(flag)
        },
      },
      loader: 'object',
    }))
  },
})
```

开发时通过环境变量控制：

```bash
FEATURE_FLAGS=KAIROS,VOICE_MODE bun run src/main.tsx
```

### 13.2.3 已知 Feature Flag 列表

从源码中提取的所有编译时 Feature Flag：

| Flag | 功能 | 影响范围 |
|------|------|----------|
| `PROACTIVE` | 主动模式 | SleepTool, proactive 命令 |
| `KAIROS` | 助手模式 | assistant 模块, SendUserFileTool, PushNotificationTool |
| `KAIROS_BRIEF` | 简要模式 | brief 命令 |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 | PushNotificationTool |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub Webhook | SubscribePRTool |
| `BRIDGE_MODE` | IDE 桥接 | bridge 命令, remoteControlServer |
| `DAEMON` | 守护进程 | remoteControlServer 命令 |
| `VOICE_MODE` | 语音输入 | voice 命令 |
| `AGENT_TRIGGERS` | 代理触发器 | CronCreate/Delete/ListTool |
| `AGENT_TRIGGERS_REMOTE` | 远程触发 | RemoteTriggerTool |
| `MONITOR_TOOL` | 监控工具 | MonitorTool |
| `COORDINATOR_MODE` | 协调器模式 | coordinatorMode 模块 |
| `HISTORY_SNIP` | 历史裁剪 | SnipTool, snipCompact |
| `CONTEXT_COLLAPSE` | 上下文折叠 | CtxInspectTool, contextCollapse |
| `CACHED_MICROCOMPACT` | 缓存微压缩 | microcompact 缓存编辑 |
| `REACTIVE_COMPACT` | 响应式压缩 | reactiveCompact |
| `TOKEN_BUDGET` | Token 预算 | tokenBudget 追踪 |
| `OVERFLOW_TEST_TOOL` | 溢出测试 | OverflowTestTool |
| `TERMINAL_PANEL` | 终端面板 | TerminalCaptureTool |
| `WEB_BROWSER_TOOL` | 浏览器工具 | WebBrowserTool |
| `TRANSCRIPT_CLASSIFIER` | 对话分类器 | autoModeState |
| `BREAK_CACHE_COMMAND` | 缓存破坏 | systemPromptInjection |
| `UDS_INBOX` | UDS 收件箱 | ListPeersTool |
| `WORKFLOW_SCRIPTS` | 工作流脚本 | WorkflowTool |
| `BG_SESSIONS` | 后台会话 | taskSummary |
| `EXPERIMENTAL_SKILL_SEARCH` | 技能搜索 | skillSearch |
| `TEMPLATES` | 模板 | job classifier |
| `CCR_REMOTE_SETUP` | 远程设置 | remote-setup 命令 |

## 13.3 运行时 Feature Flag (GrowthBook)

```typescript
import {
  initializeGrowthBook,
  refreshGrowthBookAfterAuthChange,
  getFeatureValue_CACHED_MAY_BE_STALE,
  checkStatsigFeatureGate_CACHED_MAY_BE_STALE,
} from './services/analytics/growthbook.js'
```

### 13.3.1 缓存策略

函数名 `_CACHED_MAY_BE_STALE` 暗示了一个重要设计：
- GrowthBook 值在启动时获取并缓存
- 缓存值可能过期但仍被使用
- 这避免了每次检查时的网络请求
- 认证变更后会刷新

### 13.3.2 使用场景

```typescript
// Scratchpad 功能门
function isScratchpadGateEnabled(): boolean {
  return checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_scratch')
}
```

运行时 Flag 用于：
- A/B 测试新功能
- 灰度发布
- 远程紧急开关
- 用户分群

## 13.4 环境变量控制

除了两层 Feature Flag，Claude Code 还大量使用环境变量：

```typescript
// 用户类型
process.env.USER_TYPE === 'ant'          // 内部用户

// 功能开关
process.env.CLAUDE_CODE_COORDINATOR_MODE // 协调器模式
process.env.CLAUDE_CODE_SIMPLE           // 简单模式
process.env.CLAUDE_CODE_REMOTE           // 远程模式
process.env.CLAUDE_CODE_VERIFY_PLAN      // 验证计划
process.env.ENABLE_LSP_TOOL             // LSP 工具
process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS // 禁用 CLAUDE.md

// 测试
process.env.NODE_ENV === 'test'          // 测试模式
```

## 13.5 构建时常量

```toml
# bunfig.toml
[define]
"MACRO.VERSION" = '"1.0.0-dev"'
"MACRO.BUILD_TIME" = '""'
"MACRO.PACKAGE_URL" = '"@anthropic-ai/claude-code"'
```

`MACRO.*` 常量在构建时由 Bun 的 `--define` 注入，用于嵌入版本号、构建时间等信息。

## 13.6 内部 vs 外部构建

```typescript
// 代码中有许多这样的条件
if ("external" !== 'ant') {
  // 外部构建的行为
}
```

字符串 `"external"` 在内部构建中被替换为 `"ant"`。这通过 Bun 的 define 系统实现：
- **外部构建**：`"external" !== 'ant'` → `true` → 执行外部逻辑
- **内部构建**：`"ant" !== 'ant'` → `false` → 跳过（执行内部逻辑）

## 13.7 设计启示

### 编译时 vs 运行时

两层 Flag 服务于不同目的：
- **编译时**：减小 bundle 大小，消除未使用代码
- **运行时**：支持远程控制和灰度发布

### 命名规范

`_CACHED_MAY_BE_STALE` 后缀是优秀的 API 设计——函数名本身就传达了使用注意事项。

---

**下一章**：[IDE 桥接与远程会话](./chapter-14-bridge-remote.md)

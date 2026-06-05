# 第十七章：设计模式与工程实践总结

## 17.1 架构级模式

### 17.1.1 并行预取 (Parallel Prefetch)

Claude Code 最具特色的架构模式是**利用模块加载时间进行并行预取**：

```typescript
// 在 import 语句之前启动异步 I/O
profileCheckpoint('main_tsx_entry');
startMdmRawRead();        // 启动 MDM 子进程
startKeychainPrefetch();  // 启动钥匙串读取

// 后续的 import 语句需要 ~135ms
// 此时 MDM 和 Keychain 读取在后台进行
import chalk from 'chalk';
import React from 'react';
// ...
```

**原则**：识别"免费"的等待时间，将 I/O 操作前移到该窗口。

### 17.1.2 分层延迟初始化

```
阶段 1：模块求值前 → 启动异步 I/O
阶段 2：模块求值中 → 自动完成（JS 引擎）
阶段 3：首次渲染前 → 必要的同步初始化
阶段 4：首次渲染后 → 延迟预取（用户不感知）
```

### 17.1.3 AsyncGenerator 驱动的流式处理

```typescript
async function* query(params): AsyncGenerator<StreamEvent | Message> {
  while (true) {
    yield { type: 'stream_request_start' }
    // ... API 调用 & 工具执行 ...
    yield assistantMessage
    // ... 判断是否继续 ...
  }
}
```

使用 AsyncGenerator 实现流式数据管道，允许调用方实时消费每个事件。

## 17.2 代码组织模式

### 17.2.1 条件导入与死代码消除

```typescript
const SleepTool = feature('PROACTIVE')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

**规律**：所有可选功能都使用三元表达式 + `require()` 而非顶层 `import`。这使得 Bun 的 bundler 可以在编译时完全移除未使用的模块。

### 17.2.2 懒加载打破循环依赖

```typescript
// 不使用顶层 import（会创建循环依赖）
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```

**规律**：将导入包装在函数中，只在实际调用时执行。搭配 TypeScript 的 `typeof import(...)` 保留类型安全。

### 17.2.3 Memoize 模式

```typescript
export const getSystemContext = memoize(async () => {
  // 只执行一次的重量级操作
  const gitStatus = await getGitStatus()
  return { gitStatus }
})
```

对于整个会话期间不变的数据（Git 状态、CLAUDE.md 内容），使用 `memoize` 缓存结果。

### 17.2.4 清除缓存的信号

```typescript
export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  getUserContext.cache.clear?.()    // 清除 memoize 缓存
  getSystemContext.cache.clear?.()
}
```

当底层数据变化时，手动清除 memoize 缓存。

## 17.3 类型系统模式

### 17.3.1 DeepImmutable

```typescript
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode
  alwaysAllowRules: ToolPermissionRulesBySource
  // ...
}>
```

使用 `DeepImmutable` 确保权限上下文在传递过程中不被意外修改。

### 17.3.2 自文档化类型名

```typescript
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = string
```

类型名本身就是一条检查清单。每次使用时，开发者必须 pause 并确认。

### 17.3.3 品牌类型 (Branded Types)

```typescript
import { asSessionId } from './types/ids.js'
import type { AgentId } from './types/ids.js'
```

使用品牌类型防止 ID 类型之间的混用。

## 17.4 错误处理模式

### 17.4.1 恢复循环

```typescript
const MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3

// 自动恢复，最多重试 3 次
// 恢复期间扣留错误消息，避免影响 SDK 调用方
```

### 17.4.2 优雅降级

```typescript
const truncatedStatus = status.length > MAX_STATUS_CHARS
  ? status.substring(0, MAX_STATUS_CHARS) +
    '\n... (truncated. Use BashTool for full info)'
  : status
```

超出限制时截断并提供获取完整信息的替代方案。

### 17.4.3 分类重试

```typescript
import { categorizeRetryableAPIError } from './services/api/errors.js'
import { FallbackTriggeredError } from './services/api/withRetry.js'
```

API 错误被分类为可重试和不可重试，支持自动重试和模型 fallback。

## 17.5 安全模式

### 17.5.1 纵深防御

```
1. 工具列表过滤 → 不暴露被拒工具
2. 调用时权限检查 → 每次操作验证
3. 沙箱执行 → 受限环境
4. 审计追踪 → 记录所有决策
```

### 17.5.2 隐私优先

```typescript
logForDiagnosticsNoPII(...)     // 日志不含 PII
_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS  // 类型级别隐私检查
```

### 17.5.3 信任边界

```typescript
if (!hasTrust) {
  // 不执行 Git 命令（可能通过 hooks 执行任意代码）
}
```

## 17.6 性能模式

### 17.6.1 并行化

```typescript
// 5 个 Git 命令并行
const [branch, mainBranch, status, log, userName] = await Promise.all([...])

// 启动时并行预取
startMdmRawRead()
startKeychainPrefetch()
void getSystemContext()
```

### 17.6.2 缓存感知排序

```typescript
// 内置工具排在前面，保持稳定排序
// 使 Prompt Cache 跨请求复用
return uniqBy(
  [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
  'name',
)
```

### 17.6.3 延迟加载重量级模块

```
OpenTelemetry (~400KB) → 动态 import()
gRPC (~700KB) → 动态 import()
```

## 17.7 项目规模管理

一个 512,000+ 行的 TypeScript 项目面临的挑战和应对：

| 挑战 | Claude Code 的应对 |
|------|-------------------|
| 循环依赖 | 懒加载 `require()` 函数 |
| 启动速度 | 并行预取 + 延迟初始化 |
| Bundle 大小 | Feature Flag + 死代码消除 |
| 类型安全 | 严格模式 + 品牌类型 |
| 配置演进 | 版本化迁移系统 |
| 隐私合规 | 类型级别隐私标记 |

## 17.8 总结

Claude Code 的源码揭示了构建工业级 AI 编程助手的复杂度。它不仅仅是一个"调用 API 的 CLI 工具"，而是一个包含以下系统的完整平台：

1. **查询引擎**：多层压缩管线、自动恢复、Token 预算
2. **工具系统**：40+ 工具、权限模型、缓存感知
3. **命令系统**：50+ 命令、生命周期管理
4. **协调器**：多代理编排、并行执行
5. **UI 框架**：React + Ink 的 140+ 组件
6. **安全层**：多模式权限、沙箱、审计
7. **扩展系统**：插件、技能、MCP
8. **遥测**：OpenTelemetry、成本追踪、性能分析

这些系统的设计模式——并行预取、缓存感知、分层压缩、纵深防御——不仅适用于 AI 编程助手，也可以推广到任何需要将 LLM 深度集成到软件工程流程中的应用。

---

**全书完**

> 本书基于 `szylover/claude-code` 仓库中的公开源码撰写。
> 所有原始源码的知识产权归 [Anthropic](https://www.anthropic.com) 所有。

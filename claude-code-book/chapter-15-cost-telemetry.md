# 第十五章：成本追踪与遥测

## 15.1 成本追踪系统

Claude Code 实现了精细的成本追踪系统，让用户实时了解 API 使用费用。

### 15.1.1 核心数据结构

```typescript
type ModelUsage = {
  inputTokens: number
  outputTokens: number
  cacheReadInputTokens: number
  cacheCreationInputTokens: number
  webSearchRequests: number
  costUSD: number
  contextWindow: number
  maxOutputTokens: number
}
```

### 15.1.2 成本计算

```typescript
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number {
  // 1. 更新模型使用量统计
  const modelUsage = addToTotalModelUsage(cost, usage, model)
  addToTotalCostState(cost, modelUsage, model)

  // 2. OpenTelemetry 计数器
  getCostCounter()?.add(cost, attrs)
  getTokenCounter()?.add(usage.input_tokens, { ...attrs, type: 'input' })
  getTokenCounter()?.add(usage.output_tokens, { ...attrs, type: 'output' })
  getTokenCounter()?.add(usage.cache_read_input_tokens ?? 0, {
    ...attrs, type: 'cacheRead'
  })

  // 3. 递归处理 Advisor 使用量
  for (const advisorUsage of getAdvisorUsage(usage)) {
    const advisorCost = calculateUSDCost(advisorUsage.model, advisorUsage)
    totalCost += addToTotalSessionCost(advisorCost, advisorUsage, advisorUsage.model)
  }
  
  return totalCost
}
```

### 15.1.3 Fast Mode 追踪

```typescript
const attrs = isFastModeEnabled() && usage.speed === 'fast'
  ? { model, speed: 'fast' }
  : { model }
```

Fast Mode（快速模式）下的使用量单独标记。

### 15.1.4 格式化输出

```typescript
export function formatTotalCost(): string {
  return chalk.dim(
    `Total cost:            ${costDisplay}\n` +
    `Total duration (API):  ${formatDuration(getTotalAPIDuration())}\n` +
    `Total duration (wall): ${formatDuration(getTotalDuration())}\n` +
    `Total code changes:    ${getTotalLinesAdded()} lines added, ` +
    `${getTotalLinesRemoved()} lines removed\n` +
    `${modelUsageDisplay}`
  )
}
```

输出示例：
```
Total cost:            $0.1234
Total duration (API):  12.3s
Total duration (wall): 45.6s
Total code changes:    42 lines added, 13 lines removed
Usage by model:
        claude-sonnet-4-20260514:  15,234 input, 3,456 output, 12,000 cache read, 2,000 cache write ($0.0234)
```

## 15.2 会话成本持久化

```typescript
export function saveCurrentSessionCosts(fpsMetrics?: FpsMetrics): void {
  saveCurrentProjectConfig(current => ({
    ...current,
    lastCost: getTotalCostUSD(),
    lastAPIDuration: getTotalAPIDuration(),
    lastTotalInputTokens: getTotalInputTokens(),
    lastTotalOutputTokens: getTotalOutputTokens(),
    lastSessionId: getSessionId(),
    lastFpsAverage: fpsMetrics?.averageFps,
    lastFpsLow1Pct: fpsMetrics?.low1PctFps,
    lastModelUsage: Object.fromEntries(
      Object.entries(getModelUsage()).map(([model, usage]) => [model, {
        inputTokens: usage.inputTokens,
        outputTokens: usage.outputTokens,
        cacheReadInputTokens: usage.cacheReadInputTokens,
        cacheCreationInputTokens: usage.cacheCreationInputTokens,
        webSearchRequests: usage.webSearchRequests,
        costUSD: usage.costUSD,
      }])
    ),
  }))
}
```

成本在会话切换前保存，支持通过 `/resume` 恢复成本数据。

## 15.3 OpenTelemetry 集成

```json
{
  "@opentelemetry/api": "^1.9.1",
  "@opentelemetry/api-logs": "^0.214.0",
  "@opentelemetry/core": "^2.6.1",
  "@opentelemetry/resources": "^2.6.1",
  "@opentelemetry/sdk-logs": "^0.214.0",
  "@opentelemetry/sdk-metrics": "^2.6.1",
  "@opentelemetry/sdk-trace-base": "^2.6.1",
  "@opentelemetry/semantic-conventions": "^1.40.0"
}
```

Claude Code 集成了完整的 OpenTelemetry 全家桶：
- **Traces**：请求链路追踪
- **Metrics**：Token 计数器、成本计数器
- **Logs**：结构化日志

### 懒加载优化

```
// OpenTelemetry (~400KB) 和 gRPC (~700KB) 通过动态 import() 延迟加载
// 只在实际需要发送遥测时才加载
```

## 15.4 分析事件

所有遥测事件使用 `tengu_` 前缀（Claude Code 内部代号"天狗"）：

```typescript
logEvent('tengu_startup_telemetry', {
  is_git: isGit,
  worktree_count: worktreeCount,
  gh_auth_status: ghAuthStatus,
  sandbox_enabled: SandboxManager.isSandboxingEnabled(),
  // ...
})

logEvent('tengu_managed_settings_loaded', {
  keyCount: allKeys.length,
  keys: allKeys.join(','),
})

logEvent('tengu_coordinator_mode_switched', {
  to: sessionMode,
})

logEvent('tengu_advisor_tool_token_usage', {
  advisor_model: advisorUsage.model,
  input_tokens: advisorUsage.input_tokens,
  cost_usd_micros: Math.round(advisorCost * 1_000_000),
})
```

### 安全类型标记

```typescript
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = string
```

这个冗长的类型名是一个**代码审查辅助工具**：每次将字符串作为分析元数据传递时，开发者必须显式将其断言为这个类型，确认该值不包含代码或文件路径（隐私保护）。

## 15.5 诊断日志

```typescript
import { logForDiagnosticsNoPII } from './utils/diagLogs.js'

logForDiagnosticsNoPII('info', 'git_status_started')
logForDiagnosticsNoPII('info', 'git_commands_completed', {
  duration_ms: Date.now() - gitCmdsStart,
  status_length: status.length,
})
logForDiagnosticsNoPII('error', 'git_status_failed', {
  duration_ms: Date.now() - startTime,
})
```

`NoPII`（No Personally Identifiable Information）后缀表明这些日志不包含个人可识别信息。

## 15.6 启动性能分析

```typescript
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js'

profileCheckpoint('main_tsx_entry')
// ... 模块加载 ...
profileCheckpoint('main_tsx_imports_loaded')
```

启动分析器在关键点放置检查点，测量每个阶段的耗时。

## 15.7 设计启示

### 隐私优先的遥测

`AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 和 `logForDiagnosticsNoPII` 展示了如何在类型系统层面强制执行隐私策略。

### 多维成本追踪

按模型、按 Token 类型（输入/输出/缓存读/缓存写）、按功能（主循环/Advisor）的多维追踪，使得成本分析可以精确到每个维度。

---

**下一章**：[插件与技能系统](./chapter-16-plugins-skills.md)

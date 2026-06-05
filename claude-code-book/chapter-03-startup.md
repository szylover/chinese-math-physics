# 第三章：启动流程与初始化

## 3.1 入口文件分析

Claude Code 的入口文件是 `src/main.tsx`。这是整个应用中最大的文件之一，包含了从 CLI 参数解析到 REPL 启动的完整初始化链路。

### 3.1.1 并行预取：启动优化的精髓

`main.tsx` 的前几行代码展现了 Claude Code 对启动性能的极致追求：

```typescript
// 这些副作用必须在所有其他 import 之前运行：
// 1. profileCheckpoint 在重量级模块求值开始前标记入口
// 2. startMdmRawRead 启动 MDM 子进程 (plutil/reg query)，使其与
//    后续 ~135ms 的 import 并行运行
// 3. startKeychainPrefetch 并行启动两个 macOS 钥匙串读取
//    (OAuth + 遗留 API key)

import { profileCheckpoint } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();

import { startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();
```

**关键洞察**：JavaScript 模块的 `import` 语句在执行时会触发模块求值（module evaluation），这个过程需要约 135ms。Claude Code 利用这段"免费"的时间，将 I/O 密集型操作（如读取系统设置和钥匙串）提前启动，使其与模块加载并行执行。

```
时间线：
0ms    ─── profileCheckpoint('main_tsx_entry')
       ├── startMdmRawRead()        ← 启动 MDM 子进程（非阻塞）
       ├── startKeychainPrefetch()  ← 启动钥匙串读取（非阻塞）
       │
       │   ┌──────────────────────────────────────┐
       │   │ 模块求值（~135ms）                     │
       │   │ - import chalk, React, Commander...   │
       │   │ - import services, tools, commands... │
       │   └──────────────────────────────────────┘
       │
       │   ┌──────────────────────────────────────┐
       │   │ MDM 读取 & 钥匙串读取 并行进行         │
       │   │ （在模块求值期间已完成）                 │
       │   └──────────────────────────────────────┘
       │
135ms  ─── profileCheckpoint('main_tsx_imports_loaded')
           ← 此时 MDM 和 Keychain 数据通常已就绪
```

### 3.1.2 反调试保护

在启动阶段还有一个安全措施——反调试检测：

```typescript
// 如果非内部用户使用调试/检查模式运行，直接退出
if ("external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);
}
```

`isBeingDebugged()` 检查三个信号：
1. `process.execArgv` 中的 `--inspect` / `--debug` 标志
2. `NODE_OPTIONS` 环境变量中的检查标志
3. Node.js inspector 模块是否激活

注意 `"external" !== 'ant'` 这个条件：在内部构建中，字符串 `"external"` 会被替换为 `"ant"`，使得内部用户可以自由调试。

### 3.1.3 配置迁移系统

```typescript
const CURRENT_MIGRATION_VERSION = 11;

function runMigrations(): void {
  if (getGlobalConfig().migrationVersion !== CURRENT_MIGRATION_VERSION) {
    migrateAutoUpdatesToSettings();
    migrateBypassPermissionsAcceptedToSettings();
    migrateEnableAllProjectMcpServersToSettings();
    resetProToOpusDefault();
    migrateSonnet1mToSonnet45();
    migrateLegacyOpusToCurrent();
    migrateSonnet45ToSonnet46();
    migrateOpusToOpus1m();
    migrateReplBridgeEnabledToRemoteControlAtStartup();
    // ...
    saveGlobalConfig(prev => ({
      ...prev,
      migrationVersion: CURRENT_MIGRATION_VERSION
    }));
  }
}
```

这个迁移系统的设计特点：
- **版本号控制**：只有当配置版本号不匹配时才执行迁移
- **幂等性**：每个迁移函数自身处理"已迁移"状态
- **模型演进追踪**：可以清晰看到 Anthropic 的模型命名变化历史（Sonnet 1M → Sonnet 4.5 → Sonnet 4.6、Opus → Opus 1M 等）

## 3.2 初始化流程

### 3.2.1 Commander.js 命令注册

`main.tsx` 使用 Commander.js 定义了丰富的 CLI 选项：

```typescript
const program = new CommanderCommand()
  .name('claude')
  .description('Claude Code CLI')
  .option('-p, --print <prompt>', '单次执行模式（非交互式）')
  .option('--model <model>', '指定模型')
  .option('--permission-mode <mode>', '权限模式')
  .option('--tools <preset>', '工具预设')
  .option('--add-dir <path...>', '添加额外工作目录')
  .option('--bare', '精简模式')
  .option('--verbose', '详细输出')
  // ... 更多选项
```

### 3.2.2 两种运行模式

Claude Code 有两种主要运行模式：

**Print 模式 (`-p`)** — 非交互式，单次执行：
```bash
claude -p "请解释这个函数的作用" 
```

**REPL 模式** — 交互式，持续对话：
```bash
claude  # 直接启动交互式会话
```

### 3.2.3 信任对话框

在交互模式下，首次使用时会显示信任对话框：

```typescript
function prefetchSystemContextIfSafe(): void {
  const isNonInteractiveSession = getIsNonInteractiveSession();

  // 非交互模式下，信任是隐式的
  if (isNonInteractiveSession) {
    void getSystemContext();
    return;
  }

  // 交互模式下，只在信任已建立时才预取
  const hasTrust = checkHasTrustDialogAccepted();
  if (hasTrust) {
    void getSystemContext();
  }
  // 否则等待信任建立后再获取
}
```

**安全考量**：`getSystemContext()` 会执行 `git status` 等 Git 命令。Git 命令可以通过 hooks 和配置（如 `core.fsmonitor`、`diff.external`）执行任意代码。因此，在用户确认信任当前目录之前，不会执行任何 Git 命令。

## 3.3 延迟预取

Claude Code 将非关键的预取操作延迟到首次渲染之后：

```typescript
export function startDeferredPrefetches(): void {
  // 如果只是测量启动性能，跳过所有预取
  if (isEnvTruthy(process.env.CLAUDE_CODE_EXIT_AFTER_FIRST_RENDER)) {
    return;
  }
  
  // --bare 模式：跳过所有预取
  if (isBareMode()) {
    return;
  }

  // 这些在首次渲染之后运行，不阻塞初始画面：
  // - initUser：用户信息初始化
  // - getUserContext：CLAUDE.md 读取
  // - tips：使用提示
  // - countFiles：文件计数
  // - modelCapabilities：模型能力查询
  // - change detectors：设置/技能变更检测
}
```

**设计原则**：将用户**感知到的启动时间**与**实际初始化时间**分离。用户看到界面（首次渲染）后，后台继续完成剩余初始化。

## 3.4 GrowthBook 功能开关初始化

```typescript
import { initializeGrowthBook } from './services/analytics/growthbook.js';
```

GrowthBook 是 Claude Code 使用的功能开关（Feature Flag）服务。它在启动时初始化，用于：
- 控制 A/B 测试
- 灰度发布新功能
- 远程配置管理

与 `bun:bundle` 的编译时 Feature Flag 不同，GrowthBook 提供的是**运行时功能开关**，可以在不更新版本的情况下远程开关功能。

## 3.5 会话管理

### 3.5.1 会话 ID

每次启动都会生成一个唯一的会话 ID：

```typescript
const sessionId = getSessionId(); // UUID v4
```

会话 ID 用于：
- 关联遥测数据
- 持久化会话记录
- 恢复（Resume）会话

### 3.5.2 并发会话检测

```typescript
import { countConcurrentSessions, registerSession } from 'src/utils/concurrentSessions.js';
```

Claude Code 支持多个实例同时运行，并通过文件锁机制追踪并发会话数量。

## 3.6 遥测与启动日志

```typescript
async function logStartupTelemetry(): Promise<void> {
  if (isAnalyticsDisabled()) return;
  
  const [isGit, worktreeCount, ghAuthStatus] = await Promise.all([
    getIsGit(),
    getWorktreeCount(),
    getGhAuthStatus()
  ]);
  
  logEvent('tengu_startup_telemetry', {
    is_git: isGit,
    worktree_count: worktreeCount,
    gh_auth_status: ghAuthStatus,
    sandbox_enabled: SandboxManager.isSandboxingEnabled(),
    auto_updater_disabled: isAutoUpdaterDisabled(),
    // ...
  });
}
```

注意内部代号 **"tengu"**（天狗）— 这是 Claude Code 的内部项目名。所有遥测事件都以 `tengu_` 为前缀。

## 3.7 设计启示

### 利用模块求值时间

在 JavaScript/TypeScript 应用中，模块加载是一段"不可避免"的开销。Claude Code 展示了如何将这段时间变废为宝：在模块求值开始前启动异步 I/O 操作，让它们与模块加载并行进行。

### 分层初始化

将初始化分为多个阶段：
1. **最早**：性能关键的异步 I/O（MDM、Keychain）
2. **模块加载期间**：自动完成（JavaScript 引擎的工作）
3. **首次渲染前**：必要的同步初始化
4. **首次渲染后**：延迟预取（不影响用户感知）

### 安全优先

在信任建立之前不执行可能危险的操作（如 Git 命令），体现了安全与性能之间的平衡。

---

**下一章**：[核心查询引擎 QueryEngine](./chapter-04-query-engine.md)

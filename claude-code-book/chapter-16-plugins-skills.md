# 第十六章：插件与技能系统

## 16.1 插件系统

Claude Code 的插件系统允许通过第三方扩展来增强功能。

### 16.1.1 架构

```
src/plugins/
├── bundled/           # 内置插件
│   └── index.ts       # 内置插件初始化
└── ...

src/services/plugins/
├── pluginLoader.ts    # 插件加载器
└── pluginCliCommands.ts # 插件 CLI 命令

src/utils/plugins/
├── installedPluginsManager.ts # 已安装插件管理
├── managedPlugins.ts          # 托管插件
├── pluginDirectories.ts       # 插件目录
├── cacheUtils.ts              # 缓存工具
├── orphanedPluginFilter.ts    # 孤立插件过滤
└── pluginLoader.ts            # 插件加载逻辑
```

### 16.1.2 插件加载

```typescript
import { initBuiltinPlugins } from './plugins/bundled/index.js'
import { loadAllPluginsCacheOnly } from './utils/plugins/pluginLoader.js'

// 启动时初始化内置插件
initBuiltinPlugins()

// 缓存优先加载所有插件
const { enabled, errors } = await loadAllPluginsCacheOnly()
```

### 16.1.3 插件管理命令

```typescript
import {
  VALID_INSTALLABLE_SCOPES,
  VALID_UPDATE_SCOPES,
} from './services/plugins/pluginCliCommands.js'
```

支持安装、更新、删除插件，以及按作用域（全局/项目/用户）管理。

### 16.1.4 版本管理

```typescript
import { initializeVersionedPlugins } from './utils/plugins/installedPluginsManager.js'
import { cleanupOrphanedPluginVersionsInBackground } from './utils/plugins/cacheUtils.js'
```

插件支持版本管理，并在后台清理孤立版本。

### 16.1.5 托管插件

```typescript
import { getManagedPluginNames } from './utils/plugins/managedPlugins.js'
```

"托管插件"是由 Anthropic 或企业管理员配置的插件，用户无法自行修改。

## 16.2 技能系统

技能（Skills）是可复用的工作流定义，类似于"预设的操作序列"。

### 16.2.1 架构

```
src/skills/
├── bundled/           # 内置技能
│   └── index.ts       # 内置技能初始化
└── ...
```

### 16.2.2 内置技能初始化

```typescript
import { initBundledSkills } from './skills/bundled/index.js'
```

### 16.2.3 技能工具

```typescript
import { SkillTool } from './tools/SkillTool/SkillTool.js'
```

`SkillTool` 是 LLM 调用技能的入口。通过它，模型可以执行预定义的工作流，如 `/commit`（提交代码）、`/verify`（验证变更）等。

### 16.2.4 技能发现

```typescript
// 技能发现预取——每次迭代运行
const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(
  null,
  messages,
  toolUseContext,
)
```

技能发现在查询循环的每次迭代中运行，与模型流式输出和工具执行并行。这替代了之前在 `getAttachmentMessages` 中的阻塞式发现路径。

### 16.2.5 技能搜索

```typescript
const clearSkillIndexCache = feature('EXPERIMENTAL_SKILL_SEARCH')
  ? require('./services/skillSearch/localSearch.js').clearSkillIndexCache
  : null
```

实验性技能搜索功能支持在大量技能中进行全文搜索。

### 16.2.6 斜杠命令技能

```typescript
import { getSlashCommandToolSkills } from './commands.js'
```

某些斜杠命令可以注册为技能，使得 LLM 可以自主调用它们。

## 16.3 技能变更检测

```typescript
import { skillChangeDetector } from './utils/skills/skillChangeDetector.js'
```

当技能文件发生变更时，系统能检测并重新加载，无需重启。

## 16.4 遥测

```typescript
import { logSkillsLoaded } from './utils/telemetry/skillLoadedEvent.js'
import { logPluginsEnabledForSession } from './utils/telemetry/pluginTelemetry.js'
import { logPluginLoadErrors } from './utils/telemetry/pluginTelemetry.js'

// 记录加载的技能
void logSkillsLoaded(getCwd(), getContextWindowForModel(model, getSdkBetas()))

// 记录启用的插件
logPluginsEnabledForSession(enabled, managedNames, getPluginSeedDirs())

// 记录插件加载错误
logPluginLoadErrors(errors, managedNames)
```

## 16.5 设计启示

### 技能 vs 工具 vs 命令

Claude Code 的三层扩展模型：

| 层次 | 目的 | 调用者 | 粒度 |
|------|------|--------|------|
| **工具 (Tool)** | 原子操作 | LLM | 低（读文件、执行命令） |
| **技能 (Skill)** | 工作流 | LLM / 用户 | 中（提交、审查） |
| **命令 (Command)** | 用户操作 | 用户 | 高（配置、诊断） |

### 渐进式扩展

- 内置插件/技能提供基础功能
- 用户可以添加自定义技能
- 企业可以通过托管插件统一配置
- 第三方可以发布开源插件

---

**下一章**：[设计模式与工程实践总结](./chapter-17-patterns.md)

# 第七章：命令系统 (Command System)

## 7.1 概述

命令系统管理用户通过 `/` 前缀调用的斜杠命令。这些命令提供了直接的用户交互功能，不通过 LLM 处理。命令注册表在 `src/commands.ts`（约 25,000 行）中管理。

## 7.2 命令注册

```typescript
// commands.ts — 静态导入
import commit from './commands/commit.js'
import compact from './commands/compact/index.js'
import config from './commands/config/index.js'
import doctor from './commands/doctor/index.js'
import memory from './commands/memory/index.js'
import review, { ultrareview } from './commands/review.js'
import vim from './commands/vim/index.js'
// ... 50+ 命令导入

// 条件导入（Feature Flag 控制）
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

const proactive = feature('PROACTIVE') || feature('KAIROS')
  ? require('./commands/proactive.js').default
  : null

const bridge = feature('BRIDGE_MODE')
  ? require('./commands/bridge/index.js').default
  : null

// 仅内部用户
const agentsPlatform = process.env.USER_TYPE === 'ant'
  ? require('./commands/agents-platform/index.js').default
  : null
```

## 7.3 命令分类

### 7.3.1 Git 操作

| 命令 | 功能 |
|------|------|
| `/commit` | 创建 Git 提交（自动生成提交信息） |
| `/commit-push-pr` | 提交、推送并创建 PR |
| `/diff` | 查看当前变更 |
| `/pr_comments` | 查看 PR 评论 |
| `/review` | 代码审查 |
| `/security-review` | 安全审查 |

### 7.3.2 会话管理

| 命令 | 功能 |
|------|------|
| `/compact` | 压缩对话上下文 |
| `/resume` | 恢复之前的会话 |
| `/share` | 分享会话 |
| `/session` | 会话管理 |
| `/rename` | 重命名会话 |
| `/clear` | 清除对话历史 |

### 7.3.3 配置与状态

| 命令 | 功能 |
|------|------|
| `/config` | 设置管理 |
| `/memory` | 持久记忆管理 |
| `/doctor` | 环境诊断 |
| `/cost` | 查看使用费用 |
| `/status` | 查看状态 |
| `/usage` | 查看使用量 |
| `/theme` | 切换主题 |

### 7.3.4 工具与扩展

| 命令 | 功能 |
|------|------|
| `/mcp` | MCP 服务器管理 |
| `/skills` | 技能管理 |
| `/tasks` | 任务管理 |
| `/init` | 初始化项目配置 |
| `/init-verifiers` | 初始化验证器 |

### 7.3.5 交互模式

| 命令 | 功能 |
|------|------|
| `/vim` | 切换 Vim 模式 |
| `/keybindings` | 键绑定配置 |
| `/color` | 颜色设置 |

### 7.3.6 认证

| 命令 | 功能 |
|------|------|
| `/login` | 登录 |
| `/logout` | 登出 |
| `/install-github-app` | 安装 GitHub App |
| `/install-slack-app` | 安装 Slack App |

### 7.3.7 跨平台

| 命令 | 功能 |
|------|------|
| `/desktop` | 传递到桌面应用 |
| `/mobile` | 传递到移动应用 |
| `/ide` | IDE 集成 |
| `/teleport` | 远程传送会话 |

### 7.3.8 Feature Flag 控制的命令

| 命令 | Feature Flag | 功能 |
|------|-------------|------|
| `/voice` | `VOICE_MODE` | 语音输入 |
| `/proactive` | `PROACTIVE` / `KAIROS` | 主动模式 |
| `/bridge` | `BRIDGE_MODE` | IDE 桥接 |
| `/assistant` | `KAIROS` | 助手模式 |
| `/workflows` | `WORKFLOW_SCRIPTS` | 工作流管理 |
| `/force-snip` | `HISTORY_SNIP` | 强制历史片段裁剪 |

## 7.4 命令与工具的交叉

命令系统与工具系统有一个有趣的交叉点——`getSlashCommandToolSkills()`：

```typescript
import { getSlashCommandToolSkills } from './commands.js'
```

某些斜杠命令可以被注册为"工具技能"，使得 LLM 不仅可以在用户主动调用时使用它们，还可以在认为合适时自主调用。

## 7.5 远程模式命令过滤

```typescript
import { filterCommandsForRemoteMode, getCommands } from './commands.js'
```

在远程模式下，某些不适用的命令（如 `/desktop`、`/vim`）会被自动过滤。

## 7.6 命令生命周期

```typescript
import { notifyCommandLifecycle } from './utils/commandLifecycle.js'

// 在查询循环完成后通知命令完成
for (const uuid of consumedCommandUuids) {
  notifyCommandLifecycle(uuid, 'completed')
}
```

每个命令有明确的生命周期追踪：启动 → 执行 → 完成/失败。这支持异步命令的状态追踪和错误报告。

## 7.7 设计启示

### 命令 vs 工具的边界

Claude Code 将**用户发起的操作**（命令）与**LLM 发起的操作**（工具）分离，但允许交叉。这种设计提供了清晰的心智模型：
- 命令：用户主动控制，即时反馈
- 工具：LLM 自主调用，异步执行

### 按需加载

只在需要时加载命令实现（通过 Feature Flag 控制），保持 CLI 的启动速度。

---

**下一章**：[权限与安全模型](./chapter-08-permissions.md)

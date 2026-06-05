# 第二章：技术栈与项目结构

## 2.1 为什么选择 Bun

Claude Code 选择 [Bun](https://bun.sh) 作为运行时而非 Node.js，这是一个值得深思的技术决策。

### 2.1.1 Bun 的关键优势

| 特性 | Node.js | Bun | Claude Code 的受益点 |
|------|---------|-----|---------------------|
| 启动速度 | ~40ms | ~6ms | CLI 工具对冷启动极度敏感 |
| TypeScript 支持 | 需要 ts-node/tsx | 原生支持 | 无需编译步骤即可运行 .ts/.tsx |
| 内置 bundler | 无 | 有 | 支持 `bun:bundle` Feature Flag 系统 |
| 包管理器 | npm/yarn/pnpm | 内置 (bun install) | 更快的依赖安装 |
| 文件系统操作 | 标准库 | 优化的 I/O | 大量文件读写操作受益 |

### 2.1.2 `bun:bundle` 虚拟模块

Claude Code 利用 Bun 独有的 `bun:bundle` 虚拟模块实现编译时功能开关：

```typescript
// 生产环境：Bun bundler 在编译时静态替换 feature() 调用
import { feature } from 'bun:bundle'

// 如果 VOICE_MODE 未启用，以下代码会在编译时被完全移除（死代码消除）
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

在开发环境中，通过 `plugins/bunBundleDev.ts` 插件提供运行时 polyfill：

```typescript
// plugins/bunBundleDev.ts
import { plugin } from 'bun'

const enabledFlags = new Set(
  (process.env.FEATURE_FLAGS ?? '').split(',').filter(Boolean),
)

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

开发时通过环境变量启用特定功能：

```bash
FEATURE_FLAGS=KAIROS,VOICE_MODE bun run src/main.tsx
```

## 2.2 TypeScript 配置

Claude Code 使用严格模式的 TypeScript：

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "src/*": ["./src/*"],
      "bun:bundle": ["./src/types/bun-bundle.d.ts"]
    },
    "types": ["bun-types"],
    "lib": ["ESNext"],
    "allowImportingTsExtensions": true,
    "noEmit": true
  }
}
```

关键配置点：

1. **`moduleResolution: "bundler"`**：使用 Bun 的模块解析策略。
2. **`jsx: "react-jsx"`**：支持 JSX 语法，用于 Ink 终端 UI 组件。
3. **路径别名**：`src/*` 映射允许使用绝对导入路径（如 `src/bootstrap/state.js`）。
4. **`bun:bundle` 类型映射**：将虚拟模块映射到本地类型定义文件。

## 2.3 构建时宏定义

通过 `bunfig.toml` 定义构建时注入的全局常量：

```toml
[run]
preload = ["./plugins/bunBundleDev.ts"]

[define]
"MACRO.VERSION" = '"1.0.0-dev"'
"MACRO.BUILD_TIME" = '""'
"MACRO.PACKAGE_URL" = '"@anthropic-ai/claude-code"'
"MACRO.NATIVE_PACKAGE_URL" = 'undefined'
"MACRO.FEEDBACK_CHANNEL" = '"https://github.com/anthropics/claude-code/issues"'
"MACRO.ISSUES_EXPLAINER" = '"file an issue at ..."'
"MACRO.VERSION_CHANGELOG" = '""'
```

这些 `MACRO.*` 常量在代码中直接作为全局变量使用，Bun 在构建/运行时自动替换。

## 2.4 依赖分析

### 2.4.1 核心依赖

| 分类 | 包名 | 用途 |
|------|------|------|
| **AI/API** | `@anthropic-ai/sdk` | Anthropic API 客户端 |
| | `@anthropic-ai/claude-agent-sdk` | Agent SDK |
| | `@aws-sdk/client-bedrock-runtime` | AWS Bedrock 支持 |
| | `google-auth-library` | Google Cloud 认证 |
| **终端 UI** | `react` (canary) | React 19 Canary 版本 |
| | `react-reconciler` (canary) | 自定义 React 渲染器 |
| | `chalk` | 终端着色 |
| | `cli-boxes` | 终端边框 |
| | `figures` | Unicode 符号 |
| **CLI** | `commander` | 命令解析 |
| | `@commander-js/extra-typings` | Commander 增强类型 |
| **协议** | `@modelcontextprotocol/sdk` | MCP 协议实现 |
| | `vscode-jsonrpc` | JSON-RPC 通信 |
| | `vscode-languageserver-protocol` | LSP 协议 |
| **数据处理** | `zod` | 运行时模式验证 |
| | `ajv` | JSON Schema 验证 |
| | `diff` | 文本差异比较 |
| | `marked` | Markdown 解析 |
| **工具** | `execa` | 子进程管理 |
| | `chokidar` | 文件监听 |
| | `fuse.js` | 模糊搜索 |
| | `lodash-es` | 实用函数库 |
| **网络** | `undici` | HTTP 客户端 |
| | `ws` | WebSocket |
| | `https-proxy-agent` | 代理支持 |
| **安全** | `xss` | XSS 过滤 |
| | `proper-lockfile` | 文件锁 |
| **遥测** | `@opentelemetry/*` | OpenTelemetry 全家桶 |
| | `@growthbook/growthbook` | 功能开关 |

### 2.4.2 特殊依赖：React Canary

值得注意的是，Claude Code 使用的是 React **19.3.0-canary** 版本，而非稳定版。这是因为 Ink（终端 UI 框架）需要与 `react-reconciler` 的 canary 版本匹配。

### 2.4.3 内部 Stub 包

被替换的 Anthropic 内部包（使用 `file:stubs/...` 引用）：

```json
{
  "@ant/claude-for-chrome-mcp": "file:stubs/@ant/claude-for-chrome-mcp",
  "@ant/computer-use-input": "file:stubs/@ant/computer-use-input",
  "@ant/computer-use-mcp": "file:stubs/@ant/computer-use-mcp",
  "@ant/computer-use-swift": "file:stubs/@ant/computer-use-swift",
  "@anthropic-ai/mcpb": "file:stubs/@anthropic-ai/mcpb",
  "@anthropic-ai/sandbox-runtime": "file:stubs/anthropic-ai/sandbox-runtime"
}
```

这些包在内部用于：
- **Chrome 浏览器控制** (`claude-for-chrome-mcp`)
- **计算机使用** (`computer-use-*`)：屏幕截图、鼠标/键盘操控
- **沙箱运行时** (`sandbox-runtime`)：安全隔离执行环境

## 2.5 目录结构详解

```
src/
├── main.tsx                 # 入口文件：Commander.js CLI 解析 + React/Ink 初始化
├── commands.ts              # 命令注册表：管理所有斜杠命令
├── tools.ts                 # 工具注册表：管理所有 AI 可调用工具
├── Tool.ts                  # 工具类型定义：接口、权限模型、进度类型
├── QueryEngine.ts           # 核心查询引擎：LLM API 调用、流式响应、工具循环
├── query.ts                 # 查询循环：消息处理、自动压缩、Token 管理
├── context.ts               # 上下文收集：System/User Context 生成
├── cost-tracker.ts          # 成本追踪：Token 计数、费用计算
│
├── commands/                # 斜杠命令实现 (~50 个命令)
│   ├── commit.ts            #   /commit - Git 提交
│   ├── review.ts            #   /review - 代码审查
│   ├── compact/             #   /compact - 上下文压缩
│   ├── config/              #   /config - 设置管理
│   ├── doctor/              #   /doctor - 环境诊断
│   ├── memory/              #   /memory - 持久记忆管理
│   ├── mcp/                 #   /mcp - MCP 服务器管理
│   ├── skills/              #   /skills - 技能管理
│   ├── tasks/               #   /tasks - 任务管理
│   ├── vim/                 #   /vim - Vim 模式
│   └── ...
│
├── tools/                   # 工具实现 (~40 个工具)
│   ├── AgentTool/           #   子代理生成
│   ├── BashTool/            #   Shell 命令执行
│   ├── FileReadTool/        #   文件读取
│   ├── FileWriteTool/       #   文件创建/覆盖
│   ├── FileEditTool/        #   文件部分编辑
│   ├── GlobTool/            #   文件模式匹配
│   ├── GrepTool/            #   ripgrep 内容搜索
│   ├── WebFetchTool/        #   URL 内容获取
│   ├── WebSearchTool/       #   网页搜索
│   ├── SkillTool/           #   技能执行
│   ├── MCPTool/             #   MCP 工具调用
│   ├── LSPTool/             #   语言服务器协议
│   ├── TeamCreateTool/      #   团队代理创建
│   ├── SendMessageTool/     #   代理间消息
│   ├── EnterPlanModeTool/   #   进入计划模式
│   ├── EnterWorktreeTool/   #   Git Worktree 隔离
│   └── ...
│
├── components/              # Ink UI 组件 (~140 个)
│   ├── REPL.tsx             #   主 REPL 界面
│   ├── MessageSelector.tsx  #   消息选择器
│   ├── AgentProgressLine.tsx #  代理进度显示
│   ├── Spinner.tsx          #   加载动画
│   └── ...
│
├── hooks/                   # React Hooks
│   ├── toolPermission/      #   工具权限检查
│   ├── fileSuggestions.ts   #   文件建议
│   ├── useCanUseTool.ts     #   工具使用权限
│   └── ...
│
├── services/                # 外部服务集成
│   ├── api/                 #   Anthropic API 客户端
│   ├── mcp/                 #   MCP 服务器管理
│   ├── oauth/               #   OAuth 认证
│   ├── lsp/                 #   LSP 管理器
│   ├── compact/             #   上下文压缩服务
│   ├── analytics/           #   GrowthBook 分析
│   ├── plugins/             #   插件加载
│   ├── policyLimits/        #   策略限制
│   ├── extractMemories/     #   自动记忆提取
│   └── ...
│
├── bridge/                  # IDE 集成桥
│   ├── bridgeMain.ts        #   桥主循环
│   ├── bridgeMessaging.ts   #   消息协议
│   └── ...
│
├── coordinator/             # 多代理协调器
│   └── coordinatorMode.ts   #   协调器模式
│
├── state/                   # 状态管理
│   ├── AppStateStore.tsx    #   全局应用状态
│   ├── store.ts             #   状态存储
│   └── onChangeAppState.ts  #   状态变更监听
│
├── plugins/                 # 插件系统
├── skills/                  # 技能系统
├── memdir/                  # 持久记忆目录
├── tasks/                   # 任务管理
├── voice/                   # 语音输入
├── vim/                     # Vim 模式
├── remote/                  # 远程会话
├── server/                  # 服务器模式
├── schemas/                 # Zod 配置模式
├── migrations/              # 配置迁移
├── entrypoints/             # 入口初始化逻辑
├── types/                   # 类型定义
├── utils/                   # 工具函数
├── constants/               # 常量定义
├── ink/                     # Ink 渲染器封装
├── buddy/                   # 伴侣精灵（彩蛋）
└── upstreamproxy/           # 代理配置
```

## 2.6 设计启示

### 选择合适的运行时

Claude Code 选择 Bun 而非 Node.js 的决策展示了一个重要原则：**CLI 工具应该为冷启动时间进行优化**。Bun 的 6ms 启动时间 vs Node.js 的 40ms，对于频繁调用的 CLI 工具来说影响显著。

### 虚拟模块实现编译时优化

`bun:bundle` 的 Feature Flag 系统展示了一种优雅的**死代码消除**方案。未启用的功能在编译时被完全移除，减小了最终产物的体积，同时避免了运行时检查的开销。

### 内部包的 Stub 策略

对于无法公开的内部依赖，使用 `file:stubs/` 替换为空实现是一种实用的解耦策略，使得核心代码可以在缺失内部依赖的情况下独立运行。

---

**下一章**：[启动流程与初始化](./chapter-03-startup.md)

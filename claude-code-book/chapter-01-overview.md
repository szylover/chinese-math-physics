# 第一章：概述与泄露始末

## 1.1 Claude Code 是什么

Claude Code 是 Anthropic 公司开发的官方 CLI（命令行界面）编程助手。它允许开发者直接在终端中与 Claude 大语言模型交互，执行软件工程任务——包括编辑文件、运行命令、搜索代码库、管理 Git 工作流等。

与 ChatGPT 等网页界面不同，Claude Code 的定位是**嵌入开发者工作流**的原生工具。它不是一个简单的聊天机器人，而是一个拥有完整工具链的 AI 编程代理（Agent）。它可以：

- 读写文件系统中的任意文件
- 执行 Shell 命令（Bash/PowerShell）
- 使用 ripgrep 搜索代码
- 创建 Git 提交和 Pull Request
- 生成子代理（Sub-agent）并行处理任务
- 通过 MCP（Model Context Protocol）连接外部工具
- 在 VS Code 和 JetBrains 等 IDE 中作为后端运行

## 1.2 泄露事件

### 1.2.1 时间线

2026 年 3 月 31 日，安全研究员 [Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) 在 X（原 Twitter）上公开了一个发现：

> "Claude code source code has been leaked via a map file in their npm registry!"

### 1.2.2 泄露机制

泄露的技术细节如下：

1. **Source Map 文件**：Anthropic 在将 Claude Code 发布到 npm 注册表时，发布包中包含了一个 `.map`（Source Map）文件。
2. **Source Map 中的引用**：该 Source Map 文件中包含了对完整、未混淆的 TypeScript 源码的引用路径。
3. **R2 存储桶**：这些源码文件存储在 Anthropic 的 Cloudflare R2 存储桶中，且可以公开下载。
4. **完整源码可获取**：通过解析 Source Map 中的路径，可以下载整个 `src/` 目录的完整源码。

```
npm package (.tgz)
  └── dist/
       └── bundle.js.map  ← Source Map 文件
            └── sourcesContent 引用 → R2 存储桶中的 TypeScript 源文件
```

### 1.2.3 影响范围

泄露的源码包含：
- 完整的 `src/` 目录，约 **1,900 个文件**
- 总计 **512,000+ 行** TypeScript 代码
- 所有核心模块的完整实现

**未包含的部分**：
- `@ant/*` 开头的 Anthropic 内部专用包（如 `@ant/computer-use-mcp`、`@ant/claude-for-chrome-mcp`）
- 部分构建脚本和 CI/CD 配置
- 测试文件（部分）

在公开仓库中，缺失的内部包被替换为导出空操作（no-op）的 stub 模块。

## 1.3 项目基本信息

| 属性 | 值 |
|------|------|
| 语言 | TypeScript（严格模式） |
| 运行时 | [Bun](https://bun.sh) v1.3+ |
| 终端 UI 框架 | React + [Ink](https://github.com/vadimdemedes/ink) |
| CLI 解析 | [Commander.js](https://github.com/tj/commander.js)（extra-typings） |
| 模式验证 | [Zod](https://zod.dev) v4 |
| 代码搜索 | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| 协议支持 | MCP SDK、LSP |
| API 客户端 | Anthropic SDK |
| 遥测 | OpenTelemetry + gRPC |
| 功能开关 | GrowthBook |
| 认证 | OAuth 2.0、JWT、macOS Keychain |

## 1.4 架构概览

Claude Code 的整体架构可以用一张图来概括：

```
用户输入 (终端)
    │
    ▼
┌─────────────────────────────────────────────┐
│  main.tsx (Commander.js CLI 解析)            │
│  ├── 启动优化：并行预取 MDM / Keychain       │
│  └── React/Ink 渲染器初始化                  │
└─────────────────┬───────────────────────────┘
                  │
    ▼─────────────┼──────────────▼
┌────────────┐    │    ┌────────────────┐
│  REPL 模式  │    │    │  Print 模式     │
│ (交互式)    │    │    │ (单次执行 -p)   │
└─────┬──────┘    │    └───────┬────────┘
      │           │            │
      ▼───────────┴────────────▼
┌─────────────────────────────────────────────┐
│         QueryEngine (查询引擎)               │
│  ├── 构建 System Prompt                      │
│  ├── 管理消息历史                             │
│  ├── 调用 Anthropic API                      │
│  ├── 处理 Tool Use 循环                      │
│  └── Token 计数 & 成本追踪                   │
└─────────────────┬───────────────────────────┘
                  │
    ▼─────────────┼──────────────▼
┌────────────┐    │    ┌────────────────┐
│  工具系统   │    │    │  命令系统       │
│ (40+ 工具)  │    │    │ (50+ 斜杠命令) │
└─────┬──────┘    │    └───────┬────────┘
      │           │            │
      ▼───────────┴────────────▼
┌─────────────────────────────────────────────┐
│              服务层 (Services)                │
│  ├── Anthropic API 客户端                    │
│  ├── MCP 服务器连接                          │
│  ├── OAuth 认证                              │
│  ├── LSP 语言服务                            │
│  ├── 上下文压缩 (Compact)                    │
│  ├── 分析 / 遥测                             │
│  └── 插件 / 技能加载                         │
└─────────────────────────────────────────────┘
```

## 1.5 本书的分析方法

本书采用**自顶向下**与**自底向上**相结合的方法：

1. **自顶向下**：从入口文件 `main.tsx` 开始，沿着启动流程和请求处理链路，逐步深入到每个子系统。
2. **自底向上**：对关键的类型定义（如 `Tool.ts` 中的接口）和工具实现（如 `BashTool`）进行独立的深度分析。
3. **横向对比**：将 Claude Code 的设计决策与业界常见做法（如 LangChain、AutoGPT 等）进行对比，分析其设计取舍。

每章结尾都包含**设计启示**部分，总结该模块中值得借鉴的设计模式和工程实践。

---

**下一章**：[技术栈与项目结构](./chapter-02-tech-stack.md)

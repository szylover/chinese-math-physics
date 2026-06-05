# 第十二章：终端 UI：React + Ink

## 12.1 技术选型

Claude Code 使用 **React + Ink** 构建终端 UI——这是一个非常规但极其优雅的选择。[Ink](https://github.com/vadimdemedes/ink) 是一个使用 React 语法构建命令行界面的框架，将 React 的组件模型和声明式编程带到终端。

## 12.2 为什么用 React 做终端 UI

| 传统方案 | React + Ink |
|----------|-------------|
| 命令式字符串拼接 | 声明式组件 |
| 手动管理状态更新 | React 状态管理 |
| 难以复用 UI 逻辑 | 组件化复用 |
| 难以测试 | 可测试的组件 |
| 难以实现复杂布局 | Flexbox 布局 |

## 12.3 React Canary 版本

```json
{
  "react": "^19.3.0-canary-705268dc-20260409",
  "react-reconciler": "^0.34.0-canary-705268dc-20260409"
}
```

Claude Code 使用 React 19 的 canary 版本。这是因为 Ink 需要与 `react-reconciler` 紧密配合，而 reconciler 的稳定版本可能不支持最新的 Ink 特性。

## 12.4 组件库概览

`src/components/` 目录包含约 **140 个组件**，覆盖了整个 CLI 界面：

### 12.4.1 核心组件

| 组件 | 功能 |
|------|------|
| `REPL.tsx` | 主 REPL 界面——整个应用的顶层组件 |
| `MessageSelector.tsx` | 消息选择器（选择发送哪些消息给 API） |
| `AgentProgressLine.tsx` | 代理进度显示 |
| `Spinner.tsx` | 加载动画 |

### 12.4.2 Ink 渲染器封装

```
src/ink/
├── renderer.ts        # 自定义 Ink 渲染器
└── termio/
    └── dec.ts         # 终端控制序列
```

```typescript
import { SHOW_CURSOR } from './ink/termio/dec.js'
```

Claude Code 对 Ink 渲染器进行了封装，添加了终端控制序列支持（如光标显示/隐藏）。

## 12.5 JSX 与工具 UI

工具可以通过 `setToolJSX` 设置自定义的 React UI：

```typescript
export type SetToolJSXFn = (
  args: {
    jsx: React.ReactNode | null
    shouldHidePromptInput: boolean
    shouldContinueAnimation?: true
    showSpinner?: boolean
    isLocalJSXCommand?: boolean
    isImmediate?: boolean
    clearLocalJSX?: boolean
  } | null,
) => void
```

这意味着每个工具执行时可以渲染自己的 UI 组件——例如，文件编辑工具可以显示差异视图，搜索工具可以显示实时结果。

## 12.6 React Hooks 在 CLI 中的应用

`src/hooks/` 目录包含大量 React Hooks：

```
src/hooks/
├── toolPermission/      # 工具权限相关 Hooks
├── fileSuggestions.ts   # 文件建议
├── useCanUseTool.ts     # 工具使用权限
└── ...
```

例如，`useCanUseTool` Hook 在 REPL 组件中使用，每次工具调用都通过它进行权限检查。

## 12.7 全屏界面 (Screens)

```
src/screens/
├── Doctor/      # /doctor 诊断界面
├── Resume/      # /resume 恢复界面
└── ...
```

某些命令（如 `/doctor`）需要全屏 UI。这些通过独立的 Screen 组件实现，接管整个终端。

## 12.8 FPS 追踪

```typescript
import type { FpsMetrics } from './utils/fpsTracker.js'

// 启动遥测中记录
lastFpsAverage: fpsMetrics?.averageFps,
lastFpsLow1Pct: fpsMetrics?.low1PctFps,
```

Claude Code 追踪 UI 的渲染帧率，记录平均 FPS 和 1% 最低 FPS。这反映了 Anthropic 对终端 UI 流畅度的重视。

## 12.9 输出样式

```
src/outputStyles/   # 输出样式配置
```

支持不同的输出样式——包括主题、颜色方案等，可通过 `/theme` 命令切换。

## 12.10 Vim 模式

```
src/vim/           # Vim 模式实现
```

Claude Code 甚至内置了 Vim 模式，通过 `/vim` 命令切换。这反映了其目标用户群——习惯在终端中工作的开发者。

## 12.11 Buddy（伴侣精灵）

```
src/buddy/         # 伴侣精灵（彩蛋）
```

一个有趣的彩蛋——Claude Code 包含一个"伴侣精灵"组件，为终端体验增添趣味性。

## 12.12 设计启示

### React 的通用性

React 不仅仅是一个"Web 框架"——它的核心是一个**声明式 UI 编程模型**。通过 `react-reconciler`，这个模型可以适配到任何渲染目标：浏览器 DOM、原生移动端、VR、甚至终端。

### 组件化的终端 UI

140+ 个组件的规模证明了组件化方法在复杂 CLI 应用中的可行性。传统的字符串拼接方式在这种规模下将变得不可维护。

### 性能追踪

FPS 追踪在 CLI 应用中很少见，但 Claude Code 的实时流式输出场景确实需要关注渲染性能。

---

**下一章**：[Feature Flag 与条件编译](./chapter-13-feature-flags.md)

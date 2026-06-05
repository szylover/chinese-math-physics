# 《深入剖析 Claude Code：Anthropic AI 编程助手架构与实现》

> 基于 2026-03-31 泄露源码的全面技术分析

---

## 关于本书

本书是对 Anthropic 官方 CLI 编程助手 **Claude Code** 的深度技术剖析。2026 年 3 月 31 日，Claude Code 的完整 TypeScript 源码通过 npm 注册表中的 `.map` 文件意外泄露。本书基于该公开源码，系统性地分析了 Claude Code 的架构设计、核心模块、上下文处理、工具系统、权限模型等关键技术细节。

**规模**：约 1,900 个源文件，512,000+ 行 TypeScript 代码。

---

## 目录

| 章节 | 标题 | 文件 |
|------|------|------|
| 第一章 | [概述与泄露始末](./chapter-01-overview.md) | `chapter-01-overview.md` |
| 第二章 | [技术栈与项目结构](./chapter-02-tech-stack.md) | `chapter-02-tech-stack.md` |
| 第三章 | [启动流程与初始化](./chapter-03-startup.md) | `chapter-03-startup.md` |
| 第四章 | [核心查询引擎 QueryEngine](./chapter-04-query-engine.md) | `chapter-04-query-engine.md` |
| 第五章 | [上下文处理与 System Prompt](./chapter-05-context.md) | `chapter-05-context.md` |
| 第六章 | [工具系统 (Tool System)](./chapter-06-tool-system.md) | `chapter-06-tool-system.md` |
| 第七章 | [命令系统 (Command System)](./chapter-07-command-system.md) | `chapter-07-command-system.md` |
| 第八章 | [权限与安全模型](./chapter-08-permissions.md) | `chapter-08-permissions.md` |
| 第九章 | [上下文压缩与 Token 管理](./chapter-09-context-compression.md) | `chapter-09-context-compression.md` |
| 第十章 | [多智能体协调 (Coordinator Mode)](./chapter-10-coordinator.md) | `chapter-10-coordinator.md` |
| 第十一章 | [MCP 协议与外部集成](./chapter-11-mcp.md) | `chapter-11-mcp.md` |
| 第十二章 | [终端 UI：React + Ink](./chapter-12-terminal-ui.md) | `chapter-12-terminal-ui.md` |
| 第十三章 | [Feature Flag 与条件编译](./chapter-13-feature-flags.md) | `chapter-13-feature-flags.md` |
| 第十四章 | [IDE 桥接与远程会话](./chapter-14-bridge-remote.md) | `chapter-14-bridge-remote.md` |
| 第十五章 | [成本追踪与遥测](./chapter-15-cost-telemetry.md) | `chapter-15-cost-telemetry.md` |
| 第十六章 | [插件与技能系统](./chapter-16-plugins-skills.md) | `chapter-16-plugins-skills.md` |
| 第十七章 | [设计模式与工程实践总结](./chapter-17-patterns.md) | `chapter-17-patterns.md` |

---

## 适合读者

- 对 AI 编程助手内部实现感兴趣的开发者
- 想要构建类似 CLI AI 工具的工程师
- 研究大语言模型 (LLM) 应用架构的技术人员
- 对 TypeScript 大型项目架构感兴趣的前端/全栈工程师

## 技术前提

- 熟悉 TypeScript / JavaScript
- 了解 React 基本概念
- 了解 LLM API 调用的基本流程
- 对命令行工具有基本使用经验

---

## 声明

本书仅用于技术研究与教育目的。所有分析基于已公开的源码。原始源码的知识产权归 [Anthropic](https://www.anthropic.com) 所有。

# 第二十一章：Agentic Engineering 实战 —— 大型移动端项目的 AI 工程化体系

> 基于某大型移动端项目的真实实践，2026-06

---

## 目录

1. [为什么要做 Agentic Engineering](#1-为什么要做-agentic-engineering)
2. [整体架构一览](#2-整体架构一览)
3. [Agent 开发详解](#3-agent-开发详解)
4. [Skill 开发详解](#4-skill-开发详解)
5. [Hook 质量门禁](#5-hook-质量门禁)
6. [Rules 条件加载规则](#6-rules-条件加载规则)
7. [Eval 评测体系](#7-eval-评测体系)
8. [多平台指令镜像](#8-多平台指令镜像)
9. [设计模式与最佳实践](#9-设计模式与最佳实践)
10. [从 0 到 1 创建你的第一个 Skill](#10-从-0-到-1-创建你的第一个-skill)
11. [从 0 到 1 创建你的第一个 Agent](#11-从-0-到-1-创建你的第一个-agent)
12. [反模式与踩坑记录](#12-反模式与踩坑记录)
13. [附录：该项目现有 Agent/Skill 速查表](#13-附录该项目现有-agentskill-速查表)

---

## 1. 为什么要做 Agentic Engineering

### 背景

该项目是某大厂移动端 AI 产品的核心 SDK，包含 120+ 个 Kotlin/Compose 模块和对应的 iOS SwiftUI 模块，嵌入多个宿主 App。团队日常面临：

- **重复性高**：加 feature flag、加 snapshot test、加 telemetry event 都有固定套路
- **PR review 成本大**：跨 iOS/Android 双平台，review 规则多且容易遗漏
- **值班 DRI 工作琐碎**：事件管理平台分诊、发版、周报都是机械流程
- **质量事故有规律**：线上 regression 往往因为漏跑某个 review pass

### 核心理念

> **Treat AI agents as team members with guardrails.**
>
> 给 agent 足够自主权去完成工程任务，但通过 hook、eval、review-pass 形成多层质量闭环。

这不是"用 AI 写代码"那么简单——而是构建一个 **AI 工程师团队**，每个 agent 有明确职责、工具链、知识库和约束条件。

---

## 2. 整体架构一览

```
.claude/                          ← Claude Code 的配置根目录
├── agents/                       ← Agent 定义（角色 + 工具 + 知识库）
│   ├── Nova.agent.md             ← 自主编码 agent
│   ├── Arbiter.agent.md          ← PR 质量评审 agent
│   ├── DRI.agent.md              ← 值班管理 agent
│   ├── Arbiter/                  ← Arbiter 的知识库子目录
│   │   ├── evaluation-flow.md
│   │   ├── rubric-ios.md
│   │   ├── rubric-android.md
│   │   └── scorecard-template.md
│   └── DRI/
│       ├── config.json
│       └── workflows/
│           ├── 1-dashboard-monitoring.md
│           ├── 2-incident-triage.md
│           └── ...
│
├── skills/                       ← 可组合能力模块
│   ├── pr-risk-evaluate/
│   │   ├── SKILL.md              ← 核心定义文件
│   │   ├── guardrails.md         ← 辅助知识
│   │   └── evals/                ← 评测用例
│   ├── android-add-feature-flag/
│   │   └── SKILL.md
│   └── ...（40+ skills）
│
├── hooks/                        ← 工具调用拦截器
│   ├── pr-review-gate.sh         ← 创建 PR 前检查 review pass
│   ├── pre-push-ci-gate.sh       ← 推送前检查 CI marker
│   ├── swift-lint-on-edit.sh     ← 编辑 Swift 后自动 lint
│   └── mark-review.sh            ← 记录 review 状态
│
├── rules/                        ← 条件加载的编码规范
│   ├── ios-development.md        ← 仅 ios/**/*.swift 时加载
│   ├── android-testing.md        ← 仅 android/**/test/** 时加载
│   └── pr-format.md              ← 始终加载
│
├── evals/                        ← Agent 评测脚本
│   ├── run-nova-eval.py
│   └── verify-nova-eval.py
│
├── resources/                    ← 共享知识文件
│   └── kusto_telemetry_table_reference.md
│
├── commands/                     ← 快捷命令
└── settings.json                 ← 权限 + hook 绑定
```

### 数据流

```
用户请求
    │
    ▼
┌─────────────┐     意图路由      ┌──────────────┐
│  Agent 层   │ ───────────────→ │  Skill 层    │
│  (角色决策) │                   │  (能力执行)  │
└─────┬───────┘                   └──────┬───────┘
      │                                  │
      │  调用工具（MCP/CLI/编辑文件）      │
      ▼                                  ▼
┌─────────────┐                   ┌──────────────┐
│  Hook 层    │ ←── 拦截/增强 ──→ │  Rules 层    │
│  (质量门禁) │                   │  (编码规范)  │
└─────┬───────┘                   └──────────────┘
      │
      ▼
┌─────────────┐
│  Eval 层    │  ← 事后验证 agent/skill 输出质量
│  (评测闭环) │
└─────────────┘
```

---

## 3. Agent 开发详解

### 3.1 Agent 文件结构

一个 agent 由 **一个 `.agent.md` 文件** + **可选的知识库子目录** 组成。

```
.claude/agents/
├── MyAgent.agent.md          ← 主定义文件
└── MyAgent/                  ← 知识库目录（可选）
    ├── workflow-a.md
    ├── workflow-b.md
    └── config.json
```

### 3.2 `.agent.md` 文件格式

```markdown
---
name: MyAgent
description: "一句话描述 agent 的职责和触发场景。"
argument-hint: "告诉用户可以传什么参数，如 'a PR number' or 'LOCAL_DIFF'"
tools:
  [
    "read",
    "edit",
    "search",
    "Bash(git:*)",
    "mcp__pm__*",          # MCP 工具用完全限定名
    "todos",
  ]
---

# MyAgent - 角色全称

## Instructions

@file:.claude/agents/MyAgent/workflow-a.md    ← 引用外部文件

## Overview

描述 agent 的核心职责...

## Trigger Phrases

| 触发词 |
| "do X"; "handle Y"; "run Z" |

## Step-by-Step Workflow

### Step 1: ...
### Step 2: ...
```

### 3.3 关键设计原则

#### 原则 1：工具集最小化

```yaml
# ✅ 好：只声明需要的工具
tools: ["read", "search", "mcp__pm__repo_pull_request"]

# ❌ 差：给所有权限
tools: ["*"]
```

Agent 的 `tools` 字段定义了它能使用的工具白名单。遵循最小权限原则——DRI agent 不需要编辑文件的权限，Arbiter 不需要推代码的权限。

#### 原则 2：知识按需加载（Progressive Disclosure）

```markdown
## Intent Triage

| Intent | Workflow File to Read |
|--------|------------------------|
| 事件分诊 | `.claude/agents/DRI/workflows/2-incident-triage.md` |
| 发版    | `.claude/agents/DRI/workflows/3-release-cut.md` |
```

**不要**在 agent 主文件里塞所有知识。DRI agent 有 5 个 workflow，但只在匹配意图后才加载对应的 workflow 文件。这样：
- 减少 context window 占用
- 加载更快
- 每个 workflow 可以独立迭代

#### 原则 3：结构化输出格式

```markdown
## Output Format

| Dimension | Score | Assessment |
|-----------|-------|------------|
| Scope     | X/25  | ...        |
| Semantic  | X/25  | ...        |
```

给 agent 定义严格的输出模板。这样：
- 人类可预期输出结构
- 下游 skill/hook 可以解析
- eval 可以验证格式正确性

#### 原则 4：Todo-Driven 执行

```markdown
## CREATE TODO LIST FIRST

**BEFORE starting any coding work, you MUST create a todo list.**

### Required Todo Structure
1. Understand the requirement
2. Explore codebase
3. Plan implementation
4. Create feature branch
5. Implement changes
6. Add tests
7. Run quality checks
8. Create PR
```

Nova agent 强制要求先创建 todo list 再开始编码。这提供了：
- 对用户的可见性（知道 agent 在做什么）
- 进度追踪
- 可中断/可恢复

### 3.4 现有 Agent 案例分析

#### Nova（自主编码 agent）

- **角色**：给定项目管理平台 ticket，端到端实现功能
- **知识库**：40KB+ 的 coding-workflow.md（业界最详细的 agent workflow 之一）
- **关键设计**：
  - 强制 todo list → 每一步都有可见性
  - 调用 skill（如 `android-add-feature-flag`）而不是自己写脚手架代码
  - 完成后自动触发 `pr-risk-evaluate` 和 `review-pr`

#### Arbiter（PR 质量评审 agent）

- **角色**：独立评审 PR 质量，6 维度 5 级打分
- **知识库**：平台特定 rubric（rubric-ios.md / rubric-android.md）
- **关键设计**：
  - 和 Nova 形成 **"maker-checker"** 模式——Nova 写代码，Arbiter 审代码
  - 有 scorecard 模板确保输出一致
  - 支持 `--build-free` 模式（跳过构建，只做静态分析）

#### DRI（值班管理 agent）

- **角色**：IM 事件分诊 + 发版 + 周报
- **MCP 工具链**：IM（事件管理）+ 日志查询平台 + 即时通讯（发消息）
- **关键设计**：
  - 支持 `/loop 10m @DRI check dashboard` —— 每 10 分钟自动巡检
  - 有 `config.json` 控制自动分诊行为
  - 工作流严格分离到 5 个独立文件

---

## 4. Skill 开发详解

### 4.1 Skill vs Agent 的区别

| | Agent | Skill |
|--|-------|-------|
| **比喻** | 一个有角色的工程师 | 工程师掌握的一项技能 |
| **触发** | 用户直接对话 | 被 agent 或用户调用 |
| **状态** | 有持续的角色上下文 | 无状态，执行完即结束 |
| **复杂度** | 高（多步决策、意图路由） | 中（步骤明确的标准流程） |
| **文件** | `.agent.md` + 知识库目录 | `SKILL.md` + 辅助文件 + evals |

### 4.2 Skill 文件结构

```
.claude/skills/
└── my-skill/
    ├── SKILL.md              ← 核心定义（必须）
    ├── helper-data.md        ← 辅助知识文件（可选）
    ├── reference-rules.md    ← 参考规则（可选）
    └── evals/                ← 评测用例（强烈推荐，G6 guardrail 要求）
        ├── eval-001.md
        ├── eval-002.md
        └── ...
```

### 4.3 `SKILL.md` 文件格式

```markdown
---
name: my-skill
description: Does X for Y in the project. Use when asked to "do X",
             "handle Y", or "run Z for module W".
---

# My Skill

简短描述这个 skill 做什么。

## Prerequisites

- 列出前置条件
- 需要哪些工具/环境

## Step-by-Step Instructions

### 1. 第一步
具体指令...

### 2. 第二步
具体指令...

## Output Format

定义输出格式...
```

### 4.4 SKILL.md 写作规范

根据该项目的 `skill-review` skill 中总结的 best practices：

| # | 规则 | 示例 |
|---|------|------|
| 1 | **Description 用第三人称** | ✅ `"Adds feature flag to Android module"` ❌ `"I can help add..."` |
| 2 | **Description 包含触发词** | `"Use when asked to 'add feature flag' or 'create toggle'"` |
| 3 | **Description ≤ 1024 字符** | 不含 XML tag |
| 4 | **Name 格式** | 小写 + 连字符，≤64 字符，如 `android-add-feature-flag` |
| 5 | **SKILL.md ≤ 500 行** | 超长内容拆到辅助文件 |
| 6 | **引用文件只能一层深** | SKILL.md → helper.md ✅，SKILL.md → a.md → b.md ❌ |
| 7 | **不硬编码时间/版本** | ❌ `"As of June 2026..."` |
| 8 | **MCP 工具用完全限定名** | ✅ `mcp__pm__wit_get_work_item` ❌ `pm/get_work_item` |
| 9 | **不假设工具存在** | 需要 pip 包就写 `pip install X` |
| 10 | **用 Unix 路径** | 即使在 Windows 上也用 `/` |

### 4.5 Skill 分类

该项目的 40+ skills 大致分为以下几类：

#### 脚手架类（Scaffolding）
自动化常见的代码生成任务：
- `android-add-feature-flag` — 加 feature flag
- `android-add-snapshot-test` — 加 Paparazzi snapshot test
- `android-add-telemetry-event` — 加遥测事件
- `android-add-use-case` — 加 UseCase
- `android-add-new-module` — 创建新 Gradle 模块

#### 质量检查类（Quality）
确保代码/PR 质量：
- `pr-risk-evaluate` — PR 风险评分（0-100，4 维度 + 14 条 guardrail）
- `review-pr` — PR 代码审查（规则 + swarm 并行）
- `pr-quality-check` — detekt + coverage 检查
- `code-coverage` — 覆盖率分析
- `skill-review` — 审查其他 skill 的质量

#### 流程自动化类（Workflow）
自动化日常流程：
- `create-pr` — PR 创建流程
- `address-pr-comments` — 处理 PR review 评论
- `shiproom-update` — 发版会议更新
- `triage-business-chat-bugs` — Bug 分诊

#### 调试类（Debug）
辅助问题排查：
- `pm-pipeline-analyzer` — Pipeline 失败分析
- `cud-kusto` — Kusto 查询辅助

---

## 5. Hook 质量门禁

### 5.1 Hook 是什么

Hook 是在 AI agent 执行工具调用 **前后** 自动触发的 shell 脚本。它们是 agentic guardrails 的核心实现机制。

### 5.2 配置方式

在 `.claude/settings.json` 中定义：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__pm__repo_create_pull_request",
        "hooks": [
          {
            "type": "command",
            "command": "bash \".claude/hooks/pr-review-gate.sh\"",
            "timeout": 10,
            "statusMessage": "PR-review gate: checking..."
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash \".claude/hooks/swift-lint-on-edit.sh\"",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

### 5.3 Hook 分类

| 时机 | Hook | 匹配 | 作用 |
|------|------|------|------|
| **创建 PR 前** | `pr-review-gate.sh` | `repo_create_pull_request` | 检查是否跑过 review-pr / pr-risk-evaluate |
| **推送代码前** | `pre-push-ci-gate.sh` | `Bash(git push)` | 检查本地 CI 是否通过 |
| **编辑文件后** | `swift-lint-on-edit.sh` | `Edit\|Write` | 自动对编辑的 Swift 文件跑 lint |
| **编辑文件后** | `claude-md-quality-check.sh` | `Edit\|Write` | 检查是否修改了 CLAUDE.md |
| **编辑文件后** | `cowork-swift-strict-on-edit.sh` | `Edit\|Write` | Cowork 项目严格检查 |

### 5.4 PR Review Gate 深度解析

这是最重要的 hook，让我们详细看看 `pr-review-gate.sh` 的逻辑：

```bash
# 1. 判断是否在创建 PR
case "$TOOL" in
  *repo_create_pull_request*) is_pr_create=true ;;
esac

# 2. 计算 branch diff，找到改了哪些文件
DIFF=$(git diff "$RANGE" --name-only)

# 3. 根据改动类型决定需要哪些 review pass
need pr-risk        && missing+=("pr-risk")          # 每个 PR 都需要
echo "$DIFF" | grep -qE 'ios|android' &&             # 代码 PR 需要
  need review-pr    && missing+=("review-pr")
echo "$DIFF" | grep -qiE '\.(swift|kt|png)$' &&      # UI PR 需要
  need visual       && missing+=("visual")
echo "$DIFF" | grep -qE '\.claude/skills/' &&         # Skill PR 需要
  need skill-review && missing+=("skill-review")

# 4. 如果有缺失的 pass，发出提醒
echo "{\"additionalContext\":\"请先运行 $LIST\"}"
```

关键机制：
- `mark-review.sh` 在每个 review pass 运行后写入 commit SHA 到 `.claude/.review-state/`
- `pr-review-gate.sh` 检查当前 HEAD 是否有对应的标记
- 如果有新 commit，标记过期，需要重新运行

### 5.5 Hook 的 Advisory vs. Blocking 模式

目前该项目的所有 hook 都是 **advisory**（建议模式）——输出 `additionalContext` 提醒 agent，但不阻止执行。

```bash
# Advisory（当前使用）
echo '{"additionalContext":"请先运行 review-pr"}'
exit 0

# Blocking（可选升级）
echo '{"hookSpecificOutput":{"permissionDecision":"deny","permissionDecisionReason":"..."}}'
```

设计文档中明确标注了升级路径，但出于稳定性考虑暂时保持 advisory。

---

## 6. Rules 条件加载规则

### 6.1 规则定义

Rules 是 **按文件路径条件加载** 的编码规范。只有当 agent 编辑匹配路径的文件时才会激活。

```markdown
<!-- .claude/rules/ios-development.md -->
<!-- paths: ios/**/*.swift -->

# iOS Development Rules

- 使用 Swift Concurrency (async/await)，不用 Combine
- 使用 @Observable，不用 ObservableObject
- Composable 函数前缀用 PascalCase
- ...
```

### 6.2 该项目的 Rules 体系

| Rule 文件 | 加载条件 | 内容 |
|-----------|---------|------|
| `ios-development.md` | 编辑 `ios/**/*.swift` | iOS 架构、CLI 命令、Swift 规范 |
| `ios-testing.md` | 编辑 `ios/**/Tests/**` | Snapshot test、单元测试模式 |
| `android-development.md` | 编辑 `android/**/*.kt` | Android 架构、Gradle、Kotlin 规范 |
| `android-testing.md` | 编辑 `android/**/test/**` | 单元测试、Compose test 模式 |
| `pr-format.md` | 始终加载 | PR 标题/描述格式 |

### 6.3 为什么不把所有规则放在一个文件里？

- **Context window 效率**：iOS 开发者不需要加载 Android 规则
- **维护性**：各平台团队独立维护自己的规则
- **精确性**：测试规则只在编辑测试文件时加载，避免干扰正常编码

---

## 7. Eval 评测体系

### 7.1 为什么需要 Eval

Agent 和 Skill 本质上是 **prompt engineering 的产物**。和代码一样，它们需要测试来保证质量。Eval 就是 agent/skill 的单元测试。

### 7.2 Eval 文件格式

```markdown
<!-- evals/eval-001-simple-pr.md -->

## Input
Evaluate PR #XXXXX

## Expected Behavior
- Should fetch PR metadata from PM平台
- Should compute a risk score 0-100
- Should identify platform as iOS
- Score should be in LOW RISK range (0-30)
- Output should include Dimension Breakdown table
```

### 7.3 G6 Guardrail — Eval 的强制执行

该项目的 `pr-risk-evaluate` skill 中定义了 G6 guardrail：

> **G6（agentic system）**: 当 PR 修改了 `.claude/skills/`、`.claude/agents/`、`.claude/commands/` 下的文件时触发。
>
> - 检查被修改 skill 目录下的 eval 文件数量
> - 要求 **≥ 50 个 eval** 且 **100% pass rate**
> - 不满足则 floor score 设为 70（HIGH RISK）

这形成了一个自我进化的闭环：
```
修改 skill → G6 要求 eval → 必须先写/更新 eval → eval 不过不让创建 PR
```

### 7.4 Eval 运行机制

```python
# .claude/evals/run-nova-eval.py
# 自动运行 Nova agent 的评测用例，验证输出质量
```

---

## 8. 多平台指令镜像

### 8.1 问题

该项目同时使用两个 AI 编码工具：
- **Claude Code** — 读取 `.claude/rules/*.md`
- **GitHub Copilot CLI** — 读取 `.github/instructions/*.instructions.md`

两个工具都不会自动读取对方的目录。

### 8.2 解决方案：双向镜像

```
.claude/rules/ios-development.md
    ↕ 手动保持同步
.github/instructions/ios-development.instructions.md
```

每个文件头部都有注释指向镜像文件：

```markdown
<!-- Mirror of .claude/rules/ios-development.md. When you change one, update the other. -->
```

在 `AGENTS.md` 中明确规定了同步规则：

> When you modify a file in either folder, update its counterpart in the other folder.
> Filenames pair 1:1. To add a new rule, create both files in the same change.

### 8.3 为什么不用一个共享文件？

- Claude Code 的 `paths:` 用 YAML 列表格式
- Copilot CLI 的 `applyTo:` 用逗号分隔字符串
- 两个工具都不支持 markdown 链接跨目录引用
- 接受重复，换取两个工具都能正常工作

---

## 9. 设计模式与最佳实践

### 模式 1：Maker-Checker（制造者-检查者）

```
Nova (maker) → 写代码 → Arbiter (checker) → 评审质量
```

两个 agent 形成分权制衡。Nova 写的 PR 必须经过 Arbiter 评审，就像人类工程师写代码要过 code review。

### 模式 2：Skill 组合（Composition）

```
Nova agent
  ├── 调用 android-add-feature-flag skill
  ├── 调用 android-add-snapshot-test skill
  ├── 自己写业务逻辑
  ├── 调用 pr-quality-check skill
  └── 调用 create-pr skill
```

Agent 不重复发明轮子——复用 skill 完成标准化子任务。

### 模式 3：Guard-then-Act（先守后行）

```
创建 PR (action)
    ↑
pr-review-gate.sh (guard)
    检查 review-pr ✓?
    检查 pr-risk-evaluate ✓?
    检查 design-visual-audit ✓?
```

Hook 在 action 前拦截，确保前置条件满足。

### 模式 4：从 Regression 学习（Incident-Driven Guardrails）

该项目的每条 guardrail 都关联真实的线上事故：

```
G6  (agentic system)   ← 防止未经测试的 skill 变更
G12 (iPad coverage)    ← 来自 commit xxxxxxx 的 iPad regression
G13 (flag collapse)    ← 来自 #XXXXX 的 feature flag bug
G14 (custom toolbar)   ← 来自 #XXXXX/#XXXXX 的 iOS 26 glass bug
```

这不是凭空设计的规则——每一条都是从血的教训中提炼的。

### 模式 5：Intent-Driven Workflow Loading（意图驱动加载）

```markdown
## Intent Triage

| Intent         | Workflow to Load                |
|----------------|---------------------------------|
| 事件分诊       | workflows/2-incident-triage.md  |
| 发版           | workflows/3-release-cut.md      |
| 周报           | workflows/4-weekly-report.md    |
```

不预加载所有 workflow——匹配意图后才加载，节省 context window。

### 模式 6：Swarm Review（群体审查）

```
review-pr skill
  ├── 规则审查（review-rules-*.md）
  ├── Swarm 并行审查
  │   ├── Breaker（找 bug）
  │   ├── Exploiter（找安全问题）
  │   └── Inspector（找架构问题）
  └── 合并去重 → 发 PM平台 评论
```

单一 reviewer 的盲区通过多角色并行审查来弥补。

---

## 10. 从 0 到 1 创建你的第一个 Skill

### 示例：创建一个 `android-add-string-resource` skill

#### Step 1：创建目录结构

```bash
mkdir -p .claude/skills/android-add-string-resource/evals
```

#### Step 2：编写 SKILL.md

```markdown
---
name: android-add-string-resource
description: Adds a new string resource to the Android project with
             proper localization setup. Use when asked to "add a string",
             "create string resource", or "add localized text".
---

# Add Android String Resource

Adds a new string entry to the project string resource system, including
the base `strings.xml` and the `StringResource` Kotlin constant.

## Prerequisites

- Working in the `project/android/` directory
- Module name where the string is needed

## Step-by-Step Instructions

### 1. Identify the Target Module
Determine which module needs the string (e.g., `ui`, `chat`, `business-chat`).

### 2. Add to strings.xml
Add the string entry to `<module>/src/main/res/values/strings.xml`:
```xml
<string name="app_<feature>_<description>">English text</string>
```

### 3. Add StringResource Constant
Add a constant to the module's `StringResource` provider:
```kotlin
val FeatureDescription = StringResource("app_<feature>_<description>")
```

### 4. Verify
Run: `project-cli gradle :<module>:assembleDebug`
```

#### Step 3：添加评测用例

```markdown
<!-- evals/eval-001-basic-add.md -->
## Input
Add a string "Cancel" for the chat input module.

## Expected Behavior
- Creates entry in chat module's strings.xml
- Adds StringResource constant
- Uses naming convention app_chat_cancel
```

#### Step 4：测试你的 skill

```bash
# 在 Claude Code 中
/android-add-string-resource Add a "Send" button label to the ui module
```

---

## 11. 从 0 到 1 创建你的第一个 Agent

### 示例：创建一个 `Reviewer` agent

#### Step 1：创建文件

```bash
touch .claude/agents/Reviewer.agent.md
mkdir -p .claude/agents/Reviewer
```

#### Step 2：编写 agent 定义

```markdown
---
name: Reviewer
description: "Reviews Android Compose code for performance anti-patterns.
              Use when asked to 'check compose performance' or
              'review recomposition'."
tools:
  [
    "read",
    "search",
    "mcp__pm__repo_pull_request_thread_write",
  ]
---

# Reviewer - Compose Performance Agent

## Overview
Reviews Jetpack Compose code for common performance issues:
recomposition, unstable types, missing keys, etc.

## Trigger Phrases
| "check compose performance"; "review recomposition"; "find unstable composables" |

## Workflow

### Step 1: Identify Compose Files
Search for `@Composable` in changed files.

### Step 2: Check Anti-Patterns
For each file, check:
- Missing `@Stable` / `@Immutable` on data classes passed to composables
- Lambda allocations inside composition
- Missing `key()` in `LazyColumn` items
- Unnecessary recomposition triggers

### Step 3: Report Findings
Post findings as PM平台 PR comments with severity and fix suggestions.

## Rules Reference
@file:.claude/agents/Reviewer/compose-rules.md
```

#### Step 3：添加知识库

```markdown
<!-- .claude/agents/Reviewer/compose-rules.md -->
# Compose Performance Rules

## Rule 1: Unstable Parameters
Data classes passed to @Composable must be @Immutable or @Stable.
...

## Rule 2: Lambda Allocation
Avoid creating lambdas inside composition scope.
...
```

---

## 12. 反模式与踩坑记录

### ❌ 反模式 1：Agent 文件过大

```markdown
<!-- 40KB 的 agent 文件 = 每次对话都消耗大量 context -->
```

**解决**：拆分到子目录，按需加载。Nova 的 40KB workflow 是一个已知问题。

### ❌ 反模式 2：Skill 之间链式引用

```
SKILL.md → a.md → b.md → c.md    ← Claude 不会跟超过一层
```

**解决**：所有引用文件只能从 SKILL.md 直接引用（一层深度）。

### ❌ 反模式 3：硬编码版本号

```markdown
❌ "Use Compose 1.5.4"
✅ "Use the version defined in gradle/libs.versions.toml"
```

### ❌ 反模式 4：忘记双平台同步

修改了 `.claude/rules/ios-development.md` 但忘了更新 `.github/instructions/ios-development.instructions.md`。

**解决**：在 AGENTS.md 中明确规定，每个文件头部有镜像提示。

### ❌ 反模式 5：Guardrail 没关联事故

```markdown
❌ G15: Check for magic numbers
✅ G13: New flag collapses a conditional branch — grounded in #XXXXX
```

没有事故支撑的 guardrail 缺乏说服力，容易被忽略。

### ❌ 反模式 6：重命名 Paparazzi 测试方法但不删旧快照

```
# CLAUDE.md 中明确警告：
# DO NOT rename Paparazzi test methods without deleting the old snapshot image
```

---

## 13. 附录：该项目现有 Agent/Skill 速查表

### Agents

| Agent | 角色 | 知识库 | MCP 集成 |
|-------|------|--------|----------|
| **Nova** | 自主编码 | coding-workflow.md (40KB) | PM平台 |
| **Arbiter** | PR 质量评审 | rubric-ios/android.md, scorecard | PM平台 |
| **DRI** | 值班管理 | 5 个 workflow, config.json | IM, Kusto, 即时通讯 |
| **Atlas** | 架构规划 | iPad checklist, extension points | — |
| **Jing** | — | — | — |
| **Scorpion** | — | — | — |
| **Touchstone** | — | — | — |
| **Debuggy** | iOS 调试 | — | Appium, Jupyter |
| **Hao** | — | orchestration-workflow.md | — |

### Skills（按类别）

| 类别 | Skills |
|------|--------|
| **脚手架** | android-add-feature-flag, android-add-snapshot-test, android-add-telemetry-event, android-add-use-case, android-add-new-module, android-add-provider, android-add-chat-setting, android-add-debug-setting, android-add-feature-toggle |
| **质量** | pr-risk-evaluate, review-pr, pr-quality-check, code-coverage, skill-review, review-dev-spec, review-partner-pr, project-pr-ultrareview |
| **流程** | create-pr, address-pr-comments, shiproom-update, triage-business-chat-bugs, pages-strings |
| **构建/运行** | android-build-and-run, android-build-and-run-on-mac, android-build-and-run-on-windows, android-start-emulator-on-windows, setup |
| **测试** | automated-ui-test, android-appium-setup, copilot-notebooks-android-appium, appium-setup, android-pre-push-patterns |
| **调试/分析** | pm-pipeline-analyzer, cud-kusto, exp-scorecard-analyzer |
| **审查** | accessibility, design-visual-audit (implied) |
| **Cowork** | cowork-architecture-review, cowork-flaky-test-stabilizer, cowork-pr-review, cowork-snapshot-update, cowork-swift-concurrency, android-cowork-project |
| **验证** | pacman-request-validation |

### Hooks

| Hook | 时机 | 作用 |
|------|------|------|
| `pr-review-gate.sh` | 创建 PR 前 | 检查 review pass |
| `pre-push-ci-gate.sh` | git push 前 | 检查 CI marker |
| `swift-lint-on-edit.sh` | 编辑后 | Swift lint |
| `cowork-swift-strict-on-edit.sh` | 编辑后 | Cowork 严格 lint |
| `claude-md-quality-check.sh` | 编辑后 | CLAUDE.md 质量 |
| `mark-review.sh` | review 后 | 记录 review 状态 |
| `write-green-ci-marker.sh` | CI 通过后 | 写 CI 标记 |

---

*本文档基于某大型移动端项目代码库 2026-06 版本整理。Agent/Skill 体系持续演进中。*

# Subagent-Driven Development — Mode A 子 Agent 编排

> Phase 3 的 Mode A 执行策略。Controller 不写代码，只调度 subagent 并验证。

## 适用条件

平台支持 Agent tool（如 Claude Code）时使用 Mode A。否则使用 Mode B（solo 执行，直接写代码）。

## 核心原则

> Fresh subagent per task + 独立 spec review = 高质量、快迭代。

每个 subagent 拿到干净 context，只包含当前任务信息。防止跨任务的上下文污染。

### Prompt Cache Reuse（子 Agent 缓存复用）

> 借鉴 Claude Code：fork 出的子 agent 继承父级的渲染后 prompt 字节，不重新生成。

**原理**：Controller 和所有 Worker 共享相同的静态层（rules + engineering-structure），
只在动态层注入不同的 task-specific context。这使得 API 前缀缓存在 Controller 和
Worker 之间共享，减少重复 token 消耗。

**实施方式**：

```
Controller prompt:
  [Static Layer] ← rules/coding-standards.md + engineering-structure.md（缓存命中）
  [Dynamic Layer] ← 当前 plan 概览 + 调度逻辑

Worker (Implementer) prompt:
  [Static Layer] ← 同一份 rules（前缀缓存复用 ✓）
  [Dynamic Layer] ← 单任务 description + criteria + commands

Worker (Reviewer) prompt:
  [Static Layer] ← 同一份 rules（前缀缓存复用 ✓）
  [Dynamic Layer] ← task desc + implementer report + diff
```

**规则**：
- Worker prompt 的静态层必须与 Controller 完全相同（字节级一致）
- 动态层放在 prompt 末尾，不插入静态层中间
- 不要为 Worker 生成独立的完整 rules 副本（破坏缓存）

## 单任务工作流

```
┌─────────────────────────────────────────────────────┐
│  Controller (你)                                     │
│                                                      │
│  1. 从 plan.json 读取 task                           │
│  2. 根据 task.type 选择 prompt 模板                  │
│  3. 填充所有占位符（task 完整信息）                    │
│  4. 派发 Implementer subagent ──────────────┐       │
│                                               │       │
│  5. 读取 implementer 完整报告 ◄──────────────┘       │
│                                                      │
│  6. 派发 Spec Reviewer subagent ────────────┐       │
│     (提供: task desc + criteria +             │       │
│      implementer 报告 + 变更文件)             │       │
│                                               │       │
│  7. 读取 reviewer 裁定 ◄────────────────────┘       │
│     ├── ✅ Pass → 进入验证（Step 6 of AGENTS.md）   │
│     └── ❌ Fail → 进入修复循环（step 8）             │
│                                                      │
│  8. 带修复列表再次派发 Implementer ──────────┐       │
│  9. 再次派发 Spec Reviewer ◄─────────────────┘       │
│     └── 循环直到 ✅                                  │
└─────────────────────────────────────────────────────┘
```

## 模板选择

| Task Type | 模板 |
|-----------|------|
| `backend` / `integration` | Backend Implementer Prompt |
| `ui` | UI Implementer Prompt |

Spec Review 统一使用 Spec Reviewer Prompt。

## 派发规则

### 填充 Prompt

1. **完整提供** — 替换所有占位符为 plan.json 的实际值
2. **不让 subagent 读 plan.json** — 所有信息内联到 prompt
3. **包含所有命令** — verificationCommands + qualityCommands 必须写入 prompt

### Implementer Prompt 要素

```markdown
# Task: {TASK_ID} - {TASK_TITLE}

## Description
{完整 task.description}

## Acceptance Criteria
{逐条列出 acceptanceCriteria}

## Test Requirements
- Test file: {testFile}
- Test description: {testDescription}

## Files to Modify
{filesToModify 列表}

## Context
{task.context 完整内容}

## Verification Commands
{verificationCommands 列表}

## Quality Commands
{qualityCommands 列表}

## Rules
- 使用 TDD: 先写测试（红灯）→ 实现（绿灯）→ 重构
- 跑完所有验证命令
- 输出修改的文件列表和验证结果
```

### Spec Reviewer Prompt 要素

```markdown
# Spec Review: {TASK_ID}

## Task Description
{task.description}

## Acceptance Criteria
{逐条列出}

## Implementer Report
{implementer 返回的完整报告}

## Changed Files
{git diff 或 updatedFiles}

## Your Job
逐条核对 acceptanceCriteria：
- ✅ 已满足（引用具体代码行）
- ❌ 未满足（说明缺什么）

输出: PASS 或 FAIL + 具体问题列表
```

## 修复循环

Spec Reviewer 报告 ❌ 时：

1. 提取具体问题分类：
   - `[MISSING]` — 需求未实现
   - `[EXTRA]` — 多余功能
   - `[WRONG]` — 需求理解错误
2. 再次派发 Implementer，附带：
   - 原始 task 描述
   - 具体修复列表
   - 指令："只修这些问题，不改别的"
3. 修完后再次派发 Spec Reviewer
4. 循环直到 ✅

## 禁止行为

1. **Controller 不写代码** — 你是调度者，subagent 执行
2. **不跳过 spec review** — 每个 task 必须经过独立审查
3. **不让 subagent 读 plan.json** — 信息内联到 prompt
4. **不并行派发多个 implementer** — 一次一个 task
5. **不带着 ❌ 继续** — 修复循环必须完成
6. **不信任 subagent 的验证声明** — 你自己跑验证命令（Step 6）
7. **不给部分 task 描述** — 完整文本 + 所有 criteria + 所有命令

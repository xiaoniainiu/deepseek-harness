---
name: explore-mode
description: |-
  探索模式：非结构化思考伙伴。不是 Phase，不是 Workflow，是一种姿态。
  在问题空间中自由探索，不急于结论和方案。
  来源：OpenSpec /opsx:explore — "a stance, not a workflow"。
version: 1.0.0
appendix_limit: 50
---

# 探索模式（Explore Mode）

> 不是 Phase 0.5（那有结构化追问），不是 brainstorming（那有分步流程）。
> 这是一种**姿态**——在问题空间中自由游走，追踪线索，随时转向。
> 来源：OpenSpec /opsx:explore — "A stance, not a workflow. No fixed steps, no required outputs."

## 定位

| 对比 | brainstorming | idea-exploration (0.5) | **explore-mode** |
|------|-------------|---------------------|---------------|
| 结构 | 5 步流程 | 6 追问 | **无固定步骤** |
| 产出 | requirement.md | exploration.md | **无强制产出** |
| 目的 | 澄清需求 | 质疑"该不该做" | **理解问题空间** |
| 何时用 | Phase 1 常规 | L1/L2 路由触发 | **任何时候，用户说"我想了解一下"** |

## 触发条件

- 用户说"探索一下"/"了解一下"/"想想"/"看看怎么回事"
- 用户给了一个模糊方向但不是要求（"我们的盘点流程感觉不太对"）
- AI 判断用户还在问题空间，不在解方案空间
- 用户显式说 `/explore`

**不触发**：用户已有明确需求（"给盘点单加一个字段"）→ 走 brainstorming。

## 核心原则

### 1. 绝不实现

可以读代码、读文档、画图、搜索，但**不写代码、不写 PRD、不创建文件**。

### 2. 自由跟线索

没有预定义步骤。用户说什么就跟什么，发现有趣的就追下去。

允许的动作：
- 读代码/配置/文档
- 画 ASCII 图（数据流、时序、架构）
- 搜索代码库（grep/find）
- 问用户问题（一次一个）
- 提出假设并验证
- 对比多种可能的方向

### 3. 可视化优先

> OpenSpec: "Visualizes liberally — ASCII diagrams encouraged."

用文字说不清的，画图。数据流、调用链、状态机、时序图——能画就画。

### 4. 提供但不强加结论

发现了有价值的东西时：
- **说出来**："我注意到 {X}，这可能意味着 {Y}"
- **提议但不执行**："要不要把这个发现记到一个 proposal 里？"
- **永远问用户**：不自动创建文件或进入下一个 Phase

### 5. 带着立场探索

虽然是自由探索，但反谄媚规则仍然生效（`rules/anti-sycophancy.md`）：
- 发现问题直说
- 有判断就给判断
- 不说"有趣的方向"

## 退出条件

探索模式自然退出的信号：

| 信号 | 退出到哪 |
|------|---------|
| "好，我们做这个" | → brainstorming Step 0（需求分级） |
| "写个需求吧" | → Phase 1 Mode A/B |
| "先不做了" | → 结束（可选：记录探索笔记到 `changes/<topic>/exploration-notes.md`） |
| "继续看看别的" | → 继续探索（不退出） |

## 探索笔记（可选产出）

如果探索产出了有价值的发现，**用户同意后**可以记录：

```markdown
# 探索笔记：{主题}

**日期**：{date}
**触发**：{用户原话}

## 发现
- {发现 1}
- {发现 2}

## 假设
- {假设 1}：{依据}
- {假设 2}：{依据}

## 可视化
{ASCII 图}

## 下一步建议
- {如果要做：建议从 brainstorming 开始}
- {如果不做：记录原因}
```

写入 `changes/<topic>/exploration-notes.md`。不进入 Harness 流程，不创建 phase-state.json。

## 与 Harness 流程的关系

探索模式**在 Harness 之外**运行：
- 不创建 `phase-state.json`
- 不受 Phase 状态机约束
- 不消耗评审轮次
- 探索笔记是"原材料"，不是"需求文档"

只有用户明确说"做"的时候，才从探索进入 Harness Phase 0.5 或 Phase 1。

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

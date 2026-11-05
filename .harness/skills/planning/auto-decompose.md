---
name: auto-decompose
description: |-
  检测多部分请求并自动拆解为原子任务。Phase 2.5 使用。
  触发词：拆解、decompose、多步骤、复杂需求
version: 1.0.0
appendix_limit: 50
---

# 自动任务拆解

## 触发条件

- Phase 2.5 入口（development-process.md 定义的触发条件：5+ 文件 / 3+ 独立路径 / 双端触发）
- Coordinator 在 Phase 1 检测到复杂信号

## Step 1: 信号检测

检查输入是否匹配以下模式（任一匹配则需要拆解）：

### 结构信号（来自 ai-collab decomposer.ts）

| 模式 | 正则 | 示例 |
|------|------|------|
| 编号列表 | `[1-9]\d*[)）.、]` | "1）增加校验 2）修改表单" |
| 项目符号 | `[-•]\s+` | "- 后端接口 - 前端页面" |
| 中文序词 | `首先\|其次\|再次\|最后` | "首先改表，其次改接口" |
| 分号分隔 | `[;；]` | "增加字段；修改校验" |

### 语义信号

| 信号 | 拆解方向 |
|------|---------|
| 前后端双端改动 | 按端拆分（backend / ui） |
| 数据依赖链（A→B→C） | 按依赖层级拆分 |
| 多个独立业务规则 | 按规则拆分 |
| 涉及多仓库 | 按仓库拆分 |
| 涉及多个本体实体 | 按实体边界拆分 |

### 范围信号

| 条件 | 判定 |
|------|------|
| 涉及 5+ 文件 | 必须拆解 |
| 涉及 3+ 独立业务路径 | 必须拆解 |
| 前后端双端触发 | 必须拆解 |
| 跨仓库 | 必须拆解 |

## Step 2: 上下文提取

从原始需求中提取共享上下文：
- 需求背景（所有子任务共享）
- 技术约束（所有子任务共享）
- 每个子任务 = 共享上下文 + 独立描述

## Step 3: 子任务路由

每个子任务标注：

```yaml
- id: T-001
  title: "<简明描述>"
  why: "<对齐 requirement.md 的哪条 AC 或哪个 Feature>"  # P9 六要素 WHY
  type: backend | ui | integration | review | planning
  complexity: simple | moderate | complex
  budget:                      # P9 六要素 HOW MUCH
    files: <number>            # 预估涉及文件数
    lines: "<~NNN>"            # 预估变更行数
    note: "<预算说明>"          # 如"新建类 ~100 行 + Provider 注入 ~10 行"
  dependsOn: []              # 前置子任务 ID
  filesToModify: []           # 允许变更的文件（Surgical Changes）
  verificationCommands: []    # 验证命令
  acceptanceCriteria: []      # 验收标准
  dont: []                   # P9 六要素 DON'T（不要做什么，防 Scope Creep）
```

## Step 4: 依赖排序

```
将子任务按 dependsOn 关系排列为 DAG
无依赖的任务可并行执行
有依赖的任务按拓扑排序
```

## Step 5: 输出 plan.json

```json
{
  "requirement": "<需求描述>",
  "tasks": [
    {
      "id": "T-001",
      "title": "...",
      "why": "对齐 AC-08：工作日 10:00 到期当日借用人收到提醒卡片",
      "type": "backend",
      "priority": 1,
      "budget": {"files": 3, "lines": "~130", "complexity": "medium"},
      "dependsOn": [],
      "filesToModify": ["src/..."],
      "verificationCommands": ["mvn compile"],
      "acceptanceCriteria": ["..."],
      "dont": ["不改现有 CheckNotifyTask 逻辑"]
    }
  ]
}
```

## Step 6: 人工确认

拆解结果必须在进入 Phase 3 前（[H3] 前）展示给用户确认：
- 展示任务列表和依赖关系
- 标注每个任务的估计复杂度
- 等待用户确认或调整

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

# Plan Revision Review — 计划修订评审

> **何时使用**：执行实施计划期间，发现某个任务的前置假设已失效（环境变了、依赖不可用、上游决策变了）。**强制触发本流程，禁止静默改 plan。**
> **来源教训**：本次 V2 路径前缀迁移 Task 1（cherry-pick `cc31beb4c` 到 master）发现 staging 与 master 已发散 1 万 commit，原方案不可执行 → 应该按本流程暂停 + 评估替代 + 用户审批，而不是直接在脑内改成"跳过 Task 1，从 staging 打 tag"。

---

## 核心原则

```
plan 假设失效  →  暂停  →  写 revision  →  ≥ 2 alternatives + 用户审批  →  更新 frontmatter  →  继续
```

**禁止**：
- 静默把"原方案不可执行"改成"换个方案"，不更新 plan
- 单一 alternative（"只能这样了"——必须强迫自己想出第 2 种）
- 不写 invalidated assumptions（让下次 review 失去溯源）
- 用户口头同意但不更新 plan frontmatter（plan 与执行漂移）

---

## Plan.md frontmatter schema（与 [writing-plans.md](writing-plans.md) 协议）

每份 plan 文件**必须**有 yaml frontmatter，记录假设和修订历史：

```yaml
---
plan_id: <kebab-case>
status: pending | in_progress | done | revised | abandoned
created: 2026-05-26
last_revised: 2026-05-26

assumptions:
  - id: A1
    text: "asset-eam-web master 含 InventoryV2Embed.tsx，可 cherry-pick cc31beb4c"
    validated: false
    validated_by: null
    invalidated_at: 2026-05-26
    invalidated_reason: "实测 master 完全没有 V2 嵌入功能链，与 staging 发散 1 万 commit"

  - id: A2
    text: "test 环境从 master 发布"
    validated: false
    invalidated_at: 2026-05-26
    invalidated_reason: "实测最近 9/10 个 test-mi-* tag 都从 staging 打"

revisions:
  - revision_id: R1
    triggered_by_task: T1                       # plan 中的 Task 编号
    discovered_at: 2026-05-26
    reason: "cherry-pick 失败：modify/delete 冲突，InventoryV2Embed.tsx 在 master 不存在"
    invalidated_assumptions: [A1, A2]
    alternatives_evaluated:                      # 必须 ≥ 2
      - name: "A: 跳过 master，从 staging 打 test tag"
        pros: ["staging 已含 cc31beb4c", "test 历史就从 staging 发", "本次范围仅 test 验证"]
        cons: ["prod 同步成债务，需独立工程"]
        risk_score: low
      - name: "B: 强行把 staging 整体合入 master"
        pros: ["一次性闭环", "prod 端无遗留"]
        cons: ["1 万 commit 远超本次范围", "回归风险极高", "破坏原 master 节奏"]
        risk_score: high
      - name: "C: 单独把 8 个 V2 嵌入 commit cherry-pick 到 master"
        pros: ["范围可控"]
        cons: ["8 commit 间有依赖链，需逐个 resolve", "工时不可控"]
        risk_score: medium
    chosen: "A"
    chosen_rationale: "本次仅 test 验证；prod 同步作为独立后续债务；A 方案 1 步搞定"
    approved_by: <user>
    approved_at: 2026-05-26
---
```

### Task schema 扩展

每个 Task 块加 2 个字段：

```yaml
- id: T1
  name: "asset-eam-web：iframe 修复合入 master"
  preconditions: [A1, A2]                        # 该任务依赖的假设 ID
  on_unfeasible: pause_and_revise                # 不可行时的强制行为
```

`on_unfeasible` 取值：
- `pause_and_revise`（默认）— 触发本流程
- `skip_with_note` — 仅在 plan 已明确允许时，且必须在 revisions 中登记
- 不允许 `auto_continue`（任何"自动跳过"都视为静默修订）

---

## 强制流程（5 步）

### 1. Pause（停下）

**信号**：任务执行中遇到以下任一情况：
- 前置命令失败且失败原因与计划假设矛盾（如 cherry-pick 报 modify/delete 冲突）
- 实测发现现状与计划描述不一致（commit 数发散、文件不存在、API 已废弃等）
- 上游决策已变（用户口头说"现在不要做 X 了"）
- 环境/工具不可用（依赖服务下线、token 过期等）

**触发后立刻**：
- 不再执行后续 Task
- 不在脑内改方案
- 进入 step 2

### 2. Identify invalidated assumptions（找到失效假设）

逐条对照 plan frontmatter 的 `assumptions[]`：
- 哪几条被本次发现否定了？
- 标记 `invalidated_at` 和 `invalidated_reason`
- 失效假设 ≥ 1 才有资格进入 revision；如果只是"任务 step 写错了"，走普通 plan 修订即可

### 3. Generate ≥ 2 alternatives（强制双备选）

对失效假设，必须想出 **2 种以上**替代方案。

**为什么强制 2 种**：单方案常常是"懒惰最近邻"——遇到挫折就选最像原方案的微调，但真正的好方案往往需要换个角度（如本次"跳过 master 走 staging"就比"硬合 1 万 commit"好）。

每个 alternative 必须填：
- `name`：一句话方案描述
- `pros`：≥ 2 条
- `cons`：≥ 2 条（如果想不出 cons，说明你没认真想）
- `risk_score`：low / medium / high

### 4. User approval（用户审批）

**必须的**：把 revision yaml 块完整展示给用户，问：
- 失效假设理解对吗？
- 推荐 alternative 是 ?
- 同意吗？

**禁止**：
- 自己判定"显然 A 更好"就执行
- 用户只口头说"嗯可以"但不在 plan 落地

用户回复 `approved_by` 字段记下来，落到 frontmatter。

### 5. Update plan frontmatter & resume（更新 plan 并继续）

- 把 `revisions[]` append 到 frontmatter
- 把 `assumptions[].invalidated_*` 字段填上
- 顶层 `status: revised`，`last_revised` 更新
- 受影响的 Task 块加备注：`> 本任务受 R1 修订影响，方案改为 ...`
- commit plan 文件（`docs(plan): R1 修订 — <reason>`）
- 继续执行后续任务

---

## 执行 Skill 接口（[executing-plans](../shipping/) / [subagent-driven-development](../coding/)）

执行类 skill 在每个 Task 完成 / 失败时**必须**判断：

```
if task_failed_with_assumption_violation:
    invoke plan-revision-review
    block until user approval
    
if task_succeeded:
    mark assumption.validated = true
    mark assumption.validated_by = task_id
```

简单 task 不会触发；只有"失败且失败信号涉及前置假设"才触发。

---

## 反模式

| 反模式 | 真实信号 | 正确做法 |
|---|---|---|
| 静默改方案 | "Task 1 失败了，我直接跳到 Task 2 用 staging" | Pause，写 R1，列 ≥ 2 alternatives |
| 单一 alternative | "只能从 staging 打了" | 强迫自己写第 2、3 种（哪怕明显差） |
| 用户口头同意 | "用户说嗯可以" | 写到 frontmatter `approved_by` |
| 不登记失效假设 | revision 只写"换方案" | 必须列出 invalidated_assumptions[] |
| 假设字段缺失 | plan 没有 assumptions 块 | writing-plans 必须强制 frontmatter 包含该字段 |
| 任务失败但不触发 | "这个失败跟 plan 没关系吧" | 失败原因 ∩ plan 描述 ≠ ∅ → 必须触发 |

---

## 关联 Skill / 规则

- [writing-plans.md](writing-plans.md) — 写 plan 时必须包含 assumptions / preconditions schema
- [auto-decompose.md](auto-decompose.md) — 拆分原子任务时为每个 Task 标注 preconditions
- 执行类 skill（`shipping/`）— 任务失败时调用本 skill
- [governance/](../../governance/) — revision 通过率、平均修订轮次进入 governance 指标
- [risk-score.py](../../tools/governance/risk-score.py)（G7）— alternative 的 risk_score 字段提供输入
- [c7-multi-axis-archival.md](../../rules/c7-multi-axis-archival.md)（G6）— revision 沉淀为 case 时记录"假设失效"教训轴

---

## 模板（直接 copy 到 plan 文件顶部）

```yaml
---
plan_id: <kebab>
status: pending
created: 2026-05-26

assumptions:
  - id: A1
    text: "<把每个隐性前置假设显式写出来>"
    validated: false
  - id: A2
    text: "..."
    validated: false

revisions: []  # 触发时 append
---
```

每个 Task：

```yaml
- id: T1
  name: "..."
  preconditions: [A1, A2]
  on_unfeasible: pause_and_revise
```

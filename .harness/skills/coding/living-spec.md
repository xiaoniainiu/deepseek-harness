---
name: living-spec
description: |-
  活文档协议：requirement.md 不是写完就锁的文档，实现中的发现应该回写。
  由 Checkpoint 反思触发，确保需求和实现始终对齐。
  来源：OpenSpec fluid-not-rigid + brownfield-first。
version: 1.0.0
appendix_limit: 50
---

# 活文档协议

> requirement.md 不是石碑，是活文档。编码中发现的真相比编码前的假设更重要。
> 来源：OpenSpec（"fluid not rigid — learn as you build, refine as you go"）。

## 触发条件

- Checkpoint 反思（`deep-work-protocol.md` §2）发现偏差且偏差不可接受
- 实现中发现 requirement.md 有歧义/遗漏/矛盾
- 技术约束使原方案不可行，需要调整

## Delta 操作类型（来源：OpenSpec delta spec）

每次更新使用结构化的 Delta 操作，而非自由文本：

| 操作 | 含义 | 合并顺序 |
|------|------|---------|
| **RENAMED** | AC 改名（语义不变） | 1（最先） |
| **REMOVED** | AC 删除 | 2 |
| **MODIFIED** | AC 内容变更（就地替换） | 3 |
| **ADDED** | 新增 AC | 4（最后追加） |

**冲突规则**：同一个 AC 不能同时出现在 MODIFIED 和 REMOVED 中。

## 更新协议

每次更新记录为一个 Living Spec Update，使用 Delta 格式：

```markdown
## Living Spec Update #{N}

**触发时机**: Phase 3 / Task {T-ID} / Layer {层名}
**Delta 操作**: {ADDED / MODIFIED / REMOVED / RENAMED}
**目标 AC**: AC-{NN}: {名称}
**原文**: "{原始描述}"
**更新后**: "{新描述}"（REMOVED 时为空）
**影响范围**: {受影响的 Task ID 列表}
**决策**: {自行更新 / 待用户确认}
**理由**: {为什么需要更新}
```

### Delta 操作示例

**MODIFIED**（最常见 — 发现需求遗漏，补充场景）：
```markdown
## Living Spec Update #1
**Delta**: MODIFIED
**目标 AC**: AC-14: 驾照校验
**原文**: "提交借用申请时校验驾照"
**更新后**: "提交借用申请（含首借和续借）时校验驾照"
**理由**: 实现中发现续借 submit 也需要校验，PRD "提交时"未区分首借/续借
```

**ADDED**（实现中发现新需求）：
```markdown
## Living Spec Update #2
**Delta**: ADDED
**目标 AC**: AC-25: VIN 空值跳过
**更新后**: "系统 MUST 在 VIN 为空时跳过唯一性校验"
**理由**: 非车辆类资产无 VIN，原 AC 未覆盖
```

**REMOVED**（实现中发现原需求不可行）：
```markdown
## Living Spec Update #3
**Delta**: REMOVED
**目标 AC**: AC-18: 实时同步 SAP
**原文**: "变更后 5s 内同步 SAP"
**理由**: SAP 接口 SLA 最快 30s，实时同步不可行，改为异步
```

## 自行更新 vs 需确认

| 情况 | 处理 | 示例 |
|------|------|------|
| 措辞澄清（不改语义） | 自行更新，Checkpoint 中记录 | "提交"改为"提交审批" |
| 补充遗漏细节（不改行为） | 自行更新，Checkpoint 中记录 | 补充"空值时显示 —" |
| 功能增减 | **必须用户确认** | 原说 3 种状态，实际需要 4 种 |
| 行为变更 | **必须用户确认** | 原说同步执行，改为异步 |
| 涉及其他未完成 Task | **必须用户确认** | 改了 DTO 字段名影响下游 Task |
| 技术方案不可行 | **必须用户确认** | 数据库不支持原方案的查询方式 |

## 更新格式

在 requirement.md 对应位置追加标记，**保留原文**：

```markdown
原始描述文字 [Updated in Phase 3, T-001: 发现 XX 约束，改为 YY]
```

## 更新汇总

所有 Living Spec Updates 汇总到 `changes/<req>/living-spec-log.md`：

```markdown
# Living Spec Log

| # | 时机 | Delta | 目标 AC | 原文摘要 | 更新摘要 | 决策 |
|---|------|-------|--------|---------|---------|------|
| 1 | T-001/Data Access | MODIFIED | AC-03 | "提交" | "提交审批" | 已更新 |
| 2 | T-002/Business | ADDED | AC-25 | — | "VIN空值跳过" | 已更新 |
| 3 | T-003/Business | MODIFIED | AC-14 | "提交时校验" | "首借+续借时校验" | ⏳ 等用户 |
```

## Phase 10 Spec Sync（归档时同步回 PRD）

Phase 10 归档时，Delta Log 中的变更**必须**同步回 Atom PRD：

1. **合并顺序**：RENAMED → REMOVED → MODIFIED → ADDED（与 OpenSpec 一致）
2. **合并前验证**：用 `spec-validation.md` 检查合并结果仍通过 ERROR 级别
3. **冲突检测**：同一 AC 同时 MODIFIED + REMOVED → 阻塞，人工裁决
4. **写入位置**：更新 `docs/systems/eam/domain/Atom PRD/{模块}/` 中对应文件
5. **标记**：在 PRD 文件中追加 `<!-- synced from changes/<req>/living-spec-log.md, {date} -->`

**跳过条件**：如果 Living Spec Log 为空（无 delta），跳过同步。

## 链路关系

```
Checkpoint 反思 → 发现偏差 → 触发 Living Spec
                                    │
                                    ├→ 自行更新 → 修改 requirement.md + 记录
                                    │
                                    └→ 需确认 → 暂停当前 Task → 等用户
                                                                    │
                                                用户确认 → 修改 requirement.md → 继续
                                                用户拒绝 → 保持原方案 → 调整实现
```

## 与意图验证的关系

Phase 3 结束时的意图验证（`deep-work-protocol.md` §3）会检查：
- requirement.md 的最终版本（含所有 Living Spec Updates）与实现是否一致
- 所有"待确认"的 Updates 是否已得到用户回答
- 未确认的 Update → 意图验证 FAIL → 不能进入 Phase 4

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

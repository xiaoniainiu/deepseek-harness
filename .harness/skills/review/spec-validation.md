---
name: spec-validation
description: |-
  需求规格程序化验证：检查 requirement.md 中的 AC 是否符合 spec-format 标准。
  Phase 2 评审时自动执行，输出验证报告。
  来源：OpenSpec Zod validator（SHALL/MUST + Scenario 覆盖 + 冲突检测）。
version: 1.0.0
appendix_limit: 50
---

# 需求规格验证

> 人工评审 + 程序化验证双保险。人看逻辑，程序看格式。
> 来源：OpenSpec validator（Zod-based structural validation）。

## 触发条件

- Phase 2 需求评审开始时**自动执行**
- Phase 1 输出 requirement.md 后可手动执行
- Living Spec 更新后重新执行

## 验证规则

### Level 1：结构性验证（ERROR — 不通过则阻塞 Phase 2 评审）

| # | 规则 | 检查方式 |
|---|------|---------|
| V-01 | 每个 AC 标题符合 `### AC-{NN}: {名称}` 格式 | 正则匹配 `^### AC-\d+:` |
| V-02 | 每个 AC 正文包含 **MUST** 或 **SHALL**（至少一个） | 搜索关键词（大写） |
| V-03 | 每个 AC 至少有一个 `#### Scenario:` 块 | 搜索 `#### Scenario:` |
| V-04 | 每个 Scenario 包含 **GIVEN**、**WHEN**、**THEN**（三者齐全） | 逐块检查 |
| V-05 | AC 编号无重复 | 提取所有 AC-NN，检查唯一性 |

### Level 2：完整性验证（WARNING — 不阻塞但必须在评审中讨论）

| # | 规则 | 检查方式 |
|---|------|---------|
| V-06 | 每个 AC 至少覆盖 2 种场景类型（正常 + 异常/边界） | 场景名分析（含 "异常"/"错误"/"边界"/"为空"/"超时"/"无权限"） |
| V-07 | 有状态流转的 AC 必须有 State Transition 场景 | 包含 "状态" 的 AC 检查场景覆盖 |
| V-08 | 有权限要求的 AC 必须有 Permission 场景 | 包含 "权限"/"角色" 的 AC 检查 |
| V-09 | 跨模块影响声明中提到的模块在 AC 中有体现 | 交叉引用 §跨模块影响声明 |

### Level 3：质量建议（INFO — 记录但不阻塞）

| # | 规则 | 检查方式 |
|---|------|---------|
| V-10 | AC 正文长度 ≤ 500 字符（过长说明应拆分） | 字符计数 |
| V-11 | 总 AC 数与 Feature 数的比例 ≥ 2:1（每个 Feature 至少 2 个 AC） | 计数对比 |
| V-12 | Scenario 中使用具体值而非模糊描述（"VIN=LSVAU..." 而非"某个 VIN"） | 启发式检查 |

## 执行方式

AI 在 Phase 2 评审开始前扫描 `requirement.md`：

```markdown
## 规格验证报告

**文件**: changes/<req>/requirement.md
**AC 总数**: {N}
**验证时间**: {timestamp}

### 验证结果

| 级别 | 通过 | 失败 | 详情 |
|------|------|------|------|
| ERROR | {N} | {M} | {列表} |
| WARNING | {N} | {M} | {列表} |
| INFO | {N} | {M} | {列表} |

### 失败详情

#### ❌ V-02 FAIL: AC-07 缺少 MUST/SHALL 关键词
- **位置**: AC-07 "配置项管理"
- **问题**: 正文使用"需要"而非 MUST/SHALL
- **修复**: 改为 "系统 **MUST** 在配置变更时..."

#### ⚠️ V-06 WARN: AC-04 仅有 1 种场景类型
- **位置**: AC-04 "VIN 校验"
- **问题**: 只有 Happy Path，缺少异常场景
- **建议**: 增加 "VIN 为空" 和 "网络超时" 场景

### 判定
- **ERROR = 0**: ✅ 可进入评审
- **ERROR > 0**: ❌ 需修复后重新验证
```

## 与 Living Spec 的关系

Living Spec 更新后，验证规则对更新后的 requirement.md 重新生效：
- **ADDED** 的 AC 必须符合 spec-format
- **MODIFIED** 的 AC 修改后仍然必须符合
- **REMOVED** 的 AC 需在 living-spec-log 中记录理由

## 与 Phase 10 Spec Sync 的关系

Phase 10 归档时，如果有 Living Spec 更新需要同步到 Atom PRD：
- 同步前验证：确保 delta 合并后的规格仍然通过全部 ERROR 级别规则
- 同步后验证：确保 Atom PRD 无结构破坏

## 旧格式容错

对不使用 spec-format 的旧 requirement.md：
- V-01~V-05 降级为 INFO（不阻塞）
- 输出提示："建议升级为 spec-format 以获得程序化验证"

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

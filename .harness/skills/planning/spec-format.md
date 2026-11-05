---
name: spec-format
description: |-
  可测试需求规格格式：SHALL/MUST 强制语 + Given/When/Then 场景。
  requirement.md 从自由文本升级为机器可验证格式。
  来源：OpenSpec RFC-2119 + Gherkin 场景模板。
version: 1.0.0
appendix_limit: 50
---

# 可测试需求规格格式

> 自由文本需求 → 机器可解析的规格。每条需求必须可测试，不可测 = 不算需求。
> 来源：OpenSpec（RFC-2119 SHALL/MUST + Given/When/Then scenarios）。

## 触发条件

Phase 1 输出 `requirement.md` 时，所有 L1/L2/L3 需求的验收标准**必须**使用本格式。
L4 修复型可简化（仅需 MUST + 一个场景）。

## 需求条目格式

每个验收标准（AC）改写为以下结构：

```markdown
### AC-{NN}: {需求名称}

系统 **MUST** {在什么条件下} {做什么} {产生什么结果}。

#### Scenario: {正常场景名}
- **GIVEN** {前置条件}
- **WHEN** {用户操作或系统事件}
- **THEN** {期望结果}

#### Scenario: {异常场景名}
- **GIVEN** {前置条件}
- **WHEN** {异常输入或边界条件}
- **THEN** {期望的错误处理}
```

## 关键词语义（RFC-2119）

| 关键词 | 语义 | 用法 |
|--------|------|------|
| **MUST** | 绝对要求，不实现 = 不发布 | 核心业务规则、安全约束 |
| **MUST NOT** | 绝对禁止 | 数据泄露、非法状态转换 |
| **SHALL** | 同 MUST（正式文档风格） | 合规/法规要求 |
| **SHOULD** | 强烈推荐，除非有充分理由不做 | 用户体验优化、性能目标 |
| **MAY** | 可选功能 | 增强功能、v2 考虑 |

## 场景覆盖要求

每个 AC **至少包含以下场景类型之一**：

| 场景类型 | 何时需要 | 示例 |
|---------|---------|------|
| **Happy Path** | 必须 | 正常操作成功 |
| **Boundary** | 有数值/长度/数量限制时 | 输入为空 / 恰好等于上限 / 超出上限 |
| **Error/Exception** | 有失败可能时 | 网络超时 / 权限不足 / 并发冲突 |
| **Permission** | 涉及权限时 | 无权限用户尝试操作 |
| **State Transition** | 涉及状态机时 | 从不可操作状态触发操作 |

## 示例（V08 F-02 VIN 校验改写）

改写前（旧格式）：
```markdown
- AC-04：新建资产时，如果 VIN 在全库中已存在，则提交失败并提示"VIN 已存在"
- AC-05：排除自身记录（编辑时自身 VIN 不算重复）
- AC-06：所有 7 条写入路径在汇聚点统一校验
```

改写后（新格式）：
```markdown
### AC-04: VIN 全库唯一性校验

系统 **MUST** 在资产创建或修改时校验 VIN 码全库唯一（含报废/退库/历史记录）。

#### Scenario: 新建时 VIN 重复
- **GIVEN** 数据库中已存在 VIN = "LSVAU2180N2012345" 的资产记录
- **WHEN** 用户通过任一写入路径提交 VIN = "LSVAU2180N2012345" 的新资产
- **THEN** 系统拒绝提交并返回错误提示 "VIN 已存在（关联资产编号：{existing_code}）"

#### Scenario: 编辑时排除自身
- **GIVEN** 资产 A001 的 VIN = "LSVAU2180N2012345"
- **WHEN** 用户编辑 A001 但未修改 VIN
- **THEN** 系统不报重复错误（排除自身记录）

#### Scenario: 7 条写入路径汇聚点校验
- **GIVEN** 系统存在 7 条资产写入路径（页面/导入/API/...）
- **WHEN** 任一路径提交新 VIN
- **THEN** 校验逻辑在 Repo 层汇聚点统一执行（不在各入口分别实现）

#### Scenario: VIN 为空时跳过
- **GIVEN** 非车辆类资产无 VIN 字段
- **WHEN** 提交 VIN 为 null 或空字符串的资产
- **THEN** 跳过 VIN 唯一性校验
```

## 与现有 prd-to-ai-ready.md 的关系

`prd-to-ai-ready.md` 已有 Gherkin 场景生成能力。本文件做的是：
1. **强制化**：从"可选的结构化附录"升级为"必须的 AC 格式"
2. **标准化关键词**：使用 RFC-2119 SHALL/MUST（OpenSpec 标准）
3. **可验证化**：`spec-validation.md` 可对此格式做程序化检查

## 输出位置

写入 `changes/<req>/requirement.md` 的 `## 验收标准` 区段，替代原有的列表式 AC。

## 旧格式兼容

已有 requirement.md 不需要回溯改写。新需求从此使用新格式。Phase 2 评审时，如果 AC 仍是旧格式，AI **提示但不阻塞**。

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

# 场景矩阵硬门禁（Scenario Matrix Gate）

> **触发**：Phase 1（任务拆分）完成后，hook `phase-1-complexity-check.sh` 自动评估第 5 维 `scenario_matrix_required`；命中触发条件却缺失 `scenario-matrix.md` 即**强制**阻断进入 Phase 3。
> **来源教训**：MR #583 `fix/cis-mq-car-usestatus` 涉及 carPurposeType × carPossessType × useStatus 三枚举字段，主对话仅识别"3 字段"未做 5×3×3 = 45 行笛卡尔积矩阵，导致 8 处边界条件遗漏 + 多次回退（T-R1..T-R3）。
> **核心认知**：枚举字段 / 状态机 / MQ 字段映射 涉及组合爆炸；不"先出矩阵"就"先动手"等于赌运气。

---

## 一、触发条件（任一命中即要求矩阵）

| 通道 | 信号 | 检测方式 |
|------|------|---------|
| **关键词** | `MQ` / `topic` / `状态机` / `枚举字段对照` / `跨系统映射` / `RocketMQ` | grep -iE 在 `requirement.md` |
| **文件路径** | `*Consumer*.java` / `*Listener*.java` / `*MessageHandler*` 出现在 `plan.json` 的 `filesToModify` | jq 解析 |
| **业务字段** | `carPurposeType` / `carPossessType` / `useStatus` / `carStatus` / 显式 `enum` 声明 ≥ 2 处 | grep -E 在 `requirement.md` |

任一命中 → `scenario_matrix_required = true`。

**反例（不触发）**：
- 文档专项任务（`type: docs` / 仅修改 `*.md`）
- 单字段修改（如修复一个字段的拼写）
- UI 文案调整 / CSS 样式

---

## 二、矩阵格式（强制）

```markdown
| 字段A | 字段B | 台账状态 | 期望行为 |
|-------|-------|----------|----------|
| 1     | 1     | IN_USE   | 同步更新 |
| 1     | 1     | RETIRED  | 跳过     |
| 1     | 2     | IN_USE   | 软删除   |
| ...   | ...   | ...      | ...      |
```

**强制要求**：
1. 表头第一行包含每个枚举字段名 + 至少 1 列"期望行为"
2. 数据行覆盖每个枚举字段所有取值的笛卡尔积（cartesian product）
3. 每行的"期望行为"非空且不是 `TODO` / `?` / `待定`
4. 文件位置：`changes/<task>/scenario-matrix.md`（与 `requirement.md` 同目录）

---

## 三、校验逻辑（phase-1-complexity-check.sh 新增第 5 维）

```
1. 读 requirement.md / plan.json
2. 命中触发条件 → scenario_matrix_required = true
3. IF required AND ! exist(<task>/scenario-matrix.md):
     输出违例 → exit 1
4. IF required AND exist scenario-matrix.md:
     - grep '^|.*|' 计数据行（去掉表头 + 分隔行）
     - 期望覆盖行数 = ∏ 各字段枚举数（粗略：通过 requirement.md 的 enum 声明取值数）
     - 实际行数 < 期望 50% → 警告（不阻断，但写入 violations 报告）
     - 任一行"期望行为"列为空 / TODO / ? → exit 1
5. exit 0
```

退出码语义对齐已有 4 维：
- `0` = 全部通过
- `1` = 任一维度超阈值或矩阵缺失/不合格
- `2` = plan / requirement 解析失败

---

## 四、违例场景（实测案例）

### 案例 1（MR #583）：缺矩阵直接编码

```
requirement.md 含：
  - carPurposeType (1..5)
  - carPossessType (1..3)
  - useStatus (IN_USE/RETIRED/DISPOSED)
  - 修改 CarAssetConsumer.java（命中 *Consumer*.java）

phase-1-complexity-check.sh 输出：
  scenario_matrix_required = true
  scenario-matrix.md 不存在
  ❌ 触发硬门禁：必须先在 changes/<task>/ 写 scenario-matrix.md
     期望矩阵：5 × 3 × 3 = 45 行
```

### 案例 2：矩阵存在但行数不足

```
requirement.md 命中 MQ + 状态机
scenario-matrix.md 存在，但只有 6 行（期望 45 行）
→ scenario_matrix_coverage = 13%
→ 警告：覆盖率 < 50%，建议补全
```

### 案例 3：矩阵每行"期望行为"为 TODO

```
| 1 | 1 | IN_USE | TODO |
| 1 | 1 | RETIRED | TODO |
→ ❌ 行内含 TODO，矩阵未生效，回 Phase 1 补全
```

### 案例 4（不触发）：docs-only 任务

```
plan.json: filesToModify = ['docs/foo.md']
requirement.md 不含 MQ / 状态机 / 枚举字段
→ scenario_matrix_required = false
→ 跳过本维度，直接 exit 0
```

---

## 五、与 4 维原有指标的关系

| 维度 | 来源 | 默认阈值 | 关系 |
|------|------|---------|------|
| files_count | plan 文件路径计数 | 5 | 独立 |
| repos_count | plan 仓库计数 | 2 | 独立 |
| phases_count | plan 阶段计数 | 3 | 独立 |
| est_tokens | plan 字数估算 | 50000 | 独立 |
| **scenario_matrix** | requirement / plan 信号检测 | 触发即必需 | 与上述 4 维 OR 关系 |

任一维度违例 → 全部走 subagent / 矩阵补全；矩阵违例不能用"调用 subagent"绕过——矩阵本身是输入，subagent 也得读它。

---

## 六、bypass / experimental

```
1. plan.json 含 experimental: true → 跳过本维度（与 4 维原有 bypass 一致，24h 内）
2. 环境变量 SCENARIO_MATRIX_BYPASS=1 → 跳过（仅限文档/紧急 hotfix）
3. requirement.md 含 <!-- scenario-matrix-gate: skip --> 注释 → 跳过 + 写入审计日志
```

---

## 七、配套 skill / agent

- 输入侧：`scenario-matrix-builder` skill（T-010）— 从 YAML 字段定义生成 markdown 矩阵骨架
- 评审侧：`reviewer-agent` 矩阵覆盖度检查（T-013）— review 时校验测试用例覆盖矩阵每行
- 上下文：`domain-analyst` sub-agent（T-008）— 业务字段全集分析

本文件是输出门禁；上述三者是上下游配合。

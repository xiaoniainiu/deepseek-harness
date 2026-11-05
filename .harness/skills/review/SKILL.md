---
expiration_signal:
  status: not_yet_evaluated
  last_evaluated: "2026-05-25"
  evaluation_baseline:
    intercepted_real_bugs: 0
    triggered_total: 0
    last_triggered_days_ago: null
  retire_path: dormant
---

# 评审技能：分离执行和评判

> 做事的 Agent 和评判的 Agent 分开。评审不需要更聪明，只需要不同的检查视角。

## 触发条件

- Phase 2（需求评审）/ Phase 4（编码评审）/ Phase 6（单测评审）

## 核心原则

1. **切换视角**：从"实现者"切换到"审查者"，忘掉"我写的所以一定对"
2. **结构化输出**：每条问题含 严重级别 + 位置 + 问题 + 建议
3. **轮次上限**：需求评审 ≤ 3 轮，编码评审 ≤ 2 轮
4. **超出升级**：超过轮次上限 → 人工决策

## 评审输出格式

```markdown
| 级别 | 位置 | 问题 | 建议 | 优先级 |
|------|------|------|------|--------|
| P0 | file:line | 描述 | 修复方案 | 必须修 |
| P1 | file:line | 描述 | 修复方案 | 应该修 |
| P2 | file:line | 描述 | 修复方案 | 可以改 |
```

## 两种评审模式

### 计划评审（Phase 2）
- 加载 `plan-review-template.md`
- 审查对象：`changes/<req>/requirement.md` + `tasks.md`
- 关注：需求完整性、方案可行性、风险遗漏

### 代码评审（Phase 4 / Phase 6）— 三阶段

代码评审分三个阶段依次执行，**前一阶段未通过不进入下一阶段**：

**第零阶段：Scope Drift Detection（范围漂移检测，强制首步）**

> WHY: 执行快照验证——范围蔓延是 AI Agent 最常见的毛病（多做或少做）。
> Google Code Review 指南: "First, check if the CL does what it's supposed to do."

执行步骤：
1. 从 `requirement.md` 提取验收标准清单（N 条）
2. 从 `git diff` 提取实际变更点（M 处）
3. 建立映射表：

```markdown
## Scope Drift Detection

| 验收标准 | 对应 diff 位置 | 状态 |
|---------|---------------|------|
| AC-1: xxx | file.java:L42 | ✅ COVERED |
| AC-2: xxx | file.java:L88 | ✅ COVERED |
| AC-3: xxx | （无对应 diff） | ❌ INCOMPLETE |

| diff 位置 | 对应验收标准 | 状态 |
|-----------|-------------|------|
| file.java:L42 | AC-1 | ✅ MAPPED |
| other.java:L10 | （无对应 AC） | ⚠️ DRIFT |
```

判定：
- 存在 INCOMPLETE → 实现不完整，回退 Phase 3 补充
- 存在 DRIFT → 范围蔓延，必须提供理由（重构必需/安全修复/依赖变更）
- 全部 COVERED + MAPPED → `CLEAN`，进入第一阶段

**第一阶段：Spec 合规审查**
- 加载 `code-review-template.md`
- 审查对象：git diff vs requirement.md / plan.json
- 关注：**实现是否符合需求规格**
  - 验收标准是否全部覆盖？
  - filesToModify 范围是否匹配？
  - 是否有超出需求范围的"顺手修改"？
  - 是否遗漏了需求中的边界条件？

**第二阶段：代码质量审查**
- 前提：第一阶段 PASS
- 审查对象：代码本身的实现质量
- 关注：**代码正确性和工程质量**
  - 正确性：逻辑对不对、SQL 对不对、类型一致吗
  - 安全：注入、权限、数据越界
  - 性能：N+1、全表扫描、线程池共享
  - 风格：是否匹配现有代码风格

**为什么分两阶段**：Spec 不合规的代码质量再高也无意义（方向错了）。CASE-003 证明：AI 可能写出质量很好但方向完全相反的代码。

## 门禁判定

```
PASS: P0 == 0 AND P1 <= 2（且每条有书面理由）
FAIL: P0 > 0 OR P1 > 2
```

## 主动扩展检查（Phase 4 第二阶段末尾必做）

> WHY: 执行快照验证——对每处改动主动问"如果这里出错了下游会怎样"，
> 是 FMEA（失效模式与效应分析）在代码评审中的实践。

对每个 diff hunk，填写以下表格：

```markdown
## 主动扩展检查

| 改了什么 | 扩展问（如果出错会影响什么？） | 验证命令 | 结论 |
|---------|--------------------------|---------|------|
| {变更描述} | 其他地方是否也有同名参数/常量？ | grep -r "xxx" | 安全/需关注 |
| {变更描述} | 下游是否对该值做字符串比对/解析？ | grep consumers | 安全/需关注 |
| {变更描述} | 前端/模板是否会转义或截断该输出？ | 查渲染方式 | 安全/需关注 |
```

**规则**：
- 每处改动至少一个扩展问
- 扩展问的答案必须有 grep/read 命令支撑，不可凭推理
- "需关注"项必须在 E2E（Phase 6.8）中覆盖验证

## 已有技能复用

- 评审接收心态 → `skills/receiving-code-review/SKILL.md`
- 评审请求流程 → `skills/requesting-code-review/SKILL.md`
- 完成前验证 → `skills/verification-before-completion/SKILL.md`

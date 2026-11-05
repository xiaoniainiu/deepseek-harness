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

# 编码技能：分层实现 SOP

> 按规范一层一层实现，不是凭感觉写。

## 触发条件

Phase 3（编码实现）开始时加载本文件。

## ⚠️ 执行顺序（不可违反）

```
Step 0（心智模型+Pre-flight+Gene）→ 分层编码（每层内走 TDD 红绿循环）
                                              ↑
                                    tdd-loop.md 在这里介入
```

**TDD 不在 Step 0 之前。** Step 0 是"理解要做什么"，TDD 是"怎么做"。
先理解再动手。每个分层 Step（1-6）内部走 TDD 循环。

## 执行原则

1. **自底向上实现**：Entity/Mapper → Service → Controller → Frontend
2. **每层一个 commit**：保证每层独立可验证
3. **先读再写**：改任何文件前先读其现有实现
4. **匹配现有风格**：不引入项目中不存在的模式

## 任务锁协议（手动模式强制）

Phase 3 一次只做一个任务。Scheduler 模式由外部循环强制；手动模式由以下协议强制：

```
开始任务前:
  1. 读 phase-state.json → taskLock.activeTaskId
  2. IF activeTaskId != null AND activeTaskId != 当前任务:
     → STOP。前一个任务未提交。先完成它。
  3. 设置 taskLock.activeTaskId = "T-XXX", startedAt = now

完成任务后（领下一个前必须全部执行）:
  4. git commit（一个任务至少一个 commit）
  5. 设置 taskLock.lastCommitSha = $(git rev-parse HEAD)
  6. 更新 harness-progress.json（status/passes/evidence）
  7. 设置 taskLock.activeTaskId = null

违规：未释放锁就开始新任务 = Phase 3 阻塞（GOV-009）
```

## 分层编码流程

```
Step 0: 构建心智模型 + Karpathy 四原则门禁
  → 执行 Pre-flight（已有，skills/context/SKILL.md）
  → 加载 skills/coding/deep-work-protocol.md §1
  → 输出 Mental Model（数据流/状态变迁/不变量/边界）
  → 确认心智模型与 requirement.md 一致
  → L4 修复型可简化为一句话数据流
  
  Karpathy Gate（编码前强制自检，任何一项未通过则不动手）:
  □ K1-Think: 我的假设是什么？列出。有不确认的？→ 先问再写
  □ K2-Simple: 最简实现方案是什么？如果有 50 行版本为什么要写 200 行？
  □ K3-Surgical: 我要改的每一行都直接追溯到用户请求？不相关的代码不碰
  □ K4-Goal: 我的验证标准是什么？（不是"让它工作"，是具体的 test/command/output）

Step 0.05: 影响追踪表（Pre-Flight Checklist，编码前必输出）

  <STOP>如果你正在跳过这一步，STOP。这是铁律 #3"改A查B"的执行机制，不是可选项。</STOP>

  > WHY: 执行快照验证——结构化的 pre-flight 表格比"改A查B"口号更可靠。
  > 类似航空业的飞行前检查清单，每次编码前逐项确认，不依赖记忆。
  
  对本次需要修改的**每个方法/文件**，输出以下表格：

  | 层 | 检查问题 | 验证方式 | 结论 |
  |----|---------|---------|------|
  | 调用方 | 谁调用了被改方法/接口？ | `grep -r "methodName"` | 单一/N个（列出） |
  | 数据流 | 参数/返回值变化影响哪些下游？ | 分析 SQL/ORM/DTO | 无影响/需同步改 |
  | 契约 | 方法签名/API 契约是否变化？ | `diff` 前后 | 不变/Breaking Change |
  | 渲染 | 输出在哪些界面展示？格式要求？ | `grep` 前端引用 | 列出界面 |
  | 多入口 | 哪些业务流程触发此逻辑？ | `grep` 入口调用链 | 列出流程 |
  | 开关 | 有无 feature flag / 配置可回滚？ | `grep` config/nacos | 有(列出)/无 |

  **输出要求**：
  - 每行结论必须有 grep/read 命令的实际输出支撑（不可凭记忆填写）
  - 发现调用方 > 1 时，标注为 ⚠️ 并在后续编码中逐一验证
  - 表格写入 `changes/<需求>/pre-flight.md`（可选，大改动建议保留）
  
  **简化条件**：Direct Mode 或改动 ≤ 1 个方法时，可内联到心智模型输出中不单独建表。

Step 0.1: Expert Context Load + 持久知识检索（跨 Phase + 跨会话连续性）
  → 读取 phase-state.json → taskAssignment[当前task] → expert_name
  → 读取 phase-state.json → expertContext[expert_role]
  → IF designDecisions 非空:
    输出 "设计决策承接: {decisions}" → 不重新推导
  → IF crossPhaseNotes 非空:
    输出 "跨阶段笔记: {notes}" → 关注标注的风险点
  
  跨会话知识加载（通过 Harness 持久层）:
  → harness search "{当前任务模块} {涉及文件名}" → 获取历史 CASE/Gene
  → 结果中 score > 2.0 的条目 → 高相关警告（可能有前人踩过的坑）
  → harness search "pattern:{检测到的模式名}" → 获取跨域 Capsule
  → 写入 phase-state.json → knowledgeSnapshot.search_hits: [{id, score}]

Step 0.5: Gene 检查执行（EP-007 强制）
  → 读 evolution/genes/phase-checklist-genes.yaml → 筛选 phase 包含 3 的 Gene
  → 对每条 Gene：评估 trigger 是否匹配当前任务
  → 命中的 Gene → 执行 check_command → 记录结果
  → 写入 scores.jsonl：{"type":"gene_applied","geneId":"{ID}","taskId":"{task}","phase":"3","result":"{pass/fail/na}","timestamp":"{ISO}"}
  → 写入 phase-state.json → evolutionSignals.genes_injected += [geneId]
  → 如果 check_command 发现问题 → genes_prevented_bug += [geneId]
  → 无匹配 Gene 时仍记录：{"type":"gene_applied","geneId":"NONE","taskId":"{task}","phase":"3","timestamp":"{ISO}"}

Step 1: 确认数据模型
  → 读 wiki/data-model.md
  → 确认表结构支持需求
  → 如需新表/新列 → 先写 DDL → 标注到 tasks.md

Step 2: Data Access 层
  → 加载 skills/coding/data-access-layer.md
  → Entity + Mapper/Repository

Step 3: Integration 层（如涉及外部调用）
  → 加载 skills/coding/integration-layer.md
  → External Client + DTO

Step 4: Business 层
  → 加载 skills/coding/business-layer.md
  → Service 实现

Step 5: Presentation 层
  → 加载 skills/coding/presentation-layer.md
  → Controller / API endpoint

Step 6: Frontend（如涉及）
  → 加载 skills/coding/frontend-layer.md
  → 组件 + API Hook
```

## 每层完成后：Checkpoint 反思（替代原 checklist）

完成一层后执行 `deep-work-protocol.md` §2 Checkpoint：

1. **意图对照**：原始意图 vs 实际实现 → 有无偏差？
2. **发现记录**：新理解 → 是否触发 `living-spec.md`？
3. **下一层预判**：约束传递 + 临时决策标记

**附属验证**（仍然执行，降级为 Checkpoint 的一部分）：
- [ ] 编译通过？
- [ ] 无新 warning/lint 告警？
- [ ] 匹配现有代码风格？（K3: 不改引号风格/不加类型注解/不改格式）
- [ ] 不超出需求范围？（K3: 每改一行都能追溯到用户请求）
- [ ] 没有过度抽象？（K2: 单次使用的逻辑不建类/不建接口/不建策略模式）
- [ ] commit message 关联 task？

**Expert Context Write**（每个 Checkpoint 时执行）：
- 发现新约束/偏差/风险 → 写入 phase-state.json → expertContext[my_role].crossPhaseNotes:
  ```json
  {"phase": "3", "task": "T-XXX", "decision": "一句话", "rationale": "为什么", "affects_phases": ["4", "9"]}
  ```
- 只记关键发现，不记常规实现细节
- 目的：Phase 4 评审时对照、Phase 9 验证时重点关注

L4 修复型可继续使用上述 checklist 而不执行完整 Checkpoint。

## 节律控制（Rhythm Guard）

> 来源：ai-collab 5 层节律机制。防止分析瘫痪和伪工作。

### 强制规则（Phase 3 编码期间）

```
1. Tool Ratio Gate:
   - 每 3 段输出文字之间必须有 ≥1 个 tool call（Read/Edit/Bash）
   - 连续 2 轮无 tool call → 自检：是否在分析瘫痪？→ 立即执行一个操作或报告阻塞
   - 目标: tool_calls / total_output_blocks ≥ 0.6

2. Output Cap:
   - 编码期间每轮非代码输出 ≤ 200 字
   - 超出 → 砍到行动："下一步我做 X"
   - 禁止: 列举选项不做决策 / 解释正在做什么 / 重复已知信息

3. Execute First, Report After:
   - 编码期间不解释思路，直接执行
   - 只有两种合法输出: tool call 或 "阻塞在 X，原因 Y，建议 Z"
   - 完成后报告结果（一句话）

4. 反伪工作检测:
   - 产出美观 markdown 但无 tool call = 伪工作 → 立即停止，执行一个操作
   - 列出 3+ 选项但不选择 = 分析瘫痪 → 选一个最可能的执行
   - 说"我无法"但未尝试 = 提前放弃 → 至少尝试一次
```

### 度量（写入 phase-state.json → executionTrace）

```json
"executionTrace": {
  "steps": [],
  "totalToolCalls": 0,
  "totalTextBlocks": 0,
  "toolRatio": 0.0,
  "rhythmNudges": 0,
  "consecutiveNoToolTurns": 0
}
```

每个 Checkpoint 时更新 toolRatio。Phase 10 归档时写入 scores.jsonl。

---

## 进化信号采集（C16 自进化）

Phase 3 编码过程中，**持续记录**以下信号到 `phase-state.json → evolutionSignals`：

| 事件 | 何时记录 | 记录到哪个字段 |
|------|---------|-------------|
| Phase 3 开始时注入了哪些 Gene | 心智模型构建时 | `genes_injected` |
| 实际执行中参考了哪个 Gene 的建议 | 编码决策引用 Gene 时 | `genes_actually_used` |
| 某个 Gene 的检查指令阻止了一个 bug | 执行 Gene check 发现问题时 | `genes_prevented_bug` |
| AP 匹配了当前代码模式 | Phase 入口 AP 扫描时 | `aps_matched` |
| AP 警告阻止了一个潜在问题 | 因 AP 警告修改了实现时 | `aps_prevented_bug` |
| Checkpoint 发现偏差 | Checkpoint 反思时 | `checkpoint_deviations` += 1 |
| Living Spec 触发更新 | Living Spec Update 时 | `living_spec_deltas` += 1 |

**记录方式**：编码过程中在内部追踪，每个 Checkpoint 时写入 phase-state.json（不需要每次编辑都写）。

## 已有系统优化 SOP

当任务是**优化/重构已有代码**（而非新建功能）时，分层编码流程之前插入三步前置：

```
Step 0a: 诊断（先读代码再开药）
  → 读目标模块的现有实现
  → 识别具体瓶颈（性能？耦合？正确性？）
  → 用数据支撑诊断（慢 SQL 日志、Profile、错误日志）
  → 禁止：不读代码凭感觉提方案

Step 0b: 保护测试（先建安全网再动刀）
  → 为现有行为添加测试/基准（测试覆盖的是"当前行为"不是"期望行为"）
  → 运行并记录基准数据（响应时间、数据正确性）
  → 禁止：没有保护测试就开始修改

Step 0c: 增量计划
  → 每次只改一个关注点
  → 每步完成后与基准对比
  → 退化立即回滚（不是"之后修"）
  → 禁止：顺手重构不在任务范围内的代码
```

**三条铁律（来源：Superpowers + 盘点模块 46 问题实证）**：

1. 先诊断后开药 — 不读代码就提方案 = 庸医
2. 先保护后动刀 — 没有基准测试就优化 = 裸奔
3. 最小化改动 — 顺手重构 = 埋雷

## Phase 3 完成门禁（新增）

所有 Task 完成后，进入 Phase 4 前，执行 `deep-work-protocol.md` §3 意图验证：

1. **Completeness**：requirement.md 每个功能点逐条对照实现位置
2. **Correctness**：意图匹配（不只是字面——"禁止 X"不等于"隐藏按钮"）
3. **Coherence**：设计决策一致性（各层数据模型、命名、方案选择）

三项全 PASS → 进入 Phase 4。
任一 FAIL → 修复 → 重新验证。

L1/L2 需额外执行 §4 体验对照（计划的用户旅程 vs 实际用户旅程）。

---

## 子文件索引

| 文件 | 加载时机 |
|------|---------|
| `data-access-layer.md` | Step 2 |
| `integration-layer.md` | Step 3（有外部调用时） |
| `business-layer.md` | Step 4 |
| `presentation-layer.md` | Step 5 |
| `frontend-layer.md` | Step 6（有前端时） |
| `tdd-loop.md` | 贯穿全过程 |
| `verification.md` | 每步完成后 |
| `deep-work-protocol.md` | Step 0（心智模型）+ 每层（Checkpoint）+ Phase 3 结束（意图验证+体验对照） |
| `living-spec.md` | Checkpoint 发现偏差时 |
| `self-healing.md` | 遇到错误时 |

# Phase Plan Protocol — 五步子循环

> 每个 Phase 开始时强制执行。不走这五步直接动手 = 违反流程。

## 核心理念

来源：superpowers `writing-plans` + `brainstorming` 方法论。

```
搞清楚上下文 → 想清楚做什么 → 做好计划 → 动手 → 验证
```

不是所有 Phase 都需要写完整计划文件，但所有 Phase 都必须**在对话中明确输出**前三步的结论。

## 五步详解

### Step 1: Context Load（搞清楚上下文）

**目标**：确保对当前要做的事情有足够的信息支撑。

| Phase | 必读上下文 | 主动查什么 |
|-------|-----------|-----------|
| 0-1 | PRD/口头需求 + 已有系统现状 | grep 关键实体、读 DB schema、查同类实现 |
| 2 | requirement.md + 已有方案 | 查架构文档、查外部接口文档 |
| 2.5 | requirement.md + review 结论 | 统计改动文件数、分析依赖关系 |
| 3 | requirement + review + plan + 目标代码 | grep 调用方、read 目标方法上下文 |
| 4 | diff + requirement + coding-standards | 查六层追踪每层证据 |
| 5-6 | 验收标准 + 已有测试模式 | grep 测试工具/框架用法 |
| 7 | repo-map + deploy-info + branch 状态 | git status + fetch remote |
| 9 | deploy 结果 + requirement + PRD | curl/浏览器验证 |
| 10 | 全流程 review + scores.jsonl | 对比预期 vs 实际 |

**产出**：≤5 条关键已知事实（在对话中列出）。

### Step 2: Intent Clarify（想清楚做什么）

**目标**：用一句话说清楚本阶段要达成什么。

格式：
```
[目标] 本阶段完成后，______ 应该成立。
[不确定] ______ 需要先确认。
```

**规则**：
- 如果有 ≥2 个不确定点 → 先查再动手（回到 Step 1 补充）
- 如果目标写不出来 → 说明对任务理解不够（回到 Step 1）
- 如果目标太大 → 拆分（进入 Phase 2.5 原子拆解）

### Step 3: Approach Plan（做好计划）

**目标**：列出具体步骤，每步有验证标准。

**轻量模式**（Direct Mode / 改动 ≤2 文件）：
```
1. [做什么] → [怎么验证]
2. [做什么] → [怎么验证]
```

**完整模式**（Full Mode / 改动 ≥3 文件 / 有不确定点）：

使用 `writing-plans` skill 的 plan 文件格式：
```markdown
## Implementation Plan

### File: path/to/file.ext
- [ ] Step 1: description (verify: command)
- [ ] Step 2: description (verify: command)

### File: path/to/other.ext
- [ ] Step 3: description (verify: command)

### Risks
- Risk 1 → mitigation
```

**判定规则**：

| 条件 | 模式 |
|------|------|
| Direct Mode + ≤2 文件 + 用户给了明确指令 | 轻量（对话中列步骤） |
| Full Mode Phase 1-2 | 完整（brainstorming → plan 文件） |
| Full Mode Phase 3+ 且 ≥3 文件 | 完整（plan 文件写入 changes/<req>/） |
| Full Mode Phase 3+ 且 ≤2 文件 | 轻量（对话中列步骤） |
| 有 ≥2 个不确定点 | 完整（必须写 plan 文件） |

### Step 4: Execute（按计划做）

**规则**：
- 按 Step 3 的步骤顺序执行
- 每步完成后对照验证标准确认
- 偏离计划时：先在对话中说明为什么偏离，更新计划，再继续
- **禁止静默偏离**：做了计划外的事不说 = 违反纪律

**Checkpoint 反思**（每 3-5 步或每个文件完成后）：
```
[Checkpoint] 
- 计划完成进度：N/M
- 是否偏离：是/否
- 新发现：______
- 下一步：______
```

### Step 5: Verify（对照验证）

**目标**：对照 Step 2 的目标，确认达成。

**验证内容**：
1. Step 2 的目标达成了吗？（用证据说明）
2. 有新发现吗？（写入 Living Spec 或下一 Phase 输入）
3. 有偏离计划的地方吗？（记录原因）

**验证失败时**：
- 目标未达成 → 回到 Step 3 修改计划
- 发现新问题 → 评估是否需要回退 Phase

## 各 Phase 适用示例

### Phase 1 示例

```
[Context] 用户要在车辆预约中加驾照校验。读了 requirement.md，grep 了 VehicleReserve。
[Intent] 本阶段完成后，requirement.md 应包含完整的验收标准+影响范围+风险标注。
[Plan]
1. 梳理驾照校验涉及哪些入口（预约/续借/归还） → 验证：入口清单完整
2. 确认数据工场接口可用性和返回格式 → 验证：API 文档已读
3. 输出 requirement.md → 验证：5 视角无 P0
```

### Phase 3 示例

```
[Context] 读了 requirement + review-v1 + 目标 Provider 代码。grep 了 submit 方法的所有调用方。
[Intent] 本阶段完成后，驾照校验逻辑应在 submit 汇聚点生效，覆盖预约+续借两条路径。
[Plan]
1. 在 VehicleReserveProviderImpl.submit() 插入校验调用 → 编译通过
2. 在 VehicleRenewalProviderImpl.submit() 同步加校验 → 编译通过
3. 写失败测试 → 红灯
4. 实现校验 Service → 绿灯
5. 回归跑全量单测 → 全过
[Risk] 数据工场不可用时需要降级开关 → 用 @Value 配置
```

### Phase 7 示例

```
[Context] 读了 repo-map.yaml，确认目标仓库 + tag 格式 + 分支策略。git fetch 完成。
[Intent] 本阶段完成后，代码应已推送到 test 分支并打 tag 触发 CI。
[Plan]
1. git diff 确认无漂移 → diff 行数合理
2. cherry-pick 本次 commit 到干净分支 → 无冲突
3. 推送 + 打 tag → push 成功
4. 确认 CI 触发 → pipeline URL 可访问
```

## 与现有机制的关系

| 现有机制 | 与本协议的关系 |
|---------|--------------|
| Karpathy 四问（K1-K4） | = Step 2-3 的极简版（Direct Mode 用） |
| brainstorming skill | = Step 1-2 的完整版（Phase 0.5-1 用） |
| writing-plans skill | = Step 3 的完整版（复杂任务用） |
| deep-work-protocol Checkpoint | = Step 4 中间的反思点 |
| verification skill | = Step 5 的工具化版本 |

本协议不替代上述机制，而是**统一框架**：确保无论用哪个具体 skill，五步子循环的完整性不被破坏。

## 反模式（违反 Plan-Before-Execute 的信号）

| 信号 | 说明 |
|------|------|
| 直接 Edit 文件没有先说步骤 | 跳过了 Step 3 |
| "我来修一下" 不说修什么 | 跳过了 Step 2 |
| 改了 3 个文件但没有列清单 | Step 3 应该写完整 plan |
| 做完了才说"我刚才做了 X" | 事后补计划 ≠ 先计划 |
| "这个很简单不需要计划" | Direct Mode 也需要 Karpathy 四问 |
| 偏离了计划但不说明原因 | 静默偏离 = 违反纪律 |

## 门禁设计（Phase 3 物理强制，未来实现）

```
PreToolUse Edit/Write:
  IF phase-state.json 存在且 currentPhase 有值:
    IF phase-state.json 中无 "planSteps" 字段:
      IF 当前 session 中未输出过 [Plan] 标记:
        → 输出提醒："[GATE-PLAN] 未检测到计划输出。请先完成 Phase Plan Protocol Step 1-3。"
        → 不阻断（当前为提醒模式），但记录到 observations
```

未来升级为硬阻断：`planSteps` 为空时禁止 Edit/Write。

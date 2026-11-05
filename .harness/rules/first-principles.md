# 第一性原理架构：4 层知识防护体系

> 从原始需求和问题本质出发，不从惯例或模板出发。
> 让错误在结构上不可能发生 > 精确注入必要知识 > 原理自检 > 规则兜底。

## 四层架构

```
[Layer 0] 结构性预防 — 系统自动把信息摆在面前，错误不可能发生
[Layer 1] 精确注入   — 基于任务特征向量精确匹配，宁缺毋滥
[Layer 2] 原理自检   — 5 条不变量，恒定加载，触发自检
[Layer 3] 规则兜底   — 当 L0/L1/L2 都没覆盖时才 fallback 到具体 AP/CASE
```

**优先级**：Layer 0 > Layer 1 > Layer 2 > Layer 3。能用结构预防的不靠注入，能精确注入的不靠原理推导，能原理推导的不翻规则。

---

## Layer 0: 结构性预防

> 不是"规则告诉 AI 要做什么"，而是"系统直接把信息摆在面前"。
> AI 看到完整信息，自然不会遗漏。

### 机制清单

| ID | 预防什么 | 触发条件 | 系统动作 | 取代了什么 |
|----|---------|---------|---------|-----------|
| SP-01 | 修改方法时遗漏重载/调用方 | Edit Java 方法体 | 自动 grep 展示所有同名方法签名 + 全部 caller | Phase 4 人工六层追踪"方法签名层" |
| SP-02 | 新增配置时发明新模式 | 编辑 yml/properties/Nacos | 自动 grep 展示已有同类配置模式 | AP-071/072 "搜索优先" |
| SP-03 | Bug fix 不复现就动手 | Bug fix 类任务进入 Phase 3 | phase-transition-guard 阻断直到 reproEvidence 非空 | 铁律 #23 "先复现" |
| SP-04 | 验证走不同路径 | Phase 9 验证清单生成 | phase-transition-guard 检查验证清单是否从 diff 生成 | 铁律 #26 "走相同路径" |
| SP-05 | 改了 A 漏了用 A 的 B | 任何文件修改提交后 | 自动展示该文件/方法的所有下游引用 | Phase 4 "跨模块影响验证" |
| SP-06 | 校验加在非汇聚点 | 新增校验/拦截逻辑时 | 自动追踪数据写入链路展示所有入口 | Mode B "5 个必答问题" |
| SP-07 | Phase 2.3 无本体注入就做技术设计 | Phase 转换到 2.3 | phase-transition-guard 阻断直到本体表格存在 | SOP-H4 文字约束（71% 触发率） |
| SP-08 | Phase 3 无精确注入就编码 | Phase 转换到 3 | phase-transition-guard 阻断直到注入确认存在 | GOV-013a 文字约束 |
| SP-09 | Phase 9 验证不从 diff 导出 | Phase 转换到 9 | phase-transition-guard 警告验证清单缺失 | 铁律 #26 文字约束 |
| SP-10 | E2E 观察到数据后不查 PRD 就判定 PASS | Phase 6.8/9 写入 e2e-evidence.md | prd-alignment-gate.sh 检查每个验证点是否含 PRD 原文引用，缺失则阻断 | CASE-087 "系统行为≠正确行为"教训 |
| SP-11 | AI 输出合理化短语（"应该是这样/可能是手动的"） | AI 任意输出时 | 触发词检测（见 prd-code-alignment.md §触发词），警告并要求查 PRD | 铁律 #5 "不确定必查" |

### 执行协议

```
结构性预防是自动的——不需要 AI "记得要检查"，系统直接做。
如果预防机制输出了信息，AI 必须在下一步响应中 acknowledge。
如果预防机制未触发（文件类型不匹配/工具不可用），降级到 Layer 1。
```

### 降级条件

| 条件 | 降级到 |
|------|-------|
| 工具/hook 不可用 | Layer 1（精确注入相关知识） |
| 非代码类任务 | Layer 2（原理自检） |
| 文件类型无匹配规则 | Layer 1 |

---

## Layer 1: 精确上下文注入

> 从"Phase 转换时 push 25K"变为"基于任务特征精确匹配 3 条以内"。
> 宁缺毋滥：注入 0 条 > 注入 20 条无关的。

### 精确匹配流程

```
Phase N 开始 →
  Step 1: 提取任务特征向量
    - 改动文件列表（git diff --name-only 或 filesToModify）
    - 改动实体/方法名（从 requirement.md 提取）
    - 变更类型: NEW | MODIFY | DELETE | CONFIG | BUGFIX
    - 涉及模块: inventory | vehicle | account | ...
    
  Step 2: 精确匹配（三通道并行）
    通道 A: 文件名 → code-knowledge-index 精确命中
    通道 B: 实体名 → ontology/links 精确命中 → 下游影响
    通道 C: 变更类型 → 该类型 top-3 高频陷阱
    
  Step 3: 必要性过滤（每条知识必须通过）
    ① 和本次具体改动直接相关？（不是"可能相关"）
    ② 最近 3 个任务中被标记为 useful？（效用验证）
    ③ 与代码现状一致？（最新性验证）
    任一不通过 → 丢弃
    
  Step 4: 注入（严格限制）
    - 最多 3 条
    - 每条格式: 一句话问题 + 一条验证命令
    - 总预算 ≤ 1500 chars
```

### 注入格式（强制）

```
⚠️ [文件/实体] 一句话描述风险
   验证: <可复制执行的命令>
```

示例：
```
⚠️ [updateAssetAccountWithCheck] 有 4 参数和 3 参数两个重载，Controller 调的是 3 参数版本
   验证: grep -n "updateAssetAccountWithCheck" src/ --include="*.java" -r
```

### 与旧注入系统的关系

| 旧机制 | 新机制 | 变化 |
|--------|--------|------|
| 25K budget push-all | 1.5K precision inject | 预算从 25K → 1.5K，条数从 20+ → ≤3 |
| 三阶段自检输出（3 段 markdown） | 一行确认（注入 N 条 / 0 条） | 自检输出从仪式变为信号 |
| priority_inject 不受限 | 一切受限，无例外 | 消除"不从预算扣减"特例 |
| Gene 按 Phase 全量注入 | Gene 仅在文件精确匹配时注入 | 从 Phase-level 到 file-level 精度 |
| 反复 Bug 关键词匹配 | Layer 0 SP-01 结构性覆盖 | 从"提醒"升级为"展示" |

---

## Layer 2: 五条不变量（恒定加载）

> 不是替代规则，是 AI 执行中的 self-check trigger。
> 恒定在 context 中（~400 chars），不需要动态注入。

| # | 原理 | 自检时机 | 自检产出 |
|---|------|---------|---------|
| P1 | **先看实际行为再动手** | 要写代码/改配置的瞬间 | "我确认了运行时行为吗？没有 → 用 Layer 0 工具确认" |
| P2 | **找汇聚点，改一处覆盖全部** | 设计校验/拦截方案时 | "所有入口的数据流汇聚到哪？改那里" |
| P3 | **改 A 必看所有用 A 的 B** | 任何代码修改后 | "Layer 0 已展示引用 → 我逐条确认了影响吗？" |
| P4 | **验证 = 证伪，不是证实** | Phase 9 验证时 | "我能区分'有效果'和'没效果'吗？有对比数据吗？" |
| P5 | **不确定就查，不猜** | confidence < HIGH 时 | "我正在假设什么？能查到事实代替假设吗？" |

### 与第一性原理的映射

```
用户原话: "不假设我清楚自己想要什么" → P5 (不确定就查) + 全局前置 5.1
用户原话: "目标清晰但路径不最短" → P2 (找汇聚点) + 全局前置 5.2
用户原话: "追根因不打补丁" → P1 (先看实际行为) + 全局前置 5.3
用户原话: "输出说重点" → Layer 1 注入格式（一句话+一条命令）+ 全局前置 5.4
```

---

## Layer 3: 规则兜底

> 仅当 Layer 0/1/2 都没覆盖时才触发。
> 不主动注入，由 AI 在遇到不确定性时主动 pull。

### Pull 触发条件

```
AI 执行中遇到以下信号时，主动从 wiki/anti-patterns.md 查找：
  - 编译失败且原因不明确
  - 方案选择犹豫（2+ 等价选项）
  - 与之前任务相似的模式（déjà vu 感）
  
Pull 方式：
  grep -n "关键词" wiki/anti-patterns.md → 读取匹配段落
  
不 Pull 的情况：
  - Layer 0 已展示了相关信息
  - Layer 1 已注入了精确知识
  - 任务简单且 confidence HIGH
```

### 规则淘汰机制

```
每 5 个任务检查一次：
  - 连续 5 次注入但标记为 irrelevant 的规则 → 降级为 dormant
  - 被 Layer 0 结构性预防完全覆盖的规则 → 标记 superseded
  - 6 个月未被 pull 的规则 → 归档到 wiki/archive/
```

---

## 跨层协同示例

### 场景：修改 Java Service 方法

```
[Layer 0 自动触发]
  SP-01: 检测到编辑 VehicleReserveProviderImpl.submit()
         → 自动输出：
           "submit() 有 2 个重载：submit(req,bizFunc,check) 和 submit(req,bizFunc,check,cateControl)
            调用方：VehicleReserveController.java:45, VehicleRenewalController.java:62"

[Layer 1 Phase 转换时]
  精确匹配: VehicleReserveProviderImpl → CASE-069 one-liner
  注入: "⚠️ [submit] 续借 Provider 有独立的 submit，预约改了续借也要改
         验证: grep -rn 'submit' vehicle/ --include='*Provider*.java'"

[Layer 2 恒定自检]
  P3 触发: "改 A 必看所有用 A 的 B"
  AI 确认: Layer 0 已展示 2 个 caller，Layer 1 提醒了续借 → 三处都要覆盖

[Layer 3 未触发]
  L0+L1+L2 已完整覆盖，无需翻规则
```

### 场景：新增配置项

```
[Layer 0 自动触发]
  SP-02: 检测到编辑 application-prod.yml
         → 自动输出：
           "已有配置模式:
            vehicle-cate-codes: [nacos] VEHICLE_XXX
            template-id: [yml] profile-specific
            timeout: [yml] common block"

[Layer 1]
  无精确匹配 → 不注入（宁缺毋滥）

[Layer 2]
  P1: "先看实际行为" → AI 确认看到了 Layer 0 展示的已有模式
  P5: "不确定就查" → 配置格式已确认

[Layer 3 未触发]
```

---

## 度量指标

| 指标 | 目标 | 当前基线 | 计算方式 |
|------|------|---------|---------|
| 注入精确率 | ≥ 80% | 10% → 80%(dogfood) | useful_injections / total_injections |
| 结构预防覆盖率 | ≥ 60% | 0% → 71%(dogfood) | SP_triggered / total_edits |
| 平均注入条数 | ≤ 3 | ~15 → ≤3(v3) | per-phase average |
| 重复犯错率 | 0% | — | same_case_triggered_twice_in_5_tasks |
| Layer 3 触发频率 | ≤ 20% | — | tasks_needing_rule_fallback / total |

---

## 实施状态

| 优先级 | 模块 | 状态 | 实现文件 |
|--------|------|------|---------|
| 1 | **Layer 0 落地** | ✅ SP-01/02/05 + SP-04/06 新增 | `tools/code-quality/hooks/structural-prevention.sh` |
| 2 | **Layer 1 inject.md v3** | ✅ 三通道精确匹配 + 1.5K budget | `skills/memory/inject.md` v3.0 |
| 3 | **Layer 1 + PGLite 集成** | ✅ precision-inject-bridge.sh | `tools/code-quality/hooks/lib/precision-inject-bridge.sh` |
| 4 | **Layer 2 不变量** | ✅ 全局前置 5 + prompt-assembly 硬编码 | `scheduler/lib/prompt-assembly.sh` |
| 5 | **Layer 3 淘汰机制** | ✅ R1/R2/R3 三规则 + Dream Cycle 集成 | `skills/evolution/rule-retirement.md` |
| 6 | **Scheduler 对齐** | ✅ 四层架构 + static/dynamic 分离 | `scheduler/lib/prompt-assembly.sh` |

### SP 实现清单

| SP | 预防目标 | 实现状态 |
|----|---------|---------|
| SP-01 | 方法重载遗漏 | ✅ structural-prevention.sh |
| SP-02 | 配置模式发明 | ✅ structural-prevention.sh |
| SP-03 | Bug fix 无复现 | ✅ phase-transition-guard.sh |
| SP-04 | 验证不从 diff 生成 | ✅ structural-prevention.sh --sp04 + guard 联动 |
| SP-05 | 改 A 漏 B | ✅ structural-prevention.sh |
| SP-06 | 校验加在非汇聚点 | ✅ structural-prevention.sh (新增) |
| SP-07 | 无本体注入就设计 | ✅ phase-transition-guard.sh |
| SP-08 | 无精确注入就编码 | ✅ phase-transition-guard.sh |
| SP-09 | Phase 9 验证缺失 | ✅ phase-transition-guard.sh (升级为 SP-04 联动) |

# Subagent 强制触发（Quantified Trigger）

> **触发**：Phase 1（任务拆分）完成后，hook `phase-1-complexity-check.sh` 自动评估 4 维指标；任一超阈值即**强制**调用 subagent-driven-development，禁止"我自己一个人写到底"。
> **来源教训**：本次 V2 路径前缀迁移涉及 3 仓 / 6+ 文件 / 5 个独立 Phase，主对话上下文窗口被海量 grep 输出 + 配置 yaml + 部署日志吃掉一半，到 Phase 7 时已经记不住 Phase 0 的炸点 6 层细节，多次回查浪费时间；如果在 Phase 1 拆分时就触发 subagent，每个 subagent 一个干净上下文做 1 个原子任务，效率高得多。
> **引用记忆**：[[feedback_harness-atomic-decomposition]]、[[feedback_save-progress-on-exit]]

---

## 为什么要从"推荐"升级为"强制"

Subagent 在原 Harness 设计中是 _推荐_ 项；实践数据表明：

| 场景 | 主对话独自完成 | 强制 subagent | 差异 |
|---|---|---|---|
| 跨 ≥ 5 文件改动 | 上下文 60% 用于回查；4-5 次错改后修复 | subagent 干净上下文；首改正确率 > 85% | 错改率降 70% |
| 跨 ≥ 2 仓库 | "刚才看的是哪个仓"反复确认 | 每仓 1 个 subagent，互不污染 | 沟通成本降 50% |
| 跨 ≥ 3 Phase | Phase 7 时已忘 Phase 0 结论 | 每 Phase subagent 独立产出 | 一致性 > 95% |

**核心认知**：上下文窗口是有限资源；主对话应该做"协调者"，不是"全栈实现者"。

---

## 4 维量化触发条件（OR 关系，任一命中即强制）

```yaml
subagent_trigger:
  files_count:
    threshold: 5
    measure: |
      git diff --name-only origin/main..HEAD | wc -l
      或 Phase 1 plan 中 "Files" 块声明的待修改文件数

  repos_count:
    threshold: 2
    measure: |
      Phase 1 plan 中涉及的 git 仓库数（不同 origin url 视为独立仓库）

  phases_count:
    threshold: 3
    measure: |
      Phase 1 plan 中划分的独立 Phase 数（如 §3.3 自底向上发布的 5 步）

  estimated_context_tokens:
    threshold: 50000
    measure: |
      预估输入 token 数（plan 长度 + 关键文件 + 相关 case 引用）
      粗略估算：1 字 ≈ 1.5 token；5 个文件平均 200 行 × 80 字 ≈ 12 万 token
```

| 维度 | 阈值 | 命中后果 |
|---|---|---|
| **files_count** ≥ 5 | 涉及 ≥ 5 文件 | **强制**调用 subagent-driven-development |
| **repos_count** ≥ 2 | 跨 ≥ 2 仓库 | 每仓 1 个 subagent |
| **phases_count** ≥ 3 | ≥ 3 独立阶段 | 每 Phase 1 个 subagent |
| **est_tokens** ≥ 50000 | 上下文预估 ≥ 50k | 拆分到能放进单 subagent（每个 ≤ 30k） |

任一命中即激活，多维同时命中按"最严格规则"应用。

---

## 强制行为对照表

| 项 | 旧（推荐） | 新（强制） |
|---|---|---|
| Phase 1 完成后 | 用户决定要不要 subagent | **hook 自动评估，超阈值禁止主对话直接进 Phase 3** |
| subagent 失败 | 主对话接管继续 | **必须先分析失败原因（可能是上下文不足/任务粒度大），重新派遣** |
| 跨 Phase 协调 | 主对话边写边查 | **主对话只做协调（读 plan + 派遣 + 审查），不下场写代码** |
| 上下文管理 | 自由 | **每 subagent 完工 → 主对话只保留摘要（≤ 200 字），原文丢弃** |

---

## hook 自动评估流程

```
1. Phase 1 plan.md 写完，状态置 ready
2. hook phase-1-complexity-check.sh 触发
3. 解析 plan：files / repos / phases / 预估 tokens
4. 4 维评估
5a. 全部低于阈值 → 通过，主对话可直接进 Phase 3
5b. 任一超阈值 → 输出红框警告，列出超标维度 + 建议拆分方案
6. 用户/主对话必须执行：
   - 调用 subagent-driven-development skill
   - 或者 plan 拆分为更小子 plan 直到全部维度低于阈值
7. 强制 + 拒绝绕过（除非走 G7 risk-score 条件 override）
```

---

## subagent 拆分指引

### 按文件拆（files_count ≥ 5 命中）

每个 subagent 负责 1-3 个相关文件 + 它们的测试。

```yaml
subagent_decomposition:
  task: "V2 路径前缀迁移"
  subagents:
    - id: SA1
      files: [eam-v2/vite.config.ts, eam-v2/application.yml]
      scope: "前端 base + Spring context-path"
    - id: SA2
      files: [eam-v2/CommonsProxy.java, eam-v2/ConfigsController.java, eam-v2/SpaForwardController.java]
      scope: "全仓兜底"
    - id: SA3
      files: [eam-v2/deploy/nginx-gateway.conf]
      scope: "网关样板"
    - id: SA4
      files: [asset-eam-web/InventoryV2Embed.tsx]
      scope: "iframe URL 修复"
```

### 按仓拆（repos_count ≥ 2 命中）

每仓 1 个 subagent，主对话负责跨仓协调（如发布顺序）。

### 按 Phase 拆（phases_count ≥ 3 命中）

每个独立部署阶段 1 个 subagent；主对话只做"上一 Phase 验证 → 决策是否进下一 Phase"。

---

## bypass 协议

完全不允许 silent bypass。允许的两种例外：

| 场景 | 处理 |
|---|---|
| **真单一文件**（如改文案） | 命中阈值通常不会发生；如发生，hook 输出"看起来是单文件但触发了多维（请检查 plan 字段）" |
| **探索性任务**（PoC，未来要重做） | 用户在 plan frontmatter 写 `experimental: true`；hook 跳过；但限 1 次会话内不超过 24h |

不允许"我赶时间所以不用 subagent"——赶时间恰恰是用 subagent 的最佳理由（加速）。

---

## 反模式

| 反模式 | 真实信号 | hook 反应 |
|---|---|---|
| Phase 1 plan 写得很模糊以躲触发 | files: "若干"；phases: "见下文" | 拒绝，要求显式枚举 |
| 拆 subagent 但每个都是"实现整个功能" | subagent 间内容重叠 > 70% | 拒绝，要求按文件/Phase 切分 |
| Subagent 失败立刻接管 | 主对话直接 Edit 该文件 | 警告，提示先派遣新 subagent |
| Subagent 完工后主对话保留全文 | 主对话上下文 > 60% 是 subagent 输出 | 警告，要求摘要化 |

---

## 与其他 Gate 的关系

| 关系 | 说明 |
|---|---|
| **Phase 0 G1 → G4**：blast_radius_check 中 `involved` 层 ≥ 3 → 自动 +1 复杂度信号 |
| **G3 → G4**：plan revision 中 alternative 涉及 ≥ 2 子方向 → 拆 subagent 评估 |
| **G4 → G5**：subagent 内部失败 ≥ 2 次 → 触发 verify-counter 换路 |
| **G4 → G6**：subagent 数 ≥ N → 归档时多教训轴可能性高 |

---

## 关联

- [phase-0-blast-radius.md](phase-0-blast-radius.md)（G1）— `involved` 层数纳入 phases_count 信号
- [`hooks/phase-1-complexity-check.sh`](../hooks/phase-1-complexity-check.sh) — 实际评估器
- [[feedback_harness-atomic-decomposition]] — 原子分解约束
- subagent-driven-development skill — 强制路由目标
- executing-plans skill — 备选（多会话执行）

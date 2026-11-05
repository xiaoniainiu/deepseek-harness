# 开发流程规范：双模式执行 + 11 阶段 + 门禁 + 精确回退

> 核心原则：**匹配复杂度的流程开销**——简单任务 Direct Mode 快速交付，复杂任务 Full Mode 完整保障。

## 执行模式路由（Phase 0 入口，所有任务必经）

```
收到任务 → Intent Classify（规则判定，非 LLM）:

┌─ Direct Mode（快通道）────────────────────────────────────┐
│ 触发条件（满足任一）:                                       │
│   • 单文件修改 + 用户给了明确指令                            │
│   • Bug fix + 复现路径已知 + 影响范围 ≤ 3 文件              │
│   • 配置变更 + 同类模式已有（grep 可确认）                   │
│   • 用户显式: "直接做"/"简单改"/"快速修"/"一行改动"         │
│                                                            │
│ 执行流: Gene注入 → Phase 3(编码,≤25步) → Phase 4(自审,1轮)  │
│         → Phase 7(推送) → Phase 9(验证) → Phase 10(归档)    │
│ 跳过:   Phase 0.5 / 1 / 2 / 2.3 / 2.5 / 5 / 6            │
│ 保留:   SOP-H1(验证闭环) + SOP-H2(搜索优先) + Gene注入     │
│         + **门禁1(代码扫描,每次commit,不可跳过)**            │
│ 约束:   最大 25 步 tool call / 失败 2 次自动升级 Full Mode   │
│ 人工确认: 仅 H4（推送前）                                   │
└─────────────────────────────────────────────────────────────┘

┌─ Full Mode（完整流程）──────────────────────────────────────┐
│ 触发条件:                                                   │
│   • 多文件（≥5）/ 多模块 / 有审批依赖 / 数据库变更           │
│   • 新功能 / 架构变更 / 第一次接触的模块                     │
│   • 用户不确定方案 / 需要讨论 / 明确要求走完整流程           │
│   • Direct Mode 升级（失败 2 次触发）                        │
│                                                            │
│ 执行流: 下方 11 阶段完整流程                                 │
└─────────────────────────────────────────────────────────────┘

升级协议: Direct → Full
  触发: 同一方向失败 2 次 / 发现影响范围 > 3 文件 / 遇到未知模块
  动作: 停止 Direct，从 Phase 1 开始走 Full Mode
  记录: phase-state.json → mode: "upgraded_from_direct"
```

### Direct Mode 详细规则

```
Step 0: 轻量注入 + Karpathy Gate + 门禁1激活
  - 读 phase-state.json（如有），确认无冲突
  - Gene 注入（只匹配当前 Phase 3 的 Gene，不走完整 3 阶段 LOAD）
  - 跨域知识 top-3（从 cross-domain-index.yaml）
  - SOP-H2: 先搜索已有实现模式
  - **门禁1（Direct Mode 同样强制）**：每次 commit 增量扫描，禁止 --no-verify
  - Karpathy 四问（3 秒自检）:
    K1: 假设明确了？ K2: 最简方案？ K3: 只改该改的？ K4: 怎么验证？
  - Knowledge Engine 注入（EAM 后端任务）:
    - 条件: 任务涉及 `docs/reference/eam-backend/` 下的代码
    - 动作: 自动执行 `knowledge-engine:task_context("{任务描述}")`
    - 效果: 快速建立项目全貌（涉及实体 + 代码入口 + 调用链 + 相关 PRD）

Step 1-N: 执行（≤25 步）
  - 先读再改（每个文件）
  - K3 外科手术: 不改格式、不加注释、不重构相邻代码、只改追溯到请求的行
  - K2 简洁优先: 单次使用不建抽象、50 行能解决不写 200 行
  - 节律: 每 3 段文字必须有 tool call
  - 失败处理（react-doctor binary-split 模式）:
      第 1 次失败: 读错误信息 + 修复
      第 2 次失败（全局性）: 升级 Full Mode
      第 2 次失败（局部性，可定位到具体文件）: 二分降级 ↓

  二分降级协议（Binary-Split Graceful Degradation）:
    触发: 编译/类型检查错误可定位到 ≤ N/2 个文件
    执行:
      1. 从错误输出提取 failing_files（filename:line 模式）
      2. remaining = filesToModify - failing_files
      3. IF remaining 非空:
         - 创建 sub-task-A: {files: remaining, status: continue}
         - 创建 sub-task-B: {files: failing_files, status: blocked, needs: diagnosis}
         - 继续执行 sub-task-A（直到推送）
         - sub-task-B 标记为独立后续任务
      4. 记录: phase-state.json → executionTrace.partial_decomposition
    降级: 无法区分文件边界 → 回退到 "升级 Full Mode"

Step N+1: 自审（1 轮）
  - 对照变更 diff 检查: 编译通过 + 无范围溢出 + 逻辑正确
  - 可选: Skill: review (PR 预审，独立 diff 分析)
  - P0 问题 → 立即修复（不走 Phase 3 回退）
  - P1 问题 ≤ 2 → 记录理由，继续

Step N+2: 推送
  - [H4] 暂停确认目标分支
  - Skill: ship (自动化: merge base + tests + VERSION + CHANGELOG + push + PR)
  - 或手动 git push（简单场景）

Step N+3: 验证 + 归档
  - Skill: qa (浏览器 QA: 打开页面 + 交互验证 + 截图证据)
  - 或 SOP-H1: 必须有验证命令输出（无浏览器场景）
  - scores.jsonl: mode="direct", duration, steps, toolRatio
  - CASE 归档（如有新发现）
  - harness put "traces/{task}" → 持久化执行摘要
```


## 流程总览

```
Phase 0.5         Phase 1    [H1]    Phase 2    [H2]        Phase 3    Phase 4    [H3]
想法探索(L1/L2) → 需求分析 → 🧑确认 → 需求评审 → 🧑确认 ─┬→ 编码实现 → 编码评审 → 🧑确认
  (路由表触发)     (Mode A/B/C)   (≤3轮+5视角)          │  (心智模型+   (≤2轮)
  6追问→选方案                       ↓                │  Checkpoint
       │                       超出→人工决策          │  +意图验证)
  L3/L4 跳过                                         │
                          重大变更时走分支 ─────────────┤
                                                     │
                              Phase 2.3   [H2.3]     │
                              技术设计  →  🧑确认 ─────┘
                            (条件分支)

[H3]    Phase 5    Phase 6    Phase 6.5   Phase 6.8        Phase 7    [H4]    Phase 8
🧑确认 → 单测编写 → 单测评审 → 单测执行 → 本地验证(E2E) → 代码推送 → 🧑确认 → CI验证
                                          ↑门禁4            ↑推 test 分支
                                          前端/移动端:
                                          启动dev→交互验证→截图

Phase 8    Phase 9    Phase 10         [H5]
CI验证 → 部署验证 → 用户确认+案例归档 → 🧑确认 → 完成

🧑 = 人工确认点（Agent 必须暂停等待）
```

## Phase 状态机（强制执行）

每个需求目录下 `phase-state.json` 是 Phase 级唯一权威状态源。

**状态**：`pending` → `running` → `done`（或 `skipped`）

**转换规则**：不满足前置条件，`currentPhase` 不前进。没有例外。

| 转换 | 前置条件（文件锁） | GOV |
|------|-------------------|-----|
| → Phase 1 | L1/L2：`exploration.md` 存在 + 用户已选方案；L3/L4：直接进入 | GOV-008 + **GOV-025** + SP-14 |
| → Phase 2 | `requirement.md` 存在 + gates.H1=passed + **领域知识已加载**（若匹配 AI-INDEX 项目） | GOV-021 + **GOV-026** + SP-15 |
| → Phase 3 | `review-v*.md` 存在 + P0=0 + gates.H2=passed + **功能分支已创建**（HEAD 不在 main/master） | GOV-001+002 + **GOV-027** + **GOV-031** + SP-16 |
| → Phase 2.3 | 满足触发条件（新服务/3+微服务/DB变更）+ **本体注入表格已输出**（实体+状态机+影响链） | GOV-007 |
| → Phase 3 | `review-v*.md` 存在 + P0=0 + gates.H2=passed + **本体注入摘要已输出**（SOP-H4 三段自检）+ **功能分支已创建** | GOV-001+002 + **GOV-027** + **GOV-031** + SP-16 |
| → Phase 4 | Phase 3 status=done（编译通过+代码已提交） | **GOV-028** + SP-17 |
| → Phase 5 | gates.H3=passed | **GOV-022** + SP-10 |
| → Phase 6.8 | Phase 6.5 status=done（测试全过）| GOV-013 + SP-11 |
| → Phase 7 | Phase 6.8 status=done（E2E 门禁4 PASS）或 status=skipped（**仅限纯文案/注释改动**，含跳过说明）。Direct Mode 豁免 | GOV-004 + **GOV-023** + SP-12 |
| → Phase 8 | gates.H4=passed | **GOV-024** + SP-13 |
| → Phase 9 | CI status=SUCCESS | GOV-005 + **GOV-029** + SP-18 |
| → Phase 10 | Phase 9 status=done | **GOV-030** + SP-19 |

**人工确认点**到达时 `gates.{HN}.status = "waiting"`，AI 停止执行，等用户确认后才更新为 `"passed"`。

详见 `skills/harness.md` §2.5 转换协议 和 `governance/policies/phase-gates.md`。

## 角色分配 [C1]

> 详见 `rules/agent-handoff.md`。不同 Phase 由不同角色主导。

| Phase | 角色 | 原因 |
|-------|------|------|
| 0.5, 1-2, 2.5, 10 | **Coordinator** (`agents/coordinator.md`) | 规划与综合 |
| 2.3, 3(后端), 7-9 | **Backend Executor** (`agents/senior-backend-developer.md`) | 后端执行 |
| 3(前端) | **Frontend Executor** (`agents/frontend-developer.md`) | 前端编码+前端审查清单 |
| 4, 6 | **Verifier** (`agents/verifier.md`) | 对抗性评审 + 门禁1/2/3 达标验证 |
| 5, 6, 6.8, 9 | **QA Engineer** (`agents/qa-engineer.md`) | 测试方案+接口测试+本地E2E+部署验证 |

## 增强能力索引 [C0-C7]

| 能力 | 影响的 Phase | 文件位置 |
|------|-------------|---------|
| C0 领域本体 | 1(全局), 2.3(设计), 3(精确), 4(覆盖度) | `ontology/` + `skills/ontology/` |
| C1 多角色 | 全部 | `agents/` + `rules/agent-handoff.md` |
| C2 自动拆解 | 2.5 | `skills/planning/auto-decompose.md` |
| C3 多视角评审 | 2 | `skills/review/multi-perspective.md` |
| C4 记忆+知识注入 | 全部（Phase 转换时） | `memory/` + `skills/memory/` |
| C5 治理策略 | Phase 转换 + Push | `governance/` |
| C6 可观测性 | Scheduler 全程 | `scheduler/lib/trace.sh` |
| C7 进化引擎 | 3(Gene注入), 4/6(Reflexion), 10(归档) | `evolution/` + `skills/evolution/` |
| C20-C29 | agent-memory 增强 | 见下表 | `skills/memory/` + `knowledge-map.yaml` |

### C20-C29 增强能力索引（来源：agent-memory 对标）

| 能力 | Phase | 文件 |
|------|-------|------|
| C20 Buffered Signals | 3-9(捕获), 10(消费) | `skills/memory/buffered-signals.md` |
| C21 Entity Extraction | 10(三级抽取) | `skills/memory/entity-extraction-pipeline.md` |
| C22 Three-Band Dedup | 10(归档前) | `skills/memory/deduplication.md` |
| C23 Supersession Chain | 注入(穿透), 10(建链) | `skills/memory/supersession.md` |
| C24 Reasoning Trace | 3-9(捕获), 10(归档) | `skills/memory/reasoning-trace.md` |
| C25 Graph Traversal | 3,4(检索扩展) | `knowledge-map.yaml#graph_traversal` |
| C26 Context Compression | 注入时(降级) | `knowledge-map.yaml#context_compression` |
| C27 Consolidation Ops | Doctor/5任务 | `skills/evolution/consolidation.md` |
| C28 Explicit Scoring | 注入时(排序) | `knowledge-map.yaml#scoring_formula` |
| C29 Schema Adoption | init(新项目) | `skills/memory/schema-adoption.md` |

## 研发基线对照（MIT 信息部 13 项强制要求）

> 来源：小米信息部研发基线 + 测试过程基线（P0/P1 系统强制，ODC 和自研均适用）

| 基线项 | 要求 | 对应 Phase | 门禁映射 |
|--------|------|-----------|---------|
| 1. 任务线上化 | 代码变更可关联需求/缺陷 | Phase 1 | requirement.md 必须有需求号 |
| 2. 识别影响范围 | 技术方案覆盖业务/技术上下游 | Phase 1-2, 2.3 | requirement.md 影响范围 + technical-design.md §1.2 |
| 3. 可观测性 | 方案含监控、告警、业务指标 | Phase 2-3, 2.3 | technical-design.md §8 监控报警 |
| 4. 失效分析 | 方案覆盖风险分析 | Phase 2, 2.3 | technical-design.md §1.3 设计难点 |
| 5. 代码须审查 | MR 有其他人 Approve | Phase 4 | 编码评审 PASS |
| 6. 编码规范 | 每次提交跑 CI + Sonar | Phase 8 | CI status=SUCCESS |
| 7. 落实自测 | 自测证明（截图/覆盖率/单测） | Phase 5-6 | tests>0 + passed=total |
| 8. 上线须评审 | Checklist（上线/回滚/配置/窗口） | Phase 7 [H4] | 人工确认 |
| 9. 测试方案 | P0 项目必须有测试方案，含重难点+全面类型 | Phase 5 | testing SKILL |
| 10. 测试用例须评审 | 所有功能必须有测试用例，产研评审通过 | Phase 6 | 单测评审 PASS |
| 11. 缺陷可追踪 | 线上提报，复现步骤准确，优先级明确 | Phase 10 | retrospective.md |
| 12. 测试报告 | 测试结论 + 交付风险 + 遗留问题确认 | Phase 8 | CI 产出 |
| 13. 线上验证 | 上线后必须验证，测试数据须清理 | Phase 9 | 冒烟通过 + 数据清理确认 |

### 红线（禁止项）

| 红线 | 来源基线 | Harness 强制措施 |
|------|---------|-----------------|
| 无线上化管控的变更上线 | 基线 1 | requirement.md 必须存在才能进 Phase 3 |
| 不 review 直接合并 | 基线 5 | Phase 4 门禁不通过不进 Phase 5 |
| 关闭或跳过 CI | 基线 6 | Phase 8 门禁：禁止 skip-test / --no-verify |
| 不自测、低质量提测 | 基线 7 | Phase 5 门禁：tests>0 |
| 无评审上线 | 基线 8 | [H4] 人工确认点不可跳过 |
| 上线前无有效测试报告 | 基线 12 | Phase 8 CI 必须产出测试结果 |
| 上线不验证 / 测试数据未清理 | 基线 13 | Phase 9 门禁：冒烟通过 + 数据清理确认 |
| 缺陷优先级乱标（一切 P0） | 基线 11 | issues.json severity 按标准定义 |

### L1 代码质量量化标准（通用，所有项目适用）

> 来源：`tools/code-quality/`（L1 企业级代码质量守护能力包）
> 三道防线：Git Hooks (快/确定) → Claude Hooks (深/智能) → CI/CD (全/权威)

#### 编码硬指标（Phase 3 编码 + Phase 4 评审）

| 指标 | 阈值 | 检查方式 | Phase |
|------|------|---------|-------|
| 文件长度 | ≤ 500 行 | Checkstyle / Linter | 3（PostToolUse lint） |
| 方法长度 | ≤ 80 行 | Checkstyle / PMD | 3 |
| 方法参数数 | ≤ 7 个 | Checkstyle | 3 |
| if 嵌套深度 | ≤ 3 层 | Checkstyle | 3 |
| try 嵌套深度 | ≤ 2 层 | Checkstyle | 3 |
| 圈复杂度 | ≤ 15 | PMD | 4（AI Review） |
| 认知复杂度 | ≤ 15 | PMD / SonarLint | 4 |
| 类依赖数 | ≤ 20 | PMD | 4 |

#### Git 提交规范（Phase 7 推送）

| 规则 | 标准 | 检查方式 |
|------|------|---------|
| Commit 格式 | **Conventional Commits**（`feat:` / `fix:` / `chore:` / `docs:` / `refactor:` / `test:`） | Git Hook: commit-msg |
| 密钥检测 | 零硬编码密钥/Token/API Key | Git Hook: pre-commit（detect-secrets） |
| 安全扫描 | 无已知漏洞依赖 | Git Hook: pre-push |

#### CI/SonarQube 门禁（Phase 8 CI 验证）

| 指标 | 阈值 | 卡控方式 |
|------|------|---------|
| 单测通过率 | **100%**（不允许跳过失败测试） | CI Pipeline |
| 新增代码覆盖率 | **≥ 80%** | SonarQube Quality Gate |
| 总体覆盖率 | ≥ 60% | SonarQube Quality Gate |
| 重复代码率 | **≤ 3%** | SonarQube Quality Gate |
| 新增 Blocker 异味 | **0** | SonarQube Quality Gate |
| 新增 Critical 异味 | **0** | SonarQube Quality Gate |
| 新增安全漏洞 | **0** | SonarQube Quality Gate |

#### 语言专项规范（按技术栈自动注入）

| 语言 | Linter | 关键规则 |
|------|--------|---------|
| **Java** | Checkstyle + PMD | equals/hashCode 成对、switch 有 default、禁止字符串 == 比较、空 catch 块、资源关闭、保留异常栈 |
| **TypeScript/React/Vue** | ESLint + Prettier | 禁止 any 类型、禁止 var、严格 ===、禁止 debugger/console、Prettier 格式化 |
| **Python** | Ruff | PEP 8 + Bugbear + 安全检查(Bandit) + 现代语法升级 |
| **PHP** | PHP-CS-Fixer + PHPStan | PSR-12 风格、PHPStan Level 5 静态分析 |

---

## 阶段定义

每阶段四要素：**触发条件 + Skill 加载 + Context 策略 + 质量门禁**

> **全局前置 1（using-superpowers）**：当 `superpowers.enabled=true`（默认）时，每个 Phase 开始前、每次动手前，必须检查是否有适用的 skill 并调用。禁用时仅依赖 Phase 触发器。配置：`harness superpowers on|off` 或 `.harness/config.json`。
>
> **全局前置 2（深度思考 Pre-flight）**：每个 Task 开始前强制执行 Pre-flight Check（`skills/context/SKILL.md`）。产出 High/Medium/Low/Unknown 四级 confidence 清单，驱动 context 加载。**不做 Pre-flight 就开始编码 = 违反流程。**
>
> **全局前置 3（Confidence Gate）**：每个关键决策点（写代码/改配置/调 API）前评估 confidence。LOW 时停止执行并查 Expert Routing 表或向用户提问。**禁止 confidence < HIGH 时猜测执行。**
>
> **全局前置 4（任务类型路由）**：识别任务类型后选对思维框架再动手。不同类型的第一步不同。
>
> **全局前置 5（Plan-Before-Execute 五步子循环）**：每个 Phase 内部在动手前必须完成「理解→目标→计划→执行→验证」五步。不走计划直接动手 = 违反流程。详见 `skills/planning/phase-plan-protocol.md`。
>
> ```
> Phase N 内部子循环（每个 Phase 强制）:
>
> ┌─ 1. Context Load ───────────────────────────────────────────┐
> │  读什么上下文？（前序产出 + Brain 注入 + 主动 grep/Read）     │
> │  产出：已知事实清单（≤5 条关键信息）                          │
> └─────────────────────────────────────────────────────────────┘
>        ↓
> ┌─ 2. Intent Clarify ────────────────────────────────────────┐
> │  本阶段要达成什么具体目标？（一句话）                         │
> │  不确定的点是什么？（列出，查了再动手）                       │
> └─────────────────────────────────────────────────────────────┘
>        ↓
> ┌─ 3. Approach Plan ─────────────────────────────────────────┐
> │  具体步骤是什么？（写出来，不在脑子里）                       │
> │  每步的验证标准是什么？                                      │
> │  什么情况需要停下来重新评估？                                │
> └─────────────────────────────────────────────────────────────┘
>        ↓
> ┌─ 4. Execute ───────────────────────────────────────────────┐
> │  按步骤做。每步完成对照计划验证。                             │
> │  偏离计划时：更新计划再继续，不静默偏离。                     │
> └─────────────────────────────────────────────────────────────┘
>        ↓
> ┌─ 5. Verify ────────────────────────────────────────────────┐
> │  对照 Step 2 的目标验证：达成了吗？                           │
> │  有新发现吗？（写入 Living Spec / 下一 Phase 输入）           │
> └─────────────────────────────────────────────────────────────┘
> ```
>
> **适用规则**：
> - Direct Mode：Step 1-3 可合并为 Karpathy 四问（K1-K4），不需要写文件
> - Full Mode Phase 1-2：Step 1-3 对应 brainstorming → writing-plans 技能（写 plan 文件）
> - Full Mode Phase 3+：Step 3 至少在对话中输出步骤清单，复杂任务写入 phase-state.json
> - 判定"是否需要写 plan 文件"：涉及 ≥3 文件改动 或 ≥2 个不确定点 → 必须写
> - **禁止**：脑子里有计划但不说出来就开始做（「想清楚了我直接做」= 违反流程）

### Phase 0.5: 想法探索（L1/L2 路由触发）

| 要素 | 内容 |
|------|------|
| 触发条件 | 需求分级为 L1 探索型 或 L2 新建型（由 `brainstorming.md` Step 0 路由） |
| Skill 加载 | `skills/planning/idea-exploration.md`（6 追问 / 精简 3 追问）+ `rules/anti-sycophancy.md` |
| 质量门禁 | 追问记录完整 + 2-3 方案已呈现 + 用户已选方案 + 产出 `exploration.md` |

**跳过条件**：L3 增强型 / L4 修复型 直接进入 Phase 1。

**需求分级路由**：详见 `README.md` §需求分级路由。AI 初判后向用户确认，降级保护（只能升级不能降级）。

---

### 任务类型方法论路由（Phase 1 前置）

> WHY: PUA 方法论核心——每种任务类型有独立的解题思路。修 Bug 和做新功能不应该走同一套流程。
> Harness Mode A/B/C 按规模分，本表按类型分——两者正交，先选类型再选模式。

| 任务类型 | 识别信号 | 思维框架 | **第一步** | 对应 Mode |
|---------|---------|---------|-----------|----------|
| **修 Bug** | 报错/异常/不工作/白屏 | RCA 根因分析 + 蓝军自攻击 | **先复现，不猜**（铁律 #23） | B |
| **新功能** | 新增/做一个/加一个 | 质疑→删除→简化→加速→自动化 | **先问"该不该做"** | A 或 B |
| **性能优化** | 慢/卡/超时/OOM | 数据驱动 + A/B 对比 | **先量化基线**（改前数据） | B |
| **代码审查** | review/MR/评审 | 减法优先 + 代码量合理性 | **先看 diff 量是否合理**（铁律 #29） | C |
| **重构** | 重构/优化/整理/code smell | 语义等价性验证 | **先确认不改行为**（AP-046） | B |
| **配置/集成** | 对接/配置/环境/模板 | 搜索先于一切 | **先查已有模式**（AP-071/072） | C |

**使用方法**：Phase 1 需求分析开始时，先匹配任务类型，再选 Mode A/B/C。任务类型决定"用什么思路想"，Mode 决定"花多少功夫分析"。

### Phase 1: 需求分析

| 要素 | 内容 |
|------|------|
| 触发条件 | 收到 PRD / 口头需求 / Issue |
| Skill 加载 | `skills/planning/project-understanding.md`（项目理解，Mode A/B 前置）+ `skills/planning/brainstorming.md`（探索澄清，Step 0 含 L1-L4 路由）+ `skills/planning/idea-exploration.md`（L1/L2 深度探索，由 brainstorming Step 0 触发）+ `skills/planning/spec-format.md`（可测试需求格式）+ `skills/planning/prd-to-ai-ready.md`（写 PRD）+ `wiki/anti-patterns.md#phase-1-2`（反模式检查） |
| 质量门禁 | 验收标准已列出 + 影响范围已标注（基线2）+ 需求号已关联（基线1）+ 风险已标注（基线4）+ 产出 `requirement.md` + **PRD-Code 预对齐检查**（`rules/prd-code-alignment.md` §Phase 1：对每条业务规则验证代码是否一致，偏差写入 requirement.md） |
| **领域知识门禁** | 若任务关键词匹配 AI-INDEX 已索引项目，必须确认 `harness index domain-status` 为 loaded。未加载 → GOV-021 阻断进入 Phase 2。执行 `harness index init-domain <项目名>` 加载。 |

#### Phase 1 三种模式

**模式 A：从 0 到 1 的全新需求**

必须先完整了解业务再动笔：

0. **项目理解**（首次接触仓库时）：加载 `skills/planning/project-understanding.md`，走完六阶段理解流程，产出 `project-understanding.md`，通过 [H0] 理解确认后再进入后续步骤
1. **业务背景**：这个需求属于哪个业务模块？谁在用？
2. **业务现状**：当前系统如何处理？用户现在怎么操作？
3. **痛点**：用户遇到了什么问题？为什么现有方案不够用？
4. **期望效果**：做完之后用户的操作会变成什么样？
5. **写 PRD**：加载 `skills/planning/prd-to-ai-ready.md`，输出结构化附录（功能清单 + 术语表 + Mermaid + Gherkin + 待确认项）
6. **门禁检查 → [H1]**：列出关键业务逻辑，让人工确认

**模式 B：在已有系统上改造/新增功能**

必须先充分了解现有系统再设计方案。**核心纪律：不理解完整链路不动手。**

0. **项目理解**（必须）：加载 `skills/planning/project-understanding.md`，至少完成阶段 1-3（项目扫描→依赖图谱→架构分层）+ 阶段 5（变更影响分析）。产出 `project-understanding.md` 中的关键发现（`U-NN` 编号），后续 requirement.md 必须引用。

1. **业务链路梳理**（必须穷尽，不可遗漏）：
   - 这个数据从哪些入口可以被创建/修改？（页面、接口、导入、定时任务、回调）
   - 列出**所有写入路径**，不只是"最常见的那条"
   - 每条路径标注：前端页面 → API 接口 → Controller → Service → 数据落库
   - 问自己："还有没有我没想到的入口？"

2. **技术架构理解**（找到汇聚点）：
   - 多条业务路径最终汇聚到哪一层？（Controller？Service？Provider？Repo？）
   - 校验/拦截应该加在**汇聚点**，而不是每个入口各加一份
   - 如果汇聚点在另一个仓库 → 该需求的实现仓库可能不是最初以为的那个

3. **技术现状摸底**（逐项确认）：
   - 前端代码仓库 + 相关页面/组件位置
   - 后端代码仓库（可能涉及多个微服务！）
   - API 接口清单（涉及哪些端点）
   - 定时任务（是否有相关 Job/Scheduler）
   - 系统架构（微服务间调用关系、Dubbo/HTTP/MQ）
   - 技术架构（框架版本、中间件、数据库表）
   - 已有的全局配置/开关/枚举

4. **设计方案**：基于汇聚点设计最小改动方案（一处加，所有入口覆盖）

5. **门禁检查 → [H1]**：列出改动逻辑 + **所有受影响的入口路径** + 对现有功能的影响

```
// WHY: VIN 校验首次只在 Pipeline 入口加了校验，遗漏了编辑保存、台账编辑、待验收等 5+ 个入口。
//      根因：没有梳理"所有写入台账的路径"，只看了"最明显的那条"。
//      教训：校验类需求必须找到数据写入的汇聚点，而不是在某一个入口加。
```

#### 模式 B 必须回答的 5 个问题（门禁前）

| # | 问题 | 不回答不过门禁 |
|---|------|--------------|
| 1 | 这个数据有几条写入路径？全列出来了吗？ | 遗漏入口 = 后续线上漏洞 |
| 2 | 这些路径汇聚到哪一层？校验加在汇聚点了吗？ | 分散校验 = 维护噩梦 |
| 3 | 改动涉及几个仓库？都确认了吗？ | 跨仓库需求不提前识别 = 返工 |
| 4 | **汇聚点方法有几个重载？每个重载的调用方是谁？** | 方法名相同 ≠ 同一个入口 |
| 5 | **从 Controller 到最终执行，完整调用链走过了吗？** | 只看方法名不跟调用链 = 遗漏 |

```
// WHY: VIN 校验加在了4参数的 updateAssetAccountWithCheck(req, bizFunc, isCheck, isCateControl)，
//      但 Controller 实际调用的是3参数的 updateAssetAccountWithCheck(req, bizFunc, isCheck)，
//      两个重载是独立实现，不互相调用。结果校验完全没触发。
// 教训: "汇聚点"不能只看方法名，必须追踪到具体方法签名（参数列表）。
//       Java 的方法重载意味着同名方法可能是完全独立的代码路径。
```

**模式 C：明确的小改动需求**

需求方已明确描述了要做什么，改动范围小且清晰：

1. 确认验收标准 + 影响范围 + 风险
2. 输出 `requirement.md`
3. 门禁检查 → [H1]

#### 模式判定规则

| 信号 | 选模式 |
|------|--------|
| "我们要做一个全新的XX功能" | A |
| "现有的XX流程需要加一个YY" / "在已有页面上增加ZZ" | B |
| "把这个字段加个校验" / "这个提示文案改一下" | C |
| 不确定 | 默认 B（了解现状不会错，跳过了解可能踩坑） |

#### Phase 1 [H1] 输出格式（面向业务确认人）

```
## 需求摘要（一句话）
## 关键业务逻辑（用"当…时，系统…"句式）
## 边界行为（什么情况不触发/特殊处理）
## 待确认项（如有歧义）
```

### Phase 2: 需求评审（≤ 3 轮）

| 要素 | 内容 |
|------|------|
| 触发条件 | [H1] 用户确认需求理解正确 |
| Skill 加载 | `skills/review/spec-validation.md`（程序化规格验证，评审前自动执行）+ **`tools/spec-validator/validate-spec.sh`（脚本级强制检查，ERROR 阻塞评审）** + `skills/review/plan-review-template.md`（评判视角）+ `skills/review/multi-perspective.md`（5 视角评审，含战略+设计视角按路由表触发）+ `skills/review/cross-impact-analysis.md`（影响声明）+ `wiki/anti-patterns.md#phase-1-2`（反模式检查） |
| 质量门禁 | 零 P0 + P1 有缓解方案 + 方案含可观测性设计（基线3）+ 方案含失效分析（基线4）+ **跨模块影响声明已填写** + 轮次 ≤ 3 |
| 工具触发 | 评审 PASS 后、[H2] 等待前 → 检查 `tools/feishu-notify/` 是否存在 → 存在则执行 `review-push.md`（创建飞书文档 + 发群消息）→ 不存在则跳过 |
| 升级规则 | 第 3 轮仍有 P0 → 人工决策 |

#### Phase 2 跨模块影响声明（前置分析）

> WHY: V08 五个功能同时修改 `VehicleReserveProviderImpl`，直到实现后才发现三功能改同一文件的合并风险。
> Phase 2 提前声明预期影响范围，为 Phase 4 的实际验证提供对照基线。

**触发条件**：所有需求（不可跳过，简单需求可简化填写）

**产出物**：在 `changes/<req>/requirement.md` 末尾追加 `## 跨模块影响声明` 章节

**必须回答的 4 个问题**：

| # | 问题 | 输出 |
|---|------|------|
| 1 | 本需求改动涉及哪些文件/方法？ | 改动文件清单（文件名 + 方法名） |
| 2 | 这些文件/方法还被哪些其他业务链路调用？ | 受影响链路清单（链路名 + 是否会被本次改动的行为变更影响） |
| 3 | 是否有其他需求同时在修改相同文件？ | 并行改动冲突声明（无 / 有，列出需求编号） |
| 4 | 如果本次改动引入 bug，影响的最大爆炸半径是什么？ | 爆炸半径评估（HIGH/MEDIUM/LOW + 说明） |

**影响等级判定**：

| 等级 | 条件 | 后续动作 |
|------|------|---------|
| HIGH | 改动主流程入口方法 / 影响 3+ 业务链路 / 多需求并行修改同文件 | **[H-IMPACT] 人工确认** — Agent 暂停，等用户确认影响可接受 |
| MEDIUM | 改动被 2 个链路共享的方法 / 有降级路径 | 记录到 requirement.md，Phase 4 重点审查 |
| LOW | 新增代码/独立方法 / 仅影响本链路 | 记录到 requirement.md 即可 |

**输出格式**：

```markdown
## 跨模块影响声明

### 改动文件清单
| 文件 | 改动方法 | 改动类型 |
|------|---------|---------|
| XxxProvider.java | submit() | 修改（插入校验逻辑） |
| XxxAbility.java | buildSuffix() | 新增 |

### 受影响链路
| 链路 | 入口 | 是否受本次改动影响 | 影响说明 |
|------|------|-------------------|---------|
| 车辆预约提交 | POST /submit | ✅ 是（预期行为） | 标题增加车辆编号后缀 |
| 车辆续借提交 | POST /renewal | ✅ 是（预期行为） | 同上 |
| 资产入库 | Pipeline | ❌ 否 | 不经过本方法 |

### 并行改动冲突
- 无 / F-01 和 F-04 同时修改 VehicleReserveProviderImpl

### 爆炸半径评估
- 等级：MEDIUM
- 说明：...

### 需要人工确认的风险（如有）
- [ ] ...
```

### Phase 2.3: 技术设计方案（条件分支 — 重大变更时触发）

> 来源：MIT 技术设计方案（架构+详细）标准 V1.7。较大的重构或新增功能必须输出详细技术设计方案并完成评审。

| 要素 | 内容 |
|------|------|
| 触发条件 | [H2] 用户确认方案可行 + 满足"重大变更"判定规则 |
| Skill 加载 | `skills/planning/technical-design.md`（MIT V1.7 十章模板）+ `skills/review/expert-review-template.md`（专家评审材料） |
| 质量门禁 | 十章全覆盖 + 架构图≥1 + 外部依赖接口已确认 + DB变更方案含回滚SQL + 部署/灰度/回滚方案 + 监控指标已定义 + **专家评审材料 `technical-review.md` 已产出** |
| 产出物 | `changes/<req>/technical-design.md` + `changes/<req>/technical-review.md`（专家评审材料：变更范围 + 数据链路 + 关联影响 + 风险分析 + 确认清单） |

#### 触发条件（满足任一则必须编写技术设计方案）

| 信号 | 说明 |
|------|------|
| 新服务/新模块创建 | 从零搭建一个新服务或新业务模块 |
| 3+ 微服务改动 | 变更横跨 3 个或以上服务 |
| 数据库 schema 变更 | 新建表 / 修改表结构 / 数据迁移 |
| 新外部系统对接 | 首次接入新的外部 API / MQ / RPC |
| 标记为重大重构 | 人工判定 or Phase 1 标注 |

#### 跳过条件（满足全部则跳过，直接进 Phase 2.5/Phase 3）

- 单服务内的功能新增/修改且无 schema 变更
- Bug fix / 配置调整 / UI 调整
- Phase 1 判定为模式 C（明确小改动）

#### Phase 2.3 [H2.3] 输出格式（面向技术评审人）

```
## 技术设计方案评审摘要
## 架构方案（一句话 + 架构图）
## 关键设计决策（为什么这样做而不是那样做）
## 风险与难点（已有应对措施）
## 数据库变更概要（新表/改表 + 回滚方案）
## 待确认项（如有）

等待确认：技术设计方案是否可行？
```

#### Agent 触发判定话术（Phase 2 [H2] 通过后）

```
"需求涉及 [新服务/多服务改动/DB schema变更/新外部对接]，按 MIT 研发基线要求，
需要先输出技术设计方案文档。进入 Phase 2.3。"

或

"需求为单服务内改动，无 schema 变更，跳过技术设计阶段，直接进入 Phase 2.5/Phase 3。"
```

---

### Phase 2.5: 原子任务拆解（复杂需求必须）

> WHY: 盘点计划 E2E 案例（5 分配方式 × 13 文件 × 前后端）一把梭改完无法定位问题。拆为 18 原子任务后每步独立验证，失败精确回退。

#### 触发条件（满足任一则必须拆解）

- 涉及 5+ 文件改动
- 有 3+ 条独立业务路径
- 前后端双端改动
- 存在数据依赖链（A 改完才能验 B）

#### 拆解规则

1. **按依赖关系分层**：后端修复 → 前端清除 → E2E 串联验证
2. **每个原子任务有独立验证方式**：编译/单测/页面渲染/API 响应
3. **失败回退到具体 Task**，不影响已通过的 Task
4. **不了解业务时一问一答确认**，不猜测后动手

#### Agent 建议话术（Phase 1 → Phase 2 过渡时）

```
"这个需求涉及 N 个文件 / M 条路径，建议先拆为原子任务逐个验证，最后串联。
拆解过程中如果我对业务逻辑不确定，会逐个和你确认。"
```

#### 输出格式

```markdown
## 原子任务清单

### 第 N 层：[层名]（[依赖说明]）
| Task | 描述 | 验证方式 | 文件 |
|------|------|---------|------|
| T1   | ...  | ...     | ...  |
```

---

### Phase 3: 编码实现

| 要素 | 内容 |
|------|------|
| 触发条件 | [H2] 用户确认方案可行 |
| **前置步骤** | **① 创建功能分支（GOV-031 强制，见下方）** + **② 目标仓库漂移检查（见下方）** + **③ 领域知识验证（GATE-9, 若匹配 AI-INDEX 项目）** + **④ PRD-Code 对齐三问（见 `rules/prd-code-alignment.md` §Phase 3）** |
| Skill 加载 | `skills/coding/*.md`（执行视角，含 `deep-work-protocol.md` 心智模型+Checkpoint 反思 + `living-spec.md` 活文档）+ `wiki/anti-patterns.md#phase-3`（反模式检查）+ `rules/prd-code-alignment.md`（PRD 对照协议） |
| **L1 实时守护** | **PostToolUse Hook: 每次 Edit/Write 自动增量 lint（语言自动检测）** |
| **门禁1 嵌入** | 每次 commit 前 pre-commit hook 执行增量代码扫描，阻断标准见 `rules/code-quality-gates.md` §门禁1 |
| **门禁2 嵌入** | TDD 强制模式（见下方 TDD 段落），单测在编码循环内产出，不等到 Phase 5 |
| 质量门禁 | 编译通过 + 无新 lint/warning + commit 关联 task + **六层追踪确认** + **意图验证三维 PASS（完整性/正确性/一致性）** + **门禁1 零阻断项** + **门禁2 TDD 循环覆盖率达标** |

#### Knowledge Engine 影响分析门禁（EAM 后端代码变更）

> 条件: 即将修改 `docs/reference/eam-backend/` 下的代码
> 动作: 对每个要修改的方法/类执行 `knowledge-engine:impact_analysis("{符号名}", "{修改描述}")`
> 门禁: 影响域未确认前禁止提交代码
> 与铁律#3联动: "改A查B" 由引擎自动执行

#### Phase 3 L1 实时质量守护（PostToolUse Hook）

> 来源：`tools/code-quality/hooks/post-edit-lint.sh`
> Agent 每次调用 Edit/Write 工具后，hook 自动对变更文件增量 lint。
> lint 结果作为 tool output 反馈给 Agent，Agent 在下一步自动修复。
> **不阻断编码流程**——只提示，不拒绝。Agent 自行判断是否修复。

**行为协议**：
```
Agent Edit file.java
  → PostToolUse hook 触发
  → detect-lang.sh 识别 Java
  → lint-java.sh 执行 Checkstyle + PMD
  → lint 结果输出到 stdout（Agent 可见）
  → Agent 判断：warning → 继续 / error → 自动修复
```

**支持的语言和工具**：

| 语言 | Linter | 配置 |
|------|--------|------|
| Java | Checkstyle + PMD | `tools/code-quality/config/java/` |
| TS/JS/React/Vue | ESLint + Prettier | `tools/code-quality/config/frontend/` |
| Python | Ruff | `tools/code-quality/config/python/ruff.toml` |
| PHP | PHP-CS-Fixer + PHPStan | `tools/code-quality/config/php/` |

**降级**：lint 工具未安装时 hook 静默退出（exit 0），不阻塞编码。

#### Phase 3 TDD 强制模式（门禁2 嵌入编码循环）

> 来源：FDE 四阶段门禁 · 门禁2。
> 核心思想：单测不是编码结束后的「补作业」，而是编码过程中的「驱动引擎」。
> 阈值标准：`rules/code-quality-gates.md` §门禁2。

**触发条件**（满足任一则强制 TDD）：

| 信号 | 说明 |
|------|------|
| 含业务逻辑的新增/修改 | Service / Provider / Ability 层代码 |
| 核心模块（资金/权限/状态机） | 覆盖率要求 ≥ 90% |
| writing-plans 标注为「需单测」的任务 | Phase 2.5 拆解时已标注 |
| Full Mode 所有任务 | Full Mode 默认强制 TDD |

**跳过条件**（同时满足才能跳过）：

- Direct Mode + 纯配置变更/UI 文案修改
- 无业务逻辑（只有 CRUD 透传）
- 用户显式说「不需要测试」

**TDD 循环（每个编码任务内重复）**：

```
┌─────────────────────────────────────────────────────────────┐
│  RED: 写一个失败的测试（描述期望行为）                          │
│    ↓                                                        │
│  GREEN: 写最小实现让测试通过（不多不少）                        │
│    ↓                                                        │
│  REFACTOR: 消除重复/改善结构（测试仍绿）                       │
│    ↓                                                        │
│  门禁1: pre-commit 代码扫描 → 通过才能 commit                 │
│    ↓                                                        │
│  COMMIT: 提交（测试代码和实现代码一起）                         │
│    ↓                                                        │
│  下一个任务 → 回到 RED                                       │
└─────────────────────────────────────────────────────────────┘
```

**与 Phase 5 的关系**：

| 阶段 | 产出 | 职责 |
|------|------|------|
| Phase 3（TDD 循环） | 单元测试 + 实现代码（同步产出） | 驱动实现、验证逻辑 |
| Phase 5（补充覆盖） | 补充边界/异常/覆盖率不足的用例 | 达到覆盖率阈值、补充 BCDE |
| Phase 6（评审） | — | 审查测试质量 |

**Phase 3 结束时覆盖率检查**：

```
运行覆盖率工具（jacoco / istanbul / coverage.py）
→ 新增代码覆盖率 ≥ 80% → 继续
→ 新增代码覆盖率 < 80% → Phase 5 必须补充
→ 核心模块 < 90% → Phase 5 必须补充
```

**禁止行为**：

| 禁止 | 原因 |
|------|------|
| 先写完所有实现再补测试 | 违反 TDD 原则，测试沦为「补作业」 |
| 测试只验证不抛异常 | 空断言不计入覆盖 |
| skip/pending 标记跳过 | MR 中的 skip 计为未覆盖 |
| Mock 一切外部依赖后只测胶水代码 | 失去验证意义 |

---

#### Phase 3 前置：创建功能分支（GOV-031 强制）

> WHY: 执行快照暴露——编码直接在 master 进行，Phase 7 才补救切分支。
> master 上的脏代码如果遇到紧急回滚会造成冲突，且无法隔离不同需求的改动。

**Full Mode 强制执行，Direct Mode 建议执行：**

```
1. 确认当前 HEAD 不在 main/master 上:
   current=$(git branch --show-current)
   IF current == "main" || current == "master":
     → 必须创建分支，不能直接在主干编码

2. 创建功能分支（命名规范）:
   git checkout -b feat/<需求目录名>
   # 示例: git checkout -b feat/high-freq-visit-block-adjust

3. 记录到 phase-state.json:
   "branch": "feat/<需求目录名>",
   "createdAt": "<ISO timestamp>"

4. 验证:
   git branch --show-current → 必须不是 main/master
```

**命名规范**：
- 新功能: `feat/<需求目录名>`
- Bug 修复: `fix/<需求目录名>`
- 重构: `refactor/<需求目录名>`

**跳过条件**：已经在正确的功能分支上（如从上次会话恢复）。

---

#### Phase 3 前置：六层深度追踪（编码和修 bug 都适用）

> WHY: V08 连续踩坑——`fieldValues.get(0)` 只取单值导致批量搜索全空、`GlobalValRes.value` 被误写为 `getGlobalValue()`、UMS 模板变量名 `buttonContent/url` 被猜成 `buttonText/buttonUrl`。全是没追到运行时实际行为的结果。
> **铁律 #19**：不仅 Phase 0 项目理解要追到底，编码实现和修复 bug 时每碰一个调用点都必须追到底。

**编码/修 bug 前，对涉及的每个调用点必须确认以下六层：**

| 层 | 追踪什么 | 不追的后果（实例） |
|----|---------|-----------------|
| 服务层 | Provider/Service 方法的业务语义 | 调错方法或漏掉前置校验 |
| SQL 生成层 | 框架如何将 fieldCode/queryCond 转为 SQL；fieldValues 单值还是多值 | `equalsTo` 只取 `get(0)` 导致批量搜索失败 |
| 方法签名层 | 参数类型、返回类型、Lombok 生成的 getter 名 | `getValue()` 被猜成 `getGlobalValue()` 编译失败 |
| Hook / 模板内部行为层 | 前端 hook 是否注入额外参数；消息模板的变量名 | UMS 模板 `buttonContent` 被猜成 `buttonText` 导致按钮无文案 |
| 多流程链路层 | 同一入口的多种业务路径（预约/续借/归还）是否共享逻辑 | 只改了预约漏了续借归还 |
| 执行门禁层 | 运行时条件过滤（type/status 枚举）、权限校验、配置开关 | 缺 `reserveType` 过滤导致误命中归还记录 |

**操作**：对每个调用点写一行确认记录（心里确认即可，不需要文档化）：
- `✓ equalsTo 取 fieldValues 全部值` 或 `✗ equalsTo 只取 get(0) → 需改为 IN`
- `✓ GlobalValRes getter 是 getValue()` 或 `✗ 是 getGlobalValue() → 需确认`

#### Phase 3 前置：目标仓库状态同步（多仓库防错）

> WHY: 2026-05-11 inventory-engine-v2 审查中连续 3 次踩坑 — 修改了 eam-v2 已 revert 的 context-path、
> 重复实现 eam-v2 已有的 OR fix、7/9 文件漂移未察觉。根因：没有在动手前检查目标仓库状态。

**进入编码前，如果模块注册在 `Harness/repo-map.yaml` 中且 source-of-truth ≠ pm-context，必须执行：**

```
1. 读取 Harness/repo-map.yaml → 确认 remote、branch、key-configs
2. git fetch <remote> --quiet
3. 对本次变更涉及的每个文件执行：
   diff <(git show <remote>/<branch>:<path>) <module-path>/<path>
4. 如果存在漂移：
   - 目标仓库有本地没有的改动 → 暂停，先 pull 目标仓库代码到 pm-context
   - 本地有目标仓库没有的改动 → 确认这些改动是否已被目标仓库有意移除
5. 检查 key-configs 各值：
   grep <pattern> <(git show <remote>/<branch>:<file>)
   与 expected 值对比，不匹配则暂停确认
```

**跳过条件**：source-of-truth = pm-context 的模块（如 feishu-bot）不需要检查。

### Phase 4: 编码评审（≤ 2 轮）

> **Inner Loop 钩子（编码评审失败时）**：当 evaluator 判定本 phase verify 失败（`passed=false`），由 `bin/inner-loop.sh --run --phase-state <ps.json> --phase 4` 自动接管：
> - 反思 + 重试 ≤ 2 次（scope=micro）→ 再次 verify；通过则恢复主流程
> - scope=pivot 或 max-retries 命中 → `status=blocked`，发出 `inner-loop-pivot` 事件，按铁律 #4 "2 次换路" 升级人工
> - 反思双次幻觉 → `reflection-double-hallucinated-pivot`，同样阻断
>
> 详见 `evolution/reflexion.md §1`。开关：`config/evaluators.yaml: inner_loop_enabled`，禁用时 router exit 90 直接走老路径。

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 3 门禁通过 |
| Skill 加载 | `skills/review/code-review-template.md` + `skills/review/cross-impact-analysis.md` + `skills/review/expert-review-template.md` + `wiki/anti-patterns.md#phase-4` |
| **L1 六维审查** | **`rules/coding-standards.md` §L1 六维检查** — 安全/性能/逻辑/可维护性/SonarLint/反模式（通用规范，L1 常驻上下文） |
| **独立 Verifier** | Claude 完成六层追踪 + L1 六维审查后，**必须调用 `/codex review` 做独立二审** |
| 质量门禁 | 零 P0 + P1 ≤ 2 + **L1 六维零 CRITICAL** + **跨模块影响验证 PASS** + **Codex 独立评审 PASS** + **抗合理化检查 PASS**（`rules/anti-rationalization.md`）+ 轮次 ≤ 2 |
| 工具触发 | 评审 PASS 后 → `tools/feishu-notify/code-review-push.md`（推群） |
| 升级规则 | 第 2 轮仍有 P0 → 人工决策 |

#### Phase 4 独立 Verifier 协议（C1 多角色物理隔离）

> WHY: V08 能力评审 C1 得分 5/10——手动模式下 Coordinator/Executor/Verifier 是同一个 AI，
> "自己审自己"无法发现自身盲区。引入 Codex CLI 作为物理隔离的独立 Verifier。

**执行流程**：

```
Step 1: Claude 完成六层追踪 + 跨模块影响验证 → 产出 code-review-draft.md
Step 2: 调用 /codex review → Codex 独立审查 diff（不看 Claude 的评审）
Step 3: 交叉比对：
  - Claude 发现 + Codex 发现 → 去重合并
  - Codex 独有发现 → 标记 [CODEX] 追加到 code-review.md
  - Claude 独有发现 → 保留
  - 两者矛盾的发现 → 标记 [CONFLICT] 提交人工裁决
Step 4: 合并后的 code-review.md 才是最终评审结论
```

#### Phase 4 主动扩展检查（能动性提升）

> WHY: CASE-069 续借驾照校验遗漏——六层追踪只检查了预约 submit 有校验就标 ✅，
> 没有主动扩展检查其他 Provider 的 submit。被动检查清单不够，需要主动扩展。

**六层追踪完成后，追加以下主动扩展（每条都问一遍）**：

| 改了什么 | 主动扩展问 | 实例 |
|---------|----------|------|
| 改了一个 Provider 的方法 | 其他 Provider 是否有同名/同类方法需要同步改？ | F-04 改了预约 submit → 续借 submit 也要改 |
| 修了一个 bug | 同文件同模式的其他位置是否存在相同 bug？ | VIN 校验修了 4 参数重载 → 3 参数重载也要修 |
| 加了一个校验 | 其他入口（续借/归还/编辑/导入）是否也需要？ | 驾照校验不止预约，续借也需要 |
| 新增了一个配置 | 已有配置模式是否一致？（yml/Nacos/代码默认值） | 模板 ID 应该用 yml profile 不是 @Value |
| 改了一个 SQL | 查的字段和传入参数语义一致吗？重载方法都覆盖了吗？ | queryBySn 查的确实是 sn 不是 asset_code |

**产出**：在 code-review.md 追加"主动扩展检查"章节，逐条标注 ✅ 已检查 / ⚠️ 发现问题。

#### Phase 4 代码量合理性检查（铁律 #29）

> WHY: V08 五个 Feature 产出 30 文件 +837 行，评审时追问"需要改这么多吗"才逐文件审视。
> 代码量过大可能意味着：过度工程 / 职责不清 / 可以复用但没复用 / 混入了无关改动。

**Phase 4 评审必须执行代码量合理性检查：**

```
Step 1: 统计变更量级
  → git diff --stat master..feat/<branch> | tail -1
  → 按类别拆分：业务代码 / 测试 / 配置 / CI / 接口定义

Step 2: 与需求复杂度对比
  → N 个 Feature / M 条验收标准 → 预期改动规模
  → 每个 Feature 对应的文件和行数是否合理

Step 3: 逐文件审视
  → 新建文件 > 100 行的：是否可拆分？是否有通用方法可复用？
  → 改动 > 50 行的已有文件：是否多个功能堆在一个文件？
  → 临时代码（调试接口/TODO/注释代码）：上线前必须清理

Step 4: 产出合理性结论
  → 在 code-review.md 中追加"代码量合理性"章节
  → 标注：总量合理 / 可精简 N 行（原因） / 有临时代码需清理
```

**参考基准**（经验值，非硬门禁）：

| 需求类型 | 预期改动 |
|---------|---------|
| 单功能点（校验/字段/展示） | 1-3 文件，30-100 行 |
| 中等功能（定时任务/表单扩展） | 3-8 文件，100-300 行 |
| 复杂功能（跨服务/新集成） | 8-15 文件，200-500 行 |
| 超出预期时 | 必须解释原因或拆分 |

**Codex 评审 FAIL 时**：
- Codex FAIL + Claude PASS → 检查 Codex 的 FAIL 原因，合理则回退 Phase 3 修复
- Codex PASS + Claude FAIL → 以 Claude 六层追踪为准（更深入）
- 两者都 FAIL → Phase 3 回退

**降级**：Codex CLI 不可用时，标记"独立 Verifier 未执行"到 code-review.md，不阻塞但在 [H3] 时告知用户。

#### Phase 4 跨模块影响验证（六层深度追踪）

> WHY: V08 审查发现 `VehicleReserveProviderImpl` 被 3 个功能同时修改，F-04 驾照校验在 submit 主路径插入硬拦截导致数据工场不可用时预约全阻断，
> F-02 VIN 校验的第三方同步路径可能被误拦截。这些问题只有在代码写完后逐层追踪才能发现。

**触发条件**：每次 Phase 4 编码评审（不可跳过）。简单改动可简化填写，但必须有结论。

**操作：对 diff 中每个改动方法，执行六层追踪验证并与 Phase 2 影响声明对照**

| 层 | 验证什么 | 验证方法 | 失败 = P 几 |
|----|---------|---------|-----------|
| **服务层** | 改动的 Provider/Service 方法的所有调用方是否已识别 | `grep -rn "methodName" --include="*.java"` 查所有调用 | 遗漏调用方 = **P0** |
| **SQL 生成层** | 新增/修改的查询条件是否正确转 SQL；fieldValues 单值 vs 多值 | 读框架代码确认 SQL 生成逻辑；走读 QueryCondition 处理 | SQL 逻辑错误 = **P0** |
| **方法签名层** | 改动方法的重载是否全部覆盖；Lombok getter 名是否正确 | 检查同名方法的所有重载签名 + 参数差异 | 重载遗漏 = **P0** |
| **Hook/模板内部行为层** | 前端 hook 注入的额外参数、UMS 模板变量名、BPM 表单字段名是否匹配 | 读模板定义确认变量名；读 hook 源码确认注入行为 | 变量名不匹配 = **P1** |
| **多流程链路层** | 同一入口的多种业务路径（如预约/续借/归还）是否都被覆盖 | 画出 entrypoint → 各 flow → 汇聚点的调用图 | 链路遗漏 = **P1** |
| **执行门禁层** | 运行时条件过滤（type/status 枚举、权限、配置开关）是否正确 | 确认枚举值、配置 key、过滤条件是否与 DB 数据匹配 | 过滤条件错误 = **P0** |

**额外检查维度**（超出六层追踪的跨模块专项）：

| 维度 | 验证什么 | 判定 |
|------|---------|------|
| **并行改动冲突** | 多个功能同时修改同一文件时，行号是否重叠、逻辑是否互斥 | 有冲突 → **[H-IMPACT] 人工确认** |
| **无关模块污染** | diff 中是否包含不在 requirement.md 影响声明中的文件 | 有无关文件 → **P0**（要么移除，要么补充声明） |
| **爆炸半径验证** | Phase 2 声明的爆炸半径 vs 实际代码是否一致 | 实际 > 声明 → **P1**（更新声明 + 人工确认） |
| **降级路径验证** | 新增的外部依赖（如数据工场）不可用时，系统是否仍可正常运行 | 硬拦截无降级开关 → **P1**（需评估是否可接受） |
| **内部数据转换** | DTO→Entity、Entity→VO 等转换过程中字段是否丢失或错位 | 读 converter/mapper 代码逐字段确认 | 转换遗漏 = **P1** |

**与 Phase 2 影响声明的对照**：

```
Phase 4 必须执行：
1. 打开 changes/<req>/requirement.md 的「跨模块影响声明」章节
2. 逐行对照实际代码 diff：
   - 声明了但没改的文件 → 确认是否还需要改（遗漏 or 设计变更）
   - 改了但没声明的文件 → P0（无关改动或声明遗漏）
   - 受影响链路的实际影响 vs 声明是否一致
3. 输出「影响验证结论」到 review-vN.md
```

**输出格式**（追加到 Phase 4 review 中）：

```markdown
### 跨模块影响验证

**与 Phase 2 声明对照**：
- 声明文件 {N} 个 / 实际改动文件 {M} 个 / 差异：{说明}
- 声明链路 {N} 条 / 实际受影响 {M} 条 / 差异：{说明}

**六层追踪结果**：

| 层 | 结果 | 发现 |
|----|------|------|
| 服务层 | ✅/⚠️/❌ | ... |
| SQL 生成层 | ✅/⚠️/❌ | ... |
| 方法签名层 | ✅/⚠️/❌ | ... |
| Hook/模板层 | ✅/⚠️/❌ | ... |
| 多流程链路层 | ✅/⚠️/❌ | ... |
| 执行门禁层 | ✅/⚠️/❌ | ... |

**跨模块专项**：

| 维度 | 结果 | 发现 |
|------|------|------|
| 并行改动冲突 | 无/有 | ... |
| 无关模块污染 | 无/有 | ... |
| 爆炸半径一致性 | 一致/偏差 | ... |
| 降级路径 | 有/无 | ... |
| 内部数据转换 | 正确/异常 | ... |

**影响验证判定**：PASS / FAIL（原因：...）
**需人工确认项**（如有）：
- [ ] ...
```

### Phase 5: 单测编写

| 要素 | 内容 |
|------|------|
| 触发条件 | [H3] 用户确认编码质量达标 |
| Skill 加载 | `skills/testing/*.md`（执行视角）+ `skills/testing/interface-testing.md`（**门禁3：接口测试 5 维度**，涉及接口变更时强制加载）+ `skills/testing/case-generator.md`（**用例生成：七维分析→测试点→结构化用例**，新功能/流程变更时强制加载）+ `skills/review/expert-review-template.md`（专家评审材料）+ `wiki/anti-patterns.md#phase-5`（反模式检查） |
| **门禁3 嵌入** | 涉及接口新增/变更的任务，必须按 `interface-testing.md` 5 维度编写接口测试，阈值见 `rules/code-quality-gates.md` §门禁3 |
| **用例生成门禁** | 新功能/业务流程变更时，必须按 `case-generator.md` 生成结构化用例：P0 功能三件套（主成功+边界+异常）+ 覆盖矩阵 + 综合得分 ≥ 7.0 |
| 质量门禁 | 改了哪个接口测哪个 + 真实数据结构 + 全过 + tests>0（基线7）+ 可作为自测证明（基线7）+ **门禁3 覆盖矩阵 PASS（涉及接口时）** + **专家评审材料 `test-review.md` 已产出（功能点覆盖 + 链路覆盖 + 数据源覆盖 + 边界场景 + 确认清单）** |

#### Phase 5 MIT 单测规范（Java 服务端）

> 来源：[MIT 服务端单测实战快速入门](https://mi.feishu.cn/docx/W5yAdWukdokfbzxXO3jcoFU1nQg) · [Java单元测试规约](https://mi.feishu.cn/docx/SlnXdtFChoKbrgx8P32c65eanlc) · [MR合并管控说明](https://mi.feishu.cn/wiki/JRDMwCZl2i0MIcky1nocJRWJnKc)
> 完整参考：`wiki/mit-unit-test-standards.md`（16 条规约 + CI 模板 + 覆盖率三阶段监控 + MR 问题排查）
>
> **MR 合并门禁（全部满足才允许合并）**：
> 1. CI 流水线成功
> 2. 单测用例 **100% 通过**（存量+增量）
> 3. 变更代码行覆盖率 **≥ 60%**（Sonar CI 兜底）/ **≥ 80%**（本地门禁2 标准，更严格）
> 4. Sonar 质量门禁通过
> 5. MR 已关联 Meego 需求

**单测原则（强制）**：

| 原则 | 要求 |
|------|------|
| **Automatic（自动化）** | 全自动执行，非交互式。禁止 System.out，必须用 assert |
| **Independent（独立）** | 每个方法独立测试；用例之间无依赖；外部资源全部 Mock |
| **Repeatable（可重复）** | 可稳定重复运行；每次结果相同；用 Mock/embedded 消除环境依赖 |

**用例设计原则（推荐 BCDE）**：

| 原则 | 要求 |
|------|------|
| **B**order | 边界值测试（循环边界、特殊取值、时间点、数据顺序） |
| **C**orrect | 正确输入，验证预期结果 |
| **D**esign | 结合设计文档编写用例 |
| **E**rror | 强制错误输入（非法数据、异常流程），验证预期错误处理 |

**AAA 编写模式（每个测试方法必须包含）**：

```
Arrange（准备数据）→ Act（执行被测方法）→ Assert（断言验证）
```

```java
@Test
void getList_Success() {
    // Arrange
    when(provider.selectList(any())).thenReturn(Result.success(mockData));
    // Act
    List<String> result = service.getList();
    // Assert
    assertNotNull(result);
    assertEquals(1, result.size());
}
```

**Mock 技术栈**：

| 依赖类型 | Mock 方案 | 说明 |
|---------|----------|------|
| 数据库 | H2 内存数据库 | `spring.datasource.url=jdbc:h2:mem:test` |
| Redis | embedded-redis | MIT UnitTest-Tool 提供 |
| RocketMQ | rocketMQ mock | MIT UnitTest-Tool 提供 |
| 外部 RPC/HTTP | Mockito `@Mock` + `when().thenReturn()` | 标准 Mockito 用法 |

**AI 辅助工具（推荐）**：

| 工具 | 用途 |
|------|------|
| Cursor | AI 生成单测代码（基础+进阶） |
| unitAgent（小米自研） | 自动分析代码生成单测 |
| Claude Code | Harness Phase 5 自动生成（当前工具） |

**覆盖度要求**（阈值源：`rules/code-quality-gates.md` §门禁2）：

| 维度 | 最低要求 | 说明 |
|------|---------|------|
| 新增代码行覆盖率 | **≥ 80%**（门禁2 阻断标准） | Sonar 卡控阈值 60% 为 CI 兜底，本地门禁标准更高 |
| 核心模块覆盖率 | **≥ 90%** | 资金、权限、核心业务逻辑 |
| 整体覆盖率变化 | 不得下降 > 2% | 防止大量无测试代码稀释 |
| 主路径（happy path） | 必须覆盖 | — |
| 边界值 / 空值 / 异常 | 至少各一个 | — |
| 改了哪个方法 | 测哪个方法 | 不写不相关的测试凑覆盖率 |

### Phase 6: 单测评审

> **Inner Loop 钩子（单测评审失败时）**：当 evaluator 判定测试覆盖/AAA 结构/mock 滥用 verify 失败，由 `bin/inner-loop.sh --run --phase-state <ps.json> --phase 6` 自动接管：
> - micro retry（补 case / 调断言）≤ 2 次 → 通过则进入 Phase 6.5
> - scope=pivot（如 mock 设计有根本性缺陷）→ blocked 并升级
>
> 详见 `evolution/reflexion.md §1`。

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 5 门禁通过 |
| Skill 加载 | `skills/review/code-review-template.md`（评判视角） |
| 质量门禁 | 主路径 + 边界 + 异常覆盖；无 mock 掩盖问题；符合 AAA 模式；独立可重复 |

#### Phase 6 单测评审检查清单

- [ ] 每个测试方法符合 AAA 模式（Arrange/Act/Assert 清晰分离）
- [ ] 外部依赖全部 Mock（不依赖真实 DB/Redis/MQ）
- [ ] 测试可独立运行（不依赖其他测试的执行顺序）
- [ ] 断言验证的是业务逻辑（不只是 `assertNotNull`）
- [ ] 覆盖主路径 + 至少一个边界 + 至少一个异常场景
- [ ] 无"为凑覆盖率"的无效测试（空方法、无断言、只调不验）

### Phase 6.5: 单测执行

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 6 单测评审 PASS |
| Skill 加载 | `skills/coding/verification.md` + `skills/testing/anti-pattern-driven-testing.md` |
| 质量门禁 | `status=SUCCESS` + `tests>0` + `passed=total` + **新增代码行覆盖率 ≥ 80%（门禁2）** + 核心模块 ≥ 90% + 展示完整命令输出（反幻觉第五层） |

**执行要求**：
- 必须实际运行测试命令（`mvn test` / `vitest run` / 对应框架命令）
- 必须展示完整输出（不可说"应该能过"）
- 全量测试通过（不只是新增的测试，旧测试也不能回归）
- **Java 项目**：运行 `mvn test jacoco:report`，检查 `target/site/jacoco/index.html` 确认覆盖率
- 失败 → 回退到 Phase 5 修复测试或 Phase 3 修复代码

### Phase 6.8: 本地 E2E 测试（门禁4，提交 MR 前最后防线）

> 来源：FDE 四阶段门禁 · 门禁4。
> 从用户视角验证完整业务流程，确保提交 Review 的代码已具备基本质量。
> 阈值标准：`rules/code-quality-gates.md` §门禁4。

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 6.5 单测执行通过 + 门禁1/2/3 全部 PASS |
| 前置条件 | 门禁1（代码扫描）✅ + 门禁2（单元测试）✅ + 门禁3（接口测试，如适用）✅ |
| Skill 加载 | `skills/testing/browser-qa.md` + `skills/testing/interface-testing.md`（验证覆盖矩阵）+ **`skills/testing/e2e-protocols.md`（协议路由：A/B/C/D 按平台自动选择验证方式）** |
| 质量门禁 | **P0 用例 100% 通过** + P1 用例 ≥ 95% + 新功能有对应 E2E 用例 + 回归用例无新增失败 |
| 工具依赖 | Chrome DevTools MCP（前端） / curl + jq（纯后端） |

#### 触发条件判定

| 任务类型 | E2E 要求 | 最小验证 |
|---------|---------|---------|
| 新功能（Full Mode） | **强制** | P0 场景编写并执行 E2E |
| Bug 修复 | **强制** | 复现用例 → 修复 → 确认通过 |
| 纯后端接口改动 | **强制**（API 级） | curl 验证核心流程（≥ 1 条命令输出） |
| 纯前端展示改动 | **强制**（浏览器级） | 打开页面 + 操作 + 截图证据 |
| Direct Mode ≤ 3 文件 | **强制最小验证** | 至少 1 条验证命令输出（curl/编译/页面截图） |
| **仅跳过**：纯文案/纯注释/纯空行 | 可跳过 | 在 MR 描述中说明「纯文案改动，无功能影响」 |

> **收紧规则**：原「配置变更/Direct Mode → 推荐」已升级为「强制最小验证」。
> 只有**纯文案/纯注释/纯空行**改动才允许跳过门禁4，其他一律需要验证证据。
> 无证据 = 无法进入 Phase 7。

#### P0/P1/P2 用例分级

| 级别 | 定义 | 本地执行要求 |
|------|------|------------|
| **P0** | 核心业务 happy path（没有这个功能就不能上线） | 必须自动化执行，100% 通过 |
| **P1** | 关键异常路径（用户容易踩到的错误场景） | 尽量执行，≥ 95% 通过，不足需说明 |
| **P2** | 辅助功能（边缘场景、极端数据） | 可跳过，标注「Phase 9 验证」 |

#### 不同技术栈的执行方式

| 技术栈 | E2E 工具 | 执行方式 | 必须产出 |
|--------|---------|---------|---------|
| Java 后端 | curl + jq / RestAssured | 本地启动服务 → API 调用链验证 | 命令输出 |
| Java 后端（有鉴权拦截） | 部署 test 环境 | **推送 test 分支 → test 环境验证** | test 环境验证截图/curl 输出 |
| 前端 (React/Vue) | whistle 代理 + Chrome 代理模式 | **启动 dev server → 代理配置 → 页面交互验证** | 截图证据 |
| 前端（有鉴权拦截） | whistle + test 环境鉴权 | **whistle 代理到 test API → 本地前端 → 交互验证** | 截图 + whistle Network 确认 |
| 小程序 (Taro/飞书tt) | 飞书开发者工具 | **taro build → 导入 dist → 开发者工具预览交互** | 开发者工具截图 |
| 移动端 (H5/RN) | 真机/模拟器 + DevTools | **启动本地服务 → 设备预览 → 交互验证** | 截图证据 |
| 全栈 | 上述组合 | 后端 API + 前端页面双验证 | 命令输出 + 截图 |

**前端/移动端/小程序 强制本地验证协议**：

> 代码写完不等于功能完成。前端/移动端/小程序必须**启动本地环境亲眼验证**后才能推送。

##### 协议 A：前端 Web 项目验证（含鉴权场景）

| 步骤 | 动作 | 验证点 |
|------|------|--------|
| 1. 启动 | `npm run start` / `npm run dev` | 编译无报错，dev server 启动成功 |
| 2. 代理配置 | whistle 代理规则：`e.test.mioffice.cn/xxx http://localhost:PORT` | whistle Network 中 ServerIP = 127.0.0.1 |
| 3. Chrome 代理 | `chrome --proxy-server="http://localhost:8888"` | 请求经过 whistle 转发 |
| 4. 鉴权通过 | 通过 test 环境登录态访问页面 | 页面正常渲染，非 401/403 |
| 5. 交互验证 | 执行核心操作路径 | 交互响应符合预期，无 JS 报错 |
| 6. 截图 | 对关键状态截图 | 写入 `e2e-evidence.md` |

##### 协议 B：飞书小程序验证

| 步骤 | 动作 | 验证点 |
|------|------|--------|
| 1. 环境 | Node 16 + Python 2.7（node-sass 依赖） | `node --version` = v16.x |
| 2. 编译 | `npm run start`（watch 模式）或 `npm run build:test` | 输出 `✅ 编译成功`，`dist/app.json` 存在 |
| 3. 导入 | 飞书开发者工具 → 导入项目 → 选择 `dist/` 目录 | 模拟器中小程序正常加载 |
| 4. 交互 | 在模拟器中执行改动涉及的操作路径 | 功能正常，无报错 |
| 5. 截图 | 对开发者工具模拟器截图 | 写入 `e2e-evidence.md` |

##### 协议 C：后端（有鉴权拦截，无法本地直接验证 API）

| 步骤 | 动作 | 验证点 |
|------|------|--------|
| 1. 本地编译 | `mvn package` / `gradle build` 通过 | 编译零错误 |
| 2. 推送 test | 推送到 test 分支，触发 test 环境部署 | 部署成功 |
| 3. test 验证 | 通过 test 环境 URL + 鉴权 cookie 验证 API | 接口返回预期数据 |
| 4. 证据 | curl 输出或浏览器 Network 截图 | 写入 `e2e-evidence.md` |

> **注意**：协议 C 的 Phase 7（推送 test）与 Phase 6.8 验证存在先后依赖——
> 此时 Phase 6.8 的编译验证先完成，标记 `phases["6.8"].verifyMode = "test_env"`，
> Phase 7 推送后在 Phase 9 进行完整功能验证。

##### 通用步骤（所有协议共同）

| 步骤 | 动作 | 验证点 |
|------|------|--------|
| 1. 启动 | 对应协议的启动命令 | 编译无报错，环境可访问 |
| 2. 导航 | 打开改动涉及的页面/路由 | 页面正常渲染，无白屏/崩溃 |
| 3. 交互 | 执行核心操作路径（新增/编辑/删除/提交等） | 交互响应符合预期 |
| 4. 边界 | 空值/异常输入/网络异常场景 | 错误提示友好，不崩溃 |
| 5. 截图 | 对关键状态截图作为证据 | 写入 `e2e-evidence.md` |

**跳过条件**：仅**纯文案/注释改动**可跳过。纯后端接口改动如果能本地 curl 验证则用 curl，否则走协议 C。

#### 执行流程

```
Step 1: 确认门禁1/2/3状态
  → 读取 phase-state.json → gates.gate1/gate2/gate3 = PASS
  → 任一未通过 → 阻断，回退修复

Step 2: 识别 P0 场景
  → 从 requirement.md 验收标准提取核心路径
  → 每条验收标准至少对应 1 个 P0 用例

Step 3: 编写/执行 E2E
  → 新功能：编写 P0 用例 → 执行 → 记录结果
  → Bug 修复：编写复现用例 → 确认修复前失败 → 修复后通过
  → 回归：跑已有 P0 集 → 无新增失败

Step 3.5: PRD 对照检查（硬约束，不可跳过）
  → 对每个观察到的数据特征执行 PRD 对比
  → 协议详见 rules/prd-code-alignment.md §Phase 6.8
  → 每条验证点必须有：观察值 + PRD原文引用 + 判定(PASS/DEVIATION)
  → "PRD 定义"字段为空 → 该验证点无效
  → 出现 DEVIATION → 立即停止 E2E，进入根因分析
  → 禁止：用代码行为解释PRD / 猜测"可能是手动的" / 循环论证

Step 4: 输出证据
  → 命令输出 / 截图 / Network 抓包
  → 写入 changes/<req>/e2e-evidence.md
  → 每条必须含 PRD 原文引用（见 prd-code-alignment.md 模板）

Step 5: 门禁判定
  → P0 100% + P1 ≥ 95% → PASS → 进入 Phase 7
  → P0 < 100% → FAIL → 修复后重跑
  → P1 < 95% → 告警 → 说明原因后可继续
```

#### 门禁判定

```
ALL of:
  ✅ P0 用例全部通过（100%）
  ✅ P1 用例通过率 ≥ 95%（不足需写明原因）
  ✅ 新功能有对应 E2E 用例（不允许空手过门禁）
  ✅ 已有 P0 回归用例无新增失败
  ✅ 证据已写入 e2e-evidence.md
  ✅ 每条验证点含 PRD 原文引用（prd-code-alignment 协议）
  ✅ 无 DEVIATION 未处理（有偏差必须先修复再过门禁）
→ 门禁4 PASS → 进入 Phase 7

ANY of:
  ❌ P0 用例有失败
  ❌ 新功能无 E2E 用例且无跳过说明
  ❌ 回归用例新增失败
→ 门禁4 FAIL → 回退 Phase 3 修复
```

#### 与 Phase 9 部署后验证的关系

| 维度 | Phase 6.8 本地 E2E | Phase 9 部署后验证 |
|------|-------------------|------------------|
| 时机 | MR 前（本地） | 部署后（线上） |
| 环境 | localhost / 测试环境 | 真实部署环境 |
| 目的 | 拦截逻辑错误 | 验证部署正确性 + 环境差异 |
| 覆盖 | P0 + P1 业务逻辑 | 冒烟 + 配置 + 网关 + 数据权限 |
| 关系 | **互补，不替代** | 即使 6.8 全过，Phase 9 仍必须执行 |

---

### Phase 7: 代码推送

> **Inner Loop 钩子（推送门禁失败时）**：当 PreCommit AI 检出 CRITICAL、git lint 失败或 push 被拒（non-fast-forward / hook 阻断），由 `bin/inner-loop.sh --run --phase-state <ps.json> --phase 7` 自动接管：
> - micro retry（修 commit / rebase / 调 commit-msg）≤ 2 次 → 通过则继续 Phase 8
> - scope=pivot（如 repo-map 未注册、需新建仓库）→ blocked，按 Phase 7 自动化增强里"首推新仓库强制归档"流程升级
>
> 详见 `evolution/reflexion.md §1`。

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 6.8 门禁通过（或 Phase 6.5 通过 + E2E 跳过说明） |
| **前置步骤** | **repo-map 门禁（见下方）** |
| Skill 加载 | `skills/shipping/release-plan.md` + `skills/shipping/mr-create.md` + `skills/shipping/worktree-isolation.md` + `wiki/anti-patterns.md#phase-7` + `wiki/release-process.md` |
| **L1 提交门禁** | **PreCommit Hook**: AI 审查 staged diff（六维，>1500 行跳过留给 CI）+ **Git pre-commit**: 静态 lint + 密钥检测 + commit-msg 格式 |
| 质量门禁 | PreCommit AI 零 CRITICAL + Git lint 通过 + git push 成功 + 无冲突 + MR 描述含上线/回滚（基线8） |

#### Phase 7 repo-map 推送门禁（多仓库防错）

**推送前必须执行以下 3 步。如果模块未在 `Harness/repo-map.yaml` 中注册，拒绝推送并要求先注册。**

```
Step 1: 读取 repo-map 确认目标
  → repo-map.yaml → 当前模块的 repo / branch / push-rules / key-configs
  → 模块未注册 → 拒绝推送

Step 2: 增量 diff 预检
  → clone/fetch 目标仓库 HEAD
  → 对每个要推的文件执行 diff（目标 HEAD vs 本地文件）
  → 目标已有该改动 → 跳过（避免重复推送）
  → 目标有本地没有的改动 → 警告（漂移，可能覆盖目标仓库改进）

Step 3: 配置值校验
  → 遍历 key-configs
  → 在目标仓库 HEAD 中 grep 验证每个值
  → 不匹配 → 阻止推送，要求人工确认

Step 4: 跨服务构建依赖检查（CASE-082 教训）
  → 读 repo-map.yaml → 当前模块的 notes 和 push-rules
  → IF push-rules 含 "api-jar-deploy-prerequisite" 或 notes 含 "依赖 xxx JAR":
    1. 确认上游 API JAR 已发布（grep Artifactory 或确认上游 CI deploy-api 已成功）
    2. 确认上游变更已合入能触发 deploy-api 的分支（test/master）
    3. 未确认 → 阻止打 tag，先完成上游发布
  → IF 当前模块本身有 deploy-api 阶段:
    1. 确认推送到的分支能触发 deploy-api（only: test/master）
    2. 如果推送到 feature 分支 → 警告"API JAR 不会发布，下游可能编译失败"
```

Step 5: deploy-info 逐项验证（已知问题回归检查）
  → 读 changes/<req>/deploy-info.md
  → 对每个"⚠️ 每次部署前必须验证"项，执行其验证命令
  → 验证失败 → 阻止推送，先修复再继续
  → 无 deploy-info → 首次推送，需创建（铁律 #21）
  → deploy-info 中有"已处理"但无验证命令 → 补写验证命令后再推送

  WHY（CASE-082/083 教训）：
    deploy-info 的"已处理 ✅"只是历史快照。
    代码 merge/rebase 后，"已处理"的问题可能重新引入。
    验证命令把"死文档"变成"可执行断言"——每次推送前重新验证。
```

**实例（踩过的坑）**：

| 坑 | 哪一步能拦住 |
|----|------------|
| 改了 eam-v2 已 revert 的 context-path | Step 3: grep context-path 发现是 /inventory 不是 /inventory-api |
| 重复实现 eam-v2 已有的 OR fix | Step 2: diff 发现 appendGroupedConditions 已存在 |
| 7/9 文件漂移未察觉 | Step 2: diff 输出 145+ 行差异，触发警告 |
| commons-config-api JAR 未发布 | Step 4: 跨服务依赖检查发现 JAR 不可用 |
| MSSQL 配置重新引入导致启动失败 | **Step 5**: deploy-info 验证命令 grep sqlserver 发现未注释的配置 |

#### Phase 7 部署信息确认清单

**首次推送到某仓库时，必须逐项确认以下信息。确认后写入 `changes/<req>/deploy-info.md` 供后续复用。**

##### 通用项（所有项目）

| 确认项 | 说明 | 示例 |
|--------|------|------|
| 目标仓库地址 | GitLab/GitHub URL | `git@git.n.xiaomi.com:eam/asset-stock-be.git` |
| 目标分支 | 推送到哪个分支 | `feat/vin-check` → MR to `main` |
| 分支命名规范 | 前缀规则 | `feat/xxx`、`fix/xxx`、`hotfix/xxx` |
| MR 审批要求 | 几人 Approve | 至少 1 人（基线 5） |
| CI 触发条件 | push 自动触发？MR 触发？ | push 即触发 |
| 代码扫描 | Sonar/安全扫描是否必须通过 | 必须（基线 6） |

##### Java/Spring Boot 项目

| 确认项 | 踩坑经验 | 检查方法 |
|--------|---------|---------|
| JDK 版本 | CI 用的 JDK 版本和本地必须一致 | 查 CI 配置或 Dockerfile |
| Lombok 版本 | JDK 17 需 Lombok ≥ 1.18.30 | 查 pom.xml `<lombok.version>` |
| 增量推送 vs 整文件替换 | **禁止整文件替换**（目标仓库可能有本地没有的代码） | 始终从目标仓库 HEAD 增量修改 |
| 编译验证 | 推送前本地编译 or 依赖 CI | `mvn compile -DskipTests` |
| 多仓库依赖 | 改了 A 仓库是否需要同步改 B | 确认 Dubbo 接口兼容性 |
| 配置变更 | 新增配置项需要在哪里加 | Nacos / application.yml / 环境变量 |

##### Node.js/前端项目

| 确认项 | 踩坑经验 | 检查方法 |
|--------|---------|---------|
| Node 版本 | CI 和本地版本不一致导致构建失败 | 查 `.nvmrc` 或 CI 配置 |
| 包管理器 | npm/pnpm/yarn 不能混用 | 查 lock 文件类型 |
| 构建产物 | 是否需要本地 build 后提交 dist | 通常不需要（CI 构建） |
| 环境变量 | `.env` 文件不可提交；变量在哪配置 | CI Variables / 平台配置 |

##### Python 项目

| 确认项 | 踩坑经验 | 检查方法 |
|--------|---------|---------|
| Python 版本 | 3.8 vs 3.11 语法差异 | 查 `pyproject.toml` / CI |
| 虚拟环境 | venv 不可提交 | `.gitignore` 包含 venv/ |
| 依赖锁定 | requirements.txt vs poetry.lock | 确认 CI 安装方式 |
| 启动方式 | gunicorn/uvicorn/systemd | 查部署脚本 |

#### Phase 7 部署触发方式

不同项目使用不同的 CI 触发机制，首次推送时必须确认：

| 触发方式 | 说明 | 示例 |
|---------|------|------|
| push 触发 | 推送到特定分支自动部署 | push to `main` → prod |
| MR 触发 | 创建/更新 MR 时触发 CI | MR to `main` → build + test |
| **Tag 触发** | 打特定格式 tag 触发对应环境部署 | `test-*` → 测试环境 |
| 手动触发 | 在 CI 平台手动点击 | Matrix 手动发布 |

##### Tag 触发部署（常见于微服务）

> **铁律 #20**：Tag 格式和目标分支**必须从 `repo-map.yaml` 读取**，不可凭记忆或猜测。
> WHY: asset-account-be 无 test 分支，直接在功能分支打 tag；而 inventory-engine-v2 有 test 分支。
> 不同仓库规则不同，每次都要查 repo-map。
>
> **推荐工具**：`~/.claude/harness/tools/mitag/mitag.sh`（统一 Tag 脚本，自动处理环境/序号/编译标记/冲突检测）。
> 用法：`sh ~/.claude/harness/tools/mitag/mitag.sh <test|pre|prod> [COMMIT]`
> Tag 命名规则：`{env}[-build]-YYYYMMDD-{序号}`（如 `test-20260518-00`）

**Phase 7 Tag 部署执行步骤**：

```
1. 读取 repo-map.yaml → 当前模块的 tag-format / tag-ci-regex / tag-branch
2. 确定 tag 打在哪个分支：
   - tag-branch = "test" → 先 merge 到 test 分支，在 test 上打 tag
   - tag-branch = "feature-branch-or-master" → 直接在当前功能分支打 tag
3. 生成 tag 名：按 tag-format 填充变量
   - {env} = test / gray / prod
   - {YYYYMMDD} = 当天日期
   - {HHMM} = 当前时间
   - {short-hash} = git rev-parse --short HEAD
   - 示例：test-20260512-1619-7b56f9e
4. 打 tag 前确认 HEAD：git log --oneline -1
5. 推送 tag：git push origin <tag-name>
6. 将 tag 信息写入 changes/<req>/deploy-info.md
```

**三种分支模式**（以 repo-map.yaml 的 `tag-branch` 和 `test-branch-flow` 为准）：

| 模式 | tag-branch 值 | 说明 | 示例仓库 |
|------|--------------|------|---------|
| **有专用测试分支** | `test` / `feature/staging` | merge 到测试分支 → 在测试分支打 tag | asset-account-be (`test`)、asset-eam-web (`feature/staging`) |
| **有 tag 脚本** | 任意 + `tag-script` | 合并后执行脚本自动生成 tag | asset-eam-web (`bash tag.sh`) |
| **直接打 tag** | `main` / 功能分支 | 直接在当前分支打 tag | inventory-engine-v2 |

**有专用测试分支时**（读 repo-map.yaml `test-branch-flow`）：

```bash
# 示例：asset-eam-web（测试分支 = feature/staging，有 tag 脚本）
git checkout feature/staging && git pull origin feature/staging
git merge feat/xxx --no-edit
git push origin feature/staging
bash tag.sh  # 自动生成 test-mi-YYYYMMDD-NN

# 示例：asset-account-be（测试分支 = test，手动打 tag）
git checkout test && git pull origin test
git merge master --no-edit
git push origin test
git tag test-{YYYYMMDD}-{NN}
git push origin test-{YYYYMMDD}-{NN}
```

**直接打 tag 时**：

```bash
# 确认在功能分支或 main 上
git log --oneline -1
git tag test-{YYYYMMDD}-{HHMM}-{short-hash}
git push origin test-{YYYYMMDD}-{HHMM}-{short-hash}
```

**通用规则**：
- Tag 必须打在正确的 commit 上（先 `git log --oneline -1` 确认 HEAD）
- Tag 推送后无法撤回部署（需打新 tag 或回滚）
- 生产 tag 推送前必须经过 [H4] 人工确认
- 同名 tag 不能重复推送（需删除远端旧 tag 再推）
- 有 test 分支时：先 merge 再打 tag，顺序不可反
- **deploy-info.md 必须记录每次推送的 tag**（供后续回滚和审计）

##### 部署触发确认清单（首次填写后写入 deploy-info.md + repo-map.yaml）

| 确认项 | 来源 | 示例 |
|--------|------|------|
| 仓库地址 | repo-map.yaml `repo` | `git@git.n.xiaomi.com:...` |
| 触发方式 | repo-map.yaml `deploy` | tag-trigger / matrix-manual |
| Tag 格式 | repo-map.yaml `tag-format` | `{env}-{YYYYMMDD}-{HHMM}-{short-hash}` |
| CI 正则 | repo-map.yaml `tag-ci-regex` | `^test-.*\|prod-.*` |
| Tag 打在哪个分支 | repo-map.yaml `tag-branch` | test / feature-branch-or-master |
| JDK/Node 版本 | repo-map.yaml `key-configs` | 1.8 / 18.x |

> **门禁**：首次推送到新仓库时，上述信息必须全部填入 `repo-map.yaml` 和 `deploy-info.md`。
> 后续推送直接读取，不再重复确认。缺任一项 → 拒绝推送。

#### Phase 7 首推新仓库强制归档（铁律 #21）

> WHY: 2026-05-09 向 asset-account-be 推送了 3 次 tag，但未注册 repo-map 也未创建 deploy-info.md。
> 05-12 再次推送时不知道 tag 格式和分支规则，重新摸索浪费时间。
> 根因：repo-map 是 05-11 才创建的，05-09 的推送没有这个机制；创建 repo-map 时也没有回溯已推过的仓库。

**判定方法**：`grep "<module-name>" Harness/repo-map.yaml`

```
IF 仓库不在 repo-map.yaml 中:
  → 停止推送
  → 执行首推归档：
    1. 在 repo-map.yaml 注册模块（repo/branch/deploy/tag-format/tag-ci-regex/tag-branch/key-configs）
    2. 创建 changes/<req>/deploy-info.md（仓库信息 + 部署触发 + 技术栈）
    3. 归档完成后才能继续推送
  → 回溯检查：repo-map 中是否有其他「之前推过但未注册」的仓库
    git remote -v | 对照 repo-map → 遗漏的立即补注册

IF 仓库已在 repo-map.yaml 中:
  → 读取 tag-format / tag-branch → 按规则执行
  → 推送后将 tag 追加到 deploy-info.md
```

#### Phase 7 推送红线

```
// WHY: eam-v2 连续 4 次构建失败，因为整文件从 pm-context 复制到目标仓库
1. 禁止整文件替换（始终增量修改）
2. 推送前 grep 验证关键依赖未被破坏
3. git add 指定文件（不用 git add -A / git add .）
4. 不确定目标仓库状态时先 pull 再操作
5. 新增配置项必须确认在所有环境（dev/test/prod）都已配置
6. Tag 部署前确认 HEAD 是正确的 commit
7. 生产 tag 必须经人工确认后才能推送
8. 前后端一体化项目：前端改动必须同时提交 vite build 构建产物（详见 wiki/lessons-learned.md §十三）
9. **前端构建必须在目标仓库 clone 中执行**（不在 pm-context 构建再 rsync，config 可能已漂移）
10. 推送静态产物前 grep 验证 baked 值：`grep -r "关键路径" static/ | wc -l` 预期 0
11. **首次推送新仓库必须先完成 repo-map 注册 + deploy-info.md 归档（铁律 #21）**
12. **Java 仓库确认已接入 MIT CI 模板**：`grep "mit/infra/ci-templates" .gitlab-ci.yml`（见 wiki/mit-unit-test-standards.md §三）
```

**规则 8 判定方法**：首次推送到某仓库时，`grep "frontend-maven-plugin" pom.xml` — 如果不存在或被注释，说明 CI 不构建前端，`src/main/resources/static/` 必须本地 build 后提交。

#### Phase 7 自动化增强（从 gstack /ship 提取的最佳实践）

```
推送前自动化步骤（Phase 7 入口执行）:

1. Commit 组织:
   - 检查 git status：如果有多个不相关的改动 → 建议拆分为逻辑 commit
   - 每个 commit 应可独立 revert（bisectable）
   - commit message 格式: type(scope): description

2. VERSION + CHANGELOG（项目根目录有 VERSION 文件时）:
   - 读取当前 VERSION
   - 根据变更类型建议 bump: fix→PATCH, feat→MINOR, breaking→MAJOR
   - 自动从 commit messages 生成 CHANGELOG 条目
   - 如无 VERSION 文件 → 跳过（不强制）

3. 测试框架检测:
   - 检测项目测试基础设施是否存在（package.json scripts.test / pom.xml surefire）
   - 如果完全无测试 → 提醒用户（不阻断推送，但记录到 observations）
   - 如果有测试 → 确认 CI 会执行

4. 推送执行:
   - 默认非交互（用户说"推送"意味着 DO IT）
   - 只在以下情况停下来问:
     a. 目标分支不确定
     b. 有 merge conflict
     c. CI 配置缺失
     d. 首推新仓库（铁律 #21）
```

#### Phase 7 MR 创建规范（铁律 #28）

> WHY: V08 首次 MR 将 test 分支整体合入 master，携带 1380 个积压 commit（98 文件 +3932 行），
> 大量非本次需求的改动混入 MR。根因：直接 test→master 而没有从 master cherry-pick。

**MR 创建三步协议（不可跳步）**：

```
Step 1: 拉取目标分支最新代码
  → git fetch origin master && git checkout master && git pull

Step 2: 从 master 创建干净的发布分支，只合入本次改动
  → git checkout -b feat/<需求>-release
  → git cherry-pick <本次需求的 commit>...（逐个或范围）
  → 解决冲突后验证 diff 干净度：
    git diff --stat master..HEAD
    只应包含本次需求的文件，不应有无关改动

Step 3: 推送发布分支并创建 MR
  → git push origin feat/<需求>-release
  → glab mr create --source-branch feat/<需求>-release --target-branch master
```

**禁止行为**：
- ❌ 直接将 test/dev 分支整体合入 master（携带积压改动）
- ❌ MR diff 包含非本次需求的文件
- ❌ 不拉取最新 master 就创建 MR（可能有冲突或遗漏）

**验证方法**：
```bash
# MR 创建前检查 diff 干净度
git diff --stat master..feat/<需求>-release
# 文件列表应与 requirement.md 改动范围一致
# 如果出现意外文件 → 停止，排查是否 cherry-pick 了无关 commit
```

### Phase 7.5: 上线准备（部署前置检查清单）

> WHY: V08 超期提醒代码完成后，缺 UMS 消息模板 + XMS Plan 定时任务注册 + 数据权限配置，功能无法生效。
> 很多线上事故不是代码 bug，而是"配置没跟上"。
> 参考：MIT 上线方案标准模板。

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 7 代码推送完成，[H4] 人工确认前 |
| Skill 加载 | `skills/shipping/release-plan.md`（全维度上线方案 18 章）|
| 质量门禁 | 8 大类检查清单 + **全维度上线方案（页面/API/Dubbo/Service/定时任务/DB/配置/外部集成/场景调用链路/技术设计/风险/回滚/验证/监控）** + deploy-checklist.md 已写入 + release-plan.md 已写入 |
| 工具触发 | 上线方案完成后 → 检查 `tools/feishu-notify/` → 存在则创建飞书文档 + 推群（摘要含各角色审查重点） |
| 产出物 | `changes/<req>/deploy-checklist.md` + `changes/<req>/release-plan.md` |

#### 8 大类上线准备清单

**[H4] 人工确认时必须逐项检查。缺一项不能上线。**

| 类别 | 检查项 | 实例（EAM） |
|------|--------|-----------|
| **服务部署顺序** | 多服务依赖谁先谁后？先 JAR 再服务再前端 | commons-config 先发 JAR → entry-be 后部署 → eam-web 最后 |
| **数据初始化/脚本** | DDL + 回滚 SQL + 数据迁移脚本 | `ALTER TABLE` / `INSERT INTO` / `DELETE FROM` |
| **环境变量/Nacos 配置** | 新增配置项在 dev/test/prod 都配了？ | `vehicle-cate-codes` 在 Nacos 各环境配置 |
| **全局变量/系统配置** | 业务开关、分类编码、阈值 | `VehicleCateCode`、`VehicleReserveAssetCateCode` |
| **消息模板** | UMS 飞书卡片/邮件模板已创建？变量名与代码对齐？ | 模板 `TLB032900027`（title/content/buttonUrl） |
| **审批流程/BPM** | 新增审批模型 + 审批人规则 + 回调 URL | BPM 后台配置 VEHICLE_RESERVE/RETURN/RENEWAL |
| **权限/数据权限/米盾** | 页面权限、数据权限、米盾路径认证策略 | 员工端"本人看本人" + 米盾新路径规则 |
| **定时任务** | XMS Plan 注册 + cron 正确 + 手动触发方式确认 | `@PlanTask` + `plan.auto-startup: true` + 项目 ID |

#### 上线方案文档结构（MIT 标准）

按以下 9 章输出上线方案，写入 `changes/<req>/deploy-plan.md`：

```
1. 上线说明：目的 + PRD 链接 + 技术方案链接
2. 上线范围与发布计划：关联系统 + 上线顺序 + 负责人
3. 资源申请/确认：域名/数据库/缓存/MQ/米盾——至少提前 1 天
4. 实施步骤：版本配置 + 操作指令（要让其他人也能执行）
5. 监控地址：Hera + 业务指标看板
6. 异常应对策略：每种异常的应对方案
7. 回滚方案：回退上一版本的具体步骤
8. 验证方案：业务/技术/产品/测试验证人 + 验证用例
9. 通告：执行前/执行中/完成后通知模板
```

#### 上线通告模板

```
【执行前】
大家好，{项目名} 将于今晚进行调整：
【变更内容】{描述}
【变更时间】{YYYY-MM-DD HH:mm}
【预计影响】{影响说明}
【变更人员】{负责人}

【完成后】
大家好，{项目名} 变更完成：
【变更内容】{描述} 变更成功/失败（已回滚）
【完成时间】{YYYY-MM-DD HH:mm}
【保障人员】{负责人}，如有问题及时联系
```

### Phase 8: CI 验证

| 要素 | 内容 |
|------|------|
| 触发条件 | [H4] 用户确认推送目标和环境参数 |
| Skill 加载 | — |
| **L1 最终防线** | **CI/CD = 三道防线的权威层**：SonarQube 质量门禁 + 集成测试 + 安全扫描。Phase 3/4/7 的 lint + AI Review 是前置过滤，CI 是兜底 |
| 质量门禁 | `status=SUCCESS` + `tests>0` + `passed=total(100%)` + **Sonar 新增覆盖率 ≥ 80%** + **Sonar 总体覆盖率 ≥ 60%** + **重复代码率 ≤ 3%** + **新增 Blocker/Critical 异味 = 0** + **新增安全漏洞 = 0** + 禁止 skip-test（红线）+ **deploy-info 反向校验** |

#### Phase 8 deploy-info 反向校验

> WHY: 05-09 推送 asset-account-be 通过 CI 但未记录部署信息，导致 05-12 再推时不知道 tag 规则。
> CI 通过 ≠ 流程完备。编译成功但部署信息缺失 = 下次推送时必然踩坑。

**CI 结果返回后（无论成功失败），必须确认以下 3 项**：

```
1. deploy-info.md 存在且包含本次 tag
   → grep "<tag-name>" changes/<req>/deploy-info.md
   → 不存在 → 立即补写

2. repo-map.yaml 中已注册本仓库
   → grep "<module-name>" Harness/repo-map.yaml
   → 不存在 → 立即注册（Phase 7 铁律 #21 的兜底）

3. CI 失败时：tag + 错误信息记录到 deploy-info.md
   → 供下次修复时参考，不重复排查相同问题
```

### Phase 9: 部署验证

> **Inner Loop 钩子（部署验证失败时）**：当 health check / API 冒烟 / E2E AC / Canary 任一失败，由 `bin/inner-loop.sh --run --phase-state <ps.json> --phase 9` 自动接管，与 `failure-diagnosis.md` 三层诊断协同：
> - Layer 1/2 识别为可重试错误 → inner loop 走 micro retry（≤2 次）
> - Layer 3 LLM 给出根因或 scope=pivot（如版本不兼容、配置缺失、需回滚）→ blocked 并升级
> - 反思双次幻觉（cause 引用了 evidence 不存在的字段）→ pivot
>
> 详见 `evolution/reflexion.md §1`。

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 8 门禁通过 |
| Skill 加载 | `skills/coding/verification.md` + `skills/testing/browser-qa.md`（浏览器 QA 自动化）+ `skills/testing/canary-monitor.md`（部署后 Canary 监控）+ `skills/testing/failure-diagnosis.md`（**验证失败时自动激活：三层归因引擎 + 可重试判断 + 修复建议**）+ `wiki/anti-patterns.md#phase-9`（反模式检查） |
| 质量门禁 | health check 通过 + 关键 API 冒烟通过 + 业务正确性验证 + **浏览器 QA AC 通过率 100%** + **Canary 3 轮无告警** |
| **失败诊断协议** | 验证失败时自动执行 `failure-diagnosis.md` 三层诊断：Layer 1 专家规则匹配 → Layer 2 正则分类 → Layer 3 LLM 分析。可重试错误自动重试（指数退避），不可重试错误输出诊断报告 + 修复建议 |
| 工具依赖 | Chrome DevTools MCP（E2E 验证必须） |

#### Phase 9 验证工具前置检查

进入 Phase 9 前，Agent 必须检查验证工具是否就绪。如果缺失，引导用户安装：

**Chrome DevTools MCP（必须）**

```
检测：查看当前会话是否有 mcp__chrome-devtools__* 系列工具可用

如果不可用 → 输出以下引导：
```

> **需要安装 Chrome DevTools MCP 才能进行 E2E 验证。**
>
> 安装步骤：
>
> 1. 安装 MCP Server：
>    ```bash
>    npx @anthropic-ai/chrome-devtools-mcp@latest
>    ```
>
> 2. 在 Claude Code 设置中添加（`~/.claude/settings.json` 或项目 `.claude/settings.json`）：
>    ```json
>    {
>      "mcpServers": {
>        "chrome-devtools": {
>          "command": "npx",
>          "args": ["@anthropic-ai/chrome-devtools-mcp@latest"]
>        }
>      }
>    }
>    ```
>
> 3. 重启 Claude Code 会话
>
> 4. 验证安装：尝试使用 `mcp__chrome-devtools__take_snapshot` 工具
>
> **安装后即可继续 Phase 9。**

#### Phase 9 部署后首检（Pod 内 curl，30 秒排除网关问题）

> WHY: inventory-engine-v2 context-path 不匹配网关路径，全接口 404，排查 2 小时。改用 Pod 内 curl 30 秒定位。

**部署完成后、E2E 验证前，必须先在 Pod 内 curl localhost 确认应用本身可达：**

```bash
# 进入 Pod
kubectl exec -it <pod> -- bash

# 用 context-path 测一个 health 或已知接口
curl -v http://localhost:8080/<context-path>/actuator/health
curl -v http://localhost:8080/<context-path>/<任意已知接口>

# 预期：200。如果 404 → context-path 和网关路径不匹配，见 wiki/lessons-learned.md §十四
```

#### Phase 9 E2E 验证方法

| 步骤 | 操作 | 验证标准 |
|------|------|---------|
| 0. Pod 内 curl | `curl localhost:<port>/<context-path>/...` | 200（排除网关问题） |
| 1. 打开目标页面 | `navigate_page` 到部署 URL | 页面加载成功 |
| 2. CAS 认证 | 用户手动登录米盾，或用已认证浏览器 | 不跳 /route-no-auth |
| 3. 功能冒烟 | 执行核心操作路径 | 业务数据正确（不只是 code=0） |
| 4. 对比验证 | 有条件/无条件分别请求，对比结果 | 数量/内容符合预期 |
| 5. 数据清理 | 删除测试数据 | 不影响真实业务 |

**反幻觉规则（E2E 验证特有）**：

| 错误做法 | 正确做法 |
|---------|---------|
| API 返回 200 就标 ✅ | 对比数据内容：有/无条件返回数量必须不同 |
| 只验证 API 可达 | 验证业务正确性：数据符合筛选条件 |
| 自己构造请求参数 | 先看 DTO/前端代码确认字段名 |
| 假设前后端字段名一致 | 用 Network 面板看真实 payload |

**Phase 9 覆盖率要求（来源：CASE-076 scope 全量矩阵验证）**：

| 改动类型 | 最低验证覆盖 | 验证方法 |
|---------|------------|---------|
| scope/WHERE/operator 改动 | **全部 14 种 operator** 各至少 1 个测试用例 | 创建 N 个计划，对比 totalQuantity 与数据分布的数学关系 |
| if-else 分支修改 | **所有同级分支** | 改了 EQ → 也测 LIKE/IN/ISNULL 等 |
| API 返回类型变更 | **前端所有调用方** | grep `.split(` / `.join(` 等类型依赖操作 |
| 展示类 bug 修复 | **DB 实际数据分布** | `SELECT DISTINCT column, COUNT(*) GROUP BY column` |
| 前端胶水层修改 | **浏览器完整操作 + Network 抓包** | 不能只 curl API，必须从页面点击触发 |

**Phase 3/9 铁律（#23-#31，来源：盘点 v2 修复 2026-05-12~15，见 wiki/lessons-learned.md §二十一 + CASE-073/075/076）**：

| # | 规则 | 适用阶段 |
|---|------|---------|
| 23 | **修 bug 前先在浏览器走一遍用户操作路径** — 先观察 Network 面板的真实请求路径再写修复，不凭代码阅读猜测 | Phase 3 编码前 |
| 24 | **修改 SQL SELECT/WHERE 前确认 DB schema** — `SHOW COLUMNS` 或查实体 `@TableField(exist=false)`，不凭推断 | Phase 3 编码时 |
| 25 | **selectOne 查询确认唯一性约束** — 不确定时用 selectList + LIMIT 1 + try-catch 降级 | Phase 3 编码时 |
| 26 | **验证必须走和修复相同的代码路径** — 用"有条件 vs 无条件"数量差异证明过滤生效，不能用不同路径的 API 代替 | Phase 9 验证时 |
| 27 | **一个修复一个验证** — 不批量推送。标记"已修复"必须有对比证据（修复前 X → 修复后 Y），不是 HTTP 200 | Phase 9 验证时 |
| 28 | **每个 bug fix commit 后立即归档** — 不等全部修完再批量归档。一个 commit = 一个 CASE + AP 提取 + scores 写入。Phase 10 是逐次门禁不是最终门禁 | Phase 10 归档时 |
| 29 | **展示类 bug 第一步查 DB 数据分布** — `SELECT DISTINCT column, COUNT(*) GROUP BY column`，不凭代码推断数据长什么样。映射关系要用实际数据验证不用 PRD 推导 | Phase 1 分析时 |
| 30 | **scope/WHERE/operator 改动必须全量矩阵验证** — 14 种 operator 每种至少一个测试用例，创建对照组对比 totalQuantity。不能只测改了的 2 个 | Phase 9 验证时 |
| 31 | **改一个分支必须审查所有同级分支** — 改了 if-else 链的 EQ 分支 → 审查 LIKE/IN/ISNULL 等所有分支是否有同类问题（缺 ESCAPE、缺 escapeLikeValue、缺 import） | Phase 4 评审时 |
| 32 | **反复 bug 沿调用链追 3 层以上** — 不止修报错行，追问上游传什么/下游假设什么/null 和空都处理了吗 | Phase 1 分析时 |
| 33 | **跨系统修复修在"发起方"** — 补丁打在别人系统随部署丢失，修自己控制的代码才可靠 | Phase 3 编码时 |

**物理约束机制**（从文档约束升级为代码约束，2026-05-15 进化）：

| 机制 | 文件 | 执行方式 |
|------|------|---------|
| **GOV-011 多 bug 串行** | `phase-state.json → bugQueue` | 每个 bug 独立 mini-cycle，activeIndex 强制递增 |
| **GOV-012 数据摸底门禁** | `phase-state.json → bugQueue.items[].dataProfile` | 展示类 bug 不查 DB 不让进 Phase 3 |
| **Java 8 兼容检测** | `tools/code-quality/config/java/java8-compat.sh` | post-edit hook 检测 var/text block/switch expression |
| **反复 bug CASE 加载** | `knowledge-map.yaml → symptom_keywords` + `inject.md Step 5.5` | "反复"信号触发全模块 CASE 强制加载 |
| **页面级验证检查清单** | `memory/verification-checklists.yaml` | Phase 9 按页面路由加载清单，逐项执行 |

### Phase 10: 用户确认 + 案例归档

| 要素 | 内容 |
|------|------|
| 触发条件 | Phase 9 门禁通过 |
| Skill 加载 | `skills/evolution/capability-review.md`（C0-C9 能力评审）+ `skills/evolution/auto-extract-ap.md`（含 T2 量级触发检查）+ `skills/evolution/self-evolution.md`（C16 自进化，T2/T3 条件触发） |
| 质量门禁 | [H5] 用户确认功能符合预期 + 案例归档完成 + **C0-C9 能力评审完成** + **Spec Sync 完成（如有 Living Spec delta）** + **进化信号已写入 scores.jsonl（evolutionSignals 非空）** |
| 工具触发 | 能力评审完成后 → 检查 `tools/feishu-notify/` → 存在则执行 `capability-review-push.md` |
| 最终产出 | `changes/<req>/retrospective.md` + `changes/<req>/capability-review.md` + 案例文件（如有）+ **PRD 同步记录**（如有 delta） |

#### Phase 10 Spec Sync（Living Spec → Atom PRD 同步）

> 来源：OpenSpec archive flow — "delta specs merge into main specs, source of truth evolves."

**触发条件**：`changes/<req>/living-spec-log.md` 存在且非空。

**执行步骤**：
```
1. 读取 living-spec-log.md 中所有 delta
2. 按合并顺序排列：RENAMED → REMOVED → MODIFIED → ADDED
3. 冲突检测：同一 AC 同时 MODIFIED + REMOVED → 阻塞，请求用户裁决
4. 用 spec-validation.md 检查合并结果仍通过 ERROR 级别
5. 更新 Atom PRD 对应文件（docs/systems/eam/domain/Atom PRD/{模块}/）
6. 在 PRD 文件中追加：<!-- synced from changes/<req>/living-spec-log.md, {date} -->
7. 在 retrospective.md 中记录："Spec Sync: {N} 条 delta 已同步到 Atom PRD"
```

**跳过条件**：living-spec-log.md 不存在或为空 → 跳过，记录"无 spec delta"。

---

#### Phase 10 案例归档（强制）

**每个任务结束时（无论成功还是失败），必须执行案例总结。**

##### 判定：是否需要新建案例

| 信号 | 动作 |
|------|------|
| 过程中踩了坑（调试 > 30min / 回退过 Phase / 修了 2+ 次才对） | **必须**新建失败案例 |
| 发现了有效的新方法/模式（首次成功的做法、提效的工具用法） | **必须**新建成功案例 |
| 触发了已有案例描述的场景（复现了 CASE-XXX） | **必须**更新已有案例的「发生次数」 |
| 一切顺利、无新发现 | 在 retrospective.md 记录「无新案例」即可 |

##### 案例写入流程

```
1. 扫描本次任务中的关键事件：
   - grep changes/<req>/ 中的 review 记录、回退记录
   - 回顾对话中的排查过程、误诊、方向修正

2. 对每个事件判定：
   - 是否匹配已有案例？→ 更新 cases/CASE-XXX.md 的「发生次数」
   - 是否是新模式？→ 新建 cases/CASE-XXX-<slug>.md

3. 新案例必须包含：
   - 关联规则（指向 lessons-learned.md 或 development-process.md 的具体章节）
   - 可 grep 的关键词（文件名、错误信息、配置项）
   - 诊断命令（可复制粘贴的验证命令）
   - 永久防护（如何确保不再发生）

4. 反向链接：在关联规则处追加案例引用 [CASE-XXX]

5. 更新 cases/README.md 索引
```

##### 成功案例 vs 失败案例

| 类型 | 包含什么 | 目的 |
|------|---------|------|
| **失败案例** | 症状→误诊→正确根因→修复→永久防护 | 防止重蹈覆辙 |
| **成功案例** | 场景→选择的方法→为什么有效→适用条件→局限性 | 推广有效实践 |

##### 门禁

```
Phase 10 完成条件：
  ✅ 用户确认功能符合预期
  ✅ retrospective.md 已写入
  ✅ 案例判定已执行（"无新案例" 或 案例文件已创建/更新）
  ✅ 新案例已双向链接（案例→规则、规则→案例）
  ✅ cases/README.md 索引已更新（如有新案例）
  ✅ [C7] 进化引擎 Feed：将本次任务结果（success/failed + duration）追加到 evolution/scores.jsonl
  ✅ [C7] 反模式自动提取：执行 `skills/evolution/auto-extract-ap.md`，从新 CASE 提取 AP 条目写入 anti-patterns.md + 创建 Gene 候选
  ✅ [C7] C0-C9 能力评审：执行 `skills/evolution/capability-review.md`，10 项能力 + 6 层 Context 逐项打分 + 差距 ≥ 3 的提改进方案
  ✅ [C7] capability_review 记录已追加到 scores.jsonl
  ✅ [C9] 能力评审结果已推送到飞书群（如 feishu-notify 可用）
  ✅ [W-13] follow-up 升级 case 复审：promoted_case_ids 已逐条复审 lesson + phase-state.gene_attribution / newRules 已更新（promoted_case_ids 为空时跳过）
```

---

#### Phase 10 follow-up 升级 case 复审（W-13 期1）

> 来源：W-13 期1 spec §5.2 C — review subagent 输出的 cross_task follow-up 自动升级为 brain CASE 后，Phase 10 主控复审补 lesson 纵深，防止误判过宽召回。

**触发条件**：`learnedContext.promoted_case_ids` 非空（即本任务有 review follow-up 被 case-promote.sh 升级）。

**执行步骤**：

```
1. 读 phase-state.json `learnedContext.promoted_case_ids` 列表
2. 对每个 case_id：
   a. 验证 `Harness/wiki/cases/<case_id>.md` 存在
      - 不存在 → 手动补跑 `case-promote.sh <review-output>`，然后继续
   b. 读本任务 context（commits / spec / review 输出 / phase-state）
   c. 给 lesson 末尾追加 1-2 行"误判风险 / 适用边界"，例如：
      - 误判风险：此 trigger 在 X 场景可能误命中（如 ...）
      - 适用边界：仅当 ... 时才参考此 lesson
   d. 编辑 `Harness/wiki/cases/<case_id>.md` 在"## 避免方法"末尾追加上述 1-2 行
3. 在 phase-state.json `learnedContext.gene_attribution` 末尾加：
   ```json
   {"promoted_case_ids": ["CASE-001-...", "..."]}
   ```
4. 在 phase-state.json `learnedContext.newRules` 中追加 1 条引用该 CASE 的简短 rule：
   ```
   "<规则简述> — 参考 [CASE-001-...]"
   ```
5. **W-13 期 2 sessionId 一致性校验**：
   - 读 CASE md 的 `sessionId` 字段，与本 task `phase-state.sessionId` 比对
   - 一致 → PASS，继续
   - 不一致 → 警告：case-promote 期间 phase-state 被改，记录到 retrospective.md「sessionId 漂移」段
6. **W-13 期 2 scope 校准**（可选，主控判断）：
   - 若 review subagent 误判 `single_project` 但本质跨项目级框架陷阱（同样的 lesson 在多项目复现）→ 主控手动改 md.scope = `cross_project`，emit `case_scope_calibrated` 事件
   - 反向 — 若误判 `cross_project` 但实际只是项目内经验 → 降级到 `single_project`
   - 校准事件格式：
     ```json
     {
       "type": "case_scope_calibrated",
       "case_id": "CASE-NNN-...",
       "from_scope": "single_project",
       "to_scope": "cross_project",
       "task_id": "<from env>",
       "phase": "10",
       "timestamp": "<ISO8601>",
       "calibrator": "phase10_review"
     }
     ```
```

**跳过条件**：`promoted_case_ids` 不存在或为空数组 → 跳过，在 retrospective.md 记录"无 follow-up case 升级"。

**容错**：
- 若 case md 文件被外部删除 → 手动补跑 promote.sh，不终止 Phase 10
- 若 lesson 已含"误判风险"段落（已被复审过）→ 跳过追加，记 warn
- 若 sessionId 一致性校验发现漂移（case md.sessionId ≠ phase-state.sessionId）→ 记 warn，不阻塞 Phase 10，在 retrospective.md 标注「需查 case-promote 期间 phase-state 是否被并发改动」

---

## Phase 验证框架（DoD Verifier）

> 每个 Phase 完成时，由独立 Verifier Agent 按 DoD checklist 审查。执行 Agent ≠ 审查 Agent，物理隔离防自审。

### 状态机

```
pending → running → verifying → done
                        ↓
                     blocked → (fix) → running
```

### 执行流程

```
Phase N 完成
  → phase-state.json: phases[N].status = "verifying"
  → Verifier 自动激活
  → 读取 governance/dod/phase-{N}.yaml
  → 执行三类检查:
      1. verify.sh --phase N (auto 类型, bash 物理验证)
      2. Verifier Agent 逐条审查 (agent 类型, Codex→Opus→Sonnet)
      3. 读取 gates 状态 (human 类型)
  → 输出 verification-report
  → ALL blocker PASS → "done"
  → ANY blocker FAIL → "blocked" + 精确回退建议
```

### Verifier 模型降级

| 模型 | 超时 | 触发降级条件 |
|------|------|------------|
| Codex (首选) | 120s | 超时 / HTTP 429/5xx |
| Opus (降级1) | 60s | 超时 / 连续 2 次失败 |
| Sonnet (降级2) | 30s | 仅执行 blocker 级检查 |
| 全不可用 | — | auto-only + 人工门禁升级 |

### 引用

- DoD 定义: `governance/dod/phase-{N}.yaml` (15 个文件, 130 项检查)
- Verifier Agent: `agents/verifier-dod.md`
- 执行器: `governance/dod/verify.sh`
- 配置: `governance/dod/verifier-config.yaml`

---

## 精确回退路径

不是出了问题从头来，是路由到该修的环节：

| 失败场景 | 回退目标 | 判定依据 |
|---------|---------|---------|
| CI 失败 + 测试覆盖为 0 | → Phase 5 单测编写 | `tests == 0` |
| CI 失败 + 编译错误 | → Phase 3 编码实现 | `compile_status == FAIL` |
| 编码评审发现需求理解偏差 | → Phase 1 需求分析 | 评审意见含"需求理解" |
| 编码评审发现架构设计缺陷 | → Phase 2.3 技术设计 | 评审意见含"架构""设计方案" |
| 单测评审发现测试场景遗漏 | → Phase 5 单测编写 | 评审意见含"覆盖不足" |
| 部署验证失败 | → Phase 3 编码实现 | 运行时异常 |
| 部署验证发现架构/性能问题 | → Phase 2.3 技术设计 | 性能不达标 / 架构不合理 |
| 用户确认"功能不符" | → Phase 1 需求分析 | 用户反馈 |
| **实现后发现需求不清晰/漏洞** | → Phase 1 补充 requirement | 新发现的业务规则/边界条件 |
| **实现后发现技术方案有缺陷** | → Phase 2.3 技术设计 | 接口不匹配/数据模型误解/架构不支持 |
| 需求评审超 3 轮 | → 人工决策 | `review_round > 3` |
| 编码评审超 2 轮 | → 人工决策 | `review_round > 2` |

### 流动式迭代（Fluid Iteration，来源：OpenSpec）

> 区别于"精确回退"（失败触发），流动式迭代是**主动发现**时的轻量回更。
> 原则：Phase 不是监狱，发现了更好的理解就应该更新上游产出物。

**场景**：Phase 3 编码中通过 Checkpoint 反思发现 Phase 2 的方案可以更好（不是错误，是改进）。

**处理步骤**：
```
1. Checkpoint 发现 → 触发 Living Spec Update（delta 格式）
2. Delta 操作类型决定路由：
   - MODIFIED（措辞/细节调整）→ 就地更新 requirement.md，不回退 Phase
   - ADDED（发现新需求）→ 就地追加到 requirement.md，如影响其他 Task 则暂停
   - REMOVED（发现不需要）→ 标记后继续，Phase 10 同步
   - 行为变更 → 必须用户确认 → 确认后更新 → 从当前位置继续
3. 不重置 Phase 状态机 — currentPhase 不回退
4. 更新记录在 living-spec-log.md 中留痕
```

**与精确回退的区别**：

| 维度 | 精确回退 | 流动式迭代 |
|------|---------|-----------|
| 触发 | 失败/错误 | 主动发现/改进 |
| Phase 状态 | currentPhase 回退 | currentPhase 不变 |
| 成本 | 需重新走门禁 | 就地更新，继续前进 |
| 适用 | 需求理解偏差/架构缺陷 | 措辞改进/补充场景/发现新约束 |

### 中途发现漏洞的处理协议

**场景**：Phase 7 推送后或 Phase 9 验证时发现实现有漏洞、需求有遗漏。

**处理步骤**：

```
1. 判断漏洞类型：
   - 需求层面（如"需要校验子级分类"之前没说）→ 回退 Phase 1
   - 技术层面（如"接口抛异常而非返回null"）→ 回退 Phase 3

2. 记录到 changes/<req>/review-v{N}.md：
   "Phase {X} 发现漏洞: {描述} → 回退到 Phase {Y}"

3. 在目标 Phase 补充/修正：
   - Phase 1：更新 requirement.md 补充遗漏的业务规则
   - Phase 3：修改代码，重新走 Phase 4-6.5

4. 修正后从当前 Phase 继续推进（不需要重走已通过的无关阶段）

### 四级响应模型（五步法 Step 4）

```
Level 4: 回滚 (Rollback)   — 数据损坏/不可逆操作
    ↑
Level 3: 熔断 (Circuit Break) — 连续失败/异常模式
    ↑
Level 2: 矫正 (Correction)  — 输出偏移/格式错误
    ↑
Level 1: 重试 (Retry)       — 临时故障/网络抖动
```

| 级别 | 触发条件 | Scheduler 动作 |
|------|---------|---------------|
| L1 重试 | 单次工具调用超时/网络错误 | 等待 30s 重试，最多 3 次 |
| L2 矫正 | Agent 输出格式不合规/缺必填字段 | 注入格式约束重新生成 |
| L3 熔断 | 同一任务连续失败 3 次（circuit breaker） | 标记 BLOCKED，跳过该任务，记录到 failure-history.jsonl |
| L4 回滚 | 推送后 CI 失败 / 部署后冒烟失败 | `git revert` + 告警人工 |

### 回滚快照（外部操作前必须记录）

执行不可逆外部操作（git push / API 调用 / 数据变更）前，记录回滚点：

```json
{
  "rollback_id": "rb_20260513_140500",
  "operation": "git push harness main",
  "pre_state": "commit abc123",
  "undo_action": "git revert HEAD",
  "ttl": "24h"
}
```

快照写入 `changes/<req>/rollback-points.jsonl`。TTL 过期后自动清理（由 scheduler 管理）。

### 兜底执行链

当所有自动方案均失败时的降级路径：

```
Step 1: 主工具执行（claude/codex）
  → 失败 → timeout 30s

Step 2: 备用工具执行（如主用 claude 则备用 codex）
  → 失败 → timeout 15s

Step 3: 缓存结果复用（上一次成功的类似任务产出）
  → 无可用缓存

Step 4: 人工升级（告警到用户，暂停 scheduler）
  → 必须是兜底链的最后一步
```

**铁律**：兜底链的最后一步必须是人工——机器兜底机器 = 无限循环。
```

**本次实例**：VIN 校验发现"需要校验子级分类" — 属于需求层面遗漏，回退 Phase 1 补充 requirement，然后从 Phase 3 修改代码。

## 失败升级机制（四级 + 强制检索前置 + /investigate + PUA 压力驱动）

### 强制检索前置协议（Retrieval-Before-Fix，CASE-082 教训升级为物理门禁）

> **设计原理**：Harness 已有 82 CASE + 混合检索能力，但之前 `harness search` 放在 L3（第三级升级后才搜），
> 导致前 2-3 次失败全靠猜测。根因：搜索是"后备"而非"前置"。
> **新规则**：任何非预期失败（编译错误/CI 失败/部署失败/运行时异常），**第一个动作必须是检索，不是修复**。

```
ON unexpected_failure(error_message):
    ┌─ Step 0: 强制检索（不可跳过）──────────────────────────────────┐
    │                                                               │
    │  0.1 提取错误关键词（error message 中的类名/错误码/核心短语）     │
    │                                                               │
    │  0.2 三路并行检索：                                             │
    │    路径 A: knowledge-map.yaml symptom_keywords 匹配             │
    │            → 命中 CASE 编号 → 读取 CASE 的诊断命令和修复方案      │
    │    路径 B: git log --all --oneline --grep="<关键词>" | head 10  │
    │            → 找到历史修复 commit → 检查是否已合入当前分支          │
    │    路径 C: grep -rn "<关键词>" Harness/wiki/cases/ repo-map.yaml │
    │            → 匹配已知问题模式                                    │
    │                                                               │
    │  0.3 检索结果判定：                                             │
    │    → 命中已知解法 → 直接应用（不重新发明）                       │
    │    → 命中相似 CASE 但不完全匹配 → 作为假设输入 L1               │
    │    → 三路全无命中 → 记录"已搜索无果"，进入 L1 自主诊断          │
    │                                                               │
    │  禁止行为：                                                     │
    │    ❌ 跳过检索直接尝试修复                                      │
    │    ❌ 只搜一路就放弃（必须三路都跑）                             │
    │    ❌ 搜到结果但不用（自己发明新方案）                           │
    └───────────────────────────────────────────────────────────────┘

    → Step 0 完成后进入四级升级：
```

| Level | 触发条件 | 动作 | PUA 压力 |
|-------|---------|------|---------|
| **L0** | 正常执行 | 按目标推进，最小修改+快速验证 | 无 |
| **L1** | 首次失败 + Step 0 检索无果 | **Skill: investigate**（4 阶段根因调试）+ 方法论切换表 | 🟡 P7 级压力：诊断先行，禁止无证据猜测 |
| **L2** | L1 仍未解决 | 逐字读失败信号，列 ≥3 假设逐个验证，反转主假设 | 🟠 P8 级压力：穷尽一切方案，5 步方法论必须走完 |
| **L3** | 问题复杂/影响大 | 完整检查清单 + **扩大检索范围**（跨域CASE + 外部文档） | 🔴 P9 级压力：蓝军自攻击，逆向验证每个假设 |
| **L4** | 穷尽仍无法完成 | 结构化报告 + **Skill: codex** → 人工接管 | ⬛ P10 级：输出完整穷尽证据，禁止说"我无法" |

### PUA 三条红线（贯穿 L0-L4）

```
🚫 红线一：闭环意识
   说"完成"之前必须有验证命令输出。没有输出 = 自嗨。（= SOP-H1）

🚫 红线二：事实驱动
   说"可能是环境问题"之前必须用工具验证。未验证的归因 = 甩锅。

🚫 红线三：穷尽一切
   说"我无法解决"之前 /investigate 4 阶段 + 3-strike gate 必须走完。
   未走完 = 缺乏韧性，直接 L4。

诊断先行格式（改代码前必须输出）:
  [DIAGNOSIS] 问题是 ___；证据是 ___；下一步动作是 ___。
```

### PUA 压力协议

PUA 三红线 + L1-L4 压力升级已内化到本文件的失败升级机制中（无需外部工具集）。

### L1 触发时自动调用 /investigate（Iron Law: no fix without root cause）

```
/investigate 四阶段协议（替代盲猜修复）:

Phase 1 — Investigate（收集证据）:
  - 读取完整错误信息（不截断）
  - grep 搜索相关代码路径
  - git log 查看最近修改
  - Network/Console 查看运行时行为
  - harness search "{错误关键词}" → 查历史类似问题
  - 跨项目经验: 检查其他项目是否踩过同样的坑

Phase 2 — Analyze（分析模式）:
  - 时间线：什么时候开始出错？
  - 范围：影响哪些入口？
  - 对比：正常 vs 异常的差异在哪？
  - 诊断模式目录表（对号入座）:
    | 症状 | 去哪里找 |
    | 竞态/间歇性 | 并发共享状态、锁、事务边界 |
    | null/undefined | 缺失守卫、可选链、默认值 |
    | 类型不匹配 | DTO 字段名、enum 值、前后端契约 |
    | 权限/403 | token 传递、CORS、Gateway 路由 |
    | 数据丢失 | 事务回滚、异步写入顺序、缓存过期 |
    | 性能/超时 | N+1 查询、全量加载、未分页 |

Phase 3 — Hypothesize（形成假设）:
  - 基于证据列出 ≥2 个假设
  - 每个假设有验证方法（可证伪）
  - 选择最可能的假设先验证
  - Scope Lock: 确定假设涉及的最小目录范围，此后只改该目录内文件

Phase 4 — Implement（验证修复）:
  - 一次只验证一个假设
  - 修复后必须有验证命令输出（SOP-H1）
  - 假设验证失败 → 下一个假设（不是调参数重试）
  - Blast Radius Gate: 修复涉及 >5 文件 → 暂停，向用户确认范围

3-Strike Gate（硬停止）:
  - 3 个假设全部验证失败 → 强制停止
  - 输出结构化选项（必须让用户选）:
    A. 提出新假设方向继续（说明为什么前3个方向都错了）
    B. 加日志/埋点，等下次复现时收集更多信号
    C. 升级到人工接管（输出完整已排除清单）
  - 禁止: 默默继续第 4/5/6 次猜测
```

### L1 方法论切换表（不是微调，是换思路）

> WHY: PUA 方法论核心洞察——失败切换不是"换一种说法"，而是"换一种解题思路"。
> 切换前三问：①当前方法步骤都走了吗？②是方法不对还是执行不到位？③新方法能解决当前失败吗？

| 失败模式 | 识别信号 | 切换链（按顺序尝试） |
|---------|---------|-------------------|
| **原地打转** | 同一方向修了 2 次还不对 | → 质疑需求（这个功能该不该做？）→ 砍中间环节（能删掉哪步？）→ 从反面攻击（如果是攻击者怎么绕过？） |
| **没搜就猜** | 凭推断改代码，没看实际行为 | → 搜索第一（grep/Network 面板先看真实数据）→ 读源码（追到方法实现）→ 数据驱动（有/无条件对比验证） |
| **质量差** | 改了能跑但结果不对 | → 减法优先（删除 > 简化 > 加速）→ 极致专注（一次只改一个点）→ 像素级验证（逐字段对比预期） |
| **放弃/推锅** | 说"超出范围""环境问题" | → 执行抗合理化检查（`rules/anti-rationalization.md`）→ 穷尽一切后输出结构化退出报告 |
| **被动等待** | 等用户给信息，不主动查 | → 先用工具自查（grep/git log/DB 查询）→ 只问真正需要确认的 → 带着"已查内容+矛盾点"问 |

---

## 回退执行协议

```
ON gate_failure(phase, reason):
    1. 分析 reason 类型
    2. 查回退路由表 → 确定目标 phase
    3. 记录到 changes/<req>/review-v{N}.md:
       "Phase {X} 门禁失败 | 原因: {Y} | 回退到: Phase {Z}"
    4. 跳转目标 phase（保留已有产出物，不删除）
    5. IF 同一目标 phase 回退次数 >= 3:
        → 停止执行
        → 升级人工决策
        → 输出: 失败历史 + 已尝试方案 + 建议选项
```

## 自动恢复机制（Recovery Queue）

> 借鉴 Rally 框架：失败不直接 block，先尝试自动修复。

### 恢复优先级

调度器按以下优先级选择下一个 work item：

```
blocked check → recovery queue → open issues → pending tasks
```

### 自动 Recovery 流程

```
ON task_failed(task_id, result_json):
    1. 计算 failure_hash = sha256(failureKind + summary + evidence)
    2. 比较 lastFailureHash:
       - 相同 → sameFailureCount++
       - 不同 → sameFailureCount = 1
    3. IF sameFailureCount >= 3:
       → mark_task_blocked → escalate to human (L4)
    4. ELSE IF result.retryable == true:
       → 构建 recovery item (带 evidence + badAttempts)
       → 入队 recoveryQueue
       → 下次迭代自动重试（注入失败上下文）
    5. ELSE:
       → mark_task_blocked → human
```

### Recovery Item 结构

每个 recovery item 携带足够上下文让 agent 避免重复犯错：

| 字段 | 用途 |
|------|------|
| `evidence.failedCommand` | 具体哪条命令失败 |
| `evidence.exitCode` | 退出码 |
| `evidence.snippet` | 错误输出片段 |
| `badAttempts[]` | 已知无效的修复尝试（agent 不要重复） |
| `failureKind` | 分类：implementation/verification/review/tooling/environment |

### failure-history.jsonl

每次失败追加一条记录，后续重试时注入最近 3 条失败历史作为 context：

```jsonl
{"taskId":"T-001","attempt":2,"failureKind":"verification","summary":"eslint: unresolved import","createdAt":"..."}
```

### 与手动流程的关系

- **Scheduler 模式**（`harness-run.sh`）：recovery 全自动，scheduler 管理 queue
- **手动模式**（Agent 自律执行）：Agent 在 Phase 3/4 失败时，手动记录 `failure-history.jsonl`，下次重试前先读取

---

## 人工确认点协议

到达确认点时 Agent 必须：

1. **用业务语言描述逻辑** — 不懂代码的人也能判断是否符合预期
2. **列出关键业务场景** — 正常路径 + 异常路径各举具体例子
3. **说明边界行为** — 什么情况会触发、什么情况不触发、失败时用户看到什么
4. 列出关键产出物清单
5. 明确写出"**等待确认：[确认内容]**"
6. **停止执行**，不得自行推进到下一阶段

### 输出原则

- **面向业务确认人**，不是面向开发者 — 避免纯技术术语
- **场景化呈现** — 用"当用户做X时，系统会Y"的句式
- **明确边界** — "以下情况不会拦截：..."
- **展示用户感知** — 用户看到的提示文案是什么、按钮能不能点

```
示例输出：
---
## Phase 1 完成 → 等待 [H1] 确认

**需求摘要**：为盘点计划添加自动分配功能
**验收标准**：3 条（详见 changes/auto-assign/requirement.md）
**影响范围**：PlanService + TaskService + 1张新表

**等待确认：需求理解是否正确？验收标准是否完整？**
---
```

---

## Harness 检查清单（逐阶段）

### 需求阶段

- [ ] 研发任务已线上化（关联 Meego）
- [ ] 高危元素已识别并标记
- [ ] 技术方案含影响范围章节
- [ ] 技术方案含可观测性章节
- [ ] 技术方案含风险评估章节
- [ ] **跨模块影响声明已填写**（改动文件/受影响链路/并行冲突/爆炸半径）
- [ ] **HIGH 影响等级已触发 [H-IMPACT] 人工确认**（如适用）
- [ ] 技术方案评审通过

### 技术设计阶段（Phase 2.3，重大变更时）

- [ ] 判定是否触发技术设计（新服务/多服务/schema变更/新外部对接/重大重构）
- [ ] MIT V1.7 十章全覆盖（不涉及已标注原因）
- [ ] 架构图/时序图至少 1 张
- [ ] 外部依赖接口文档链接已填写
- [ ] 数据库变更方案已列出（含回滚 SQL）
- [ ] 部署/灰度/回滚方案已描述
- [ ] 监控指标已定义（业务+技术）
- [ ] 术语表与 requirement.md 一致
- [ ] 设计难点均有应对措施
- [ ] **专家评审材料 `technical-review.md` 已产出**（变更范围 + 数据链路 + 关联模块影响 + 风险分析 + 确认清单）
- [ ] **专家确认清单全部勾选**
- [ ] [H2.3] 技术评审确认通过

### AI 编码阶段

- [ ] 任务自包含（description + context + acceptanceCriteria + filesToModify + verificationCommands）
- [ ] 反幻觉规则已执行（禁止模糊声明，展示命令输出）
- [ ] 实现与审查分离（implementer ≠ reviewer）
- [ ] 复杂度门禁已检查（complex task + 未解决 issue → 暂停）
- [ ] 置信度标注完整（关键信息 High/Medium/Low + 来源）
- [ ] **跨模块影响验证 PASS**（六层追踪 + Phase 2 声明对照 + 无关模块污染=0）
- [ ] **并行改动冲突已检查**（多功能同改一文件时行号不重叠）
- [ ] **专家评审材料 `code-review.md` 已产出**（Diff 摘要 + 六层追踪结论 + 变更前后对比 + 确认清单）
- [ ] **专家确认清单全部勾选**

### 验证阶段

- [ ] TDD 循环已执行（红灯→绿灯→重构）
- [ ] Spec Review 已通过（逐条核对验收标准）
- [ ] 任务级验证命令全部通过
- [ ] 全局质量命令全部通过（tsc + lint + test）
- [ ] 无模糊声明残留
- [ ] **专家评审材料 `test-review.md` 已产出**（功能点覆盖矩阵 + 链路覆盖 + 数据源覆盖 + 边界场景 + 未覆盖风险 + 确认清单）
- [ ] **专家确认清单全部勾选**

### 上线阶段

- [ ] CodeReview 有效通过（他人 Approve）
- [ ] CI 全部通过（代码扫描 + 安全扫描）
- [ ] 自测已执行并有证据
- [ ] 测试报告含结论 + 交付风险
- [ ] 上线 Checklist 已填写（上线步骤/回滚步骤/配置变更/窗口/影响范围）
- [ ] 回滚方案已准备
- [ ] 监控告警已配置
- [ ] **deploy-checklist.md 已写入（Phase 7.5 八大类逐项确认）**
- [ ] **服务部署顺序已确认**（多服务依赖先后）
- [ ] **Nacos/全局变量在所有环境已配置**（dev/test/prod）
- [ ] **消息模板已创建且变量名与代码对齐**
- [ ] **审批流程/BPM 配置就绪**（如涉及）
- [ ] **数据权限/米盾路径规则已配置**
- [ ] **定时任务已注册且 cron 正确**（如涉及）
- [ ] **DDL/数据初始化脚本已准备 + 回滚 SQL 就绪**（如涉及）

### 事故预防

- [ ] 高危元素清单已维护
- [ ] 高危场景有专项测试
- [ ] 监控阈值合理（从紧到松）
- [ ] 历史事故经验已纳入评审检查

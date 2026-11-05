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
| → Phase 1 | L1/L2：`exploration.md` 存在 + 用户已选方案；L3/L4：直接进入 | GOV-008 |
| → Phase 2 | `requirement.md` 存在 + gates.H1=passed + **领域知识已加载**（若匹配 AI-INDEX 项目） | GOV-021 |
| → Phase 3 | `review-v*.md` 存在 + P0=0 + gates.H2=passed | GOV-001+002 |
| → Phase 2.3 | 满足触发条件（新服务/3+微服务/DB变更）+ **本体注入表格已输出**（实体+状态机+影响链） | GOV-007 |
| → Phase 3 | `review-v*.md` 存在 + P0=0 + gates.H2=passed + **本体注入摘要已输出**（SOP-H4 三段自检） | GOV-001+002 |
| → Phase 4 | Phase 3 status=done（编译通过+代码已提交） | — |
| → Phase 5 | gates.H3=passed | **GOV-022** + SP-10 |
| → Phase 6.8 | Phase 6.5 status=done（测试全过）| GOV-013 + SP-11 |
| → Phase 7 | Phase 6.8 status=done（E2E 门禁4 PASS）或 status=skipped（**仅限纯文案/注释改动**，含跳过说明）。Direct Mode 豁免 | GOV-004 + **GOV-023** + SP-12 |
| → Phase 8 | gates.H4=passed | **GOV-024** + SP-13 |
| → Phase 9 | CI status=SUCCESS | GOV-005 |
| → Phase 10 | Phase 9 status=done | — |

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


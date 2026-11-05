---
name: qa-expert-agent
description: |-
  QA Expert Agent：贯穿全 Harness 流程的测试专家角色。
  不只是 Phase 9 验证——从需求评审就开始介入，驱动可测试性设计。
  初始化时声明知识源，执行时按项目配置加载上下文。
  触发词：全面验证、QA 验证、测试覆盖、跑一遍、验证全部功能点
version: 1.0.0
appendix_limit: 50
---

# QA Expert Agent

> 模拟一个从需求评审到上线验收全程参与的测试架构师。
> 不是"打开页面看看"——是**设计驱动、经验加权、闭环修复**的系统化验证。

---

## 项目初始化配置

> 新项目接入 Harness 时（Phase 0），必须填写 QA 配置块。
> 不同项目的知识源、认证方式、工具链不同——这里声明，后续自动路由。

### 配置文件位置

`Harness/changes/_template/qa-config.yaml` 或项目 `CLAUDE.md` 中的 `qa:` 块。

### 配置 Schema

```yaml
qa:
  # ─── 知识源（从哪加载业务理解）───
  prd_source: ""          # PRD 文件目录/路径模式
  ontology: ""            # 领域本体文件（状态机/实体关系）
  state_machines: []      # 需要验证的状态机枚举类名
  business_rules: ""      # 业务规则文件
  glossary: ""            # 术语表（防止误解）

  # ─── 经验源（从哪加载历史教训）───
  cases_dir: ""           # 历史 CASE 目录
  risk_heatmap: ""        # 风险热力图
  genes: ""               # Gene 检查指令来源
  known_bugs: []          # 已知未修复问题

  # ─── 测试环境（在哪执行）───
  test_env_url: ""        # 测试环境 URL
  local_start_cmd: ""     # 本地启动命令
  auth_method: ""         # cas-cookie | token | basic | none
  auth_config: ""         # 认证配置文件路径

  # ─── 工具链（用什么执行）───
  browser_tool: ""        # chrome-devtools-mcp | playwright-mcp | playwright-cli
  api_tool: ""            # curl | httpie | custom
  db_access: ""           # jdbc-url | none（是否可直查 DB 验证数据）

  # ─── 约束（哪些不可自动验证）───
  known_stubs: []         # Stub 化的外部依赖（不可 E2E）
  manual_only: []         # 必须人工验证的路径（扫码/硬件/审批）
  data_constraints: ""    # 测试数据约束（如"不可创建生产数据"）
```

### 示例：盘点引擎 v2

```yaml
qa:
  prd_source: "docs/systems/eam/domain/Atom PRD/plan/"
  ontology: "docs/systems/eam/domain/ontology.md"
  state_machines: ["PlanStatus", "TaskStatus", "ReportStatus(string)"]
  business_rules: "docs/systems/eam/domain/business-rules.md"
  glossary: "docs/systems/eam/domain/glossary.md"

  cases_dir: "Harness/wiki/cases/"
  risk_heatmap: "Harness/memory/risk-heatmap.yaml"
  genes: "Harness/evolution/genes.json"
  known_bugs: ["ADD_INFO无出口", "IN_DRAW无提交路径"]

  test_env_url: "https://mieam-asset.test.mioffice.cn/inventory/"
  local_start_cmd: "cd inventory-engine-v2 && bash run-dev.sh"
  auth_method: "cas-cookie"
  auth_config: ".env.token"

  browser_tool: "chrome-devtools-mcp"
  api_tool: "curl"
  db_access: "none"

  known_stubs: []
  manual_only: ["BPM审批回调", "飞书通知卡片", "移动端扫码盘点"]
  data_constraints: "测试环境可自由创建/删除数据"
```

---

## 全流程介入点

> QA Expert 不是 Phase 9 才出场——每个阶段都有 QA 视角输出。

### Phase 2 需求评审：可测试性审查

QA 视角的评审问题：

```
□ 每个功能点是否有明确的验收标准（Given/When/Then）？
□ 验收标准是否可程序验证（不是"体验良好"这种模糊表述）？
□ 状态流转是否完整定义（每个状态的进入/退出条件）？
□ 异常路径是否定义（超时/拒绝/中断后如何恢复）？
□ 外部依赖是否可 mock 或有降级方案？
□ 性能约束是否量化（N 条数据在 M 秒内完成）？
```

**输出**：在评审意见中增加"测试风险"章节。

### Phase 2.3 技术方案：测试策略前置

QA 视角的技术评审：

```
□ 关键路径能否端到端自动化验证？
□ 并发场景是否有锁/幂等保护？能否构造并发测试？
□ 数据迁移/兼容性如何验证（新旧共存期）？
□ 回滚方案是否可验证（回滚后数据一致性）？
□ 监控告警是否覆盖关键失败路径？
```

**输出**：在技术方案中增加"验证策略"章节（哪些自动化、哪些人工、哪些需要特殊环境）。

### Phase 4 代码评审：测试覆盖审查

QA 视角的代码评审：

```
□ 新增的条件分支是否每个分支都有对应 Case？
□ 状态转换的前置条件检查是否充分（防止非法状态）？
□ 错误处理是否有可观测性（日志/监控/告警）？
□ 是否有遗漏的边界：空集合、null、并发修改？
□ 历史 CASE 中标记的文件，本次修改是否注意了已知陷阱？
```

**输出**：评审意见中标注"测试建议"（哪些路径需要重点验证）。

### Phase 9 部署验证：系统化执行

进入完整的测试执行循环（见下方）。

---

## Phase 9 测试执行方法论

### Step 1: 上下文加载（读取 qa-config）

```
1. 加载 PRD → 提取功能点列表
2. 加载本体 → 提取状态机转换表
3. 加载 CASE/Gene → 标记高危路径
4. 加载 known_stubs → 标记不可自动验证项
5. 确认测试环境可达 + 认证有效
```

### Step 2: Case 矩阵设计

从知识源自动生成 Case，按 4 层展开：

| 层 | 来源 | 规则 |
|----|------|------|
| L1 Happy Path | PRD 验收标准 | 每个功能点 ≥ 1 条 |
| L2 Branch Path | 状态机 + 条件分支 | 每条转换边 1 条 |
| L3 Edge Case | 业务规则 + 通用边界 | 空/极值/并发/权限 |
| L4 Experience | CASE + Gene + risk-heatmap | 历史雷区全覆盖 |

**Case 格式**：

| ID | PRD# | Layer | Priority | Precondition | Steps | Expected | Method |
|----|------|-------|----------|-------------|-------|----------|--------|
| TC-001 | P07 | L1 | HIGH | Plan WAIT_SUBMIT | 点击生成任务 | Status→CREATING, 任务数>0 | E2E |
| TC-002 | P12 | L2 | HIGH | Plan 无任务 | 点击发布 | 拦截提示 | E2E |

**Priority 规则**：CRITICAL（历史复现）> HIGH（主路径+集成）> MEDIUM（分支）> LOW（展示）

### Step 3: 覆盖度自审

```
PRD 功能点覆盖率 = covered_prd / total_prd ≥ 80%
状态机转换覆盖率 = covered_transitions / total_transitions = 100%
高危路径覆盖率 = covered_risks / total_risks = 100%

覆盖率不达标 → 回到 Step 2 补充
```

### Step 4: 执行测试

按 priority 降序执行。每条 Case 记录：
- PASS / FAIL / BLOCKED / SKIP
- FAIL 时：截图 + 根因 + 修复方案

### Step 5: 修复驱动循环

```
WHILE (FAIL cases with priority ≥ HIGH):
  1. 定位根因（代码审计/调试）
  2. 输出修复方案（文件:行号 + diff）
  3. 执行修复 OR 交给开发者
  4. 回归验证（重跑 FAIL Case + 关联 Case）
  5. PASS → next; STILL_FAIL → 升级
```

### Step 6: 验收报告

```markdown
# QA 验收报告

## 结论：{SHIP / HOLD / REJECT}

| 指标 | 值 |
|------|-----|
| Case 总数 | N |
| PASS | X (X/N%) |
| FAIL(已修复) | Y |
| FAIL(遗留) | Z |
| BLOCKED | W |
| PRD 覆盖率 | % |

## 待人工验证（known_stubs + manual_only）
{列表}

## 新发现 → 归档
{CASE / Gene / risk-heatmap 更新}
```

---

## 与现有 Skills 的协作

| 本 Skill 步骤 | 调用 |
|-------------|------|
| 执行 E2E（交互式） | `skills/testing/browser-qa.md` |
| 执行 E2E（自动化） | `skills/testing/playwright.md` |
| 修复驱动 | `skills/coding/tdd-loop.md` |
| 知识加载 | `skills/memory/inject.md` |
| 归档经验 | `skills/evolution/phase10-feedback.md` |
| 影响传播 | `skills/review/impact-propagation.md` |

---

## 门禁

Phase 9 使用本 Skill 时的通过条件：
- Case 矩阵已生成且覆盖度评审通过
- CRITICAL/HIGH Case 全部 PASS
- 无 P0 遗留 FAIL
- 验收报告已输出
- 新发现已归档（CASE/Gene/risk-heatmap）

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

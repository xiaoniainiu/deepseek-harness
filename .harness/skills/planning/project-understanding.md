# 项目理解 — 重构/新增前的系统认知协议

> Phase 1 Mode A/B 前置步骤。不理解完整系统不动手。
> 灵感来源：Understand-Anything（知识图谱驱动的代码理解）+ GSD（决策覆盖门禁）

---

## 何时触发

| 场景 | 是否需要项目理解 |
|------|:---:|
| 重大重构（改架构/换框架） | **必须** — 完整六阶段 |
| 已有系统新增功能（Phase 1 Mode B） | **必须** — 至少阶段 1-3 |
| 首次接触的代码仓库 | **必须** — 完整六阶段 |
| 熟悉的仓库小改动（Mode C） | 跳过 |
| Bug fix（明确定位） | 跳过（但可选阶段 2 理解调用链） |

---

## 六阶段理解流程

> 原则：**结构提取先于 LLM 推理**。先用确定性工具扫描事实，再用 LLM 补充语义。

```
阶段 1        阶段 2        阶段 3        阶段 4        阶段 5        阶段 6
项目扫描 → 依赖图谱 → 架构分层 → 业务流程 → 变更影响 → 理解确认
(事实)     (事实)     (推理)     (推理)     (推理)     (人工)
```

### 阶段 1：项目扫描（确定性，不用 LLM）

**目标**：建立项目事实清单。

```bash
# 技术栈识别
cat pom.xml package.json build.gradle pyproject.toml 2>/dev/null | head -50

# 目录结构
find . -type f -name '*.java' -o -name '*.ts' -o -name '*.py' | head -100
tree -L 3 -I 'node_modules|target|dist|.git|__pycache__'

# 模块清单
ls src/main/java/com/mi/*/  # Java
ls src/views/ src/components/ src/api/  # 前端

# 配置文件
cat application.yml application-*.yml .env.example 2>/dev/null
```

**产出**：

```markdown
## 项目事实清单

- **仓库**：git@xxx
- **技术栈**：Java 8 + Spring Boot 2.3.7 + MyBatis-Plus
- **前端**：React 17 + Vite 5 + HiUI v4
- **模块数**：N 个 Java 包 / M 个前端页面
- **配置文件**：application.yml + application-{local,test,prod}.yml
- **数据库**：MySQL 8，N 张表
- **外部依赖**：BPM / SAP / 飞书 / Redis
- **部署方式**：Matrix / K8s / JAR
```

### 阶段 2：依赖图谱（确定性 + 轻量 LLM）

**目标**：理清模块间调用关系，找到高扇入（基础模块）和高扇出（编排模块）。

```bash
# Java：找出所有 import 关系
grep -rn "^import com.mi" src/main/java/ | awk -F: '{print $1, $NF}' | sort -u

# 前端：找出所有 import 关系
grep -rn "from ['\"]@/" src/ | awk -F: '{print $1, $NF}' | sort -u

# API 接口清单
grep -rn "@RequestMapping\|@GetMapping\|@PostMapping\|@PutMapping\|@DeleteMapping" src/ | head -50

# 数据库表关系
grep -rn "@TableName\|@Table\|@Entity" src/ | head -30

# 外部调用识别
grep -rn "RestTemplate\|WebClient\|Feign\|Dubbo\|@DubboReference\|HttpClient" src/
```

**产出**：

```markdown
## 依赖图谱

### 高扇入模块（基础层，改了影响大）
- AssetAccountMapper（被 12 个 Service 引用）
- BaseResult（所有 Controller 依赖）

### 高扇出模块（编排层，业务逻辑集中）
- PlanService（调用 7 个 Mapper + 3 个 External）
- TaskGenerationService（调用 5 个 Service）

### 外部系统调用点
- BpmClient → /process/submit（审批）
- SapClient → ZEAMFI01（转固）
- FeishuMessageClient → 飞书消息推送

### 数据库表关系
- asset_account → asset_inventory_task（1:N via plan_id）
- inventory_plan → inventory_task（1:N）
```

### 阶段 3：架构分层（LLM 推理）

**目标**：将模块归入架构层，验证层间调用是否合规。

参照 `rules/engineering-structure.md` 的五层架构：

| 层 | 职责 | 典型目录 |
|----|------|---------|
| Presentation | HTTP 路由、参数校验 | controller/ |
| Business | 业务编排、事务管理 | service/ |
| Domain | 实体、枚举、VO | entity/, dto/ |
| Data Access | CRUD、查询构建 | mapper/, repository/ |
| Integration | 外部调用、协议转换 | external/, client/ |

**检查**：
- Presentation 是否直接调了 Data Access？（违规）
- Integration 是否反向调了 Business？（违规）
- 有没有"万能 Service"（一个 Service 横跨所有层）？

### 阶段 4：业务流程映射（LLM 推理）

**目标**：把代码结构映射到业务流程，理解"用户做 X 时系统做了什么"。

**方法**：选取 3-5 个核心业务流程，逐步追踪：

```
用户操作 → 前端页面 → API 请求 → Controller → Service → Mapper/External → 数据变更
```

**⚠️ 追踪深度要求（必须到运行时层面）**：

追踪不能停在 Service 层。以下场景必须追到底：

| 场景 | 必须追到 | 为什么 |
|------|---------|--------|
| 搜索/查询功能 | 最终 SQL（fieldCode → 列名映射、Mapper XML） | fieldCode 和 DB 列名可能不一致（V08 案例：`factoryCode` → `factory_code` 但实际列名是 `sn`） |
| 外部系统调用 | 方法签名（参数类型、返回类型） | 泛型类型不匹配在编译期才报错（V08 案例：`Map<String,String>` vs `Map<String,Object>`） |
| 前端 hook/组件 | hook 内部行为（是否转换/注入额外参数） | hook 可能不透传输入（V08 案例：`handleSearch` 额外注入 `contains` 条件） |
| 定时任务 | 查询条件覆盖的数据范围 | 同表多类型数据需要 type 过滤（V08 案例：`reserveType` 未过滤导致误命中归还记录） |

```
// WHY: V08 三个运行时 bug 都因为追踪深度不够：
//   1. 搜索框架 fieldCode→SQL 映射没追到 CommonMapper.xml → 列名猜错
//   2. MessageService 方法签名没确认 → 参数类型不匹配
//   3. useSearchCombinationFilter hook 内部行为没分析 → 条件被注入 contains
// 教训：阶段 4 追到 Service 层只完成了 50%，必须追到 SQL/签名/运行时行为
```

**示例**：

```markdown
### 流程：创建盘点计划

1. 用户点击「新建计划」→ PlanCreate 页面
2. 填写表单 → POST /api/plan/create
3. PlanController.create() → 参数校验
4. PlanService.createPlan() → 
   - 生成计划编号（Redis INCR）
   - 保存计划主体（PlanMapper.insert）
   - 根据分配方式生成任务（TaskGenerationService）
   - 发送飞书通知（FeishuMessageClient）
5. 数据落库：inventory_plan + inventory_task
```

### 阶段 5：变更影响分析（LLM 推理）

**目标**：基于阶段 2-4 的理解，预判本次改动的影响范围。

```markdown
## 变更影响分析

### 计划改动
- 修改 PlanService.createPlan() 的任务生成逻辑

### 直接影响
- PlanController（调用方）
- TaskGenerationService（被调用方）
- inventory_task 表（数据变更）

### 间接影响
- TaskPublishService（消费 task 数据）
- 飞书通知模板（task 数量变化可能影响文案）
- 盘点计划列表页（task 统计数字）

### 不受影响（确认排除）
- 资产台账模块（无调用关系）
- 审批流程（plan 创建不触发审批）
```

### 阶段 6：理解确认（人工门禁 [H0]）

**输出格式**（面向业务确认人 + 技术确认人）：

```markdown
## 项目理解确认

### 我理解的系统架构
[一段话总结 + 简图]

### 我理解的核心业务流程
[3-5 个流程的一句话描述]

### 本次改动的影响范围
[直接影响 + 间接影响 + 明确排除]

### 我的疑问（需要确认）
1. [具体问题]
2. [具体问题]

**等待确认：以上理解是否正确？有遗漏吗？**
```

---

## 与 Phase 1 的关系

```
项目理解（本 Skill）→ Phase 1 需求分析 → Phase 2 需求评审 → ...
```

- **首次接触的仓库**：必须先走完六阶段，再进 Phase 1
- **熟悉的仓库 Mode B**：跳过阶段 1（已知事实），从阶段 2 开始
- **Mode A 全新需求**：阶段 1-3 帮助理解系统现状，阶段 4 可跳过
- **Mode C 小改动**：跳过本 Skill

## 产出物

写入 `changes/<req>/project-understanding.md`，后续 Phase 1-3 引用。

## 工具支持

| 工具 | 用途 | 阶段 |
|------|------|------|
| `grep -rn` / `find` | 结构扫描（确定性） | 阶段 1-2 |
| `tree` | 目录结构 | 阶段 1 |
| Chrome DevTools MCP | 前端页面结构 | 阶段 4 |
| `git log --oneline -20` | 近期变更历史 | 阶段 1 |
| `git blame <file>` | 特定文件变更者 | 阶段 5 |
| Understand-Anything 插件 | 自动化知识图谱（可选） | 阶段 1-3 |

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 只看"最明显的入口" | 遗漏 5+ 个写入路径 | 阶段 2 穷举所有调用关系 |
| 用 LLM 猜测架构 | 幻觉"这个类应该负责XXX" | 先 grep 确认事实，再推理 |
| 跳过影响分析 | 改了 A 忘了 B | 阶段 5 列出直接+间接+排除 |
| 不确认就动手 | Phase 3 实现后发现理解偏差 | 阶段 6 人工确认 |
| 用整个仓库喂 LLM | 上下文窗口溢出 | 按阶段逐步加载最小文件集 |

---

## GSD 框架借鉴

### 决策覆盖门禁

从 GSD 的 Decision Coverage Gate 借鉴：项目理解阶段产生的每个关键发现（标记 `U-NN`），必须在后续 Phase 1 requirement.md 或 Phase 2.3 technical-design.md 中被引用。

```markdown
## 关键发现清单

- U-01: PlanService 有 3 个重载方法，必须确认改哪个
- U-02: 任务生成依赖 Redis INCR，需要确认并发安全
- U-03: 飞书通知在 createPlan 内同步调用，可能阻塞

→ Phase 1 requirement.md 必须引用 U-01~U-03
→ 未引用的 U-NN 项在 Phase 2 评审时视为遗漏
```

### 节点修复策略

从 GSD 的 Node Repair 借鉴三种修复策略（更新 Harness 回退路径）：

| 策略 | 适用场景 | 动作 |
|------|---------|------|
| **RETRY** | 临时性失败（网络/环境） | 原任务带失败上下文重试 |
| **DECOMPOSE** | 任务过于复杂 | 拆为 2-3 个更小的子任务 |
| **PRUNE** | 任务不可行或不再需要 | 标记为 skipped，记录原因 |

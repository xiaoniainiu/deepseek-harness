---
name: code-navigation
description: |-
  结构化代码导航——调用链追踪、定义跳转、引用搜索、近符号上下文检索。
  替代盲目 grep，用代码结构理解影响范围。Phase 3/4 编码和评审时使用。
version: 1.0.0
trigger: Phase 3 影响范围分析 / Phase 4 跨模块影响验证 / 任何需要理解调用关系时
appendix_limit: 50
---

# Code Navigation（结构化代码导航）

> 对标 GBrain Code Cathedral II：code-callers/callees/def/refs/near-symbol walk。
> 工程适配：用代码结构理解影响范围，替代盲目 grep。

## 核心能力

| 操作 | 引擎命令（优先） | Fallback（grep） |
|------|----------------|-----------------|
| **谁调用了这个符号** | `harness code-callers SymbolName` | `grep -rn "methodName(" --include="*.java"` |
| **这个符号调用了什么** | `harness code-callees SymbolName` | 读取方法体，提取 `.method()` 调用 |
| **符号定义在哪** | `harness code-def SymbolName` | `grep -rn "class\|def\|function SymbolName"` |
| **所有引用点** | `harness code-refs SymbolName` | `grep -rn "SymbolName" --include="*.java"` |
| **近符号上下文** | `harness query "问题" --near-symbol X --walk-depth 2` | 手动 callers+callees 组合 |

### 引擎能力（tree-sitter + call-graph，36 种语言）

```bash
cd ~/.claude/harness/engine

# 代码需要先索引到 brain
GBRAIN_HOME=../.harness/brain bun run src/cli.ts import /path/to/project/src

# 然后使用结构化导航
GBRAIN_HOME=../.harness/brain bun run src/cli.ts code-callers PlanService
GBRAIN_HOME=../.harness/brain bun run src/cli.ts code-callees PlanService.finishPlan
GBRAIN_HOME=../.harness/brain bun run src/cli.ts code-def ScopeQueryBuilder
GBRAIN_HOME=../.harness/brain bun run src/cli.ts code-refs BpmClient

# 近符号语义检索（混合：向量 + call-graph 邻居）
GBRAIN_HOME=../.harness/brain bun run src/cli.ts query "N+1 handling" --near-symbol BrainEngine.searchKeyword --walk-depth 2
```

**技术栈**：tree-sitter WASM（36 语言 grammar）+ edge-extractor（call/import/extends/implements/type_refs）+ symbol-resolver（qualified name across scopes）+ two-pass retrieval（anchor + neighbors）

**支持的语言**（tree-sitter WASM grammar）：
Java, TypeScript, TSX, JavaScript, Python, Go, Rust, Ruby, C, C++, C#, Kotlin, Scala, Swift, PHP, Dart, Elixir, Lua, OCaml, Vue, YAML, JSON, HTML, CSS, Bash, SQL, Zig, Solidity, TOML, Elm + 更多（共 36 种）

**Fallback**：项目代码未导入 brain 时，自动降级为 grep 模式（上表右列）。

## 结构化影响分析协议

Phase 4 评审的"影响传播分析"使用此 skill 替代简单 BFS：

```
给定修改的方法 M:

Step 1: Callers（谁调用了 M）
  grep -rn "M(" --include="*.java" src/ | grep -v "def \|class \|interface "
  → 产出 caller_list[]

Step 2: Callees（M 调用了什么）
  读取 M 的方法体，提取 this.xxx() / injected.xxx() / static.xxx()
  → 产出 callee_list[]

Step 3: 同接口实现（M 是接口方法？有几个实现？）
  grep -rn "implements.*InterfaceName\|extends.*ClassName" --include="*.java"
  → 产出 impl_list[]

Step 4: 影响矩阵
  对 caller_list 每个 caller:
    - 是 Controller → 影响 API 层
    - 是 Scheduled Task → 影响定时任务
    - 是 Event Handler → 影响异步流
    - 是另一个 Service → 继续递归（depth ≤ 2）

Step 5: 输出
  | 影响层 | 文件 | 调用路径 | 状态 |
  |--------|------|---------|------|
  | API | XxxController.java:42 | Controller→Service→M | 需验证 |
  | 定时任务 | XxxTask.java:18 | Task→Service→M | 需验证 |
  | 下游 Service | YyyService.java:95 | Yyy→M | 已覆盖 |
```

## 近符号上下文检索（Near-Symbol Walk）

```
目标: 理解一个符号的"局部生态"——不只看定义，还看谁用它、它用什么、相关的测试。

walk(symbol, depth=2):
  1. 找到 symbol 定义（class/method/function）
  2. depth=1: 
     - callers（谁直接调用它）
     - callees（它直接调用什么）
     - 同文件的其他 public 方法（sibling context）
  3. depth=2:
     - callers 的 callers
     - callees 的 callees
     - 相关的 test 文件（grep test 目录中对 symbol 的引用）
  4. 收集所有找到的文件+行号
  5. 按距离排序（depth=1 优先）
  6. 截取前 50 个结果（neighbor cap）

输出格式:
  anchor: "PlanService.finishPlan" (src/service/PlanService.java:234)
  depth_1_callers: ["PlanController.closePlan:42", "ScheduledFinisher.run:18"]
  depth_1_callees: ["TaskService.closeAllTasks", "ReportService.generateFinalReport", "BpmClient.complete"]
  depth_2_callers_of_callers: ["HTTP POST /api/plan/{id}/finish", "Cron 每日 23:00"]
  related_tests: ["PlanServiceTest.testFinishPlan", "PlanControllerIT.testClosePlan"]
```

## 前端适配

TypeScript/React 项目的代码导航：

```
组件引用链:
  grep -rn "import.*ComponentName\|<ComponentName" --include="*.tsx" --include="*.ts" src/

Hook 调用链:
  grep -rn "useHookName\(" --include="*.tsx" --include="*.ts" src/

Store 消费者:
  grep -rn "useAppSelector.*sliceName\|dispatch.*actionName" --include="*.tsx" --include="*.ts" src/

路由注册:
  grep -rn "path.*routePath\|component.*ComponentName" --include="*.tsx" --include="*.ts" src/routes/
```

## 与现有机制的集成

| 现有机制 | 增强方式 |
|---------|---------|
| S3 影响传播（ontology/links/ BFS） | Code Navigation 提供代码级调用链，与实体级 BFS 互补 |
| Phase 4 跨模块影响验证 | 用 callers/callees 替代人工推理"还有谁用了这个" |
| Gene GENE-P1-001 (all-write-paths-enumeration) | Code Navigation 是其执行工具 |
| SOP-H5 影响传播必查 | Code Navigation 输出即 BFS 的代码级补充 |

## 使用时机

- Phase 3 Step 0: 心智模型构建时，walk(target_method, depth=2) 理解上下文
- Phase 4: 评审时对每个修改的方法执行 callers 分析
- Phase 2.3: 技术设计时用 near-symbol 理解现有架构
- Debug 时: 从报错点向上追溯调用链

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

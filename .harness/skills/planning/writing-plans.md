# Writing Plans — 将需求评审产出转化为 plan.json

> Phase 2 产出物。将 requirement.md 的需求分析转化为 scheduler 可执行的结构化任务列表。

## 触发时机

- Phase 2 评审通过（[H2] 用户确认方案可行）
- 用户要求生成 plan.json

## 输入

| 来源 | 必须 | 说明 |
|------|------|------|
| `requirement.md` | 是 | 需求分析文档（Phase 1 产出） |
| `tasks.md` | 否 | 人工拆解的任务列表（可作为参考） |
| API 文档 | 否 | OpenAPI/Swagger 文件 |

## plan.json 结构

```json
{
  "project": "项目名",
  "branchName": "feat/功能名",
  "description": "一句话描述",
  "qualityCommands": {
    "compile": "编译命令",
    "test": "测试命令",
    "lint": "lint 命令"
  },
  "tasks": []
}
```

## 自包含任务标准

每个 task 必须可被 fresh agent 独立执行，不依赖隐含上下文：

| 字段 | 要求 |
|------|------|
| `id` | T-001 格式，连续递增 |
| `title` | 动词开头，一句话 |
| `type` | backend / ui / integration — 按验证策略选择 |
| `description` | 起始状态 + 目标行为 + 关键文件。禁止"参考现有模式"不给路径 |
| `acceptanceCriteria` | 每条可二值判定（pass/fail），禁止"正确处理""合理支持" |
| `testDescription` | 具体输入、输出、边界、mock 策略 |
| `testFile` | 精确测试文件路径 |
| `filesToModify` | 涵盖所有预期变更文件（含新建） |
| `verificationCommands` | 直接证明任务完成的命令。每条可附 `concurrencyType: "readonly"\|"mutating"`（默认 mutating）。readonly 命令可并行执行，mutating 强制串行。见 `scheduler/lib/tool-partitioning.md` |
| `context` | 框架版本、ORM、关键文件位置、既有模式（内联描述） |
| `dependsOn` | 前置 task ID 数组 |
| `complexity` | simple / moderate / complex |
| `priority` | 数字越小越优先 |
| `status` | pending（初始态） |

## 任务拆分原则

1. **Foundation first** — 数据层 → 服务层 → 接口层 → 前端
2. **一个任务一个关注点** — 禁止一个 task 跨多层
3. **依赖明确** — `dependsOn` 反映真实顺序，scheduler 据此调度
4. **TDD by default** — 每个 backend/integration task 必须有 testDescription + testFile
5. **UI 任务不强制 TDD** — 用 typecheck + lint + build 验证

## 任务粒度标准

每个 Step 应在 **2-5 分钟内完成**。粒度参照：

```
写失败的测试      → 1 步
运行确认失败      → 1 步
写最简实现        → 1 步
运行确认通过      → 1 步
提交              → 1 步
```

如果一个 Step 预估 > 5 分钟，必须继续拆分。

## 红旗模式扫描（计划自检）

写完 plan.json 后，必须扫描以下**禁止出现**的模式：

| 红旗 | 示例 | 为什么禁止 |
|------|------|-----------|
| 占位符 | "TBD"、"TODO"、"implement later" | 子 Agent 拿到无法执行 |
| 省略引用 | "类似 Task N"、"参考上一步" | 子 Agent 无隐含上下文 |
| 模糊处理 | "添加适当的错误处理" | 无法二值判定 |
| 无路径 | "参考现有模式实现" | 不给文件路径等于没说 |
| 无验证 | 只说做什么不说怎么验证 | 无法证明完成 |

扫描方法：`grep -inE "TBD|TODO|implement later|类似 Task|参考上[一个]步|适当|按需" plan.json`

## 描述质量检查

**Bad**:
- "参考现有模式实现" — 不给具体路径
- "正确处理错误" — 无法判定什么是"正确"
- "按需添加测试" — 不具体

**Good**:
- "参考 `src/service/ReportService.java:exportExcel()` 的 EasyExcel 导出模式"
- "`planId` 不存在时返回 HTTP 404 + `{code: 'PLAN_NOT_FOUND'}`"
- "测试：输入存在的 planId 返回 200+xlsx header；不存在返回 404"

## 输出

1. 写入 `changes/<req>/plan.json`
2. 输出摘要：任务数量、依赖图、复杂度分布
3. 提示用户审核后启动 scheduler

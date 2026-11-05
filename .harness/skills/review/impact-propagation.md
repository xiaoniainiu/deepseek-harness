---
name: impact-propagation
description: "Phase 4 评审时从 ontology 实体关系图推导影响传播链，自动识别下游受影响组件"
version: 1.0.0
trigger_phase: [4]
appendix_limit: 50
---

# Impact Propagation（影响传播模拟）

## 触发条件

- Phase 4 编码评审（自动触发）
- Phase 2 需求评审（手动触发 — 关键字"影响分析"）

## 执行流程

```
1. 获取 diff 文件列表:
   git diff --name-only <base>..HEAD

2. 读 ontology/links/inventory-links.yaml → file_entity_map
   匹配 diff_files → 受影响实体列表

3. FOR EACH 受影响实体:
   从 links[] 中找 from=该实体 的所有关系
   沿 propagation 方向 BFS 遍历（max_depth=3）
   收集 downstream 中的组件

4. 去重 + 按风险排序:
   如果下游组件在 code-knowledge-index.yaml 中 risk=CRITICAL → 标 HIGH
   如果下游组件在当前 diff 中也被修改 → 标 COVERED
   如果下游组件未被修改 → 标 NEEDS_VERIFICATION

5. 输出传播链（使用 phase4_output_template）
```

## 输出格式

```markdown
## 影响传播分析

| 修改文件 | 对应实体 | 传播路径 | 下游受影响 | 状态 |
|---------|---------|---------|-----------|------|
| PlanService.java | InventoryPlan | Plan→ScopeQuery→TaskGen | TaskGeneration | NEEDS_VERIFICATION |
| ScopeQueryBuilder.java | ScopeQuery | Scope→TaskGen→AssetList | AssetList, UnassignedAssets | NEEDS_VERIFICATION |

### 必须额外验证的下游:
- **TaskGenerationService**: scope 变更直接影响资产筛选结果
- **UnassignedAssets**: 诊断逻辑依赖 scope 筛选结果的完整性
```

## 预算控制

- 传播分析输出最多 1000 字符
- 超出时只保留 NEEDS_VERIFICATION 的链路
- 不在 Phase 4 primary 预算中扣减（与 C7 AP 注入同机制）

## 与六层追踪的关系

影响传播是六层追踪的**自动化前置**：
- 六层追踪需要人工逐层确认
- 影响传播自动列出"应该追踪哪些层"
- 如果传播输出中有 NEEDS_VERIFICATION 但六层追踪未覆盖 → 评审 FAIL

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

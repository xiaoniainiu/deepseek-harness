---
name: expert-review-template
description: |-
  技术方案/代码/测试用例三阶段专家评审模板。为人工专家提供结构化的评审材料，
  30秒定位该看什么，5分钟完成有效评审。
version: 1.0.0
appendix_limit: 50
---

# 专家评审材料模板

> 每个评审阶段产出一份结构化文档，放在 `changes/<req>/` 目录下。
> 专家只需要判断"检查做对了没"和"还有没有遗漏"。

---

## 一、技术方案评审（Phase 2.3 [H2.3]）

文件名：`changes/<req>/technical-review.md`

```markdown
# 技术方案评审材料

## 1. 变更概述（30秒读完）

| 维度 | 内容 |
|------|------|
| 一句话 | {改什么、为什么改} |
| 触发原因 | {用户报告 / 监控告警 / 需求迭代} |
| 紧急程度 | {P0线上故障 / P1功能缺陷 / P2优化} |

## 2. 变更范围（专家首先看这里）

### 改动文件清单

| 文件 | 改动类型 | 改动方法/函数 | 影响行数 |
|------|---------|-------------|---------|
| FunQueryService.java | 修改 | getComboValuesUncached | +17 |
| scope.tsx | 修改 | transformToConditionSearchData | +6 |

### 涉及的数据表

| 表名 | 操作 | 影响 |
|------|------|------|
| amg_dict | 新增 SELECT 查询 | 只读，无写入风险 |
| amg_business_line | 新增 WHERE 条件 | 缩小查询范围，不改数据 |

### 涉及的 API

| API | 变更 | 调用方 |
|-----|------|--------|
| GET /fun/fields | 返回的 filedSource 从 null 变为有值 | 前端 scope.tsx ConditionList |

## 3. 完整数据链路（专家重点审查）

```
用户操作: 打开盘点计划编辑页 → 盘点范围 → 条件筛选
    ↓
前端请求: GET /inventory/fun/fields?funId=asset_account
    ↓
后端处理: FunQueryService.getFields()
    → getFieldsUncached() L170: getComboValues(controlCode, businessLine)
    → getComboValuesUncached() 查找链:
       1. ENUM_COMBO_MAP（内存硬编码） → 无
       2. amg_business_line（业务线专用） → 仅 businessLine 字段匹配
       3. amg_fun_dict（v2 自定义字典） → 7 个字段命中
       4. amg_dict（字典主表） → 11 个字段命中 ← 新增
    ↓
返回数据: filedSource: [{valueData, displayData}, ...]
    ↓
前端渲染: ConditionList → CheckSelect data={selectData}
```

## 4. 关联模块/功能影响评估

### 直接影响

| 功能 | 页面/入口 | 影响描述 | 风险等级 |
|------|---------|---------|---------|
| 盘点计划-盘点范围条件筛选 | 编辑页 Step 2 | 下拉选项从空变为有值 | LOW（修复行为） |
| 盘点计划-列表页列标题 | 列表页 | 列标题从 DB 名变为中文 | LOW |

### 间接影响（需专家确认）

| 功能 | 可能的影响 | 需要验证的点 |
|------|----------|------------|
| 盘点任务详情页 | 也调用 fun/fields，filedSource 变化可能影响列渲染 | 确认任务详情的 select 列是否正常 |
| 盘点报告页 | 同上 | 确认报告页的筛选条件是否正常 |
| 其他管理线的盘点计划 | businessLine 过滤可能影响 | 切换管理线验证 |

## 5. 风险分析

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| amg_dict 表无 sort 字段 | 低 | SQL 报错 | ORDER BY sort, id → sort 不存在时改用 ORDER BY id |
| 缓存 5 分钟内看到旧数据 | 确定 | 用户短暂看到空下拉 | 部署后等 5 分钟或重启服务 |
| 业务线过滤遗漏某些合法数据 | 低 | 下拉缺选项 | manage_line_code 为空时返回全量 |

## 6. 专家确认清单

- [ ] 数据链路完整性：从用户操作到 DB 查询，每一跳都追踪到了？
- [ ] 变更范围准确性：改动文件清单是否完整？有没有遗漏的文件？
- [ ] 关联影响评估：间接影响的模块都识别到了？
- [ ] 回滚方案：如果出问题怎么回退？（回退 tag / 无 DDL 变更）
```

---

## 二、代码评审（Phase 4 [H3]）

文件名：`changes/<req>/code-review.md`

```markdown
# 代码评审材料

## 1. Diff 摘要

| 文件 | +/- 行数 | 改动概要 |
|------|---------|---------|
| FunQueryService.java | +20/-8 | amg_dict fallback + businessLine 透传 |
| scope.tsx | +6/-6 | fieldValues 逗号/分号拆分 |

## 2. 六层追踪验证

| 层 | 检查项 | 结论 | 证据 |
|----|--------|------|------|
| 服务层 | 改动方法的所有调用方 | 2处已更新 | grep "getComboValues" → L87, L171 |
| SQL 层 | 新增查询的 SQL 安全性 | 参数化查询 | `WHERE parent_code=?` |
| 签名层 | 方法重载兼容性 | 保留无参版本 | getComboValues(code) → getComboValues(code, null) |
| 缓存层 | 缓存 key 是否正确隔离 | 含 businessLine | cacheKey = srcCode + "\|" + bl |
| 多链路层 | 不同入口是否都覆盖 | scope/列表/详情 | 无 businessLine 时走全量 |
| 门禁层 | 运行时条件是否正确 | manage_line_code 过滤 | 空值不过滤 |

## 3. 变更前后对比

### 变更前
```
getComboValuesUncached("amg_check_status")
  → amg_fun_dict: 无数据
  → fetchComboValuesFromRemote: 被 SecProxy 拦截
  → 返回空 → filedSource=null → "暂无数据"
```

### 变更后
```
getComboValuesUncached("amg_check_status", "car_asset")
  → amg_fun_dict: 无数据
  → amg_dict: 6 条数据 ✅
  → 返回 [{code:"wait_accept", name:"待验收"}, ...]
```

## 4. 专家确认清单

- [ ] 每个 diff 改动行都能追溯到需求？（无无关改动）
- [ ] SQL 无注入风险？（参数化 / 白名单校验）
- [ ] 异常处理完备？（try-catch + 降级逻辑）
- [ ] 向后兼容？（旧调用方不受影响）
- [ ] 性能影响？（新增 DB 查询的频率和数据量）
```

---

## 三、测试用例评审（Phase 6）

文件名：`changes/<req>/test-review.md`

```markdown
# 测试用例评审材料

## 1. 覆盖矩阵

### 功能点覆盖

| 功能点 | 场景 | 测试用例 | 覆盖状态 |
|--------|------|---------|---------|
| F1: 验收状态下拉 | 正常显示 6 个选项 | TC-01 | ✅ |
| F1: 验收状态下拉 | 选择/取消选择 | TC-02 | ✅ |
| F2: 业务线下拉过滤 | car_asset 管理线只显示 car | TC-03 | ✅ |
| F2: 业务线下拉过滤 | 不传 businessLine 显示全量 | TC-04 | ✅ |
| F3: fieldValues 拆分 | 逗号分隔拆分 | TC-05 | ✅ |
| F3: fieldValues 拆分 | 分号分隔拆分 | TC-06 | ✅ |
| F3: fieldValues 拆分 | 正常数组不处理 | TC-07 | ✅ |

### 链路覆盖

| 链路 | 入口 | 测试用例 | 覆盖状态 |
|------|------|---------|---------|
| 新建计划 → 默认条件 | filterDefaultValues | TC-08 | ✅ |
| 编辑计划 → 回填条件 | complexInventoryScope | TC-09 | ✅ |
| 切换管理线 → 业务线变化 | bizLine change | TC-10 | ⚠️ 手动验证 |

### 数据源覆盖

| 数据源 | fieldSourceCode | 测试用例 | 覆盖状态 |
|--------|----------------|---------|---------|
| amg_fun_dict（有数据） | amg_intact_status | TC-11 | ✅ |
| amg_dict（fallback） | amg_check_status | TC-12 | ✅ |
| amg_business_line（特殊） | amg_business_line | TC-13 | ✅ |
| 不存在的字典 | amg_nonexistent | TC-14 | ✅ 返回空 |

## 2. 边界场景

| 场景 | 输入 | 期望输出 | 测试用例 |
|------|------|---------|---------|
| 缓存刚过期 | 5分钟后首次请求 | 重新查 DB，返回最新数据 | TC-15 |
| businessLine 为空 | getComboValues("amg_business_line", null) | 返回全量业务线 | TC-04 |
| fieldValues 含尾部分号 | ["accepted,fixed,pre_fixed;"] | 拆为 ["accepted","fixed","pre_fixed"] | TC-06 |
| amg_dict 表为空 | parent_code 不存在 | 返回空数组，前端显示空 | TC-14 |

## 3. 未覆盖的风险（需专家判断）

| 风险场景 | 未覆盖原因 | 建议 |
|---------|----------|------|
| 并发修改 amg_dict | 低概率，无锁竞争 | 接受风险 |
| 大量 fieldSourceCode 的性能 | 每个 code 独立查 DB | 有 5 分钟缓存兜底 |
| 其他管理线切换后缓存互串 | 缓存 key 含 businessLine | 已隔离，低风险 |

## 4. 验证命令（专家可复验）

```bash
# E2E 验证：打开 V2 盘点计划编辑页
# URL: /inventory/asset/inventory/plan/edit?id=3602&embed=1&bizLine=car_asset
# 检查：验收状态下拉应有 6 个选项，使用状态下拉应有选项

# API 验证：
curl -b cookie.txt 'https://mieam-asset.test.mioffice.cn/inventory/fun/fields?funId=asset_account' \
  | jq '.data.list[] | select(.fieldCode=="checkStatus") | .filedSource | length'
# 期望：6

# DB 验证：
SELECT COUNT(*) FROM amg_dict WHERE parent_code='amg_check_status' AND is_deleted=0;
# 期望：6
```

## 5. 专家确认清单

- [ ] 功能点全部覆盖？（对照需求清单逐项 check）
- [ ] 正常路径 + 异常路径 + 边界值都有用例？
- [ ] 数据链路中每个节点都有验证？（前端→API→后端→DB）
- [ ] 回归风险已评估？（改动不影响已有功能）
- [ ] 性能场景已考虑？（数据量、并发、缓存）
```

---

## 使用规范

| Phase | 产出文件 | 专家角色 | 评审重点 |
|-------|---------|---------|---------|
| 2.3 | technical-review.md | 架构师 | 数据链路 + 变更范围 + 关联影响 |
| 4 | code-review.md | 高级开发 | 六层追踪 + Diff 逐行 + 安全性 |
| 6 | test-review.md | 测试专家 | 覆盖矩阵 + 边界场景 + 可复验性 |

**铁律**：每个 `- [ ]` 确认项必须由专家逐项 check。Agent 不能自行勾选。

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

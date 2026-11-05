# 知识元数据规范

> 知识库膨胀后，精准注入 = 精准标记。每条知识都必须带元数据标签，
> 支持按项目/需求/领域/环节/用户维度动态过滤和按需加载。

## 必须标记的 6 个维度

| 维度 | 字段名 | 类型 | 示例 | 用途 |
|------|--------|------|------|------|
| **项目** | `project` | string | `eam-v2` / `asset-eam-web` | 按项目过滤相关知识 |
| **领域** | `domain` | string | `eam` / `vehicle` / `inventory` | 按业务领域加载本体 |
| **环节** | `phase` | int[] | `[3, 4]` | Phase 转换时只注入对应环节的知识 |
| **需求** | `requirement` | string | `V08-vehicle-borrow` | 追溯知识的来源需求 |
| **用户** | `author` | string | `tankunshan` | 知识贡献者（组织级知识图谱） |
| **标签** | `tags` | string[] | `[SecProxy, RestTemplate, 302]` | 关键词匹配注入 |

## 可选标记

| 维度 | 字段名 | 类型 | 示例 |
|------|--------|------|------|
| **技术栈** | `tech_stack` | string[] | `[Java, React, MyBatis]` |
| **微服务** | `service` | string | `asset-account-be` |
| **严重性** | `severity` | string | `critical` / `high` / `medium` |
| **适用范围** | `scope` | string | `universal` / `domain-specific` / `project-specific` |

## CASE 文件 frontmatter 格式

```yaml
---
importance: 0.8
tags: [SecProxy, RestTemplate, 302, 远程HTTP]
access_count: 0
last_accessed: 2026-05-14
# === 知识元数据（必填） ===
project: eam-v2
domain: eam
phase: [3, 7]
requirement: inventory-v2-audit
author: tankunshan
scope: universal          # universal = 所有项目适用
tech_stack: [Java, Spring Boot]
---
```

## Gene 元数据格式

```yaml
- id: GENE-P3-001
  name: no-new-resttemplate
  phase: [3]
  # === 知识元数据 ===
  project: eam-v2          # 首次发现的项目
  domain: eam              # 首次发现的领域
  author: tankunshan       # 提炼者
  scope: universal         # 适用范围
  tech_stack: [Java]       # 适用技术栈
  requirement: CASE-067    # 来源需求/案例
```

## 本体事件元数据

```jsonl
{
  "type": "correction",
  "domain_id": "eam",
  "target_entity": "vehicle",
  "project": "eam-v2",
  "author": "tankunshan",
  "requirement": "V08",
  "phase": 3,
  "task_id": "T-068",
  "timestamp": "2026-05-14T10:00:00Z"
}
```

## 按维度过滤的查询场景

| 场景 | 查询条件 | 返回 |
|------|---------|------|
| Phase 3 编码注入 | `phase contains 3 AND (project=当前项目 OR scope=universal)` | 当前项目+通用知识 |
| 新项目冷启动 | `scope=universal AND domain=当前领域` | 通用+同领域经验 |
| 本体加载 | `domain=当前领域` | 领域本体实体 |
| 用户贡献排行 | `GROUP BY author COUNT(*)` | 知识贡献度 |
| 技术栈过滤 | `tech_stack contains Java` | Java 项目的知识 |

## 知识注入优先级

当知识库 > 100 条时，按以下优先级排序注入（budget 有限时截断低优先级）：

```
1. 当前项目 + 当前 Phase + 高 importance  → 最高优先
2. 当前领域 + 当前 Phase + universal       → 高优先
3. universal + 当前 Phase                  → 中优先
4. 当前领域 + 其他 Phase                   → 低优先（仅在 L3 按需加载时）
5. 其他领域 + universal                    → 最低（仅关键词精确匹配时）
```

## 自动标记协议

Phase 10 归档时，Agent 必须自动填充以下字段（不等用户输入）：

| 字段 | 自动获取方式 |
|------|------------|
| `project` | 从 `.harness/config.json` 读取 `projectName` |
| `domain` | 从 `repo-map.yaml` 或 `.harness/config.json` 读取 |
| `phase` | 从当前 Phase 编号 |
| `requirement` | 从 `changes/<req>/` 目录名 |
| `author` | 从 `git config user.name` |
| `scope` | Agent 判断：涉及特定表名/字段 → `project-specific`；涉及技术模式 → `universal` |
| `tech_stack` | 从 `.harness/config.json` 的 `techStack` |

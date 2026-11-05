# C7 多教训轴归档（Multi-Axis Case Archival）

> **触发**：Phase 10 自动归档时，对每个候选 case 执行 4 强制问题；如答案显示 ≥ 2 个独立教训轴，**强制拆分为多 case + 双向 related 链**。
> **来源教训**：本次 V2 路径前缀迁移收尾时本应沉淀 2 个独立 case（CASE-094 双源陷阱、CASE-095 FQDN 统一鉴权），如果按"单 case 写到底"会埋葬其中一个；事实上手工拆出来的 2 个 case 正是分别承载"配置层多源"和"统一鉴权入口"两个完全不同的认知。
> **引用记忆**：[[feedback_c7-auto-execute]]、[[feedback_capture-implementation-journey]]

---

## 为什么"单 case 一站式"会埋教训

Bug / 问题往往同时触发多个独立失效面，比如：

| 单一现象 | 表面看一个 case | 实际上 N 个独立教训轴 |
|---|---|---|
| `/inventory-v2/configs/*` 502 | "V2 502 修复" | (a) baseUrl 双源（yml vs K8s env）<br>(b) FQDN 鉴权统一入口<br>(c) URL 拼接 `/openapi → /api` 黑洞 |
| 部署后空白页 | "前端部署问题" | (a) 改 vite base 后未 commit dist<br>(b) CDN 旧 HTML 缓存未清<br>(c) `<base href>` 未跟前缀联动 |

如果合在一个 case 里：
- 标题只能写一个
- 索引检索只命中一次
- 下一次"症状像"但根因不同时无法精准命中
- Gene 提炼只能挑一个轴

**必须拆**：每个独立教训轴对应 1 个 case，case 间用 `related[]` 双向链接。

---

## 4 强制问题（C7 自动归档前必答）

收尾时每个候选 case 必须答这 4 题。**至少有一题答"是"且独立**，就要拆。

```yaml
multi_axis_check:
  Q1_independent_root_causes:
    question: "本 case 中是否存在 ≥ 2 个根因，且根因之间不是因果关系（互为前提 / 互不依赖）？"
    answer: yes | no
    rationale: <text>
    if_yes:
      axes:
        - axis_name: "baseUrl 配置多源"
          root_cause: "yml default 与 K8s deployment env 同时存在，env 优先级高"
          standalone: true   # 是否能独立写成一个完整 case
        - axis_name: "FQDN 统一鉴权"
          root_cause: "service name 直连绕过公网鉴权 + cookie domain 失配"
          standalone: true

  Q2_independent_lessons:
    question: "本 case 教训是否包含 ≥ 2 类独立的'下次怎么避免'？（即建议清单分属不同行动主语 / 不同时机 / 不同代码层）"
    answer: yes | no
    rationale: <text>
    if_yes:
      lesson_categories:
        - "Phase 0 探查必须涵盖 6 层炸点"
        - "PR 描述必须含运行时证据"
        - "改 baseUrl 前 grep 团队既有实践对齐"

  Q3_independent_search_keywords:
    question: "本 case 是否同时被 ≥ 2 组完全不同的关键词检索（trigger keywords 没有公共子串）？"
    answer: yes | no
    if_yes:
      keyword_groups:
        - ["yml default", "K8s env", "deployment", "matrix UI"]
        - ["FQDN", "service name", "cookie domain", "公网鉴权"]

  Q4_different_phases:
    question: "本 case 涉及的踩坑分布在 ≥ 2 个独立 Phase（如 Phase 3 编码 + Phase 7 部署）？"
    answer: yes | no
    if_yes:
      phase_distribution:
        - phase: 3
          axis: "代码 URL 拼接黑洞"
        - phase: 7
          axis: "K8s env 双源覆盖"
```

### 拆分判定

| 回答情况 | 处置 |
|---|---|
| 4 题全 `no` | **单 case**——按原计划写一份 |
| Q1 yes 且 axes ≥ 2 standalone | **强制拆 N 份**（N = standalone axes 数） |
| 仅 Q2/Q3/Q4 之一 yes，但根因相同 | **单 case**，但在 case 文件中分章节标注 lesson categories / phase distribution，**Gene 提炼时按轴分独立 Gene** |
| Q1 yes 但 axes 仅 1 个 standalone（其他从属） | **单 case**，从属轴在文件中作为子节 |

---

## cases-index.yaml schema 扩展（向后兼容）

现有 schema 是 flat-key map（`CASE-XXX:` 顶层 key），新增**可选**字段 `teaching_axes`：

```yaml
CASE-094:
  trigger: baseUrl k8s env override yml default ...
  one_liner: "baseUrl 双源陷阱：代码改 yml 不等于改运行时"
  severity: P1
  type: configuration
  phase: ['7']
  module: [eam-v2, deploy]
  symptom: "..."
  root_cause: "..."
  file: CASE-094-baseurl-strip-reappend-vs-k8s-env-dual-source.md
  use_count: 0
  last_used: '2026-05-26'
  effectiveness: 0
  lifecycle: active
  agent_role: backend

  # 新增字段（可选，向后兼容）
  teaching_axes:
    - axis_id: T-094-A
      name: "Spring Boot 配置多源"
      keywords: [yml, K8s env, deployment, ${ENV:default}]
    - axis_id: T-094-B
      name: "URL 拼接黑洞"
      keywords: [openapi, /api, replaceFirst, RestTemplate]
  related:
    - CASE-095   # 同源拆出去的另一个 case
    - CASE-020   # 历史平台 env 覆盖

CASE-095:
  ...
  related:
    - CASE-094   # 双向链
    - CASE-020
```

### 兼容性

- 缺 `teaching_axes` 字段 → 视为单轴 case（不影响现有 99+ case）
- 缺 `related` 字段 → 视为无关联（不影响现有索引）
- C7 旧版本不识别新字段时仍能运行，只是不享受多轴检索

---

## 拆 case 时的双向 related 强制规则

拆出 N 个 case 后，**每个 case 文件**必须：

1. 顶部"关联"章节列出所有兄弟 case（双向）
2. cases-index.yaml 中 `related: [...]` 互相指向

**禁止**：A 里写了 `related: [B]`，B 里没写 `related: [A]`——会破坏检索召回率。

C7 自动归档时检查：对每对 (A, B) 满足 `B in A.related ⇔ A in B.related`，否则拒绝归档。

---

## 教训轴命名规范

`teaching_axes[].name` 必须满足：
- 4-12 个字 / 词
- 名词短语，不是动词句
- 跨 case 同名轴可复用 `axis_id`，便于按轴聚合检索

**好例子**：
- "Spring Boot 配置多源"
- "FQDN 统一鉴权入口"
- "URL 拼接黑洞"
- "vite base 与 dist 联动"

**坏例子**：
- "改 baseUrl 时要小心"（不是名词）
- "环境配置问题"（太泛）
- "CASE-094 教训"（不是轴名）

---

## C7 自动归档执行流（Phase 10）

```
1. 触发：Phase 9 验证通过 + 用户确认修复
2. 收集：本次会话产生的根因 / 教训 / 关联 case
3. 答 4 题：multi_axis_check
4. 判定：拆 / 不拆
5a. 不拆：写 1 份 case + 1 个 Gene（按教训轴）
5b. 拆：
   - 写 N 份 case，文件名 CASE-XXX-A、CASE-XXX-B（或独立编号）
   - 每份 case 顶部 "关联" 章节列出兄弟
   - 写 N 个 Gene（每轴 1 个）
   - cases-index.yaml: append N 条记录，related[] 互相指向
6. Gene 注入 brain：按 teaching_axes 关键词分别注入
7. 用户可见摘要：
   "本次归档 N 个 case（轴：X / Y / Z）"
```

---

## 反模式

| 反模式 | 信号 | 正确做法 |
|---|---|---|
| 单 case 塞所有教训 | "这次踩了 3 个坑，我合并写一份吧" | 答 4 题；Q1 yes → 拆 |
| 拆了但不双向 link | A 文件写了 related: [B]，B 没写 | C7 检查双向，拒绝归档 |
| 教训轴命名为动词 | "改前要小心" | 改名词："Spring Boot 配置多源" |
| 拆出来的 case 互为子集 | A 完整覆盖 B 的教训 | B 应作为 A 的子节，不拆 |
| Gene 只写一个 | 拆了 N 个 case 但 Gene 只 1 条 | 必须 N 条 Gene（每轴 1 条） |

---

## 关联

- [[feedback_c7-auto-execute]] — Phase 10 必须自动执行
- [[feedback_capture-implementation-journey]] — 试错路径要保留
- [phase-0-blast-radius.md](phase-0-blast-radius.md)（G1）— 6 层炸点检查产生的"涉及层"是常见教训轴候选
- [plan-revision-review.md](../skills/planning/plan-revision-review.md)（G3）— revision 中的"假设失效"是独立教训轴
- evolution skill — Gene 注入 brain 入口

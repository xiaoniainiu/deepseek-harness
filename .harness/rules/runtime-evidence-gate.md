# GATE-RUNTIME — 运行时证据门禁（Runtime Evidence Gate）

> **触发**：PR / 推送时，**diff 命中触发模式**（见下方）即强制要求 PR 描述含 `## Runtime Evidence` 块，否则 pre-push hook 拒绝。
> **来源教训**：CASE-094 第 1/2 次修复都"代码改了 + commit 了 + PR 标题写了'已修复'"，但运行时仍 502——根因是只改了 yml default，K8s env 在覆盖；如果当时强制要求"启动日志显示生效值"+"curl 验证"两条证据，第 1 次就发现 yml 改动没生效。
> **引用记忆**：[[feedback_baseurl-dual-source]]、[[feedback_eam-baseurl-fqdn]]、[[feedback_verify-before-claim.md]]
> **关联 G1**：[phase-0-blast-radius.md](phase-0-blast-radius.md) Phase 0 中 `involved + pending` 的层在本门禁中必须升级为含 verified 链接

---

## 为什么"代码 commit + 测试通过"不够

Spring Boot / SPA / K8s 复合栈下，编译期与运行期是两个世界：

| 阶段 | 能验证什么 | 验证不了什么 |
|---|---|---|
| 编译 | 语法正确 | 启动后实际生效值（`${ENV:default}` 哪边赢） |
| 单测 | 函数级行为 | 网关 / CDN / cookie domain 一致性 |
| 集成测试 | 模块边界 | 真实流量下的鉴权 / 限流 / 灰度 |
| **运行时证据** | **生产/预发实际状态** | — |

**核心认知**：CASE-094 / CASE-095 共 5 次部署仍 502，根因都不是"代码错"，而是"运行时未验证"——本门禁就是把"验证"从习惯升级为门禁。

---

## 触发条件（hook 自动判断）

PR diff 命中以下任一模式即激活 GATE-RUNTIME：

```bash
# 1. 后端 baseUrl 拼接 / 远程调用类
git diff --name-only origin/main..HEAD | grep -E '(Controller|Proxy|Client)\.java$'
git diff origin/main..HEAD -- '*.java' | grep -E '(RestTemplate|WebClient|FeignClient|HttpClient|OkHttpClient)'

# 2. yml 配置 baseUrl / host / endpoint 类
git diff --name-only origin/main..HEAD | grep -E 'application(-\w+)?\.ya?ml$'
git diff origin/main..HEAD -- '*.yml' '*.yaml' | grep -iE '(remoteHost|baseUrl|base-url|host:|url:|endpoint)'

# 3. K8s deployment env 类
git diff --name-only origin/main..HEAD | grep -E '(deployment|deploy)\.ya?ml$'
git diff origin/main..HEAD -- '*.yml' '*.yaml' | grep -E '_API_URL|_BASE_URL|_REMOTE_HOST|_ENDPOINT'

# 4. 网关配置类
git diff --name-only origin/main..HEAD | grep -E '(nginx|ingress|gateway).*\.(conf|ya?ml)$'

# 5. 前端 vite base / iframe src 类
git diff origin/main..HEAD -- 'vite.config.*' | grep -E '\bbase:'
git diff origin/main..HEAD -- '*.tsx' '*.vue' | grep -E '\biframe.*src=|\.src\s*='
```

任一命中 → 进入 GATE-RUNTIME。

---

## PR 描述强制 schema

PR 描述必须包含 `## Runtime Evidence` 一级章节，下含 3 个子章节，**全部填齐**：

```markdown
## Runtime Evidence (GATE-RUNTIME)

### 1. 启动日志证据（生效值，不是配置文件值）

<粘贴启动日志关键行，证明 6 层中 involved 层的实际生效值>

示例：
```
2026-05-26 14:23:01 INFO [main] EamApplication - Starting on port 9090
2026-05-26 14:23:02 INFO [main] EamConfig - Effective remoteApiUrl=https://eam-asset-test.f.mioffice.cn (source: env=EAM_REMOTE_API_URL)
2026-05-26 14:23:02 INFO [main] EamConfig - Effective context-path=/inventory-v2
```

### 2. 实际生效配置（不是仓库内 yml）

<对每个 involved 层提供取证命令输出>

示例：
- L3 K8s env：
  ```bash
  $ kubectl get deployment inventory-engine-v2 -n eam-test -o yaml | grep -A1 EAM_REMOTE
  - name: EAM_REMOTE_API_URL
    value: https://eam-asset-test.f.mioffice.cn
  ```
- L4 ingress：
  ```bash
  $ kubectl get ingress eam-ingress -n eam-test -o yaml | grep -A2 inventory-v2
  - path: /inventory-v2/
    backend:
      service:
        name: inventory-engine-v2
  ```

### 3. 真实流量验证（curl / 浏览器抓包）

<对每个修复目标提供命令 + 输出>

示例：
- 修复目标 1：`/inventory-v2/asset/inventory/plan` 返回 V2 SPA
  ```bash
  $ curl -sI 'https://eam-asset-test.f.mioffice.cn/inventory-v2/asset/inventory/plan' | head -3
  HTTP/2 200
  content-type: text/html
  set-cookie: ...
  ```
- 修复目标 2：iframe.src 已切到新前缀（浏览器 Network 抓包截图链接）
  - https://internal-screenshot/abc123.png
```

### 字段约束

| 子章节 | 必填内容 | 缺失则 hook 拒绝 |
|---|---|---|
| 1. 启动日志证据 | ≥ 1 行实际生效值；含日期 + INFO/DEBUG 级别 + key=value | ✗ |
| 2. 实际生效配置 | 对 Phase 0 中每个 `involved` 层有对应取证命令输出 | ✗ |
| 3. 真实流量验证 | ≥ 1 条 curl / wget / kubectl exec 输出 + 状态码 | ✗ |

---

## hook 执行流程（pre-push）

```
1. git push 触发 hooks/pre-push-runtime-evidence.sh
2. 检查 PR 描述（gh pr view --json body 或 .git/PULLREQ_EDITMSG）
3. diff 命中触发模式吗？
   否 → 跳过 GATE-RUNTIME，正常 push
   是 → 进入 4
4. PR 描述含 `## Runtime Evidence` 章节吗？
   否 → 拒绝 push，提示填写 schema
   是 → 进入 5
5. 3 个子章节都有内容吗（不是占位符）？
   否 → 拒绝 push，列出缺失子章节
   是 → 进入 6
6. 子章节内容含命令输出（不是"已验证"一句话）吗？
   否 → 拒绝 push，要求附输出
   是 → 通过，正常 push
```

---

## warn-only 期 → enforce 期

新门禁上线 **第 1 周** 为 warn-only 期：

```bash
# Harness/governance/policies/gate-runtime.yaml
gate_runtime:
  status: warn_only       # 第 1 周
  start_date: 2026-05-26
  enforce_after: 2026-06-02
  warn_only_message: |
    ⚠️ GATE-RUNTIME 即将上线（2026-06-02 起强制）
    本次 PR 已命中触发模式，请补充 Runtime Evidence 章节练手
```

第 2 周起 `status: enforce`，缺失即拒绝 push。

---

## bypass 协议

紧急修复（生产事故 P0）允许 bypass，但代价高：

```bash
# 用户输入
$ git push --runtime-bypass="P0-incident-12345"

# hook 行为
1. 写入 audit log: Harness/governance/audit_log.yaml
2. 自动创建 issue：要求 24h 内补 Runtime Evidence 到 PR
3. 通知 governance owner
4. 计入 risk-score（G7）：bypass 频次 > 阈值 → 团队总评下调
```

非 P0 不允许 bypass。普通"我赶时间"不通过。

---

## 反模式

| 反模式 | 真实信号 | hook 反应 |
|---|---|---|
| PR 描述写"已验证，没问题" | 无命令输出 | 拒绝 |
| 把单测覆盖率截图当 Runtime Evidence | 单测 ≠ 运行时 | 拒绝 |
| 三个子章节都填"待补充" | 占位符 | 拒绝 |
| L3 K8s env 不取证只凭"我看了 matrix UI" | 无 kubectl 命令 | 拒绝 |
| curl 输出截断只剩状态码 | 看不出业务正确性 | 拒绝（要求至少 3 行） |
| 同一证据复用到多个 PR | 时间戳不在本次部署窗口 | 拒绝（要求新鲜证据） |

---

## 与其他 Gate 的关系

| 关系 | 说明 |
|---|---|
| **G1 → GATE-RUNTIME**：Phase 0 中 `involved + pending` 的层 → PR 时必须升级为有 verified 链接 |
| **GATE-RUNTIME → Phase 9**：本门禁是 Phase 9 验证的形式化前置 |
| **GATE-RUNTIME → G6 归档**：每次 GATE 触发 + 通过 → 进入 case 的"运行时证据完整性"教训轴 |
| **GATE-RUNTIME → G7 governance**：bypass 频次 / 缺失率 → 团队 risk-score 输入 |

---

## 关联

- [phase-0-blast-radius.md](phase-0-blast-radius.md)（G1）— 6 层炸点的 verified 字段在此被强制兑现
- [c7-multi-axis-archival.md](c7-multi-axis-archival.md)（G6）— 归档时本门禁触发记录进入 case
- `tools/governance/runtime-evidence-check.sh` — 实际检查器
- `hooks/pre-push-runtime-evidence.sh` — git pre-push 钩子

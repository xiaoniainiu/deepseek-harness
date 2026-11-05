# Phase 0 — 炸点 6 层探查（Blast Radius Check）

> **触发**：任何涉及 baseUrl / host / endpoint / 鉴权 / 路由前缀 / SPA 静态托管 的需求，进入 Phase 0 时强制执行本检查。
> **来源教训**：CASE-094（baseUrl 双源陷阱：yml default + K8s env 双源）、CASE-095（service name 绕过公网鉴权）— 修复 3 次仍 502，根因是只查了"代码层"，未查另外 5 层。
> **引用记忆**：[[feedback_baseurl-dual-source]]、[[feedback_eam-baseurl-fqdn]]

---

## 为什么是 6 层

Spring Boot / SPA / K8s ingress 复合栈下，**任何 baseUrl 类配置都同时存在 6 个独立的物理生效层**。任修一层都不能保证运行时正确，必须 6 层都对齐。

| 层 | 物理位置 | 优先级 | 失配后症状 |
|---|---|---|---|
| 1. 代码 | `*Controller.java` `RestTemplate.exchange(...)` 拼接逻辑 | — | URL 改写错误（如 `/openapi` → `/api`） |
| 2. yml default | `application-{env}.yml` `${ENV:default}` 中的 default | 低（被 env 覆盖） | 改了仅在无 env 时生效 |
| 3. K8s deployment env | matrix UI / deployment.yaml `env:` | **高（覆盖 yml）** | pod 启动注入旧值，yml 完全失效 |
| 4. 网关 conf | k8s ingress / nginx `location` + `sub_filter` | — | path-prefix 路由错误、HTML 注入失败 |
| 5. CDN-DNS | DNS 解析、CDN 缓存的旧 HTML | — | 浏览器拿到旧 dist；强刷+清缓存才生效 |
| 6. cookie-token domain | `Set-Cookie: domain=.test.mioffice.cn` + token check | — | service name/IP 直连绕过该 domain → cookie 不带 → 鉴权失配 |

**核心认知**：
- `${ENV_VAR:default}` 模式下，**改 yml 不等于改运行时**——env 单方面决定（CASE-094）
- 公网 FQDN 是唯一能命中所有鉴权 / 限流 / 灰度 / 监控的入口；service name 是"假快"（CASE-095）
- 6 层中只要任一层未对齐，部署完成 ≠ 修复完成

---

## 强制 schema（Phase 0 产出物）

Phase 0 报告必须包含 `blast_radius_check` 块，**6 层全部填齐**。每层状态必须是以下 3 种之一：

```yaml
blast_radius_check:
  layer_1_code:
    status: involved | not_involved
    files: [path:line, ...]            # involved 时必填
    verified: <evidence_link>          # involved+verified 时必填（grep 输出 / 启动日志 / 测试用例）
    rationale: <text>                  # not_involved 时必填，说明为什么本次不涉及

  layer_2_yml_default:
    status: ...
    files: [src/main/resources/application-*.yml:line, ...]
    verified: ...
    rationale: ...

  layer_3_k8s_deployment_env:
    status: ...
    matrix_ui_url: <link>              # involved 时必填，截图或 URL
    env_keys: [EAM_REMOTE_API_URL, ...]
    verified: ...
    rationale: ...

  layer_4_gateway_conf:
    status: ...
    files: [deploy/nginx-gateway.conf:line, ingress yaml path, ...]
    verified: ...
    rationale: ...

  layer_5_cdn_dns:
    status: ...
    cdn_paths: [...]                   # involved 时必填
    cache_ttl: <text>
    verified: ...
    rationale: ...

  layer_6_cookie_token_domain:
    status: ...
    cookie_domain: <text>              # 如 .test.mioffice.cn
    token_check_endpoint: <fqdn>
    verified: ...
    rationale: ...
```

### 状态契约

每层必须落到下表 3 种状态之一，**不允许第 4 种"待补充 / TODO / 之后再说"**：

| 状态 | 含义 | 必填字段 |
|---|---|---|
| `involved + verified` | 本次涉及该层，已查证生效值 | `files` / `matrix_ui_url` / `cdn_paths` / `cookie_domain`（视层而定）+ `verified` |
| `involved + pending` | 本次涉及该层，待 Phase 3 编码或 Phase 7 部署后补 verified；Phase 9 验证前必须升为 `involved + verified` | `files` 等 + `verified: pending` |
| `not_involved` | 本次不涉及该层 | `rationale`（一句话说清为什么） |

**禁止**：`status: unknown` / 任何字段填 `[待补充]` / 跳过某一层不写。

---

## 执行步骤（Phase 0 内）

1. 读 PRD / 技术方案，识别本次需求是否涉及 baseUrl / host / endpoint / 鉴权 / 路由前缀 / SPA 静态托管 → **任一命中即必须执行本检查**
2. 对每一层执行**最小取证动作**：
   - L1：`grep -rn "remoteApiBaseUrl\|targetUrl\|RestTemplate" src/main/java`
   - L2：`grep -rn "remote-api\|baseUrl\|host:" src/main/resources/application*.yml`
   - L3：matrix UI 截屏或 `kubectl get deployment <svc> -o yaml | grep -A2 EAM_REMOTE`
   - L4：`grep -n "location\|sub_filter\|proxy_pass" deploy/nginx-gateway.conf`
   - L5：`dig <fqdn>` + 询问运维 CDN 缓存策略；如无 CDN 标 `not_involved + rationale: 本系统未接入 CDN`
   - L6：`curl -I <fqdn>` 看 `Set-Cookie: domain=` + 读团队既有 `auth.token.remoteHost` 配置确认
3. 把每层结果填入 yaml schema
4. 出现 `involved + pending` 时，挂入 Phase 7 / 9 验证 checklist
5. **进入 Phase 1 前**：6 层非 `involved + pending` 即 `verified` 即 `not_involved + rationale`

---

## 反模式（GATE 拒绝）

| 反模式 | 真实案例 | GATE 反应 |
|---|---|---|
| 只查 L1（代码）就声称"已分析" | CASE-094 第 1 次修复 | 拒绝，要求补 L2-L6 |
| L3 写 `not_involved` 但无 rationale | "应该不涉及 K8s env 吧" | 拒绝，必须 grep K8s deployment 确认 |
| 任一层 `[待补充]` / `TODO` | "这层我后面再查" | 拒绝，Phase 0 不允许遗留 |
| L6 cookie domain 未查就直接动 baseUrl | CASE-095 service name 绕过鉴权 | 拒绝，要求 `curl -I` 取证 |
| 只填层名不填 verified 链接 | 自认"查过了" | 拒绝，必须有 grep 输出 / 截图 / 命令链接 |

---

## 与其他 Phase 的接口

- **→ Phase 1（任务拆分）**：`involved` 层数 ≥ 3 → 自动触发 [G4 subagent-trigger.md](subagent-trigger.md) 检查
- **→ Phase 3（编码）**：每个 `involved` 层在编码阶段必须有对应 commit 或部署动作
- **→ Phase 7（推送）**：`involved + pending` 项进入 [G2 runtime-evidence-gate.md](runtime-evidence-gate.md)，PR 描述必须包含 `## Runtime Evidence`
- **→ Phase 9（验证）**：所有 `involved + pending` 必须升级为 `verified` 才能宣称完成
- **→ Phase 10（归档）**：本检查结论进入 case，作为 [G6 c7-multi-axis-archival.md](c7-multi-axis-archival.md) 的"炸点层覆盖"教训轴候选

---

## 模板（直接复制到 Phase 0 报告）

```yaml
blast_radius_check:
  layer_1_code:
    status: not_involved
    rationale: 本次仅改前端文案，不动代理/网关代码

  layer_2_yml_default:
    status: not_involved
    rationale: 同上

  layer_3_k8s_deployment_env:
    status: not_involved
    rationale: 同上

  layer_4_gateway_conf:
    status: not_involved
    rationale: 同上

  layer_5_cdn_dns:
    status: involved
    cdn_paths: [/static/inventory-v2/]
    cache_ttl: "5min"
    verified: pending  # Phase 7 部署后强刷验证

  layer_6_cookie_token_domain:
    status: not_involved
    rationale: 同上
```

---

## 关联

- [[feedback_baseurl-dual-source]] — 双源陷阱（K8s env 优先级高于 yml default）
- [[feedback_eam-baseurl-fqdn]] — FQDN 统一鉴权政策
- [CASE-094](../wiki/cases/CASE-094-baseurl-strip-reappend-vs-k8s-env-dual-source.md)
- [CASE-095](../wiki/cases/CASE-095-eam-baseurl-fqdn-unified-auth.md)
- [CASE-020](../wiki/cases/CASE-020-platform-env-var-override.md)

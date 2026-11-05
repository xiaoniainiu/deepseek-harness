# 编码规范：硬约束

> 一切不可被机器验证的约束 = 无效约束。每条规则对应一个历史失败案例。
> 本文件同时落实 MIT 信息部研发基线 12 项要求中的编码相关条目。

## 研发基线红线（不可违反）

```markdown
## 红线：代码变更必须关联需求号
// WHY: 基线1 — 无线上化管控的变更禁止上线；审计时无法追溯变更动机
// 验证: git log --oneline | grep -v "MEEGO\|#\|fix\|feat" → 应为空（每条 commit 含需求标识）

## 红线：禁止跳过 CI 流程
// WHY: 基线6 — 关闭或跳过自动 CI = 红线违规；曾因跳过扫描导致安全漏洞上线
// 验证: git log | grep -i "no-verify\|skip-ci\|skip.test" → 必须为空

## 红线：MR 必须有他人 Approve
// WHY: 基线5 — 不 review 直接合并 = 红线违规；自审自合曾导致线上事故
// 验证: MR approvals_count >= 1

## 红线：上线前必须有测试报告
// WHY: 基线12 — 上线前无有效测试报告 = 红线；曾因无测试报告无法定位回归问题
// 验证: CI pipeline 产出 test-report artifact
```

---

## AI 反幻觉七层验证体系（Rally）

> 反幻觉是 AI 编程最核心的信任问题。目标：从"不可信"变为"可审计"。

### 七层门禁

| 层 | 检查 | 通过标准 |
|---|------|---------|
| 1 | TDD 循环 | 红灯→绿灯→重构，每步有证据 |
| 2 | Spec Review | 验收标准逐条标记 ✅/❌/⚠️ |
| 3 | verificationCommands | 任务专属验证命令全过 |
| 4 | qualityCommands | tsc + eslint + 全量测试通过 |
| 5 | 反幻觉规则 | 无模糊声明，命令输出作证据 |
| 6 | 提交+问题追踪 | commit + issues.json 更新 |
| 7 | 外部 Reviewer | 独立 reviewer 子代理审查 |

### 禁止的模糊声明

```
禁止：should pass / probably works / likely correct / 看起来没问题 / 理论上应该可以 / 测试应该能通过
必须：展示完整命令输出 + 具体成功/失败信息
不可：因为"已经知道结果"而跳过运行
```

### 置信度三级标注（VAF）

| 级别 | 标注 | 适用 |
|------|------|------|
| **High** | 确信 | 来源于已验证的代码/文档 |
| **Medium** | 有依据但需确认 | 来源于相关但非直接匹配的上下文 |
| **Low** | 推测性 | AI 推断，需人工验证 |

**规则**：
- 关键信息（接口名、方法名、类名、表名）必须标注置信度
- 必须标注 `[来源: 文件名:行号]`，无来源的接口清单视为不合格
- 知识库中未找到的信息必须标注 `[待确认]`，禁止编造

### SDD 子代理红线

| 红线 | 说明 |
|------|------|
| 不可自己实现 | 控制器只编排验证，不写实现代码 |
| 不可跳过 spec review | 每个 task 必须经审查 |
| 不可让子代理读 plan 文件 | 所有上下文通过 prompt 传递 |
| 不可并行派发多个 implementer | 串行执行保持可控 |
| 不可信任子代理验证结果 | 控制器必须自己运行验证命令 |
| 不可使用共享状态 | 子代理间不共享，每次独立 |

---

## 高危元素管控

> 涉及资金、订单、价格、库存、支付等核心业务数据的变更需特别管控。

```markdown
## 规则：高危元素关联代码变更需逐行级 Review
// WHY: 付款单公司代码字段未纳入管理，导致 2.63 亿资金风险
// 验证: MR 标记含"高危"标签 → reviewer 必须逐行 approve

## 规则：高危场景必须有防重机制
// WHY: 返利计提明细 Job 缺少防重，重复生成 11.4 亿
// 验证: 涉及金额计算的 Job/定时任务 → grep "idempotent\|防重\|去重" > 0

## 规则：监控阈值从紧到松，禁止直接设大值
// WHY: 芯片报关价格阈值从 5% 调到 100%，异常 2700 万未触发告警
// 验证: 告警配置变更 MR → 必须有业务确认
```

---

## 可观测性规则（基线 3）

```markdown
## 规则：新接口/新流程必须有关键路径日志
// WHY: 基线3 — 技术方案必须包含可观测性；线上问题排查时无日志 = 盲猜
// 验证: 新增 Service 方法 → grep "log\." 同方法内 > 0

## 规则：异常路径必须有告警触达能力
// WHY: 基线3 — 监控告警是可观测性必备；曾有外部系统挂了 2 小时无人知
// 验证: catch 块中有 log.error/warn 或 metric 上报
```

---

## 通用规则（跨语言）

### 错误处理

```markdown
## 规则：异常必须记录或重新抛出，禁止吞掉
// WHY: Agent 写了空 catch 块"为了编译通过"，线上出问题时无任何日志可查
// 验证: grep -rn "catch.*{" --include="*.java" | xargs grep -L "log\|throw" → 应为空
```

### 配置管理

```markdown
## 规则：环境相关配置从配置文件/环境变量读取，禁止硬编码
// WHY: Agent 错误推测部署参数（端口/数据库地址），导致服务启动失败
// 验证: grep -rn "localhost\|127.0.0.1\|:3306\|:8080" --include="*.java" src/main/ → 应仅在 test 目录
```

### 类型安全

```markdown
## 规则：API 返回值使用强类型 DTO，禁止 Map/Object
// WHY: Agent 用 Map<String,Object> 返回数据，前端无法获得类型提示，字段拼写错误无编译期检查
// 验证: grep -rn "Map<String.*Object>" controller/ | wc -l == 0
```

### 超时降级

```markdown
## 规则：外部调用必须设置超时 + 失败降级策略
// WHY: Agent 调外部系统没设超时，对方挂了后线程池耗尽，服务整体不可用
// 验证: grep -rn "RestTemplate\|HttpClient\|FeignClient" --include="*.java" | xargs grep -c "timeout\|Timeout" > 0
```

---

## 后端规则（{{TECH_STACK}} 适配区）

### 方法重载与调用链

```markdown
## 规则：在重载方法中加逻辑前，必须确认所有重载版本的调用关系
// WHY: updateAssetAccountWithCheck 有3参数和4参数两个版本，它们是独立实现不互相调用。
//      校验加在4参数版本，但 Controller 调的是3参数版本，校验完全没触发。
// 验证: grep 方法名 → 列出所有重载 → 确认哪些被外部调用 → 每个被调用的重载都要加

## 规则：修改汇聚点方法时，从 Controller 实际发起一次完整调用链追踪
// WHY: 只看方法名"看起来对了"但实际调用的是另一个重载。编译不报错、运行不报错、就是不生效。
// 验证: 从 Controller 的 @PostMapping 开始 → 跟到 provider.method() → 确认参数个数 → 定位到具体实现
```

### 事务

```markdown
## 规则：@Transactional 只加在 Service 写方法上，不加在 Controller 或读方法上
// WHY: Agent 在 Controller 加了 @Transactional，事务范围过大，包含了 HTTP 响应写入时间
// 验证: grep -rn "@Transactional" controller/ | wc -l == 0
```

### 并发

```markdown
## 规则：并发写入场景必须用分布式锁或乐观锁
// WHY: 两个用户同时提交盘点报告，不加锁导致数据覆盖
// 验证: 涉及 UPDATE 的 Service 方法 → 检查是否有 lock 注解或 version 字段
```

### 数据访问

```markdown
## 规则：查询条件使用类型安全的构建器，不拼字符串
// WHY: Agent 拼 SQL 字符串时字段名写错（驼峰/下划线混淆），编译通过但运行时报错
// 验证: grep -rn "\"select\|\"update\|\"delete" service/ | wc -l == 0
```

### 字段映射

```markdown
## 规则：新增实体字段前必须确认数据库表已有该列
// WHY: Agent 在 Entity 加了字段但忘记写 DDL，编译通过但运行时字段不存在
// 验证: Entity 新增字段 → wiki/data-model.md 有对应列 OR 有配套 DDL 脚本
```

---

## 前端规则（如适用）

### 组件

```markdown
## 规则：组件 Props 必须用 TypeScript interface 定义
// WHY: Agent 用 any 类型传 props，下游组件 refactor 后无编译期错误但运行时崩溃
// 验证: grep -rn "props: any\|: any" --include="*.tsx" | wc -l == 0
```

### 状态管理

```markdown
## 规则：API 调用状态使用 hook 封装，不在组件内直接 fetch
// WHY: 多个组件重复写 fetch + loading + error 逻辑，修一处漏一处
// 验证: grep -rn "fetch(\|axios\." views/ | wc -l == 0（应在 api/ 或 hooks/ 中）
```

---

## AI Coding 防御性编码规则

> 来源：AI 生成代码的系统性弱点。"防御性编程不是技巧，是伤疤。"

```markdown
## 规则：所有对外接口入口做参数 sanitize
// WHY: AI 生成的 24 个参数全无 null 检查，生产环境 null 崩溃。AI 只走 happy path，异常输入毫无防备
// 验证: 接口入口方法有 @Valid / Objects.requireNonNull / 三元守卫

## 规则：外部服务调用必须有错误降级（不可直接崩溃）
// WHY: LLM 工具调用返回 400 时 bot 直接崩溃。空的模型列表被缓存后永久失效
// 验证: 外部调用 catch 块有降级逻辑（fallback/默认值/重试），不是 throw 或空

## 规则：缓存失败状态必须加过期机制
// WHY: LLM 调用一次失败后空列表被永久缓存，后续所有请求跳过该模型无法自愈
// 验证: 缓存写入含 TTL 或定时清理逻辑

## 规则：URL/路径拼接前做规范化
// WHY: compat_openai base_url 已含 /v1，代码又追加 /v1，双重路径导致 400
// 验证: 拼接路径前 strip trailing slash 或检查前缀

## 规则：前后端交界处必须有命名转换层
// WHY: camelCase/snake_case 混用导致 scope 校验 key 不匹配
// 验证: API 入口有统一的命名转换（@JsonNaming / request interceptor）

## 规则：AI 写完代码后专门做"调用方不守规矩"的 review
// WHY: AI 代码在正常输入下完美，异常输入（null/超长/错格式/重复调用）毫无防备
// 验证: review checklist 含"异常输入"维度
```

---

## 必须做到 + 禁止做的（速查）

### 必须做到

1. 异常必须记录或抛出
2. 外部调用必须有超时 + 降级
3. 配置从文件/环境变量读取
4. API 返回强类型
5. 写操作有事务
6. 并发场景有锁
7. 新字段确认表结构
8. 组件 Props 有类型
9. 接口入口参数 sanitize（null/类型/范围）
10. 缓存失败状态有 TTL

### 禁止做的

1. 空 catch 块
2. 硬编码环境参数/密钥
3. Map/Object 作为 API 返回
4. Controller 写业务逻辑
5. Data Access 层写业务判断
6. 无超时的外部调用
7. 字符串拼接 SQL
8. any 类型的 Props
9. 整文件替换（必须增量编辑）
10. 不检查就声称"应该能过"

---

## L1 代码质量六维检查（通用，所有语言适用）

> 来源：`tools/code-quality/prompts/review-prompt.md`
> 适用时机：Phase 3 实时 lint → Phase 4 AI Review → Phase 7 PreCommit → Phase 8 CI
> 只报真问题，不报格式/空白/import 顺序。

### 维度 1: 安全（Security）

| 检查项 | 严重性 | 适用语言 |
|--------|--------|---------|
| SQL 注入（字符串拼接 SQL） | CRITICAL | Java/PHP/Python |
| XSS / CSRF | CRITICAL | Frontend |
| 硬编码密钥/凭据/API Key | CRITICAL | ALL |
| 不安全的反序列化 | HIGH | Java/PHP |
| 路径穿越 / 命令注入 | CRITICAL | ALL |
| 系统边界缺少输入校验 | HIGH | ALL |
| SSRF（服务端请求伪造） | HIGH | Java/Python |

### 维度 2: 性能（Performance）

| 检查项 | 严重性 | 适用语言 |
|--------|--------|---------|
| N+1 查询 | HIGH | Java/Python |
| 异步上下文中的阻塞 IO | HIGH | ALL |
| 无界集合 / 内存泄漏 | HIGH | ALL |
| 列表 API 缺分页 | MEDIUM | ALL |
| 循环内的昂贵操作 | MEDIUM | ALL |

### 维度 3: 逻辑与正确性（Logic & Correctness）

| 检查项 | 严重性 | 适用语言 |
|--------|--------|---------|
| Null/undefined 处理 | HIGH | ALL |
| 竞态条件 / 并发 bug | CRITICAL | Java |
| 不完整的状态机转换 | HIGH | ALL |
| 集成点缺少错误处理 | HIGH | ALL |
| Off-by-one 错误 | MEDIUM | ALL |

### 维度 4: 可维护性（Maintainability）

| 检查项 | 严重性 | 适用语言 |
|--------|--------|---------|
| 上帝方法（> 50 行） | MEDIUM | ALL |
| 死代码 / 不可达分支 | LOW | ALL |
| 误导性命名 | MEDIUM | ALL |
| 职责混合 | MEDIUM | ALL |

### 维度 5: SonarLint 关键规则

**Bug 检测**：
- 资源未在 finally 关闭 (java:S2095)
- 空指针解引用 (java:S2259)
- 类型不兼容的 equals (java:S2159)
- 运算符两侧相同表达式 (java:S1764)
- 忽略无副作用函数的返回值 (java:S2201)

**漏洞检测**：
- 弱加密密钥 (java:S4426)
- 弱 SSL/TLS 协议 (java:S4423)
- HTTP 响应头注入 (java:S5167)
- ReDoS 正则风险 (java:S5852)

**代码异味**：
- 认知复杂度过高 (> 15) (java:S3776)
- 日志中使用字符串拼接 (java:S2629)
- 使用原始类型 (java:S3740)
- Optional 不应作为参数 (java:S3553)

### 维度 6: 设计反模式检测

**通用反模式（所有语言）**：

| 反模式 | 信号 | 建议 |
|--------|------|------|
| God Class | > 500 行 或 > 20 方法 | 拆分 |
| God Method | > 50 行 或认知复杂度 > 15 | 提取子方法 |
| Feature Envy | 方法大量使用其他类的数据 | 移动到数据所在类 |
| Data Clump | 同组字段/参数在多处出现 | 提取为值对象 |
| Primitive Obsession | 用 string/int 表示领域概念 | 用值对象 |
| Long Parameter List | 方法 > 4 参数 | 参数对象或 Builder |

**Java 特有**：
- Service Locator（静态获取依赖而非 DI）
- Anemic Domain Model（Entity 只有 getter/setter）
- Catch-and-Ignore（空 catch 或仅 log.warn）
- Mutable Shared State（共享可变对象无同步）
- 违反迪米特法则（`a.getB().getC().doX()`）

**前端特有（React/Vue/TS）**：
- Prop Drilling（跨多层传 props → 用 context/store）
- Giant Component（> 300 行 → 拆分）
- 组件内写业务逻辑（→ hooks/composables/services）
- JSX 中内联回调（每次渲染创建新函数）
- 列表缺 key
- React/Vue 中直接操作 DOM

**Python 特有**：
- Bare Except（`except:` 不指定类型）
- 可变默认参数（`def f(x=[]):`）
- 全局可变状态
- 业务逻辑用 isinstance 而非多态

**PHP 特有**：
- Static Everything（过度使用静态方法）
- Array-Driven Development（无类型数组代替 DTO）
- Fat Controller（Controller 写业务逻辑）
- 缺少类型声明

---

## 三道防线集成（L1 × Harness Phase）

```
Phase 3 编码中
  │ 每次 Edit/Write → PostToolUse Hook → 增量 lint（语言自动检测）
  │ Agent 收到 lint 结果 → 自动修复 → 继续编码
  │
Phase 4 编码评审
  │ Claude 自审（六层追踪 + 上述六维检查）
  │ + /codex review（独立第二视角）
  │ 合并两方意见 → PASS / 返回修复
  │
Phase 7 推送
  │ PreCommit Hook → AI 审查 staged diff（上述六维）
  │ + Git pre-commit → 静态 lint + 密钥检测 + 格式
  │ 不通过 → 阻止提交
  │
Phase 8 CI
  │ SonarQube 质量门禁 + 集成测试 + 安全扫描
  │ 最终权威 → 上述三层全是前置过滤
```

### 语言自动检测

PostToolUse 和 PreCommit 根据文件扩展名自动选择 linter：

| 扩展名 | 语言 | Lint 工具 | 配置文件 |
|--------|------|----------|---------|
| `.java` | Java | Checkstyle + PMD | `tools/code-quality/config/java/` |
| `.ts` `.tsx` `.js` `.jsx` `.vue` | Frontend | ESLint + Prettier | `tools/code-quality/config/frontend/` |
| `.py` | Python | Ruff | `tools/code-quality/config/python/ruff.toml` |
| `.php` | PHP | PHP-CS-Fixer + PHPStan | `tools/code-quality/config/php/` |

### 审查输出格式

```
[CRITICAL|HIGH|MEDIUM|LOW] file:line - [category] description
```

Category 标签：`security`、`performance`、`logic`、`maintainability`、`sonarlint`、`anti-pattern`

### 规则

- **不报**：格式/空白/import 顺序/注释建议/测试建议
- **只报**：会导致 bug、宕机、安全事故的问题
- 反模式只报明确有害的，不报边界情况

---

## 填充指引

1. 替换 `{{TECH_STACK}}` 为实际技术栈
2. 按项目实际补充 `// WHY` 踩坑案例
3. 调整 `// 验证` 命令为项目实际可执行的检查
4. 删除不适用的语言/框架规则
5. 记住：**没有踩坑故事的规则不写入**

---

## 密钥扫描门禁与 bypass 协议（自动注入）

本仓在三层注入了密钥扫描门禁，规则共用 `Harness/config/secret-patterns.yaml`：

| 层 | 触发 | 命令 | 退出码语义 |
|----|------|------|-----------|
| 写时 | PostToolUse on Edit\|Write | `git-secret-scan.sh --file <path>` | 2=阻断，0=放行 |
| 推时 | PreToolUse on Bash `git commit/push` | `git-secret-scan.sh --staged` | 2=阻断，0=放行 |
| CI | 流水线 test stage | `git-secret-scan.sh --full --strict` | 2=阻断（warning 也升级为 blocker） |

模式库内 `(?i)` 内联标志由扫描脚本预处理（剥前缀+追加 `-i`），跨 grep 引擎稳定（CASE-096）。

### Bypass 协议（红线）

- **本地 false positive**：在命中行末加 `// secret-scan:allow`（或语言对应注释如 `# secret-scan:allow`），单行豁免。
- **整文件豁免**：把文件 glob 加入 `secret-patterns.yaml` 的 `allowlist.files`。已默认覆盖测试/示例文件（`**/*.test.*`、`**/.env.example` 等）。
- **`--no-verify` 跳过 git hook**：**禁止使用，除非用户在当前对话中显式 ack**。即便 ack，commit message 必须说明 bypass 原因；CI 层 `--strict` 仍会拦截。
- **删除/修改模式库**：必须经代码评审，不得单方面降低门槛；新模式应配测试 fixture。

### 触发约束

- PreToolUse Bash 仅匹配 `Bash(git commit*)` / `Bash(*git push*)`，不污染常规 Bash 调用。
- PostToolUse Edit\|Write 由脚本内置扩展名白名单过滤（`.java/.py/.ts/.tsx/.js/.go/.rs/.yml/.json/.sh` 等），其余文件直接跳过。
- CI 默认在 `merge_request_event` 与默认分支推送时触发，`--strict` 把 warning 模式升级为阻断。

### CI snippet 模板

- GitLab：`Harness/templates/ci/secret-scan-gitlab.yml`
- GitHub Actions：`Harness/templates/ci/secret-scan-github.yml`

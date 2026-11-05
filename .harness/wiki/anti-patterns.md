# 反模式检查清单

> Phase 转换时由 knowledge-inject 自动加载对应分组。AI 只需看当前 phase 的表格（~10-20 行），
> 命中时才 `Read CASE-{id}.md` 获取完整故事和修复方案。
>
> **维护规则**：Phase 10 归档新 case 时，同步在此文件对应 phase 分组追加 AP 条目。

## Phase 1-2（需求分析 / 评审）

| AP | 检查指令 | 触发信号 | CASE |
|----|---------|---------|------|
| AP-003 | 否定词需求必须确认修改方向 | 不展示/不显示/移除/隐藏 | 003 |
| AP-005v | 同名字段在不同实体中含义可能不同 | 配置/Rule/同名字段 | 005 |
| AP-042 | 高危字段是否有枚举约束或审批流 | 金额/公司代码/高危字段 | 042 |
| AP-047 | 5+ 文件变更时先拆原子任务再编码 | 复杂需求/双端变更/5+ 文件 | 047 |
| AP-048 | 功能需求先问澄清问题再给方案 | 新功能/功能设计/需求不明确 | 048 |
| AP-049 | 检查依赖的生命周期事件是否真实存在 | session:end/lifecycle/hook | 049 |
| AP-055 | 汇聚点必须 grep 验证所有调用方是否经过 | Provider/Ability/写入路径 | 055 |
| AP-058v | 同入口不同业务路径逐一确认数据来源 | 预约/续借/归还/多流程 | 058 |
| AP-069 | "提交时校验"需追问续借/修改/重新提交等变体是否也需要 | 校验/拦截/硬拦截/提交时 | 069 |
| AP-071 | 新增配置先查已有配置模式（yml profile vs Nacos vs 代码默认值），不自己发明 | templateBizId/botBizId/外部URL/环境配置/消息模板 | 071 |
| AP-072 | SQL/数据库查询必须在 Mapper 层，app/domain 层不写 SQL 字符串 | SELECT/INSERT/UPDATE/DELETE/SQL拼接/字符串查询 | 072 |
| AP-062 | DB 配置表驱动的逻辑？列出完整数据依赖链 | 业务线/字典/枚举/配置表 | 062 |
| AP-064v | Phase 0 必须追踪 DB schema 确认数据在哪张表 | DB schema/三张表/fallback | 064 |

### 硬编码 SOP（CLAUDE.md 每次会话强制加载）

| SOP | 检查指令 | 触发信号 |
|-----|---------|---------|
| SOP-H1 | 说"已完成"前必须有验证命令输出 | 已完成/已修复/PASS/done |
| SOP-H2 | 修改前先搜已有实现模式 | 新增配置/新增方法/新增文件 |
| SOP-H3 | 失败 2 次后按切换表换思路 | 第二次失败/同一方向/微调无效 |

## Phase 3（编码实现）

| AP | 检查指令 | 触发信号 | CASE |
|----|---------|---------|------|
| AP-001v | 非构建文件是否有构建后自动恢复机制 | emptyOutDir/closeBundle/post-build | 001 |
| AP-002 | 远端不可达时 fallback 是否返回空而非异常 | fallback/emptyList/gateway | 002 |
| AP-004 | 常量/白名单修改后检查所有引用点已同步 | ALLOWED_/whitelist/枚举/常量 | 004 |
| AP-005 | 配置读取的实体来源是否正确（计划级 vs 全局） | Rule/config/endCheckType | 005 |
| AP-006 | 大表统计用 DB 聚合而非全量加载 | findAll/selectList/COUNT/OOM | 006 |
| AP-007 | 内部服务调用是否绕过 CAS 网关 | proxy/CAS/302/k8s 内部调用 | 007 |
| AP-009 | useMemo/useCallback 依赖数组是否完整 | useMemo/useCallback/stale closure | 009 |
| AP-010 | 异步 Select 是否用受控 value 回显 | Select/setFieldsValue/异步选项 | 010 |
| AP-011 | 空值日期是否守卫为 null 而非 new Date | DatePicker/Invalid Date/new Date('') | 011 |
| AP-024 | fetch 是否设置 credentials:include | fetch/credentials/401/withCredentials | 024 |
| AP-026 | 请求是否设置 X-Requested-With 头 | X-Requested-With/米盾/HTML 响应 | 026 |
| AP-027 | UserDetail 字段是否处理 cas: 前缀 | X-Proxy-UserDetail/cas:user/Base64 | 027 |
| AP-029 | iframe src 是否包含 base path 前缀 | iframe/base path/404/vite-embed | 029 |
| AP-030 | CORS allowedOrigins 端口是否覆盖全 | CORS/allowedOrigins/403/WebConfig | 030 |
| AP-032 | iframe 参数是否通过 URL 传递非 localStorage | bizLine/localStorage/iframe/URL param | 032 |
| AP-033 | 实体字段是否有对应 DB 列和 migration | TableField/exist=false/Unknown column | 033 |
| AP-035 | 批量写入是否用 BATCH 而非循环 INSERT | batchInsert/rewriteBatched/逐行 INSERT | 035 |
| AP-036 | 不同业务异步操作是否用独立线程池 | AsyncConfig/@Async/thread pool | 036 |
| AP-037 | 多值字段调用外部接口前是否拆分 | getDeptCode/分号/多值/IN clause | 037 |
| AP-037v | 任务生成结果为 0 时是否有 WARN 日志 | zero tasks/0 条任务/生成结果为空 | 037 |
| AP-038 | 服务间调用走内部 URL 非外部域名 | k8s/svc/CAS 网关/服务间调用 | 038 |
| AP-039 | coordinator 是否移除执行工具权限 | disallowedTools/coordinator/多 agent | 039 |
| AP-040 | prompt 静态层与动态层是否分离 | CACHE_BOUNDARY/prefix cache | 040 |
| AP-041 | 自动重试是否有 max-retry 熔断机制 | circuit_breaker/max failures/熔断 | 041 |
| AP-043 | 批量 Job 是否有幂等键防重复执行 | Job/幂等/idempotency key/重复生成 | 043 |
| AP-043v | 金额操作是否有阈值校验和告警 | 金额阈值/告警/异常金额 | 043 |
| AP-045 | 树形查询是否一次加载禁止 N+1 递归 | N+1/部门树/recursive/CTE | 045 |
| AP-046 | 重构是否改变了逻辑语义等价性 | code smell/重构/优化/条件分支 | 046 |
| AP-053 | buildRes 复制字段时逐行核对 DTO 是否遗漏 | buildRes/copyProperties/Tab 为空 | 053 |
| AP-054 | 字典驱动字段必须有 rawValue fallback | BUSINESS_TYPE_DESC/字典/枚举翻译 | 054 |
| AP-055v | MapStruct 复制时检查需清空的业务字段 | copyAssetAccount/MapStruct/@Mapping | 055 |
| AP-056 | 查询方法 WHERE 字段与传入参数语义一致 | queryByAssetCodes/queryBySn/查错字段 | 056 |
| AP-058 | DTO 构建实体时确认字段数据来源是否有值 | buildReserveItems/vehicleCode/续借 | 058 |
| AP-059 | 跨模块 DTO 字段类型必须点进类确认 | Integer/String/businessLine/枚举类型 | 059 |
| AP-060 | 同名方法检查所有重载签名的调用方 | overload/重载/3 参数/4 参数 | 060 |
| AP-061 | FunColumn 的 name 必须填翻译名非 DB 列名 | amg_fun_column/fieldName/列标题 | 061 |
| AP-062v | 关键字段后端必须校验不依赖前端表单 | businessLine/非空校验/checkPlanReq | 062 |
| AP-063 | 多值字段返回前端前必须 split 为数组 | fieldValues/split/逗号分隔/暂无数据 | 063 |
| AP-064 | 字典查询确认数据在哪张表再写代码 | amg_dict/amg_fun_dict/getComboValues | 064 |
| AP-065 | 同库数据用 JDBC 直查不绕道远程 HTTP | RestTemplate/SecProxy/302/CAS | 065 |
| AP-066 | 修 bug 前必须抓 Network 请求做完整链路追踪 | 猜测修复/Network 面板/链路追踪 | 066 |
| AP-066v | 第一次修复无效必须停下做全链路追踪 | 连续猜测/连续推送/10 分钟循环 | 066 |
| AP-067a | 前端权限判断用 split+includes 不用子串匹配 | drawUserCode/includes/权限判断 | 067 |
| AP-067b | External 客户端必须用 safeRestTemplate 禁 302 | new RestTemplate/302/External 客户端 | 067 |
| AP-067c | BPM 变量无法解析时回退到创建人非硬编码 | car_03/BPM 变量/硬编码/审批人 | 067 |
| AP-067d | 远端不可用时 fail-closed 不授予高权限 | authority-account/super_admin/fail-open | 067 |
| AP-087 | operator 归一化必须有单一来源（`normalizeOperator`），禁止 if-else 链 | "EQ".equals/equal/queryCond/buildScopeWhereSql | 073 |
| AP-089 | fallback 路径不可无条件降级为空/默认值，至少做 DB 兜底查询 | fallback/userName/工号当姓名/user-info | 073 |
| AP-090 | 枚举 code 变更需兼容旧数据（legacyCode 别名或数据迁移） | BusinessType/fromCode/GROUP/枚举/v1旧数据 | 073 |
| AP-091 | 前端 URL 参数传给后端必须确认类型匹配（String vs Integer） | ?id=PJH.../save/PlanReq.id/Jackson/NumberFormatException | 075 |
| AP-092 | SQL ESCAPE 反斜杠必须考虑 MySQL 转义规则（Java 8个=SQL 2个=MySQL 1个） | ESCAPE/LIKE/backslash/MySQL/queryForList | 076 |
| AP-094 | Java 8 项目禁止 var/switch expression/text block（本地 JDK 17 不报错但 CI 失败） | var/java.version=1.8/编译失败 | — |
| AP-095 | 跨框架事件竞争必须用 capture 阶段 + stopImmediatePropagation | addEventListener('click')/React 共存/embed-inject/白屏 | 078 |
| AP-096 | 反复 bug 必须沿调用链追 3 层：上游传什么/当前假设什么/null 和空都处理了吗 | 反复出现/split/null/inventoryRule/planDimension | 077 |
| AP-097 | 跨系统修复修在"发起方"不修"接收方"——接收方补丁随部署丢失 | embed-inject/removeChild 补丁/旧系统重部署丢失 | 078 |

## Phase 4（代码评审）

| AP | 检查指令 | 触发信号 | CASE |
|----|---------|---------|------|
| AP-003v | 改动后逐条对照 PRD 验证方向正确 | PRD/字段/展示/可见 | 003 |
| AP-004 | 改了一处，全局搜索所有同构 | 白名单/配置/常量 | 004 |
| AP-006 | 大表查询是否意外全量加载 | findAll/selectList/COUNT | 006 |
| AP-035 | 批量写入是否用 BATCH | batchInsert/rewriteBatched | 035 |
| AP-036 | 异步操作是否线程池隔离 | @Async/线程池 | 036 |
| AP-039 | agent 权限是否物理隔离 | coordinator/worker/多 agent | 039 |
| AP-041 | 重试是否有熔断上限 | 重试/circuit_breaker | 041 |
| AP-043 | Job 是否幂等 | Job/幂等/批量 | 043 |
| AP-046 | 重构是否改变语义 | 重构/优化/条件分支 | 046 |
| AP-055 | 汇聚点所有调用方是否经过校验 | Provider/Ability/入口 | 055 |
| AP-056 | 查询方法实际 WHERE 与参数语义一致 | queryBy/WHERE/查错字段 | 056 |
| AP-060 | 同名方法所有重载签名的调用方 | overload/重载 | 060 |
| AP-066v | 连续修复无效时是否停下做链路追踪 | 连续猜测/连续推送 | 066 |
| AP-067b | External 客户端是否用 safeRestTemplate | new RestTemplate/302 | 067 |
| AP-067d | 远端失败时 fail-closed 还是 fail-open | super_admin/fail-open | 067 |
| AP-088 | API 返回类型变更（String→List）必须双端同步 | splitSemicolon/split/viewableFields/类型变更 | 073 |

## Phase 5（测试）

| AP | 检查指令 | 触发信号 | CASE |
|----|---------|---------|------|
| AP-002v | 权限接口是否有本地兜底数据测试 | permissionResources/stub-configs | 002 |
| AP-011 | 空值日期边界是否覆盖 | DatePicker/null/Invalid Date | 011 |
| AP-012 | HiUI 下拉组件是否用 body 全局选择器 | Playwright/Select/portal/hi-v4 | 012 |
| AP-037 | 多值字段拆分后覆盖空值/单值/多值场景 | 分号/多值/IN clause | 037 |
| AP-045 | 树形查询用接近生产数据量压测 | N+1/部门树/性能 | 045 |
| AP-051 | HiUI 状态检查用 CSS class 不用 DOM property | indeterminate/checkbox/HiUI | 051 |
| AP-052 | vendor 组件升级后 grep 旧文案确认无残留 | vendor/组件升级/getByText/文案 | 052 |
| AP-062 | 构造脏数据场景：配置表多余/缺失记录 | amg_business_line/脏数据/自动赋值 | 062 |

## Phase 7（推送 / 部署）

| AP | 检查指令 | 触发信号 | CASE |
|----|---------|---------|------|
| AP-001 | build 后验证非构建文件未被删除 | emptyOutDir/vite build/static/ | 001 |
| AP-007 | 内部服务调用是否绕过 CAS 网关 | proxy/CAS/302/k8s | 007 |
| AP-008 | 是否从目标仓库 HEAD 增量修改 | git show/整文件替换/编译失败 | 008 |
| AP-008v | 推送前 grep 目标仓库独有符号是否存在 | escapeLikeValue/跨仓库推送 | 008 |
| AP-013 | CI 是否包含前端构建步骤 | frontend-maven-plugin/vite build | 013 |
| AP-013v | 推送前确认 static/ 下有最新构建产物 | src/main/resources/static | 013 |
| AP-014 | context-path 与网关转发路径是否一致 | context-path/404/nginx/servlet | 014 |
| AP-015 | 前端构建是否在目标仓库执行 | API_BASE/config.ts/双仓库 | 015 |
| AP-015v | 构建后验证 baked 路径前缀是否正确 | baseURL/API_BASE/dist/assets | 015 |
| AP-016 | 基础镜像 GLIBC 版本是否满足运行时 | GLIBC/CentOS/Node 18/openEuler | 016 |
| AP-017 | 依赖安装是否在启动脚本而非构建层 | start.sh/MODULE_NOT_FOUND | 017 |
| AP-018 | 首次部署健康检查超时是否足够长 | healthCheck/initialDelay/首次部署 | 018 |
| AP-019 | 部署环境是否避免 pnpm 符号链接 | pnpm/symlinks/node_modules | 019 |
| AP-020 | 部署后检查环境变量是否被平台覆盖 | EAM_REMOTE_API_URL/MiONE/env | 020 |
| AP-021 | pip 是否配置内网镜像源 | pip install/PyPI/mirror/超时 | 021 |
| AP-022 | CI 脚本是否 POSIX sh 兼容 | Kaniko/sh/bash/[[/POSIX | 022 |
| AP-023 | SQLite 部署路径是否在 NFS 上 | SQLite/WAL/NFS/database is locked | 023 |
| AP-025 | 新路径是否配置 SecProxy 规则 | SecProxy/route-no-auth/米盾 | 025 |
| AP-030 | CORS allowedOrigins 端口覆盖全 | CORS/allowedOrigins/403 | 030 |
| AP-033 | 实体字段有对应 DB 列和 migration | TableField/Unknown column | 033 |
| AP-034 | 部署前对比实体字段与目标 DB 表结构 | Unknown column/migration/ALTER TABLE | 034 |
| AP-044 | Volume mountPath 与镜像 COPY 目标不重叠 | PVC/Volume/mount/initContainer | 044 |
| AP-050 | 跨服务 API 变更按依赖顺序部署 JAR | mvn deploy/SNAPSHOT/跨服务依赖 | 050 |
| AP-057 | 首推新仓库必须先注册 repo-map | repo-map/deploy-info/tag/新仓库 | 057 |

## Phase 9（E2E 验证）

| AP | 检查指令 | 触发信号 | CASE |
|----|---------|---------|------|
| AP-002 | 远端不可达时 fallback 是否返回空 | fallback/gateway/远端不通 | 002 |
| AP-003v | 改动后逐条对照 PRD 验证方向正确 | PRD/展示/可见 | 003 |
| AP-010 | 异步 Select 回显是否正确 | Select/回显/异步选项 | 010 |
| AP-014 | context-path 与网关路径一致 | 404/context-path | 014 |
| AP-015v | baked 路径前缀是否正确 | API_BASE/SPA 404 | 015 |
| AP-020 | 环境变量是否被平台覆盖 | env/MiONE | 020 |
| AP-028 | E2E 必须手动 CAS 登录不可注入 cookie | CDP/setCookie/CAS | 028 |
| AP-029 | iframe src 包含 base path 前缀 | iframe/404 | 029 |
| AP-031 | 获取 token 用 CDP API 不从面板复制 | Network.getCookies/CDP/unicode | 031 |
| AP-053v | E2E 验证必须浏览器截图不止 API 检查 | API 正确/UI 空白/条件渲染 | 053 |
| AP-054v | stub 模式验证必须检查字典依赖字段渲染 | stub/字典为空/fallback | 054 |
| AP-093 | scope operator 验证必须覆盖全部 14 类，不能只测修改的 | scope/WHERE/operator/like/equal/全量矩阵 | 076 |
| AP-061 | 走用户实际路径验证不只调 API | amg_fun_column/列标题/fieldName | 061 |
| AP-062 | DB 配置变更后验证前端自动赋值逻辑 | amg_business_line/自动赋值/脏数据 | 062 |

## 前端专项（来源：agents/frontend-developer.md + V08 实践）

| AP | 检查指令 | 触发信号 | 来源 |
|----|---------|---------|------|
| AP-FE-01 | linkNavigate 前缀匹配是否过宽（SPA vs 整页跳转） | startsWith/inventory/window.location | V08 |
| AP-FE-02 | 新路由是否注册到 routes/ 且不与已有路由冲突 | path/route/navigate | 通用 |
| AP-FE-03 | 前端校验和后端校验是否双重保障（不只依赖前端） | required/validate/Form.rules | AP-062v |
| AP-FE-04 | 组件 props 是否与后端 DTO 字段名对齐 | fieldCode/fieldName/column | AP-053 |
| AP-FE-05 | 配置驱动的 UI 控制（必填/选填/隐藏）是否正确读取配置 | isRequired/isHide/config | V08 F-05 |

## QA 专项（来源：agents/qa-engineer.md + 反模式驱动测试）

| AP | 检查指令 | 触发信号 | 来源 |
|----|---------|---------|------|
| AP-QA-01 | 测试用例是否覆盖所有 AC（AC 覆盖率 ≥ 95%） | AC/验收标准/测试用例 | 通用 |
| AP-QA-02 | 反模式防护用例是否生成（AP 扫描 → 防护用例） | AP 命中/防护/anti-pattern | V08 Phase 5 |
| AP-QA-03 | 回归场景是否覆盖（非本次改动的核心功能冒烟） | 回归/冒烟/未改动功能 | 通用 |
| AP-QA-04 | 脏数据场景是否覆盖（配置缺失/枚举不匹配/关联不存在） | 脏数据/配置缺失/null | AP-062 |
| AP-QA-05 | E2E 验证是否走用户实际路径（不只 API 调用） | E2E/浏览器/截图/用户路径 | AP-053v/061 |

---

**统计**: 77 case → 136 AP 条目（含跨 phase 变体 + QA 分组），按 phase 分布：

| Phase | AP 数 |
|-------|-------|
| 1-2 需求 | 10 |
| 3 编码 | 59 |
| 4 评审 | 15 |
| 5 测试 | 13 |
| 7 推送 | 24 |
| 9 验证 | 15 |

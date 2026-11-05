# 经验教训归纳

> 从 memory 和实践中提取的踩坑经验。Agent 在评审和编码阶段按需查阅。

---

## 一、AI 反幻觉规则

### 四种幻觉模式

| 模式 | 表现 | 对策 |
|------|------|------|
| 没干的说实现了 | 编译通过就标"已修复" | 必须有证据：curl 正确 data / 截图 / 单测 |
| 没解决的说解决了 | 修了 A 处忘了 B 处 | 修一处，全局搜索所有同构位置 |
| 没理解的说理解了 | "不展示"被反向理解 | 中文歧义先问，不猜 |
| 没看到的说看到了 | 未读旧代码就断言 | 先读旧系统确认正确行为 |

### 强制审查规则

1. 标记完成前必须有证据（不是 code=0，是数据内容正确）
2. 一问一答逐层深入（不猜测）
3. 修一处检查所有同构（全局搜索关键符号）
4. 对照旧系统验证
5. 从用户操作路径端到端验证
6. 中文歧义先问

### 具体案例

| 案例 | 教训 |
|------|------|
| Bug 方向搞反（添加了应该去掉的） | "不展示"有歧义，先确认意图 |
| PlanService 白名单没同步修 | 全局搜索 `ALLOWED_SCOPE_COLUMNS` 所有位置 |
| finishPlan 从错误实体读配置 | 区分"计划级"和"全局"配置来源 |
| 80000 行全量加载内存 | 旧系统已因此超时，必须 DB 聚合 |
| proxy 到旧后端假设能工作 | k8s 内部 CAS 拦截，必须本地实现 |

---

## 二、前端调试经验

### Bug 模式速查

| 模式 | 症状 | 修复 |
|------|------|------|
| useMemo stale closure | 表单保存时字段为空 | 去掉 useMemo 或补全依赖数组 |
| HiUI Select 异步回显失败 | 下拉显示"请选择"而非已选值 | 显式传 `value` prop |
| Invalid Date → 500 | DatePicker 空值传入 new Date() | 三元守卫：非空才构造 |
| Playwright HiUI Select | 选不到选项 | 下拉在 body portal，用 `body .hi-v4-select__option` |
| HiUI Checkbox indeterminate | 测试 `input.indeterminate` 永远 false | 用 CSS class querySelector [CASE-051] |
| vendor 组件文案断言过时 | `getByText` 超时 | 升级后 grep 旧文案，对齐 texts.js [CASE-052] |

### 关键规则

- ESLint `react-hooks/exhaustive-deps` 被禁用 → 不能依赖 linter 发现 closure 问题
- DatePicker range 两次确认（开始→OK→结束→OK）
- Redux dispatch 可作为 fallback 设值手段
- HiUI v4 状态通过 CSS class 表达（`__icon--indeterminate`），不设原生 DOM property [CASE-051]
- vendor 组件升级后必须 `grep -r "旧文案" __tests__/` 验证测试同步 [CASE-052]

---

## 三、代码推送规则

### 铁律：禁止整文件复制到另一仓库

**WHY**：eam-v2 仓库有 pm-context 中不存在的代码，整文件替换导致连续 4 次构建失败。

### 规则

1. 始终从目标仓库 HEAD 做增量修改（sed/patch/手动）
2. 推送前验证关键依赖未被破坏：`grep escapeLikeValue`
3. 用 `git add <specific-files>` 不用 `git add -A`
4. 大量改动在目标仓库本地 clone 中操作
5. Lombok 版本：eam-v2 CI 用 JDK 17，需 1.18.36+
6. **前端构建产物必须本地 build 后提交**（见下方"CI 部署"章节）

---

## 四、Agent 治理六原则

| 原则 | 含义 |
|------|------|
| 结构约束 > prompt 约束 | 物理移除能力，不是"请不要用" |
| Coordinator 只调度不执行 | 物理上不能执行 |
| Worker 上下文隔离 | 每个 worker 只看自包含 prompt |
| 产出合同 | 结构化格式（Scope/Result/Key files/Issues） |
| 多层节奏控制 | token budget + maxTurns + stop hooks |
| Synthesis 不可委托 | 绝不 "based on your findings, fix the bug" |

**判断标准**：如果 agent 的主要产出是消息不是结果物，它不是在工作，它在开会。

---

## 五、E2E 验证方法论

### Chrome DevTools 接入

```bash
# 关所有 Chrome → 独立 profile 启动 → 手动 CAS 登录 → MCP 操作
pkill -f "Google Chrome"; sleep 3
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-e2e-profile \
  --no-first-run "https://target-url" &>/dev/null &
```

### 验证方法论

| 错误做法 | 正确做法 |
|---------|---------|
| code=0 就标 ✅ | 对比数据内容 |
| 只验证 API 可达 | 验证业务正确性 |
| 自己构造请求参数 | 先看 DTO/前端代码确认字段名 |
| 假设字段名一致 | 用 Network 面板看真实 payload |

### 关键坑点

- SecProxy 不能靠注入 cookie 绕过，必须完整 CAS 登录
- Token 有效期 ~25h
- curl 能工作但浏览器不能 = SecProxy 层差异

---

## 六、v2 嵌入实践

### 关键配置

- Vite base: `/inventory/`
- BrowserRouter basename: `import.meta.env.BASE_URL`
- iframe src 前缀: `/inventory` + 路由 + `?embed=1`
- bizLine 传递: URL param 优先 → localStorage 回退

### 常见坑

| 坑 | 解法 |
|---|------|
| iframe 缺前缀 → 404 | iframeSrc 加 `/inventory` |
| CORS 端口不匹配 → 403 | WebConfig.java 加实际端口 |
| Token JSON 转义 → 401 | CDP WebSocket 取原始字节 |
| bizLine 不继承 | iframe URL 传 `&bizLine=` |

---

## 七、事故模式与复盘（血泪史）

### 典型事故模式

| 模式 | 案例 | 金额 | 根因 |
|------|------|------|------|
| 高危元素遗漏 | 付款单公司代码字段未管理 | 2.63亿 | 认知偏差+流程缺失 |
| 防重机制失效 | 返利计提 Job 重复生成 | 11.4亿 | 缺防重+缺金额校验 |
| 脚本违规执行 | 生产环境运行未 review 脚本 | 69万 | 流程不规范 |
| 缓存不一致 | 新用户下单地址找不到 | 82万 | 新旧接口不兼容 |
| 影响范围评估不足 | 注释支付代码只评估印度 | 56万 | 未评估西欧 |
| 监控阈值不合理 | 报关价格阈值从5%调到100% | 2700万 | 异常未触发告警 |
| 代码语义被修改 | code smell 优化改了退货逻辑 | 14.9万 | 优化≠重构 |

### 经验总结

**事前预防**：
1. 需求阶段识别高危元素，建清单持续迭代
2. 技术方案评估历史数据兼容性
3. 高危关联文件逐行 Review
4. 高危场景有专项测试

**事中控制**：
1. 灰度发布（先一个区域确认再全量）
2. 监控告警分级（一级：资金异常→立即；二级：可用性→30min；三级：非核心→工作时间）
3. 关键节点值班
4. 异常第一时间拉应急小组

**事后复盘**：
1. 根因分析（技术+管理双维度）
2. 明确"如果重来会怎么做"
3. 经验纳入检查清单
4. 改进措施有跟踪验证

### 事故复盘模板

```
## 基本信息
- 事故时间 / 影响范围 / 涉及金额 / 涉及高危元素

## 事故经过（时间线）

## 暴露问题
1. 高危元素管理：
2. 产研交付管理：
3. 监控告警机制：

## 根本原因（系统层面 + 管理层面）

## 如果重来（事前/事中/事后）

## 改进措施
| 措施 | 责任人 | 截止日期 | 状态 |
```

---

## 八、米盾（SecProxy）接入踩坑

> 来源：ai-collab 部署实战（2026-05-08），12 小时排查形成。适用于所有接入米盾 CAS 认证的服务。

### 核心原理

```
浏览器 → 米盾网关(CAS认证) → 后端服务
         ↑                    ↓
    未登录: 跳CAS        注入 X-Proxy-UserDetail header
    已登录: 放行+注入     后端解析获取用户信息
```

### X-Proxy-UserDetail 解析

格式：`RSA签名#Base64(JSON)` — 按第一个 `#` 分割，后半 Base64 decode 得到 JSON。

字段带 `cas:` 前缀：`cas:user`, `cas:name`, `cas:email`, `cas:uid`, `cas:departmentName`, `cas:avatar`

### 踩坑速查表

| 坑 | 现象 | 根因 | 解法 |
|----|------|------|------|
| API 401 | 前端请求返回 401 | fetch 默认不带跨域 cookie | 加 `credentials: "include"` |
| 跳 /route-no-auth | 访问新路径直接跳无权限页 | 米盾未对新路径配置认证策略 | 米盾后台添加路径规则 |
| 米盾返回 HTML 而非 JSON | 前端无法解析 401 响应 | 缺少 `X-Requested-With` header | 加 `X-Requested-With: xmlhttprequest` |
| 字段映射错误 | 后端 400/401，用户名为空 | 字段带 `cas:` 前缀 | 映射取 `u["cas:user"] \|\| u.username` |
| UserDetail 格式错误 | parseUserDetail 返回 null | 格式是 `签名#Base64`，不是纯 Base64 | 先按 `#` split 取后半 |
| Cookie 不转发 | 容器收不到浏览器 cookie | 米盾消费 `_aegis_cas` 后不转发 | 不依赖 cookie，用 header |
| CORS 错误 | 浏览器阻止带 cookie 跨域请求 | 后端未配 `credentials: true` | `cors({ origin: true, credentials: true })` |

### 前端必须配置

```typescript
// 所有 API 请求必须带：
const res = await fetch(url, {
  headers: { "X-Requested-With": "xmlhttprequest" },
  credentials: "include",
});
```

### 米盾管理后台 Checklist

- [ ] 域名已接入米盾（DNS 指向米盾 IP）
- [ ] 路径规则已添加（如 `/collab`）
- [ ] 认证方式设为 CAS
- [ ] 公钥已获取并配置到 `CAS_PROXY_PUBLIC_KEY`
- [ ] 健康检查路径（`/health`）设为 bypass
- [ ] 退出 URL：`https://<domain>/_aegis/cas/logout`

---

## 九、Matrix 部署踩坑

> 来源：同 ai-collab 部署实战。适用于小米 Matrix 平台 Node.js 服务部署。

### 避坑规则（11 条铁律）

| # | 规则 | WHY |
|---|------|-----|
| 1 | 选 openEuler/Debian，不选 CentOS 7 | GLIBC 2.17 不兼容 Node 18+ |
| 2 | 安装构建放启动命令（start.sh） | 代码安装脚本在构建层，运行容器不保留 |
| 3 | 首次部署关闭健康检查 | install+build 3-5 分钟，检查超时杀容器 |
| 4 | 写 start.sh 脚本，不在配置框写命令 | 复杂命令在 Matrix 框里转义出错 |
| 5 | 用 npm 不用 pnpm 部署 | pnpm 符号链接硬编码绝对路径，跨机器断裂 |
| 6 | 自动探测路径，不硬编码 | monorepo 工作目录可能嵌套 |
| 7 | prebuild 走内网镜像，不依赖 GitHub | 内网不能直连外网 GitHub |
| 8 | `--ignore-scripts` + `npm rebuild` 两步走 | 分离安装和原生编译 |
| 9 | 容器必须声明暴露端口 | 否则路由下拉框为空 |
| 10 | SecProxy 域名新路径需后台配置 | 否则直接跳 /route-no-auth |
| 11 | 502 按层排查 | 容器curl → 端口 → 路由 → 健康检查 → SecProxy |

### 部署 Checklist

- [ ] 运行环境选 openEuler 24.3 或 Debian Bookworm
- [ ] 代码安装脚本只装系统依赖（dnf/yum）
- [ ] deploy/start.sh 提交到 git
- [ ] start.sh 自动探测 APP_DIR
- [ ] 容器暴露端口已声明
- [ ] 路由转发已配置（路径 → 空间 → 端口）
- [ ] 健康检查路径可达（nginx 代理到后端）
- [ ] 首次部署健康检查关闭或延迟 300s+
- [ ] 环境变量已注入
- [ ] 前端 fetch 带 `credentials: "include"` + `X-Requested-With`
- [ ] 后端 CORS 配置 `credentials: true`
- [ ] 米盾公钥已配置 + bypass 路径已设置

---

## 十、从 Demo 到生产的六维度鸿沟

> 来源：[产品经理 AI Coding 实战](https://mi.feishu.cn/wiki/HWbywjOIliLxSAkMqpacdswanfc)。
> 每踩一个坑，都能对应到某个维度停在了 Demo 级。Harness 的各阶段门禁正是为了系统性跨越这道鸿沟。

### 六维度总览

| 维度 | Demo 级（踩坑表现） | 生产级（交付标准） | Harness 对应 |
|------|-------------------|------------------|-------------|
| 需求 | 设计稿 3.25/10、交互缺失、虚构功能规格 | 交互走查通过、异常场景覆盖 | Phase 1 模式 A/B/C |
| 架构 | 前后端分离?、Dubbo 隐式依赖、宽表混淆 | ADR 记录、完整依赖图、架构评审 | Phase 2 评审 |
| 编码 | null 崩溃、并发过载、命名混用 | 全入口 sanitize、并发控制、命名统一 | Phase 3 TDD + verification |
| 联调 | proxy 路由错、超时断连、数据量差异 | 真实数据量、超时保护、proxy 逐条审查 | Phase 6.5 + Phase 9 |
| 部署 | MR 不合规、密钥硬编码、pip 超时 | CR 合规、密钥加密、版本锁定 | Phase 7 部署确认 |
| 运行 | 性能爆炸、Volume 覆盖、N+1 查询 | 压测通过、监控告警、启动 fail fast | Phase 9 验证 |

### 需求维度避坑

| 坑 | 教训 | 对策 |
|----|------|------|
| P43 设计稿 AI 生成后 3.25/10 分 | AI 能画页面但不理解"用户怎么用" | AI 生成的设计稿必须经过交互走查 |
| Bot 拆分子查询后重复回答 6 次 | "代码能跑 ≠ 业务逻辑对" | 涉及多步合并输出的场景，显式约束合并规则 |
| 为不存在的功能生成规格说明 | AI 不区分"文档里有"和"实际实现了" | PRD 标注实现状态，不要让 AI 对着旧文档臆想 |

### 架构维度避坑

| 坑 | 教训 | 对策 |
|----|------|------|
| AI 默认推荐前后端分离（最流行） | 内部工具不分离更合适，省掉整层成本 | 问"有没有更简单的方案"而非"最佳实践" |
| 1:1 翻译漏掉 18 个 Dubbo RPC 功能 | 只看显式代码，不知道系统间暗线 | 重构前列完整依赖图（REST + RPC + MQ + Job） |
| 120+ 字段宽表 DB 和计算字段混淆 | `@TableField(exist=false)` AI 当 DB 列处理 | CLAUDE.md 中标注 DB 字段 vs 计算字段 |
| 架构决策无记录，后续 AI 无法理解"为什么" | 沉没成本导致无法掉头 | 每个决策记 ADR（背景/选项/权衡/结论） |

### 编码维度避坑

| 坑 | 教训 | 对策 |
|----|------|------|
| null 崩溃（24 个参数无 sanitize） | AI 只走 happy path | 所有对外接口入口做参数 sanitize |
| compat_openai 双重 /v1 导致 400 | base_url 可能已含版本路径 | URL 拼接前做路径规范化 |
| camelCase/snake_case 混用 | 前后端各自命名规范不统一 | 交界处统一命名转换层 |
| LLM 400 时 bot 直接崩溃 | 外部服务异常路径无防护 | 外部调用必须有错误降级策略 |
| 空模型列表被缓存导致永久失效 | 一次失败永久影响后续请求 | 缓存失败状态加过期机制 |

### 联调维度避坑

| 坑 | 教训 | 对策 |
|----|------|------|
| webpack proxy 把中台接口路由到本地 | "所有 /api 都转发"的通用模式不适用 | proxy 配置逐条审查 |
| LLM 多级降级超时叠加 30s+ | 每级独立超时叠加超出用户容忍 | 对外接口有总超时保护 |
| 本地几条数据秒级，生产几万条爆炸 | 小数据量无法暴露性能问题 | 联调用接近真实的数据量 |

### 部署维度避坑

| 坑 | 教训 | 对策 |
|----|------|------|
| MR 未关联 Meego、未邀请 review | AI 不知道公司 CR 合规规则 | MR 前检查：关联需求 + 邀请 reviewer |
| .dockerignore 排除了运行时必需文件 | AI 按通用模板排除 docs/ | 构建后检查运行时必需文件是否存在 |
| pip install 内网超时（10+ 次提交修） | 内网不通 PyPI | 内部镜像 + offline wheelhouse 双保险 |
| Kaniko 用 sh 不是 bash | AI 写 bash 语法，目标环境是 POSIX sh | CI 构建确认 shell 类型 |
| Python 版本不匹配（cp312 vs cp311） | 本地和 CI 版本不一致 | 锁定 Python/Node 版本，CI 与本地一致 |
| 密钥硬编码到 .env 提交仓库 | AI 不会主动建议密钥管理 | 走加密文件 + 运行时解密 |

### 运行维度避坑

| 坑 | 教训 | 对策 |
|----|------|------|
| N+1 查询（部门树递归展开） | 本地少量数据无法暴露 | 上线前压测，接近生产数据量和并发 |
| K8s Volume 挂载覆盖容器内文件 | 空 PVC 挂载 = 清空目标目录 | Volume 不挂载到含镜像数据的目录 |
| ONNX pthread_setaffinity_np 错误 | 容器 CPU 核心数与默认线程数不匹配 | 容器环境线程数/内存限制显式设置 |
| 问题出在"根本没想到会出问题的地方" | 错误信息误导排查方向 | Pod 启动后校验关键文件存在性，fail fast |

### AI Coding 特有的反模式

| 反模式 | 表现 | Harness 防护 |
|--------|------|-------------|
| AI 不会增量迁移只会重写 | 每次重写修一个旧 bug 带两个新 bug | 禁止整文件替换，必须增量编辑 |
| AI 用最新版本不管兼容性 | requirements.txt 用了 CI 不支持的版本 | 锁定版本号 |
| AI 按 happy path 写代码 | 异常输入毫无防备 | review 时专门做"调用方不守规矩会怎样"的检查 |
| AI 推荐最流行方案 | 不考虑团队约束和交付成本 | 架构选型问工程师确认 |
| AI 生成文档看似完整 | 流程图冲突、手册遗漏 | AI 文档同样需要核查 |
| AI 代码维护时无人理解 | "AI 写的我大致 review 过"半年后变黑盒 | review 深度要足够理解"为什么这样写" |

---

## 十一、Claude Code 内部架构启示（上下文工程）

> 来源：claude-code-analysis 逆向分析（2026-05）。六项可落地的架构洞察。

### 已落地到 Harness 的改进

| 改进 | 实现位置 | 核心洞察 |
|------|---------|---------|
| 分段 Prompt + 缓存边界 | `scheduler/lib/prompt-assembly.sh` | 静态层（rules）在前、动态层（task context）在后；中间用 CACHE_BOUNDARY 标记分隔；静态层跨迭代复用前缀缓存 |
| 压缩后状态重注入 | `scheduler/lib/run-task.sh` Compaction Recovery Block | 长会话压缩后自动重注入：task ID、branch、last commit、state dir；防止 agent "失忆" |
| 会话级断路器 | `scheduler/lib/state.sh` circuit_breaker_* | 同类失败 3 次 → 禁用该失败路径的自动恢复；防止无限重试死循环（Claude Code 同模式避免 25 万/日死锁调用） |
| 自动 Checkpoint | `scheduler/harness-run.sh` write_checkpoint() | 任务完成/阻塞时自动写 checkpoint 文件；每 3 个完成任务触发模式提取标记 |
| 工具并发分区 | `scheduler/lib/tool-partitioning.md` | readonly 命令可并行、mutating 命令强制串行；默认 mutating（fail-closed） |
| 子 Agent 缓存复用 | `skills/coding/subagent-driven-development.md` | Controller 和 Worker 共享相同静态层（字节级一致），只在动态层注入不同 task context |

### 尚未落地但值得关注的 Claude Code 机制

| 机制 | Claude Code 做法 | Harness 潜在应用 |
|------|-----------------|-----------------|
| Session Memory 阈值 | 累计 10K tokens + 间隔 5K + 间隔 3 次工具调用后才提取 | scheduler 的 progress.txt 写入频率可参考 |
| 输出 Token 上限 | 默认 8K，p99 实际 4.9K；截断后 64K 重试一次 | 任务执行超长输出截断策略 |
| PTL 洋葱剥皮 | 压缩超限时迭代删除 20% 最旧消息 | 长任务失败历史的渐进清理 |
| 团队共享 Memory | `.claude/teams/{team}/inboxes/{agent}.json` 邮箱通信 | 多 agent 协作的状态共享模式 |

---

## 十二、研发基线速查（MIT 13 项）

> 来源：小米信息部研发基线 + 测试过程基线

| # | 要求 | 工具 | 红线 |
|---|------|------|------|
| 1 | 任务线上化 | Meego | 无关联禁止上线 |
| 2 | 识别影响范围 | 技术方案 | — |
| 3 | 可观测性 | Hera | — |
| 4 | 失效分析 | 技术方案 | — |
| 5 | 代码须审查 | GitLab MR | 不审查禁止合并 |
| 6 | 编码规范 | Sonar + CI | 禁止跳过 CI |
| 7 | 落实自测 | 单测/Teego/截图 | 禁止不自测 |
| 8 | 上线须评审 | Checklist | 无评审禁止上线 |
| 9 | 测试方案 | Meego | P0 项目必须有 |
| 10 | 用例须评审 | Teego/文档/单测 | 产研评审通过 |
| 11 | 缺陷可追踪 | Meego | 优先级必填 |
| 12 | 测试报告 | Teego/文档 | 含结论+风险，无报告禁止上线 |
| 13 | 线上验证 | Teego | 上线必须验证+数据清理 |

---

## 十三、CI 部署踩坑（前后端一体化项目）

> 来源：inventory-engine-v2 部署实战（2026-05-09~11）。适用于前后端一体化项目（Java + Vite 前端，单 JAR 部署）。

### 铁律：前端源码变更 ≠ 前端部署

**WHY**：推送 `.tsx`/`.ts` 源文件到 eam-v2 main 后，测试环境部署完前端毫无变化。排查发现 CI 只编译 Java（`mvn compile`），不构建前端 — `frontend-maven-plugin` 已被移除（CI 环境无外网，不能 `pnpm install`）。前端构建产物 `src/main/resources/static/` 必须本地 `vite build` 后提交。

### 规则

| # | 规则 | WHY |
|---|------|-----|
| 1 | **推送前端改动时，必须同时提交 `src/main/resources/static/` 构建产物** | CI 不构建前端，只打包 JAR |
| 2 | 推送前本地 `vite build` 验证成功 | 确保产物可用 |
| 3 | 用 `rsync --delete` 同步 static 目录，不手动挑选文件 | Vite build 的文件名带 hash，每次构建都变 |
| 4 | **首次推送到某仓库时，确认 CI 是否有前端构建能力** | 查 pom.xml 中 `frontend-maven-plugin` 是否存在且未注释 |
| 5 | **`vite build` 后验证非构建文件未被删除** | `emptyOutDir:true` 会清空整个 static/，删掉手写文件 — 已踩 2 次，见 [CASE-001](cases/CASE-001-vite-emptyoutdir-deletes-non-build-files.md) |
| 6 | **部署后验证域内权限自注入** | 远端 gateway 不通时本地必须兜底 — 见 [CASE-002](cases/CASE-002-remote-gateway-down-permissions-empty.md) |

### 判定方法

```bash
# 检查 CI 是否会构建前端
grep "frontend-maven-plugin" pom.xml
# 如果找不到或被注释 → 前端产物必须本地构建后提交
# 如果存在且未注释 → CI 会自动构建，不需要提交 static/
```

### 推送检查清单（前后端一体化项目）

- [ ] Java 代码已增量修改（不整文件替换）
- [ ] 前端源码（.tsx/.ts/.scss）已修改
- [ ] 本地 `vite build` 成功
- [ ] `src/main/resources/static/` 构建产物已 `rsync --delete` 同步到目标仓库
- [ ] **非构建文件未被删除**：`ls static/embed-inject.js`（[CASE-001](cases/CASE-001-vite-emptyoutdir-deletes-non-build-files.md)）
- [ ] 构建产物已 `git add` + commit
- [ ] **不要**提交 `node_modules/` 或 `.vite/` 缓存

---

## 十四、context-path 与网关路由不匹配（全接口 404）

> 来源：inventory-engine-v2 部署事故（2026-05-11）。改 context-path 后全接口 404，排查 2 小时，根因 30 秒修复。

### 事故经过

重构盘点服务并重新部署后，所有 API 接口返回 404。应用本身正常运行（定时任务 SQL 执行成功），但浏览器请求全部失败。

**根因**：Spring Boot `context-path` 是 `/inventory`，但 EAM 平台网关直接透传 `/inventory-api/` 路径到 Pod，没有做路径重写。Pod 收到 `/inventory-api/configs/...`，和 `/inventory` 不匹配 → Tomcat 返回 404。

**迷惑点**：代码仓库里有 `deploy/nginx-gateway.conf` 写了正确的路径重写规则（`/inventory-api/` → `/inventory/`），但**实际部署环境没有使用这个 nginx 配置**。

### 诊断方法（30 秒定位）

```bash
# 在 Pod 内直接 curl，排除网关干扰
curl -v http://localhost:8080/inventory/configs/...      # 200 → 应用正常
curl -v http://localhost:8080/inventory-api/configs/...   # 404 → 路径不匹配

# 200 + 404 = 网关透传路径，context-path 不匹配
```

### 修复

`application.yml` 中 `context-path: /inventory` → `context-path: /inventory-api`，重新打包部署。

### 铁律

| # | 规则 | WHY |
|---|------|-----|
| 1 | **改 context-path 前，先确认网关实际发送什么路径到 Pod** | 配置文件描述的 ≠ 实际生效的 |
| 2 | **部署后 ALL 404 → 先 Pod 内 curl localhost** | 30 秒排除"应用问题"还是"网关问题" |
| 3 | **不要相信仓库里的网关配置文件** | nginx-gateway.conf 可能只是参考文档，实际网关由平台管理 |
| 4 | **context-path 变更属于部署级变更，不只是代码变更** | 必须同步确认：网关路由、SPA base path、embed 脚本路径 |

### 排查决策树

```
全接口报错？
├─ 应用没启动 → 看启动日志（Bean 创建失败 / 端口占用 / DB 连不上）
├─ 应用启动了 + 全部 500 → 公共 Filter/Interceptor 异常
├─ 应用启动了 + 全部 404 → Pod 内 curl localhost
│  ├─ localhost 也 404 → context-path / Controller 注册问题
│  └─ localhost 200 → **网关路径不匹配**（本次案例）
└─ 应用启动了 + 部分接口报错 → 具体接口的依赖/数据问题
```

### 二次事故：改 context-path 导致 SPA 404

**经过**：第一次修复改 context-path 为 `/inventory-api` → API 恢复，但 SPA 页面 `/inventory/...` 全部 404。因为 SPA 路由和 API 路由用了两个不同前缀。

**根因**：前端构建产物 bake 了错误的 `API_BASE`。pm-context 里 config.ts 是 `/inventory-api`，eam-v2 里已经改成 `/inventory`，但 `vite build` 在 pm-context 执行，把旧值 bake 进了 JS。

**正确修复**：context-path 保持 `/inventory`，前端 `API_BASE` 也用 `/inventory`（统一前缀），重新 build 后推送。

### 根本原因：双仓库漂移

| 反模式 | 后果 | 防护规则 |
|--------|------|---------|
| **在 A 仓库构建，往 B 仓库部署** | 两边 config 不同步时，bake 错误的值 | 前端构建必须在目标仓库（eam-v2 clone）中执行 |
| **rsync 静态产物时不验证内容** | 覆盖了 eam-v2 正确的产物 | 推送前 `grep` 验证构建产物里的关键配置值 |
| **pm-context 和 eam-v2 源码漂移** | 哪边是 source of truth 不清楚 | 双仓库源文件变更后，另一边必须同步 |

### 更新后的铁律（追加 #5-7）

| # | 规则 | WHY |
|---|------|-----|
| 5 | **前端构建必须在目标仓库 clone 中执行，不在 pm-context** | 两边 config 可能已漂移，在哪构建 = bake 哪边的值 |
| 6 | **推送静态产物前，grep 验证关键 baked 值** | `grep "inventory-api" static/` — 0 匹配才可推 |
| 7 | **context-path 和前端 API_BASE 必须一致** | 否则 SPA 和 API 不能用同一个前缀 |

---

## 十五、多仓库漂移防护（2026-05-11 Harness 全流程审查）

### 事故场景

| # | 踩坑 | 浪费时间 | 根因 |
|---|------|---------|------|
| 1 | 改 context-path 为 /inventory-api，但 eam-v2 已 revert 回 /inventory | 30min 排查 + revert | 没看目标仓库 HEAD |
| 2 | 实现 FunQueryService OR 修复，但 eam-v2 已独立实现 appendGroupedConditions | 20min 白写 | 没 diff 目标仓库 |
| 3 | code review 基于 pm-context 代码，但 7/9 文件 eam-v2 更完整 | review 结论部分失效 | 不知道 source of truth 是哪边 |

### 防护方案

引入 `Harness/repo-map.yaml` —— 模块仓库注册表，声明每个代码目录对应的部署仓库、推送规则和关键配置值。

**Phase 3（编码前）**：强制 drift check —— fetch 目标仓库 HEAD，diff 变更涉及的文件，漂移则暂停。

**Phase 7（推送前）**：3 步门禁 —— 确认目标 → diff 预检 → 配置值校验。

### 铁律（追加 #8-9）

| # | 规则 | WHY |
|---|------|-----|
| 8 | **编码前先 fetch 目标仓库 HEAD 并 diff** | 不知道目标仓库状态 = 盲改 |
| 9 | **模块未注册在 repo-map.yaml 则拒绝推送** | 没有明确的目标 = 推错仓库 |

---

## 十六、Phase 0 追踪深度不足导致运行时 bug（2026-05-11 V08 车辆运营）

### 事故场景

V08 借用归还流程优化（T1-T4+T6），首次接触 asset-entry-be / asset-eam-web / asset-commons-config 三个仓库。Phase 0 在编码后补做，补做时仍遗漏 3 个运行时 bug。

| # | 踩坑 | 报错现象 | 根因 | 浪费 |
|---|------|---------|------|------|
| 1 | `Map<String,String>` 传给 `sendUmsFlyLetter(Map<String,Object>)` | CI 编译失败 | 没确认 `MessageService` 方法签名，猜测参数类型 | 1 轮 CI + hotfix |
| 2 | `fieldCode: factoryCode/deviceCode` 生成 SQL `factory_code`/`device_code`，但实际列名是 `sn`/`asset_code` | 运行时 `Unknown column 'factory_code'` | 没追踪 `queryConditionReqs.fieldCode → CommonMapper.xml → SQL WHERE` 的映射规则 | 1 轮部署 + hotfix |
| 3 | `handleSearch`（hook）额外注入 `contains` 条件 | 后端校验拦截返回 400 | 没分析 `useSearchCombinationFilter` hook 的内部行为，假设它透传输入 | 1 轮部署 + hotfix |

### 根因分析

Phase 0 阶段 4（业务流程映射）追踪深度不够——**只追到 Service/Provider 层就停了**，没有追到 SQL 生成、方法签名、hook 内部行为这些运行时层面。

```
追到 Service 层 = 理解了 50%
追到 SQL/签名/运行时 = 理解了 100%
```

三个 bug 都发生在 "Service 层以下"：
- SQL 列名映射在 Mapper XML + 搜索框架中
- 方法签名在 Feign/Dubbo 接口定义中
- hook 行为在前端 npm 包内部逻辑中

### 防护方案

Phase 0 阶段 4 新增**追踪深度要求**（已更新 `skills/planning/project-understanding.md`）：

| 功能类型 | 必须追到 | 验证方法 |
|---------|---------|---------|
| 搜索/查询 | 最终 SQL（fieldCode → 列名映射） | `grep` Mapper XML 或对照 DB 表结构 |
| 外部调用 | 方法签名（参数类型、返回类型） | 读接口定义文件，不猜 |
| 前端 hook | hook 内部行为（转换/注入） | 读 hook 源码或 npm 包文档 |
| 定时任务 | 查询数据范围（type/status 过滤） | 对照数据模型确认是否需要 type 过滤 |

### 铁律（追加 #10-14）

| # | 规则 | WHY |
|---|------|-----|
| 10 | **搜索功能的 fieldCode 必须对照实际 DB 列名，不猜** | `factoryCode` ≠ `factory_code`，实际列名是 `sn`（V08） |
| 11 | **调用外部 Service 方法前确认签名，不假设泛型类型** | `Map<String,String>` ≠ `Map<String,Object>`（V08） |
| 12 | **前端 hook/组件不假设透传，必须确认内部是否转换输入** | `handleSearch` 会注入额外条件（V08） |
| 13 | **搜索框架的 connRelate: OR 不等于 SQL OR，多字段 OR 必须用 customSql** | 框架将多个 queryConditionReqs 逐条 AND 拼接，connRelate 仅控制括号分组而非条件间连接（V08） |
| 14 | **部署脚本自带 push 的，不要额外手动 push** | asset-eam-web 的 `tag.sh` 第一行就是 `git push`，先 push 再 tag.sh = 触发两次 CI（V08） |

---

## 十七、搜索框架 OR 语义与 customSql 绕行（2026-05-11 V08 T3 员工端精确搜索）

### 事故场景

员工端车辆精确搜索需要"车辆编号 OR VIN OR 资产编号"匹配任一字段。前端发送 3 个 `equalsTo` 条件并设置 `connRelate: 'OR'`，期望生成 `vehicle_code = 'X' OR sn = 'X' OR asset_code = 'X'`。

实际生成的 SQL：`AND vehicle_code = 'X' AND sn = 'X' AND asset_code = 'X'`——全部 AND，永远返回 0 条。

### 根因

EAM 搜索框架（JXS `CommonMapper`）将 `queryConditionReqs` 中的每个条件逐条拼接为 `AND`。`connRelate` 字段控制的是条件与括号分组的关系，**不是条件之间的 OR 连接**。框架不支持跨条件 OR。

### 修复方案

绕过 `queryConditionReqs`，将 OR 逻辑拼为 customSql：

```java
List<String> orClauses = new ArrayList<>();
for (QueryConditionReq condition : queryConditionReqs) {
    String value = condition.getFieldValues().get(0).replace("'", "");
    String column = camelToSnake(condition.getFieldCode());
    orClauses.add(column + " = '" + value + "'");
}
String orSql = "(" + String.join(" OR ", orClauses) + ")";
queryConditionReqs.clear();
shareSql = shareSql + " AND " + orSql;
```

最终 SQL：`... AND (共享资产规则) AND (vehicle_code = 'X' OR sn = 'X' OR asset_code = 'X')`

### 踩坑链（同一功能连续 4 个 bug）

| 序号 | Bug | 修了几轮 | 本质 |
|------|-----|---------|------|
| 1 | `handleSearch` hook 注入 `contains` → 后端 400 | 1 轮 | 不了解 hook 内部行为 |
| 2 | `factory_code`/`device_code` 列名不存在 | 1 轮 | 不了解 fieldCode→SQL 映射 |
| 3 | `connRelate: 'OR'` 不生效，3 条件 AND | 1 轮 | 不了解搜索框架 OR 语义 |
| 4 | 前端 push + tag.sh 触发 2 次 CI | 0 轮（浪费） | 不了解部署脚本行为 |

4 个 bug 的共同根因：**Phase 0 阶段 4 追踪深度不够**。

---

## 十八、同名变量不同来源——同构假设陷阱（2026-05-11 V08 T2 续借/归还 BPM 标题）

### 铁律（追加 #15）

| # | 规则 | WHY |
|---|------|-----|
| 15 | **对多个流程做相同改动时，必须分别追踪每个流程中变量的实际赋值来源，不因变量名相同就假设数据一致** | `reserveItems` 在预约中来自 DB、在续借/归还中来自 DTO，字段填充完全不同（V08） |

---

## 十九、系统已有实现不复用——从头造轮子陷阱（2026-05-11 V08 T6 驾照校验）

### 铁律（追加 #16-19）

| # | 规则 | WHY |
|---|------|-----|
| 16 | **新增功能前先搜索系统中已有的同类实现，复用已有链路而非从头构造** | `ShuCangAbility.execute` 已有完整的数据工场查询模式，从头写导致风格不一致+资源释放问题（V08 T6） |
| 17 | **跨服务 API 变更必须按依赖顺序部署：API 提供方先 deploy JAR，消费方再构建** | commons-config 加了 `DriverLicenseProvider` 接口但未 deploy JAR，entry-be 直接引用导致 Docker 构建 `cannot find symbol`（V08 T6） |
| 18 | **Artifactory deploy 用 curl REST API + 实际用户名，不依赖 Maven settings.xml** | settings.xml 的 reference token 只能读不能写；Maven deploy-plugin 版本/URL/凭证三重坑；curl 一次成功（CASE-050） |
| 19 | **编码和测试验证前必须深度追踪已有系统的六层行为，不能停在 Service/Provider 表面** | `equalsTo` 只取 `fieldValues.get(0)` 导致批量搜索全空；`GlobalValRes` 字段名是 `value` 不是 `globalValue`；UMS 模板变量名是 `buttonContent/url` 不是 `buttonText/buttonUrl`——全是没追到运行时实际行为的结果（V08 T1-T6） |

---

## 二十、V08 全流程复盘——执行纪律不足的系统性教训（2026-05-11）

### 全景

6 个功能点、3 个仓库、一个会话完成需求→编码→部署→验证。产出 6 个功能上线测试环境，但伴随 11 个 bug、10+ 次 hotfix、3 次 CI 失败。

### 铁律（#20-#22）

| # | 规则 | WHY |
|---|------|-----|
| 20 | **原子拆解到数据流：每个独立写入路径+展示端=一个原子** | T4"归还照片"实际是 4 个原子，漏了 2 个 |
| 21 | **门禁必须真正暂停，不自行推进** | 形式门禁 = 没有门禁 |
| 22 | **发现 bug 后立即扩散检查所有同构位置** | 续借 vehicleCode 空 → 归还也空；JAR 失败 → 第二次还犯 |

---

## 二十一、盘点 v2 批量修复——"编译通过≠修复完成"的系统性教训（2026-05-12）

### 全景

24 条测试 bug、19 项代码修复、2 次推送、1 次 hotfix。Harness 验证走了 3 遍才真正完成：第 1 遍编译通过就标"已修复"被打回；第 2 遍 Phase 4 自审发现 P0（manage_user_code 列不存在）回退；部署后 Phase 9 E2E 发现报告详情 500 回归（businessLineMapper.selectOne 多结果异常），再推 hotfix。

### 三层能力缺陷

| 层 | 表现 | 根因 |
|-----|------|------|
| **业务理解缺失** | 给 SELECT 加了 DB 不存在的列；写了永远不会被调用的代理 Controller；把手动分配当成自动生成的 bug | 没有先走一遍用户操作路径就动手改代码 |
| **链路追踪不完整** | 只验证了 scope 保存到请求体，没验证 DB 持久化→SQL 生成→任务拆分→资产分配整条链 | 修的是前端 stale closure，验证只在前端层面，没有端到端 |
| **测试方法有漏洞** | 用 queryConditionReqs（FunQuery 路径）验证 scope 过滤，但实际任务生成走的是 buildScopeWhereSql（不同路径）| 验证走的代码路径和修复走的不是同一条 |

### 铁律（#23-#27）

| # | 规则 | WHY |
|---|------|-----|
| 23 | **修 bug 前先在浏览器走一遍用户操作路径：看到现象 → 分析根因 → 写修复 → 再走一遍确认** | 盘点 v2 批量修复中，CommonsProxyController 代理了 `/commons/template/url`，但实际下载模板走的是 `/inventory/asset/sn-import-template`，代理从未被调用。如果先在浏览器点一次"下载模板"看 Network 面板，根本不需要写这个文件 |
| 24 | **修改 SQL SELECT/WHERE 前必须确认 DB schema：`SHOW COLUMNS` 或查实体 `@TableField(exist=false)` 注解** | 给 `AssetLedgerReader` 加了 `manage_user_code` 列，CLAUDE.md 明确写了台账表没有此列，Phase 4 自审+用户确认后回退。如果部署上去，所有台账查询全崩（P0 回归） |
| 25 | **selectOne 查询必须确认唯一性约束，不确定时用 selectList + LIMIT 1 + try-catch** | `businessLineMapper.selectOne(eq(lineCode, xxx))` 在 lineCode 有多条记录时抛 TooManyResultsException，导致报告详情页 500。部署后 Phase 9 E2E 才发现，推了 hotfix |
| 26 | **验证必须走和修复相同的代码路径，"有条件 vs 无条件"对比数量差异** | 用 `queryConditionReqs`（FunQuery 路径）验证了 scope 过滤，但实际任务生成走的是 `buildScopeWhereSql`（另一条路径）。碰巧两条路径结果一致（都是 20），但如果不一致就会漏掉 bug |
| 27 | **一个修复一个验证，不批量推送。标记"已修复"必须有对比证据，不是 HTTP 200** | 19 个修复一起推，出了 500 无法定位是哪个改动引入。后来逐项 E2E 验证，每项都有"修复前→修复后"的数据对比（如 113→20 资产数过滤），才算真正证明修复有效 |

### 验证方法改进

**修复前**：

| 做法 | 问题 |
|------|------|
| 编译通过 → 标"已修复" | 语法正确 ≠ 逻辑正确 |
| API 返回 200 → 标"验证通过" | 状态码正确 ≠ 数据正确 |
| 读代码推断行为 → 写修复 | 推断可能错，运行时行为才是真相 |

**修复后**：

| 做法 | 标准 |
|------|------|
| 浏览器操作 + Network 面板 → 确认实际请求路径 → 再写修复 | 先观察再动手 |
| "无条件 vs 有条件"数量对比 → 差值有业务含义 | 113→20 = 93 条被过滤 |
| 同一条代码路径端到端：前端发送 → DB 持久化 → SQL 查询 → 任务生成 → 资产分配 | 链路完整，不跳环 |
| 每个修复单独验证，截图/API 响应留证据 | "我证明了" ≠ "我认为" |

## 二十二、盘点 v2 scope 全量验证——"只测改了的 ≠ 测了全部"的系统性教训（2026-05-15）

### 全景

5 个用户反馈 bug → 修复+推送 → 声称"全部通过" → 用户指出"之前反复反馈始终没修好" → 重新审计发现修复方向偏了 2 个 + 又新发现 2 个 bug（save 500 + LIKE ESCAPE SQL 语法错误）→ 12 组全量矩阵验证 → 总计 7 bug 真正修完。

### 四层执行缺陷

| 层 | 缺陷 | 具体表现 | 铁律 |
|-----|------|---------|------|
| **验证覆盖面** | 只测修改的 operator，不测全量 | 修了 equal/notEqual，没测 like/notLike/leftLike → 漏了 LIKE ESCAPE SQL 语法错误（Bug 7） | #30 |
| **数据驱动** | 凭 PRD 推断数据而非查 DB | Bug 3 加了 GROUP/DEPARTMENT_ANNUAL 别名，但 DB 里根本没有这些值；真实脏数据是 inventory/annual/car/engineering | #29 |
| **同构审查** | 改了一个 if 分支不审查其他分支 | 修了 EQ/NE 匹配但没看 LIKE 分支缺 escapeLikeValue + ESCAPE | #31 |
| **归档纪律** | 修完 5 个 bug 不归档，等用户催 | Phase 10 每次 commit 后应立即归档 CASE + AP，不是攒到最后批量做 | #28 |

### 铁律（#28-#31）

| # | 规则 | WHY |
|---|------|-----|
| 28 | **每个 bug fix commit 后立即归档** | 修了 5 个 bug 不归档，声称"全部完成"。用户指出时才发现 Bug 2/3 方向偏了。如果每个 bug 归档时做反思，第一个就会发现数据假设错误 |
| 29 | **展示类 bug 第一步查 DB 数据分布** | Bug 3 "业务类型展示不全" — 凭 PRD 猜旧 code 是 GROUP，加了别名。实际 DB 的 business_type 值是 inventory/annual/car/engineering + 22 条空值，别名没命中任何一条 |
| 30 | **scope/operator 改动必须全量矩阵验证** | 只测了 equal/notEqual 2 个 → 漏了 like/notLike/leftLike 3 个。12 组全量矩阵覆盖后才发现 ESCAPE 反斜杠导致 SQL 语法错误 |
| 31 | **改一个 if-else 分支必须审查所有同级分支** | 修了 EQ 分支加 `\|\| "EQUAL"`，但没看 LIKE 分支——PlanService 的 LIKE 缺 escapeLikeValue 和 ESCAPE 子句，TaskGenerationService 的 ESCAPE 反斜杠在 MySQL 中被解释为转义引号 |

### Harness 框架改进

| 缺口 | 修复内容 | 位置 |
|------|---------|------|
| Phase 10 不强制逐次归档 | 追加铁律 #28："每个 commit 后立即 CASE + AP" | development-process.md |
| Phase 9 验证无覆盖率度量 | 追加覆盖率表：scope→14 operator、分支→全部同级、展示→DB 分布 | development-process.md Phase 9 |
| AP 缺"展示类 bug 先查 DB" | 新增 AP-091~093 | anti-patterns.md |
| 同构搜索范围不明确 | 铁律 #31 明确"改一个分支→审查所有同级" | development-process.md |

## 二十三、反复出现的问题必须治本——"修了又犯"的系统性反思（2026-05-15）

### 全景

两个"反复出现"的 bug：(1) 任务详情 tab 缺失，(2) v2→旧菜单白屏。两者都不是第一次报告，之前都"修过"但没根治。

### 为什么修了又犯？

| 问题 | 之前的修法 | 为什么没根治 | 这次的根治方法 |
|------|---------|------------|------------|
| Tab 缺失 | Bug 5 回退 splitSemicolon | 只修了类型，没处理 inventoryRule 为 null 和 planDimension 为 null 的情况 | 三层防御：null guard + Array.isArray + 默认值 |
| 白屏 | CASE-068 removeChild 补丁在旧系统 | 补丁在旧系统重新部署时丢失；且是"治标"（容忍错误）不是"治本"（阻止错误发生） | 从源头阻止：capture 阶段拦截事件，不让 React 有机会处理 |
| var 编译失败 | 每次发现改一个 | 没有 AP 约束，下次写新方法又用 var | 新增 AP-094 禁止 var |

### 自我反思：AI Agent 的三个执行缺陷

**缺陷 1：修了表象不修根因。** Bug 5 回退了 splitSemicolon，但没问"还有没有其他地方假设了类型？inventoryRule 为空怎么办？planDimension 为空怎么办？"——只修了报错的那一行，没沿着调用链追问。

**缺陷 2：修在了错误的层。** CASE-068 在旧系统打 DOM 补丁，但问题的发起方是 embed-inject.js（v2 侧）。在发起方修比在接收方补丁更可靠——因为发起方自己控制，接收方的补丁随时可能被部署覆盖。

**缺陷 3：修完不验证"反复场景"。** 修了 Bug 5 后只验证了"页面能打开"，没验证"tab 是否显示"、"各状态数量是否正确"。如果验证了完整的用户操作路径，当场就能发现 tab 缺失。

### 铁律（#32-33）

| # | 规则 | WHY |
|---|------|-----|
| 32 | **反复出现的 bug 必须沿调用链追 3 层以上** — 不止修报错的那一行，要问"上游传什么？下游假设什么？null/空/类型错误都覆盖了吗？" | Bug A 只修了 split 那一行，没查 inventoryRule 为 null 和 planDimension 为 null |
| 33 | **跨系统修复必须修在"发起方"而非"接收方"** — 补丁打在别人系统上会随部署丢失；修在自己控制的代码中才可靠 | Bug B 的 CASE-068 补丁在旧系统 index.tsx，每次旧系统部署就丢。改在 embed-inject.js（v2 控制）才是根治 |

### Harness 框架自身诊断——五个维度

> 以下不是对单个 bug 的反思，而是对框架机制本身的诊断：哪个环节该挡住但没挡住？

#### 维度一：流程拆分不够细——Phase 1 缺"数据摸底"步骤

**表现**：Bug 3"业务类型展示不全"——凭 PRD 猜旧数据是 `GROUP`，加了别名映射，但 DB 里根本没有。Bug A-8"英文值"——不知道 manage-line API 没嵌套 children，导致 `BUSINESS_TYPE_DESC` 为空对象。

**诊断**：Phase 1（需求分析）只要求"理解需求 + 确认范围"，没有强制"查 DB 数据分布"。对于"展示不对"类 bug，不看真实数据就写代码 = 瞎猜。

**改进**：Phase 1 增加 **Data Profiling 步骤**（铁律 #29 已加但执行力不足）：
```
展示类/数据类 bug 进入 Phase 3 前必须完成：
1. SELECT DISTINCT column, COUNT(*) GROUP BY column — 知道数据长什么样
2. 调实际 API 看返回结构 — 知道前端拿到的是什么
3. 在浏览器看渲染结果 — 知道用户看到的是什么
三步都做完才能写修复代码
```

#### 维度二：任务拆分不够细——"修 5 个 bug"应拆为 5 个独立原子任务

**表现**：第一天修 Bug 1-5，一次性分析→一次性修→一次性提交→一次性声称"全部完成"。结果 Bug 2/3 方向偏了、Bug 7 被漏掉、Bug 6 直到验证时才发现。

**诊断**：Harness Phase 2.5（原子拆解）要求"5+ 文件变更时强制拆"，但我把 5 个 bug 当成一个大任务处理。每个 bug 应该是独立的 Harness 循环：分析→修→验证→归档→下一个。

**改进**：
```
多 bug 修复的执行纪律：
- 每个 bug 是一个独立的 mini-Harness 循环（Phase 1→3→9→10）
- 一个 bug 的 Phase 9 验证通过 + Phase 10 归档完成后，才能开始下一个
- 禁止"批量分析→批量修→批量验证"——这会导致根因假设互相污染
```

#### 维度三：Agent SOP 不够细——缺少"写代码前检查清单"

**表现**：`var` 编译失败反复出现（3 次+）。Java 8 不支持 `var`，但每次写新方法都会无意识用，本地 JDK 17 不报错。

**诊断**：Harness 有 AP-094 禁止 `var`，但 AP 是被动检查（Phase 4 评审时扫描），不是主动防御（写代码时就阻止）。Agent 没有"写 Java 代码前的 pre-flight checklist"。

**改进**：Phase 3 编码阶段增加 **Pre-Code Checklist**（每次开始写 Java 之前执行）：
```
Java Pre-Code Checklist:
□ pom.xml java.version 是什么？（1.8 → 禁止 var/switch expression/text block）
□ 目标编译环境是什么？（CI 用 JDK 8？17？）
□ 有没有本地编译验证脚本？（run-local.sh / mvn compile）
□ 新增的 import 在目标 JDK 版本中是否存在？
```

也可以用 **Harness Hook（C9 工具层）**自动化：`post-edit` hook 检测 `var ` 关键字并报警。

#### 维度四：上下文注入不充分——反复 bug 没触发历史 CASE 加载

**表现**：Bug B（白屏）是 CASE-068 的完全重现，但我处理时没加载 CASE-068，从头排查了一遍。如果一开始就读了 CASE-068，直接知道"问题在 embed-inject.js 的事件阶段"，可以省 90% 排查时间。

**诊断**：Harness C4（知识注入）有 Phase 转换时自动匹配 AP 的机制，但匹配是基于关键词的——用户说"页面空白"，AP 里没有直接匹配这个关键词的条目。CASE-068 的关键词是 `removeChild`/`portal`/`react-activation`，和用户描述差异大。

**改进**：
```
反复 bug 触发规则（Phase 1 自动注入）：
- 用户说"反复出现"/"之前修过"/"又坏了" → 强制搜索全部 CASE 的"症状"字段
- 匹配到相似症状 → 加载完整 CASE 到上下文
- 特别是：同一个页面/组件/模块的历史 CASE 全部加载

关键词扩展：
- CASE 的"关键词"不仅是技术词（removeChild），还应包含用户描述词（"页面空白"/"白屏"/"无响应"）
```

#### 维度五：验证标准不够严——"能打开"≠"功能正确"

**表现**：修了 Bug 5（split 崩溃）后验证"页面能打开"就标通过。但页面能打开 ≠ tab 显示 ≠ 数据正确。Bug A 就是这样漏掉的。

**诊断**：Phase 9 的验证标准是"用户路径端到端验证"，但没有定义每个页面的 **功能完整性检查清单**。"盘点任务详情页"应该验证的不止是"不崩溃"，还有：tab 菜单是否显示、各状态数量是否正确、资产列表是否加载、操作按钮是否可用。

**改进**：
```
每个页面的验证检查清单（加入 Phase 9）：
盘点任务详情页：
□ 页面不崩溃（零 JS error）
□ 基础信息完整（任务单号/计划/业务类型/盘点人）
□ Tab 菜单显示（应盘/正常/异常/盘亏/盘盈/未盘）
□ Tab 数量正确（与 API 返回一致）
□ 资产列表加载（非空表格）
□ 操作按钮可见（提交盘点结果）

盘点计划编辑页：
□ 表单回显完整（所有必填项有值）
□ 下拉选项有数据（业务线/业务类型/通知方式）
□ 暂存成功（code=0）
□ 生成任务成功
□ scope 条件保存（equal/like 等 14 种）
```

### 总结：不是铁律不够多，是执行纪律不够强

回看 Harness 现在有 33 条铁律、97 条 AP、78 个 CASE——**规则够多了**。问题不在缺规则，在于：

1. **写了不执行**：铁律 #29（先查 DB）写了但没做；AP-094（禁 var）写了但下次还犯
2. **执行了不到位**：Phase 9 验证了"不崩溃"但没验证"功能完整"
3. **上下文没注入**：历史 CASE 存在但排查时没加载

**根治方向**：把关键规则从"文档约束"升级为"物理约束"——
- `var` 禁止 → Hook 自动检测（不靠人记）
- 数据摸底 → Phase 1 门禁（不查 DB 不让进 Phase 3）
- 反复 bug → 自动加载相同模块的历史 CASE（不靠关键词匹配，靠模块/文件路径匹配）
- 验证完整性 → 每个页面的检查清单是门禁而非建议

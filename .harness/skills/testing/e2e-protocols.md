# E2E 验证协议分类与路由

> Phase 6.8 本地 E2E 验证时，根据变更类型自动选择验证协议。
> WHY: 执行快照暴露——不同平台（Web/小程序/纯API）的验证方式完全不同，
> 缺少分类体系导致 Agent 自由发挥，验证质量不可预期。

## 协议路由表

| 协议 | 适用场景 | 验证工具 | 证据形式 |
|------|---------|---------|---------|
| **A** | Web SPA（浏览器可直连） | Chrome DevTools MCP | 截图 + console 无报错 + 网络请求断言 |
| **B** | 小程序 / Hybrid / 飞书内嵌 | 模拟器 / 真机 / 用户确认 | 录屏 / 截图 / 用户口头确认 |
| **C** | 纯后端 API（无 UI 变更） | curl / httpie / jq | 请求-响应 JSON 断言 |
| **D** | 前后端联动 + 复杂鉴权 | 协议 A/B + 协议 C 组合 | 双证据（API + UI） |

## 路由判定逻辑

```
1. 分析本次变更的文件类型:
   - 只有 .java/.py/.go + 无模板/消息变更 → 协议 C
   - 只有 .tsx/.vue/.html + 无 API 变更 → 协议 A
   - 前后端都有改动 → 协议 D
   - 涉及小程序/SDK/飞书组件 → 协议 B

2. 检查验证环境可用性:
   - Chrome DevTools MCP 可连接？→ A 可用
   - 本地 dev server 可启动？→ A 可用
   - 只有远程 test 环境？→ 需先部署（升级到 Phase 7→8→9）

3. 降级策略:
   - A 不可用（无浏览器环境）→ 降级为 C + 用户确认 UI
   - B 不可用（无模拟器）→ 降级为 C + [H-E2E] 用户手动验证确认
```

## 协议 A: Web SPA 验证

**前置条件**: Chrome DevTools MCP 已连接 OR 本地 dev server 可启动

```
Step 1: 启动目标页面
  → navigate_page → 目标 URL
  → 等待页面加载完成（wait_for 关键元素）

Step 2: 执行 P0 场景
  → 按 requirement.md 验收标准逐条操作
  → 每个操作后 take_snapshot / take_screenshot 收集证据

Step 3: 验证断言
  → console 无 error 级别日志（list_console_messages filter=error）
  → 网络请求状态码正确（list_network_requests → 无 4xx/5xx）
  → 页面元素状态正确（snapshot 中关键元素存在且内容正确）

Step 4: 输出证据
  → 截图存入 changes/<需求>/e2e-evidence/
  → 写入 e2e-evidence.md（P0 场景 × 操作路径 × 预期 × 实际）
```

## 协议 B: 小程序/Hybrid 验证

**前置条件**: 变更已部署到 test 环境 OR 本地模拟器可用

```
Step 1: 确认验证入口
  → 小程序: 飞书开发者工具 / 测试版入口
  → Hybrid: 飞书 App 内嵌页 → test 环境 URL
  → 明确告知用户验证路径

Step 2: 关键技术验证（如有）
  → HTML 标签渲染行为（rich-text/dangerouslySetInnerHTML）
  → 跳链行为（scheme/universal link/飞书内置浏览器）
  → 组件兼容性（小程序 API 限制）

Step 3: 用户确认
  → 输出验证清单给用户
  → 请求用户在真机/模拟器上执行并确认
  → [H-E2E] 标记为人工确认点

Step 4: 输出证据
  → 用户确认文本 / 截图 / 录屏链接
  → 写入 e2e-evidence.md
```

## 协议 C: 纯后端 API 验证

**前置条件**: 可直接调用目标 API（本地或远程）

```
Step 1: 构造测试请求
  → 从 requirement.md 提取 P0 场景的输入参数
  → 构造 curl/httpie 命令（含鉴权 token）

Step 2: 执行并断言
  → 发送请求，捕获响应
  → 断言 HTTP 状态码
  → 断言响应 body 关键字段
  → 断言边界条件（空值/越界/无权限）

Step 3: 回归验证
  → 用正常参数调用同一接口
  → 确认不影响正常流程

Step 4: 输出证据
  → curl 命令 + 响应 JSON
  → 写入 e2e-evidence.md
```

## 协议 D: 前后端联动验证

**前置条件**: 协议 A/B + 协议 C 的前置条件均满足

```
Step 1: 先执行协议 C（验证 API 层）
Step 2: 再执行协议 A 或 B（验证 UI 层）
Step 3: 验证联动
  → UI 操作触发的 API 请求参数正确？
  → API 响应在 UI 上正确渲染？
  → 错误场景的前端展示正确？

Step 4: 输出双证据
  → API 证据（curl + response）
  → UI 证据（截图/snapshot）
  → 写入 e2e-evidence.md（分 API/UI 两节）
```

## 门禁 4 判定标准（统一）

| 指标 | 要求 | 适用协议 |
|------|------|---------|
| P0 用例通过率 | 100% | 全部 |
| 新功能有 E2E 用例 | 是 | 全部 |
| 回归无新增失败 | 是 | 全部 |
| 证据完整 | 每个 P0 场景有对应证据 | 全部 |

**不满足 → 不能进入 Phase 7。**

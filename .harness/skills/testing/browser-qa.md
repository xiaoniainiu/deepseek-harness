---
name: browser-qa
description: |-
  浏览器自动化 QA 测试：利用 Chrome DevTools MCP 走用户路径、验证交互、
  截图取证、检查控制台错误。Phase 9 部署验证 + 日常 QA 均可使用。
  触发词：浏览器测试、QA 测试、E2E 验证、browser qa、走一遍用户流程
version: 1.0.0
appendix_limit: 50
---

# 浏览器 QA 自动化测试

> 利用 Chrome DevTools MCP 工具集做真实浏览器测试。
> 不是模拟——是真实点击、真实填写、真实截图。

## 触发条件

- Phase 9 部署验证时（强制）
- Phase 3 体验对照时（C11 deep-work §4，涉及 UI）
- 用户说"测一下"、"走一遍"、"浏览器验证"

## 可用工具清单

| 工具 | 用途 |
|------|------|
| `navigate_page` | 打开 URL / 前进后退 / 刷新 |
| `take_snapshot` | 获取页面 a11y 树（文字版，推荐） |
| `take_screenshot` | 截图取证 |
| `click` / `hover` / `drag` | 交互操作 |
| `fill` / `fill_form` | 表单填写 |
| `press_key` / `type_text` | 键盘输入 |
| `evaluate_script` | 执行 JS（检查状态/数据） |
| `list_console_messages` | 控制台错误检查 |
| `list_network_requests` | 网络请求检查 |
| `wait_for` | 等待文字出现 |
| `lighthouse_audit` | 无障碍/SEO/最佳实践审计 |
| `emulate` | 设备模拟（移动端/弱网） |

## 测试流程

### Step 1：准备

```
1. 确认测试 URL（部署环境/本地 dev server）
2. 打开页面: navigate_page → url
3. 等待加载完成: wait_for → 关键文字
4. 截图基线: take_screenshot → "baseline.png"
```

### Step 2：走 AC 路径

对每个验收标准（AC），按 Given/When/Then 执行：

```
FOR EACH AC in requirement.md:
  1. GIVEN: navigate 到起始页面 + 确认前置状态
  2. WHEN: 执行操作（click/fill/submit）
  3. THEN: take_snapshot 检查结果文字 + take_screenshot 截图取证
  4. 检查 console: list_console_messages → types: ["error", "warn"]
  5. 记录: AC-{NN} ✅PASS / ❌FAIL + 截图路径
```

### Step 3：边界场景

```
1. 空态: 页面无数据时显示什么？
2. 错误态: 提交无效数据 → 错误提示正确？
3. 权限: 无权限用户能否访问？（如可测试）
4. 并发: 快速双击提交 → 是否重复？
```

### Step 4：性能 + 可访问性（可选）

```
1. lighthouse_audit → mode: "snapshot" → 输出 a11y/SEO/best-practices 分数
2. 低于 80 分的维度 → 记录为 P2 问题
```

### Step 5：输出报告

```markdown
## 浏览器 QA 报告

**URL**: {测试环境 URL}
**时间**: {timestamp}
**AC 覆盖**: {passed}/{total}

### 通过的 AC
| AC | 操作 | 结果 | 截图 |
|----|------|------|------|
| AC-01 | 点击提交 → 审批标题包含车辆编号 | ✅ | baseline-ac01.png |

### 失败的 AC
| AC | 操作 | 期望 | 实际 | 截图 |
|----|------|------|------|------|
| AC-19 | 0 张照片点提交 | 拦截提示 | 提交成功 | fail-ac19.png |

### 控制台错误
| 级别 | 消息 | 页面 |
|------|------|------|
| ERROR | "TypeError: Cannot read property..." | /vehicle/return |

### Lighthouse 分数（如执行）
| 维度 | 分数 |
|------|------|
| Accessibility | 85 |
| Best Practices | 72 ← P2 |

### 健康度评分
- AC 通过率: {X}%
- Console 错误数: {N}
- 网络失败请求: {N}
```

## 与 Phase 9 的集成

Phase 9 部署验证时，本 skill 产出的报告写入 `changes/<req>/qa-report.md`。
AC 通过率 < 100% → Phase 9 FAIL → 修复后重新验证。

## 与 C11 体验对照的集成

Phase 3 结束前的体验对照（deep-work §4），用本 skill 实际走用户旅程，
对照 Phase 2 设计质量评审中描述的预期操作步数和信息层级。

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

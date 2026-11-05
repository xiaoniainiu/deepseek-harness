---
name: canary-monitor
description: |-
  部署后持续 Canary 监控：定期截图、检查控制台错误、
  对比性能基线、检测页面异常。Phase 9+ 使用。
  触发词：canary、部署后监控、watch deploy、持续检查
version: 1.0.0
appendix_limit: 50
---

# Canary 部署后监控

> 部署不等于上线成功。Canary 持续监控发现"部署了但用户体验变差"的问题。
> 利用 Chrome DevTools MCP 定期巡检。

## 触发条件

- Phase 9 部署验证通过后（建议执行 3-5 轮）
- 用户说"监控一下"、"盯着生产"、"canary"
- 重大变更上线后（L1/L2 需求）

## 监控流程

### 每轮巡检（约 30 秒/轮）

```
1. navigate_page → 生产 URL
2. wait_for → 关键文字（确认页面加载）
3. list_console_messages → types: ["error"]
   → 新增 error = 告警
4. take_screenshot → canary-{round}-{timestamp}.png
   → 与基线对比（视觉异常 = 告警）
5. evaluate_script → 检查关键数据是否正常
   → 例: document.querySelectorAll('tr').length > 0
6. list_network_requests → resourceTypes: ["fetch", "xhr"]
   → 非 2xx 响应 = 告警
```

### 告警输出

```markdown
## 🚨 Canary 告警 — Round {N}

**时间**: {timestamp}
**URL**: {url}

| 类型 | 详情 |
|------|------|
| Console Error | {error message} |
| 网络失败 | {url} → {status} |
| 页面异常 | {描述} |

**截图**: canary-{N}.png
**建议**: {回滚 / 排查 / 可忽略}
```

### 健康报告（所有轮次完成后）

```markdown
## Canary 监控报告

**监控时间**: {start} ~ {end}
**轮次**: {N} 轮
**间隔**: {M} 秒

| 指标 | 结果 |
|------|------|
| Console Errors | {N} 个（新增 {M} 个） |
| 网络失败请求 | {N} 个 |
| 页面加载成功率 | {%} |
| 视觉异常 | {N} 处 |

**判定**: ✅ 健康 / ⚠️ 需关注 / 🚨 需回滚
```

## 与 Phase 9 的集成

Phase 9 部署验证完成后，建议执行 3 轮 Canary（每轮间隔 60 秒）。
报告写入 `changes/<req>/canary-report.md`。
出现 🚨 告警 → Phase 10 不可进入，需先处理。

## Appendix（执行强调，不引入新规则）
<!-- managed-by: skill-evolution; do-not-edit-manually -->

_无_

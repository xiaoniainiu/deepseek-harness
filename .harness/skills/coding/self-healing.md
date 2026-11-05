# Self-Healing — Issue 自动修复

> Phase 3 中的自动修复机制。Issues 来自 reviewer 或 task executor，优先于新任务处理。

## 概述

Scheduler 在选择下一个 work item 时，open issues 的优先级高于 pending tasks。每次迭代只修一个 issue。

## issues.json 结构

```json
{
  "issues": [
    {
      "id": "ISS-20260509120000-0001",
      "taskId": "T-001",
      "severity": "high",
      "description": "缺少空 email 输入的校验",
      "source": "reviewer",
      "status": "open"
    }
  ]
}
```

## 严重性

| 级别 | 含义 | 优先级 |
|------|------|--------|
| high | 行为错误、功能缺失、安全问题 | 第一个修 |
| medium | 代码质量问题、边界未覆盖 | 其次 |
| low | 风格问题、小改进 | 最后 |

## 修复流程

```
1. 读 issues.json → 找最高 severity 的 open issue
2. 理解问题 → 读关联的 task 和文件
3. TDD 修复:
   - 写/更新一个暴露问题的测试
   - 验证测试 FAILS（确认问题存在）
   - 修复代码
   - 验证测试 PASSES
4. 跑 ALL qualityCommands
5. 提交: fix(harness): resolve issue - [description]
6. STOP — 一次修一个
```

## 规则

- **Issues 优先于新 tasks** — 有 open issue 就先修
- **一次一个** — 修完一个 issue 就结束本次迭代
- **必须 TDD** — 即使修 issue 也要先写测试
- **跑完整验证** — 修完后 verificationCommands + qualityCommands 全跑
- **不扩大范围** — 只修 issue 描述的问题，不顺手改别的

## 与 Recovery Queue 的关系

| 来源 | 处理方式 |
|------|---------|
| reviewer 发现的 issue | 通过 self-healing 修复 |
| task 执行失败 | 通过 recovery queue 重试（带失败 context） |
| recovery 重试仍失败 | 生成新 issue 或 escalate |

Scheduler 的选择优先级：`recovery queue > open issues > pending tasks`

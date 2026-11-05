# Verification — 反幻觉验证门禁

> 贯穿 Phase 3-8 的强制验证规则。证据先于声明。

## 铁律

```
没有本次迭代的验证证据，就不能声称完成。
```

上一次运行的结果不算。Agent 重启后、修改后，必须重新跑验证。

## 验证五步法

```
1. IDENTIFY — 哪条命令能证明这个声明？
2. RUN      — 执行完整命令
3. READ     — 读完整输出 + 检查 exit code
4. VERIFY   — 输出是否确认了声明？
5. CLAIM    — 只有通过了才能声明
```

## 禁止用语

出现以下用语 = 幻觉。立即停下，跑验证命令：

| 禁止 | 替代 |
|------|------|
| "应该能过" | 跑命令，贴输出 |
| "大概率没问题" | 跑命令，贴输出 |
| "看起来正确" | 跑命令，贴输出 |
| "基于我的修改" | 跑命令，贴输出 |
| "我很确定" | 跑命令，贴输出 |
| "这个应该能修复" | 跑命令，贴输出 |

## 声明-证据映射

| 声明 | 需要的证据 |
|------|-----------|
| 测试通过 | 测试命令输出，0 failures |
| 编译成功 | 编译命令 exit code 0 |
| Lint 清零 | Linter 输出 0 errors |
| Bug 已修 | 红灯测试 → 修复 → 绿灯测试 |
| 任务完成 | 所有 verificationCommands + qualityCommands 通过 |
| 功能正常 | curl/截图/数据内容正确（不是 code=0） |

## 四种任务完成状态

每个任务的验证结果必须归入以下四种状态之一（不是简单的 pass/fail）：

| 状态 | 含义 | 后续动作 |
|------|------|---------|
| **DONE** | 所有验证通过，任务完成 | → 进入审查 |
| **DONE_WITH_CONCERNS** | 验证通过但有疑虑（如性能未测、边界未覆盖） | → 先评估疑虑再审查 |
| **NEEDS_CONTEXT** | 缺少信息无法继续（如不确定业务规则、缺少 API 文档） | → 补充上下文后重新执行 |
| **BLOCKED** | 无法完成（如依赖服务不可用、权限不足、任务拆分有误） | → 评估阻塞原因，可能需要修改 plan |

**DONE_WITH_CONCERNS 的疑虑必须记录**：

```json
{
  "status": "DONE_WITH_CONCERNS",
  "concerns": [
    "未测试 12K 资产量下的任务生成性能",
    "deptCode 多值分隔符假设为分号，但 PRD 未明确"
  ]
}
```

Coordinator 评估疑虑后决定：接受（进入审查）/ 补充验证 / 返回修改。

**BLOCKED 的四种子原因**：

| 子原因 | 示例 | 处理 |
|--------|------|------|
| 上下文不足 | 不清楚业务规则 | 向用户提问 |
| 能力限制 | 需要 DBA 执行 DDL | 升级到人工 |
| 任务过大 | 预估 > 30 分钟 | 回到 auto-decompose 再拆 |
| 计划有误 | 依赖的任务产出与预期不符 | 修改 plan.json |

## Scheduler 模式下的验证

在 scheduler 模式（harness-run.sh）中，Agent 必须在 JSON 结果中填充 `verification`：

```json
{
  "verification": {
    "status": "DONE",
    "commandsRun": [
      {"command": "mvn test -Dtest=PlanMapperTest", "exitCode": 0, "status": "passed"},
      {"command": "mvn compile -q", "exitCode": 0, "status": "passed"}
    ],
    "concerns": []
  }
}
```

Scheduler 检查规则：
- `status` 必须是四种状态之一
- DONE / DONE_WITH_CONCERNS：`commandsRun` 不为空 + 所有命令 passed
- NEEDS_CONTEXT / BLOCKED：`commandsRun` 可为空，但必须有 `reason` 字段
- 有遗漏的 verificationCommand → 强制判定为 NEEDS_CONTEXT

## E2E 验证工具：Chrome DevTools MCP

Phase 9（部署验证）和功能联调需要通过浏览器验证业务正确性。Agent 必须使用 Chrome DevTools MCP 工具。

### 前置检查

进入 E2E 验证前，检查 `mcp__chrome-devtools__take_snapshot` 工具是否可用。

**如果不可用，输出以下引导**：

```
需要安装 Chrome DevTools MCP 才能进行 E2E 验证。

1. 在项目 .claude/settings.json 中添加：
   {
     "mcpServers": {
       "chrome-devtools": {
         "command": "npx",
         "args": ["@anthropic-ai/chrome-devtools-mcp@latest"]
       }
     }
   }

2. 重启 Claude Code 会话

3. 验证：尝试 mcp__chrome-devtools__take_snapshot
```

### E2E 验证反幻觉规则

| 错误做法 | 正确做法 |
|---------|---------|
| API 200 就标 ✅ | 对比数据内容：有/无条件返回数量必须不同 |
| 只验证可达性 | 验证业务正确性：数据符合筛选条件 |
| 自己构造参数 | 先看 DTO/前端代码确认字段名 |
| 假设前后端字段名一致 | 用 Network 面板看真实 payload |
| 截图看"页面加载了" | 检查具体元素/数据/状态是否正确 |

### 验证流程

```
1. navigate_page → 目标 URL
2. take_snapshot → 确认页面结构
3. 执行业务操作 (click/fill/submit)
4. take_snapshot → 对比操作前后变化
5. list_network_requests → 检查 API 请求参数和响应
6. 结论必须基于观察到的数据，不可猜测
```

## 底线

跑命令。读输出。然后才能声称结果。无例外。

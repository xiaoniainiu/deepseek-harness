# Agent 角色交接协议

## 角色-阶段映射

| Phase | 主角色 | 原因 |
|-------|--------|------|
| 0.5 (想法探索) | Coordinator | 需求分级 + 深度追问 + 方案对比 |
| 1 (需求分析) | Coordinator | 规划与理解 |
| 2 (需求评审) | Coordinator | 多视角综合 |
| 2.3 (技术设计) | Executor | 技术方案执行 |
| 2.5 (任务拆解) | Coordinator | 编排与分配 |
| 3 (编码实现) | Executor | 代码执行 |
| 4 (编码评审) | Verifier | 对抗性审查 |
| 5 (单测编写) | Executor | 测试执行 |
| 6 (单测评审) | Verifier | 对抗性审查 |
| 7-9 (推送/CI/部署) | Executor | 操作执行 |
| 10 (用户确认) | Coordinator | 综合归档 |

## 角色文件

| 角色 | 文件 | 加载时机 |
|------|------|---------|
| Coordinator | `agents/coordinator.md` | Phase 0.5, 1, 2, 2.5, 10 |
| Executor | `agents/senior-backend-developer.md` | Phase 2.3, 3, 5, 7-9 |
| Verifier | `agents/verifier.md` | Phase 4, 6 |

## 交接协议

Phase 转换时，当前角色必须：

1. **输出交接摘要**：当前阶段完成了什么、关键决策、待注意事项
2. **传递上下文文件**：列出下一角色需要读取的文件
3. **标明下一角色**：明确写出 "交接给 {Coordinator|Executor|Verifier}"

格式：
```markdown
## 交接: Phase N → Phase N+1

**当前角色**: Coordinator
**下一角色**: Executor
**阶段产出**: requirement.md, review-v2.md, plan.json
**关键决策**: [简要列出]
**待注意事项**: [风险/依赖/约束]
**传递文件**: 
- changes/<req>/requirement.md
- changes/<req>/plan.json
```

## 轮次预算

| 角色 | 单 Phase 最大轮次 | 超出处理 |
|------|------------------|---------|
| Coordinator | 8 | 输出当前状态，请求用户指导 |
| Executor | 无限制（由 scheduler 控制） | 受 `MAX_TASK_RETRIES` 限制 |
| Verifier | 5 | 输出 PARTIAL 判定，标注未完成项 |

## Scheduler 模式下的角色路由

`scheduler/lib/run-task.sh` 中 `task_prompt_source()` 按 work item 的 `type` 字段路由：

| work item type | 加载的 agent 文件 |
|---------------|------------------|
| `review` | `agents/verifier.md` |
| `planning` / `decomposition` | `agents/coordinator.md` |
| 其他（默认） | `agents/senior-backend-developer.md` |

## 升级条件

当同一 Phase 内角色交接超过 3 次（ping-pong）时：
1. 停止交接
2. 输出冲突摘要
3. 请求用户介入

## 对齐训练原则（Prompt 工程）

来源：Harness Engineering 五步法 Step 1。

### 三层注入架构

```
Layer 1: 角色定义（agents/*.md）   — "我是谁"
Layer 2: 行为边界（rules/*.md）    — "什么能做什么不能做"
Layer 3: 工具约束（skills/*.md）   — "怎么做"
```

### 提示词设计铁律

1. **4K token 上限**：每个 agent 文件的核心规则控制在 4K token 以内。超过后模型遵从率显著下降。辅助信息放到 skills/ 按需加载。
2. **正向引导 > 负向禁止**："不要做 X"比"要做 Y"更难遵从。优先用正向表述。
   - Bad: "不要生成超出范围的代码"
   - Good: "只修改 filesToModify 列出的文件"
3. **规则冲突必须有优先级**：当两条规则矛盾时，模型会"和稀泥"。必须显式声明优先级。
   - 安全红线 > 流程规范 > 代码风格 > 效率优化
4. **按需加载**：不要一次性注入全部上下文。L1（agent + rules）常驻；L2（skills）按 Phase 触发；L3（wiki/cases）按需查阅。
5. **可维护性**：规则在 markdown 文件中，热更新不重启。禁止硬编码到 scheduler 脚本里。

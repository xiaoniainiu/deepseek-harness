# MR Review — 结构化 MR 代码评审

> Phase 4（编码评审）和 Phase 7（推送后评审）的自动化 code review。

## 概述

对 GitLab MR 执行结构化代码评审：分析 diff、加载依赖上下文、逐文件检查、发布行级评论。

## 触发时机

- MR 创建后自动触发
- 用户手动请求评审
- Scheduler `--review` 模式下任务完成后

## 工作流

### 1. 初始化

```bash
# 获取 MR 信息
glab mr view <MR_IID> --repo https://git.n.xiaomi.com/<PROJECT_PATH> --output json
```

保存关键字段：`project_id`, `mr_iid`, `source_branch`, `target_branch`, `base_sha`, `head_sha`

### 2. 获取 Diff

```bash
# 使用 GitLab API 获取精确 diff
glab api projects/<PROJECT_ID>/merge_requests/<MR_IID>/changes \
  --hostname git.n.xiaomi.com --output json
```

解析统一 diff 格式，为每个文件建立行号映射。

### 3. 任务拆分

为每个变更文件生成一个评审任务：
- 文件路径 + 变更类型（新增/修改/删除）
- 变更行范围列表
- 优先级（核心逻辑 > 工具 > 配置）

### 4. 逐文件评审

对每个文件：

**a) 加载上下文**
- 读当前文件完整内容（head_sha 版本）
- 分析 import 收集项目内依赖
- 按类型加载依赖（HOOK > CONFIG > UTIL > COMPONENT）

**b) 执行检查**

| 维度 | 检查内容 |
|------|---------|
| 正确性 | acceptanceCriteria 是否满足，逻辑错误 |
| 架构 | 分层是否正确，依赖方向 |
| 安全 | 注入、XSS、敏感信息泄露 |
| 性能 | N+1 查询、不必要的循环、内存泄漏 |
| 编码规范 | 命名、注释、复杂度 |
| 测试 | 覆盖是否充分，mock 是否合理 |

**c) 输出评审结果**

每个问题包含：
- 文件路径 + 精确行号
- 严重性（BLOCKER / MAJOR / MINOR / INFO）
- 问题描述 + 代码证据
- 修复建议

### 5. 发布评论

使用 GitLab Discussions API 发布行级评论：

```bash
curl -s -X POST \
  "https://git.n.xiaomi.com/api/v4/projects/<ID>/merge_requests/<IID>/discussions" \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "body": "[MAJOR] 问题描述",
    "position": {
      "base_sha": "<base>",
      "start_sha": "<start>",
      "head_sha": "<head>",
      "position_type": "text",
      "old_path": "<path>",
      "new_path": "<path>",
      "new_line": <line>
    }
  }'
```

### 6. 输出报告

汇总所有问题，按严重性排序：
- BLOCKER: 必须修复才能合并
- MAJOR: 强烈建议修复
- MINOR: 建议改进
- INFO: 知识分享

## 严重性标准

| 级别 | 标准 | 阻塞合并 |
|------|------|---------|
| BLOCKER | 功能缺失、数据丢失、安全漏洞、生产风险 | 是 |
| MAJOR | 性能问题、边界未处理、代码质量严重问题 | 建议 |
| MINOR | 命名不佳、小优化机会、风格问题 | 否 |
| INFO | 学习建议、最佳实践分享 | 否 |

## 与 Harness 门禁的关系

| Phase | 使用方式 | 门禁标准 |
|-------|---------|---------|
| Phase 4 编码评审 | 全面 review | P0=0, P1≤2 |
| Phase 7 MR 评审 | 推送后 review | BLOCKER=0 |
| Scheduler --review | 自动 review | issues=0 则 PASS |

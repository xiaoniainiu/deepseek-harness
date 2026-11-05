# Git Worktree 隔离开发

> Phase 3 开始前创建隔离工作区，主分支永不脏。来源：Superpowers using-git-worktrees。

## 触发条件

- Phase 2 评审通过 [H2]，即将进入 Phase 3 编码
- 变更级别 ≥ A 级（单模块功能以上）

## 为什么隔离

| 问题 | 隔离后 |
|------|--------|
| 编码一半发现方向错误，主分支已被污染 | worktree 直接删除，主分支干净 |
| 多个需求并行开发，互相冲突 | 每个需求独立 worktree |
| 部署验证失败需要回滚 | 主分支没动过，无需回滚 |

## 操作流程

### 创建隔离工作区

```bash
# 在项目根目录执行
REQ_NAME="2026-05-13-vin-check"
git worktree add .harness/worktrees/$REQ_NAME -b feat/$REQ_NAME

# 进入隔离工作区
cd .harness/worktrees/$REQ_NAME
```

### 初始化验证

```bash
# 确认分支正确
git branch --show-current    # 应显示 feat/$REQ_NAME

# 确认测试基线干净
$CMD_COMPILE && $CMD_TEST    # 编译 + 测试必须通过
```

### 在隔离工作区中开发

Phase 3-6 的所有工作在 worktree 中完成。commit 正常提交到 `feat/$REQ_NAME` 分支。

### 完成后合并

Phase 7 推送时，从 worktree 分支合并回主分支：

```bash
# 回到主工作区
cd /path/to/project

# 合并（推荐 squash，保持主分支整洁）
git merge --squash feat/$REQ_NAME
git commit -m "feat: $REQ_NAME"

# 清理 worktree
git worktree remove .harness/worktrees/$REQ_NAME
git branch -d feat/$REQ_NAME
```

### 放弃时清理

如果方案被否定或需求取消：

```bash
git worktree remove --force .harness/worktrees/$REQ_NAME
git branch -D feat/$REQ_NAME
```

主分支完全不受影响。

## Scheduler 集成

`harness-run.sh` 在 `--worktree` 模式下自动创建和管理 worktree：

```bash
harness-run.sh --tool claude --review --worktree
```

Scheduler 行为：
1. 任务开始前：`git worktree add` 创建隔离分支
2. 所有任务在 worktree 中执行
3. 全部任务完成 + 审查通过：自动 merge --squash 回主分支
4. 任何任务 BLOCKED：保留 worktree，等待人工决策

## 不使用 worktree 的场景

| 场景 | 原因 |
|------|------|
| S 级变更（文案/配置） | 改动太小，隔离开销不值得 |
| 热修复（紧急 bug） | 需要直接在主分支修复 |
| 已有分支策略的项目 | 项目自己管分支，Harness 不干预 |

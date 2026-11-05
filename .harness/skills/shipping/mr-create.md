# MR Create — 创建 GitLab 合并请求

> Phase 7（代码推送）的结构化 MR 创建流程。

## 触发时机

- Phase 6 单测评审通过
- 用户确认推送 [H4]
- scheduler `finishing-branch` 选择 Option 2

## 前置检查

```bash
# 1. glab CLI 可用
glab --version

# 2. 已认证（私有 GitLab）
glab auth status --hostname git.n.xiaomi.com

# 3. 工作目录干净
git status

# 4. 不在默认分支上
git branch --show-current
```

## 流程

### 1. 收集上下文

```bash
# 找到默认分支
DEFAULT_BRANCH=$(glab repo view --output json 2>/dev/null | jq -r '.default_branch')
# 备选
DEFAULT_BRANCH=$(git remote show origin | grep "HEAD branch" | awk '{print $NF}')

# 分析提交
git log origin/$DEFAULT_BRANCH..HEAD --oneline --no-decorate

# 查看变更文件
git diff origin/$DEFAULT_BRANCH..HEAD --stat
```

### 2. 构建 MR 信息

从以下来源推断：
- 提交消息 → 变更类型（feat/fix/refactor）
- 分支名 → 功能描述
- `requirement.md` → 需求背景和验收标准
- `plan.json` → 任务列表和完成状态

### 3. Rebase + Push

```bash
git fetch origin
git rebase origin/$DEFAULT_BRANCH
git push -u origin HEAD
```

### 4. 创建 MR

```bash
# 写 body 到临时文件避免 shell 转义问题
cat > /tmp/mr_body.md <<'EOF'
## 变更内容

- 功能描述

## 关联需求

Closes #xxx

## 测试方案

- 单测覆盖情况
- 手动验证步骤

## 自测清单

- [ ] 编译通过
- [ ] 单测通过
- [ ] Lint 通过
- [ ] 关键路径手动验证
EOF

glab mr create \
  --title "feat(scope): 简短描述" \
  --description "$(cat /tmp/mr_body.md)" \
  --target-branch $DEFAULT_BRANCH \
  --remove-source-branch

rm /tmp/mr_body.md
```

### 5. 创建后

- 显示 MR URL
- 默认触发 code review（除非用户明确说跳过）
- 提醒 CI 流水线将自动运行

## MR 标题规范

遵循 Conventional Commits：
- `feat(盘点): 新增计划导出功能`
- `fix(台账): 修复分页查询超时`
- `refactor(调拨): 重构审批流程`

## 错误处理

| 问题 | 处理 |
|------|------|
| 无新提交 | 提示用户检查是否在正确分支 |
| 分支未推送 | 先执行 `git push -u origin HEAD` |
| MR 已存在 | 显示已有 MR，问是否更新 |
| 合并冲突 | 引导 rebase 解决 |

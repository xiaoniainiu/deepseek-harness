# Finishing Branch — 分支完成工作流

> Phase 7-10 的分支收尾操作。验证完成 → 选择操作 → 执行。

## 触发时机

- Scheduler 完成所有任务（exit 0）
- 用户手动确认功能完成
- Phase 6 单测评审通过后

## 前提检查

```bash
# 所有 task 必须 done
jq '[.tasks[] | select(.status != "done")] | length' plan.json
# → 必须为 0

# 无 open issues
jq '[.issues[] | select(.status != "resolved")] | length' issues.json
# → 必须为 0
```

**不满足则停止**，报告剩余内容。

## Step 0: 确认功能分支存在

**Phase 3 编码前必须创建功能分支**，推送前验证当前不在 main/master：

```bash
current_branch=$(git branch --show-current 2>/dev/null)
if [ "$current_branch" = "main" ] || [ "$current_branch" = "master" ]; then
  echo "BLOCKED: 禁止在 main/master 上直接开发"
  echo "动作: git checkout -b feature/<需求简称>"
  exit 1
fi
```

## Step 0.5: 读取部署配置

**必须从 repo-map.yaml 和项目配置中确认部署方式**：

```bash
# 读取 repo-map.yaml 确认推送目标和 CI 配置
cat Harness/repo-map.yaml | grep -A 20 "<module-name>"

# 按项目类型确认部署路径：
# - backend: git push → CI tag → Pod 部署
# - miniprogram: git push → 构建 → 预览码 → 体验版
# - frontend: git push → CI → CDN 部署
# - docs: 文档平台发布
```

IF 项目有 `deploy-config.yaml` 或 `Dockerfile` → 读取确认部署参数
IF 是小程序项目 → 确认 `project.config.json` 中的 appid 和环境
IF 是后端项目 → 确认 `application.yml` 中的 profile 和端口

## 流程

### Step 1: 跑质量命令

执行 plan.json 的所有 `qualityCommands`：

```bash
# 示例
mvn compile -q
mvn test -q
npx eslint . --quiet
```

**任何失败 → 停止，修复后再来。**

### Step 2: 确定基分支

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

### Step 3: 四选一

```
所有任务完成，质量检查通过。选择操作：

1. 本地合并回 <base-branch>
2. 推送并创建 Merge Request
3. 保留分支不动（我稍后处理）
4. 丢弃本次工作

选哪个？
```

### Step 4: 执行

#### Option 1: 本地合并

```bash
git checkout <base-branch>
git pull
git merge <feature-branch>
# 在合并结果上重跑质量命令
<qualityCommands>
# 通过后删除功能分支
git branch -d <feature-branch>
```

#### Option 2: 推送 + 创建 MR

```bash
git push -u origin <feature-branch>
```

然后调用 `skills/shipping/mr-create.md` 创建 MR。

#### Option 3: 保留

报告："分支 `<name>` 已保留。需要时再次触发 finishing-branch。"

#### Option 4: 丢弃

**必须二次确认：**
```
即将永久删除:
- 分支 <name>
- 自 <base-branch> 以来的所有提交

输入 'discard' 确认。
```

确认后：
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

## 禁止行为

- 质量命令失败时不推进
- 合并后不验证就认为安全
- 不经确认就删除工作
- 不经明确请求就 force-push

## 输出产出物

完成后生成 `changes/<req>/retrospective.md`：
- 需求摘要
- 完成日期
- 任务清单（全部 done）
- 发现的问题和修复记录
- learned patterns / rules（从 progress.txt 提取）

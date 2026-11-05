# EAM2.0 发布流程规范

- 来源：团队发布流程文档 v1.0（2026-05-13）
- 适用：EAM 全部后端微服务 + 前端仓库
- Harness 关联：Phase 7（代码推送）、Phase 7.5（上线准备）

## 分支策略

```
master (生产分支)
  ├── feature/xxx (功能分支，从 master 创建)
  │     └── 开发完成后合并到 test
  ├── test (测试分支，集成测试验证)
  └── release/vx.x (发布分支，可选)
```

### 分支命名

| 类型 | 格式 | 示例 |
|------|------|------|
| 功能 | `feature/功能名-日期` 或 `feature/功能名-开发者` | `feature/验收明细报表-0513` |
| 热修复 | `hotfix/问题描述-日期` | `hotfix/修复登录BUG-0513` |
| 标签 | `v主版本.次版本.修订版本` | `v1.2.0` |

### 分支保护

| 分支 | 规则 |
|------|------|
| master | 禁止直接推送，需 PR/Merge |
| test | 禁止强制推送，只能合并 |
| feature/* | 可自由推送和删除 |

## 测试环境发布（前后端一致）

```
1. git checkout master && git pull origin master
2. git checkout -b feature/功能名-开发者
3. 开发 → git add → git commit -m "feat: 功能描述"
4. git push -u origin feature/功能名-开发者
5. git checkout test && git pull origin test
   git merge feature/功能名-开发者 && git push origin test
6. (可选) 前端在 test 分支执行 ./tag.sh 触发部署
```

## 生产环境发布

### Step 1：同步 master 到功能分支（重要！）

```
git checkout feature/功能名
git fetch origin master
git merge origin/master          # 解决冲突
本地编译通过 + 单元测试通过
git push origin feature/功能名
```

### Step 2：合并到 master 并打 tag

```
git checkout master && git pull origin master
git merge feature/功能名         # 解决冲突
git push origin master
./tag.sh                         # 创建生产发布标签
```

### Step 3：验证

- 检查 tag 创建成功
- 确认发布版本号
- 生产环境部署验证

## 合并前检查清单

- [ ] 拉取了远端 master 最新代码
- [ ] 功能分支代码已提交
- [ ] 本地编译/构建通过
- [ ] 单元测试通过
- [ ] 无未解决的冲突
- [ ] 提交信息规范（feat/fix/docs/style）

## 版本号规范

```
v主版本.次版本.修订版本
```

| 位置 | 升级时机 |
|------|---------|
| 主版本 | 架构重构、API 不兼容变更 |
| 次版本 | 新功能上线（向后兼容） |
| 修订版本 | Bug 修复（向后兼容） |

## 冲突处理 SOP

```bash
git fetch origin master
git merge origin/master
# 解决冲突文件
git add .
git commit -m "merge: 解决合并冲突"
git push origin feature/功能名
```

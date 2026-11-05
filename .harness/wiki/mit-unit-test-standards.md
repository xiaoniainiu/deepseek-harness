# MIT 单测标准与 CI 门禁参考

> 来源文档（飞书）：
> - [Java单元测试规约](https://mi.feishu.cn/docx/SlnXdtFChoKbrgx8P32c65eanlc)（阿里 Java 开发手册黄山版）
> - [CI模板标准化修改](https://mi.feishu.cn/docx/ZcSydITU1otUNdxq2LfcPLOrnYg)
> - [MR合并管控说明](https://mi.feishu.cn/wiki/JRDMwCZl2i0MIcky1nocJRWJnKc)
> - [研发单测覆盖工具使用方案](https://mi.feishu.cn/docx/GqeIdnNaOoSAVsxm6XYc8MQgn1f)
> - [MIT 服务端单测实战快速入门](https://mi.feishu.cn/docx/W5yAdWukdokfbzxXO3jcoFU1nQg)

---

## 一、MR 合并管控条件（门禁）

MR 需要**全部满足**以下条件才允许合并：

| 条件 | 说明 |
|------|------|
| CI 流水线执行成功 | Pipeline 绿色 |
| 单元测试用例 **100% 通过** | 全量（存量+增量）单测通过率 100% |
| 变更代码行覆盖率 **≥ 60%** | 新增+修改代码的行覆盖率（Sonar 统计） |
| Sonar 质量门禁通过 | 包括 bug、漏洞、坏味道等规则 |
| MR 已关联 Meego 需求 | 任务线上化（MIT 基线 1） |

---

## 二、Java 单元测试规约（16 条）

### 强制（6 条）

| # | 规约 |
|---|------|
| 1 | **AIR 原则**：Automatic（自动化）+ Independent（独立性）+ Repeatable（可重复） |
| 2 | 全自动执行，非交互式。禁止 System.out 人肉验证，必须用 **assert** |
| 3 | 用例之间不互相调用，不依赖执行顺序 |
| 4 | 不受外部环境影响。依赖改为注入，测试时用 Mock/内存实现 |
| 5 | 测试粒度至多类级别，一般**方法级别** |
| 6 | 核心业务、核心应用、核心模块的增量代码必须确保单测通过 |

### 推荐（8 条）

| # | 规约 |
|---|------|
| 7 | 测试代码在 `src/test/java`，不在业务代码目录 |
| 8 | **语句覆盖率 ≥ 70%**；核心模块语句+分支覆盖率 **100%** |
| 9 | **BCDE 原则**：Border（边界）+ Correct（正确输入）+ Design（结合设计文档）+ Error（强制错误输入） |
| 10 | 数据库测试：不假设数据存在，用程序插入/导入准备数据 |
| 11 | DB 测试设定自动回滚机制，或使用明确前后缀标识测试数据 |
| 12 | 不可测代码适时重构，使其变得可测 |
| 13 | 设计评审阶段确定单测范围，覆盖所有测试用例（UC） |
| 14 | 项目提测前完成单测，不建议发布后补 |

### 参考（2 条）

| # | 规约 |
|---|------|
| 15 | 避免：构造方法做太多事、过多全局变量/静态方法、过多外部依赖、过多条件语句 |
| 16 | 单测不是测试同学的事；代码不多余；需要维护；与线上故障有直接关系 |

---

## 三、CI 模板接入（Java 工程）

### `.gitlab-ci.yml` 配置

**模板 1：Sonar 扫描 + 单测**（推荐）

```yaml
include:
  - project: mit/infra/ci-templates
    ref: master
    file: maven/scan.gitlab-ci.yml
stages:
  - scan
```

包含：安全扫描 + Sonar 扫描（含编译）+ 单测执行

**模板 2：Sonar + 单测 + Checkstyle**

```yaml
include:
  - project: mit/infra/ci-templates
    ref: master
    file: maven/scan+checkstyle.gitlab-ci.yml
stages:
  - scan
```

**模板 3：单测 + Sonar + 部署打包（Matrix）**

```yaml
include:
  - project: mit/infra/ci-templates
    file: maven/maven.gitlab-ci.yml
    ref: master
```

### 常用 CI 变量

| 变量 | 说明 | 默认值 |
|------|------|--------|
| `MAVEN_OPTS` | Maven JVM 参数 | `-Xmx2g` |
| `SONAR_PROJECT_KEY` | Sonar 项目标识 | 自动从 pom.xml 读取 |
| `JACOCO_INCLUDES` | JaCoCo 覆盖范围 | `**/*.class` |
| `JACOCO_EXCLUDES` | JaCoCo 排除 | 配置类/DTO 等 |

---

## 四、覆盖率三阶段监控

### 阶段 1：本地开发（随时查看）

```bash
# IDEA 配置 JaCoCo 覆盖率引擎
# 右键测试类 → Run with Coverage
# 查看 Coverage 面板中的行覆盖率

# 或命令行方式
mvn test jacoco:report
# 报告位置: target/site/jacoco/index.html
```

### 阶段 2：代码提交（分支全量）

推送到远端后 CI 自动执行，结果上传到 SonarQube：
- Sonar 地址：CI 日志中 `ANALYSIS SUCCESSFUL, you can browse` 后的 URL
- 查看整个分支的全量覆盖率

### 阶段 3：MR 提交（增量卡控）

发起 MR 时自动触发 Pipeline：
- MiCR 机器人评论：覆盖率结果 + 是否达标
- Sonar 增量分析：仅统计变更代码的覆盖率
- **不达标 → MR 无法合并**

---

## 五、MR 无法合并常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| 覆盖率不达标 | 新增代码未写单测 | 补充单测，确保变更代码行覆盖率 ≥ 60% |
| 单测通过率不达标 | 存量/增量单测失败 | 修复失败的测试用例（100% 通过率要求） |
| Sonar 门禁不通过 | bug/漏洞/坏味道超标 | 按 Sonar 提示修复代码问题 |
| Pipeline 失败 | 编译错误/环境问题 | 查看 CI 日志定位具体错误 |
| MR 未关联需求 | 未绑定 Meego | 在 MR 描述中关联 Meego 任务 |
| 测试依赖外部环境 | 未 Mock 导致不稳定 | 用 Mock/H2/embedded-redis 替代 |

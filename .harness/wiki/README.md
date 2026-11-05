# Wiki：知识库（L3 按需查阅）

> Agent 遇到具体问题时自主查阅，不主动加载。

## 文件索引

| 文件 | 内容 | 何时查阅 |
|------|------|---------|
| `mify-setup.md` | Mify 网关接入指南（API Key/Claude Code/Cline/Codex 配置）+ 可用模型速查 | **新人首次配置 AI 工具时** |
| `dev-environment.md` | 研发环境搭建（Java/Node/Git/Docker/代理）+ 系统请求链路 + 自测环境 + 常见问题 | Phase 3 本地构建 + Phase 9 部署验证 + 新人环境搭建时 |
| `tech-knowledge.md` | 外部依赖（BPM/SAP/TSP/HROD/Workbench/Feishu）+ 项目架构 + 启动方式 + 踩坑速查 | Phase 3 编码涉及外部系统或架构问题时 |
| `lessons-learned.md` | AI反幻觉 + 前端调试 + 推送规则 + Agent治理 + E2E验证 + 嵌入实践 + Claude Code 架构启示 + 研发基线 | Phase 4/6 评审时 + 回退修复时 + Scheduler 优化时 |
| `methodology.md` | 知识流动 + PRD规范 + 设计方法论 + 上下文工程 + 复盘模板 + 探索优先 | Phase 1 需求分析 + Phase 10 复盘时 |

## 按阶段查阅指引

| 阶段 | 推荐查阅 |
|------|---------|
| Phase 1 需求分析 | `methodology.md` §探索优先 + §PRD规范 |
| Phase 2 需求评审 | `methodology.md` §设计方法论 |
| Phase 3 编码实现 | `tech-knowledge.md`（按需）+ `dev-environment.md`（本地构建问题时） |
| Phase 4 编码评审 | `lessons-learned.md` §反幻觉 + §推送规则 |
| Phase 5 单测编写 | `lessons-learned.md` §前端调试（如涉及前端） |
| Phase 9 部署验证 | `lessons-learned.md` §E2E验证 + `dev-environment.md` §自测环境 + `tech-knowledge.md` §启动方式 |
| Phase 10 复盘 | `methodology.md` §复盘模板 + §知识流动 |

## 外部知识源路由（更详细的原始文档位置）

| 知识类型 | 原始位置 |
|---------|---------|
| EAM 术语表 | `docs/systems/eam/domain/glossary.md` |
| EAM 本体/实体关系 | `docs/systems/eam/domain/ontology.md` |
| 原子 PRD | `docs/systems/eam/domain/Atom PRD/` |
| 配置注册表 | `docs/systems/eam/domain/Atom PRD/config-registry.md` |
| 业务流程图 | `docs/systems/eam/domain/flowcharts/` |
| 前端路由 | `docs/reference/routes.md` |
| 前端组件 | `docs/reference/components.md` |
| QA 知识库 | `docs/systems/eam/domain/Atom PRD/qa.md` |

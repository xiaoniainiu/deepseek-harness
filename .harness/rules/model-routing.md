# 多模型路由策略：按阶段分配模型

> 不同阶段使用不同模型，降低 token 成本 + 多视角交叉检查。
> 前提：通过 Mify 网关统一接入，见 `wiki/mify-setup.md`。

---

## 设计原则

1. **强推理 → 高成本模型**：代码实现、架构设计需要精确推理
2. **交叉检查 → 不同模型家族**：让 GPT 检查 Claude 写的代码，Claude 检查 GPT 的方案
3. **简单执行 → 低成本模型**：格式化、文档生成、信息提取不需要顶级推理
4. **私部模型优先**：xiaomi owned 模型免费，优先用于非关键任务

## 阶段 → 模型映射

| 阶段 | 推荐模型 | Provider | 理由 |
|------|---------|----------|------|
| **Phase 1 需求分析** | DeepSeek-R1 / glm-4-flash | xiaomi / zhipuai | 信息提取和结构化，不需要强推理 |
| **Phase 2 需求评审** | gpt-4.1 / gpt-5 | azure_openai | 全局视角审查需求完整性 |
| **Phase 2.3 技术设计** | gpt-5 / gpt-4.1 | azure_openai | 架构设计需要强全局推理 |
| **Phase 3 编码实现** | claude-opus-4-6 | ppio | 最强代码生成 + 工具调用 |
| **Phase 4 编码评审** | gpt-5 / gpt-4.1 | azure_openai | **交叉检查**：用 GPT 审 Claude 的代码 |
| **Phase 5 单测编写** | claude-opus-4-6 | ppio | 测试代码也需要精确推理 |
| **Phase 6 单测评审** | gpt-4.1 / gemini-2.5-pro | azure_openai / vertex_ai | 交叉检查测试覆盖度 |
| **Phase 7 代码推送** | glm-4-flash / mimo-v2-flash | xiaomi | 简单 git 操作，无需强推理 |
| **Phase 8 CI 验证** | DeepSeek-v3.1 | xiaomi | 解析 CI 输出，判断 pass/fail |
| **Phase 9 部署验证** | gpt-4.1 | azure_openai | E2E 验证需要理解业务逻辑 |
| **Phase 10 复盘** | glm-4-flash / DeepSeek-R1 | xiaomi / zhipuai | 文档整理，低成本 |

## Scheduler 模型配置

`plan.json` 中每个 task 可指定 `model` 字段，覆盖默认值：

```json
{
  "tasks": [
    {
      "id": "T-001",
      "title": "实现 VIN 校验",
      "type": "backend",
      "model": "ppio/pa/claude-opus-4-6"
    },
    {
      "id": "T-002",
      "title": "编码评审",
      "type": "review",
      "model": "azure_openai/gpt-4.1"
    }
  ]
}
```

### 环境变量配置

```bash
# 默认模型（不指定时使用）
export HARNESS_DEFAULT_MODEL="ppio/pa/claude-opus-4-6"

# 按阶段覆盖
export HARNESS_MODEL_DESIGN="azure_openai/gpt-5"        # Phase 2/2.3
export HARNESS_MODEL_IMPLEMENT="ppio/pa/claude-opus-4-6" # Phase 3/5
export HARNESS_MODEL_REVIEW="azure_openai/gpt-4.1"      # Phase 4/6
export HARNESS_MODEL_SIMPLE="deepseek-v3.1"              # Phase 1/7/8/10

# Mify 网关
export MIFY_API_KEY="sk-你的KEY"
export MIFY_BASE_URL="http://model.mify.ai.srv/v1"
```

## 交叉检查矩阵

> 核心理念：实现者和审查者必须来自不同模型家族。

| 实现者 | 审查者 | 为什么 |
|--------|--------|--------|
| Claude (Anthropic) | GPT (OpenAI) | 不同训练数据 → 不同盲点 |
| GPT (设计方案) | Claude (编码实现) | GPT 偏全局，Claude 偏精确 |
| 任意模型 | Gemini (长文档审查) | Gemini 超长上下文，适合审查大文档 |
| 任意模型 | GLM / DeepSeek | 中文语境下的逻辑检查补充 |

### 检查什么

| 审查视角 | 适合的模型 | 原因 |
|---------|-----------|------|
| 代码正确性 | Claude Opus | 最精确的代码理解 |
| 架构合理性 | GPT-5 / o3 | 强全局推理 |
| 逻辑一致性 | Gemini 2.5 Pro | 超长上下文，看到全貌 |
| 中文文档质量 | GLM-4.7 / DeepSeek | 中文原生训练 |
| 安全漏洞 | GPT + Claude 双审 | 安全问题需要多视角 |

## 成本优化策略

### 模型降级规则

| 条件 | 动作 |
|------|------|
| 简单的 grep/sed/git 操作 | 降级到 glm-4-flash 或 mimo-v2-flash |
| 格式化/文档生成 | 降级到 DeepSeek-v3.1 |
| 重试第 2+ 次（同任务） | 保持原模型（不降级，需要同等推理力） |
| 评审未发现问题（连续 3 个） | 可考虑降级评审模型 |

### 私部模型优先

xiaomi owned 模型在 Mify 上免费，优先用于：
- 信息提取（Phase 1 需求分析）
- 文档整理（Phase 10 复盘）
- CI 输出解析（Phase 8）
- 简单 git 操作（Phase 7）

```
成本公式：
  高成本阶段（Opus/GPT-5）：Phase 2.3 + Phase 3 + Phase 4 + Phase 5
  低成本阶段（私部/Flash）：Phase 1 + Phase 7 + Phase 8 + Phase 10
  中等成本（GPT-4.1）：Phase 2 + Phase 6 + Phase 9
```

## 失败升级路由（GSD 模式）

> 借鉴 GSD 框架的 Dynamic Routing：先用低成本模型尝试，失败后自动升级到高能力模型。

```
失败升级链：light → standard → heavy
                ↑        ↑
            首次尝试  首次失败后升级
```

| 层级 | 模型 | 触发条件 |
|------|------|---------|
| **light** | DeepSeek-v3.1 / glm-4-flash / mimo-v2-flash | 默认首次尝试（简单任务） |
| **standard** | gpt-4.1 / claude-sonnet-4-6 | light 层失败后升级 |
| **heavy** | claude-opus-4-6 / gpt-5 / o3 | standard 层失败后升级 |

**规则**：
- `max_escalations = 2`（最多升级 2 次，避免无限升级）
- 升级时携带前一次的失败上下文（`badAttempts` + `evidence`）
- 升级不重置 `sameFailureCount`（断路器仍然生效）
- 编码实现（Phase 3/5）直接从 standard 开始（不从 light 起步）

**Scheduler 配置**（plan.json 任务级别）：

```json
{
  "id": "T-001",
  "title": "格式化文档",
  "modelTier": "light",
  "maxEscalations": 2
}
```

---

## 与 Harness Scheduler 集成

Scheduler 通过 `tool-adapters.sh` 的 `select_model_for_task()` 函数自动选择模型。
优先级：task.model > 环境变量(按阶段) > HARNESS_DEFAULT_MODEL。

手动模式下，Agent 在 Phase 切换时参考本文件选择模型。不强制（用户可覆盖），但建议遵循交叉检查原则。

## 任务粒度路由（来源：Superpowers subagent-driven-development）

在 plan.json 的每个 task 级别，根据 `complexity` 字段自动选择模型层级：

| complexity | 特征 | 默认模型层级 | 示例 |
|-----------|------|------------|------|
| `simple` | 1-2 文件，明确规格，纯机械实现 | light | 加一个字段、改一个配置 |
| `moderate` | 3-5 文件，需要理解上下文 | standard | 新增 API 端点、修复业务 bug |
| `complex` | 5+ 文件，需要架构判断或跨模块协调 | heavy | 流程重构、跨服务接口变更 |

Scheduler 在 `select_model_for_task()` 中组合 `complexity` 和 `type` 两个维度：

```
模型 = task.model           # 显式指定，最高优先级
     || route(task.type)    # 按任务类型（review→GPT, backend→Claude）
     || tier(task.complexity) # 按复杂度（simple→light, complex→heavy）
     || HARNESS_DEFAULT_MODEL
```

**成本预估**（以 10 任务为例）：

```
无路由（全用 Opus）：  10 × Opus   = 10 单位成本
有路由（3 simple + 5 moderate + 2 complex）：
  3 × light + 5 × standard + 2 × heavy ≈ 5.5 单位成本（节省 ~45%）
```

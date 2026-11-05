# Mify 大模型推理网关接入指南

> 新用户环境搭建时查阅。来源：Cline+Mify 文档 + 模型清单。
> 参考文档：https://mi.feishu.cn/wiki/NVyCw8WxciLpVWkhsKxc3RuEnNg
> 模型清单：https://mi.feishu.cn/wiki/QAGHwPr4kiD3G1kv3bKcaPjbnSf

---

## 概览

公司统一使用 **Mify 大模型推理网关** 接入所有 LLM，包括 Claude、GPT、DeepSeek、Gemini、GLM 等。

- **网关地址**：`http://model.mify.ai.srv/v1`（OpenAI 兼容）
- **Anthropic 专用**：`http://model.mify.ai.srv/anthropic`（Claude Code 专用）
- **API Key**：个人密钥，从 https://mify.mioffice.cn/gateway 获取
- **模型格式**：`provider/model`（如 `ppio/pa/claude-opus-4-6`）或纯 `model`（需配 header）

> 注意：Mify 平台应用的 key 与"模型推理"的 API key **不互通**，必须在"模型推理"页面单独申请。

---

## Step 1: 获取 API Key

1. 访问 https://mify.mioffice.cn/gateway
2. 在「模型推理」页面创建 API Key
3. 复制 `sk-xxxx` 格式的密钥

## Step 2: 连通性测试

```bash
curl -X POST 'http://model.mify.ai.srv/v1/chat/completions' \
  --header 'api-key: sk-你的KEY' \
  --header 'Content-Type: application/json' \
  --header 'X-Model-Request-Id: test-connectivity' \
  --data '{
    "model": "azure_openai/gpt-4o-mini",
    "messages": [{"role": "user", "content": "你好"}],
    "stream": true
  }'
```

返回流式 JSON 即为成功。

---

## Step 3: 配置 Claude Code

Claude Code 使用 Anthropic 原生协议，需指向 `/anthropic` 路径（非 `/v1`）。

### 方式 A：环境变量（推荐）

```bash
# ~/.zshrc 或 ~/.bashrc
export ANTHROPIC_BASE_URL="http://model.mify.ai.srv/anthropic"
export ANTHROPIC_API_KEY="sk-你的KEY"
```

### 方式 B：apiKeyHelper 脚本

```bash
# ~/.claude/print-api-key.sh
#!/bin/bash
echo "sk-你的KEY"
```

```json
// ~/.claude/settings.json
{
  "apiKeyHelper": "/Users/你的用户名/.claude/print-api-key.sh"
}
```

### 模型选择

Claude Code 支持三个模型槽位（opus/sonnet/haiku），通过环境变量覆盖：

```bash
# 全部使用 Opus（当前推荐配置）
export ANTHROPIC_DEFAULT_OPUS_MODEL="ppio/pa/claude-opus-4-6"
export ANTHROPIC_DEFAULT_SONNET_MODEL="ppio/pa/claude-sonnet-4-6"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="ppio/pa/claude-haiku-4-5-20251001"
```

Mify 网关上可用的 Claude 模型（provider: `ppio`）：

| 模型 ID | 说明 |
|---------|------|
| `ppio/pa/claude-opus-4-6` | 最强推理，适合代码实现 |
| `ppio/pa/claude-sonnet-4-6` | 平衡性能和速度 |
| `ppio/pa/claude-sonnet-4-5-20250929` | 旧版 Sonnet |
| `ppio/pa/claude-haiku-4-5-20251001` | 最快，适合简单任务 |

### 验证配置

```bash
source ~/.zshrc
claude --version    # 确认 claude 命令可用
claude "你好"       # 测试基本对话
```

---

## Step 4: 配置 Cline / Cursor

用于 VS Code / Cursor 的 Cline 插件：

1. 安装 Cline 插件
2. 配置：
   - **API Provider**：OpenAI Compatible
   - **Base URL**：`http://model.mify.ai.srv/v1/`
   - **API Key**：你的 Mify API Key
   - **Model ID**：`provider/model` 格式（如 `azure_openai/gpt-4.1`）

---

## Step 5: 配置 Codex CLI（OpenAI）

```bash
export OPENAI_BASE_URL="http://model.mify.ai.srv/v1"
export OPENAI_API_KEY="sk-你的KEY"

# 测试
codex "你好"
```

---

## 可用模型速查（按用途分类）

> 完整列表通过 `curl https://api.llm.mioffice.cn/v1/models -H 'Authorization: Bearer <KEY>'` 查询。
> 或访问 https://cnbj1-fds.api.xiaomi.net/mify-models/index.html

### 代码实现（需要强推理 + 工具调用）

| 模型 | Provider | 说明 |
|------|----------|------|
| `pa/claude-opus-4-6` | ppio | 最强代码生成，Harness 默认 |
| `pa/claude-sonnet-4-6` | ppio | 平衡选择 |
| `gpt-5-codex` | azure_openai | GPT 代码专用 |
| `gpt-4.1` | azure_openai | 强推理 |

### 技术设计 / 架构评审（需要全局视角 + 长上下文）

| 模型 | Provider | 说明 |
|------|----------|------|
| `gpt-5` | azure_openai | 强全局推理 |
| `gpt-4.1` | azure_openai | 均衡 |
| `gemini-2.5-pro` | vertex_ai | 超长上下文 |
| `o3` | azure_openai | 深度推理 |

### 简单执行 / 格式化 / 文档生成

| 模型 | Provider | 说明 |
|------|----------|------|
| `DeepSeek-R1-0528` | xiaomi（私部） | 免费，推理能力强 |
| `deepseek-v3.1` | xiaomi（私部） | 免费，通用对话 |
| `mimo-v2-flash` | xiaomi（私部） | 免费，快速 |
| `glm-4-flash` | zhipuai | 快速，低成本 |
| `gemini-2.5-flash` | vertex_ai | 快速，大上下文 |
| `gpt-4o-mini` | azure_openai | 低成本通用 |

### 多视角交叉检查

| 模型 | Provider | 适合检查什么 |
|------|----------|------------|
| `glm-4.7` | zhipuai | 中文语境下的逻辑检查 |
| `gemini-2.5-pro` | vertex_ai | 长文档一致性 |
| `gpt-5` | azure_openai | 架构合理性 |
| `pa/claude-opus-4-6` | ppio | 代码正确性 |

---

## 常见问题

| 问题 | 解法 |
|------|------|
| GPT-5 返回错误 | `temperature` 必须设为 1 |
| 401 Unauthorized | 确认使用的是"模型推理"的 key，不是 Mify 平台应用的 key |
| 模型不存在 | 检查 provider/model 格式是否正确，用 `/v1/models` 查询 |
| Claude Code 连不上 | 确认 BASE_URL 是 `/anthropic` 不是 `/v1` |
| 请求超时 | 内网环境需通过 VPN 或在办公网络内 |
| 需要排查问题 | 请求头加 `X-Model-Request-Id: 自定义ID`，联系网关研发定位 |

---

## 模型 ID 格式

两种方式指定模型：

**方式一：model 字段含 provider 前缀**（推荐）
```json
{ "model": "azure_openai/gpt-4.1" }
```

**方式二：model 字段 + header**
```json
// header: X-Model-Provider-Id: azure_openai
{ "model": "gpt-4.1" }
```

---

## 参考链接

- Mify 网关管理后台：https://mify.mioffice.cn/gateway
- 模型列表页面：https://cnbj1-fds.api.xiaomi.net/mify-models/index.html
- 体验中心（测试模型）：https://llm.mioffice.cn/playground
- API 接口文档：https://mi.feishu.cn/wiki/TORBwJoCNizSVIkg3yzcFUwGni7
- Cline 接入文档：https://mi.feishu.cn/wiki/NVyCw8WxciLpVWkhsKxc3RuEnNg

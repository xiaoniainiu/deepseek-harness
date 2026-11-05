# Harness 新手开箱即用指南

> 目标：不懂技术的人，跟着做，30 分钟内完成所有配置，开始用 AI 写代码。
>
> 基础环境详细参考：[研发环境安装指南](https://mi.feishu.cn/docx/DpeUdgxKvom9SVxMZZ6c3Xb0nug)

---

## 两条路径

| 路径 | 适合谁 | 耗时 |
|------|--------|------|
| **自动化**：`harness-init.sh` 交互式向导 | 已有基础环境，只需初始化项目 | 5 分钟 |
| **手动**：按 7 阶段逐步操作 | 全新机器，从零开始 | 30 分钟 |

### 自动化路径（推荐）

```bash
# 进入你的项目目录
cd /path/to/your-project

# 运行 Harness 初始化向导（6 步交互式引导）
/path/to/Harness/scheduler/harness-init.sh .
```

向导会自动完成：
1. **项目检测** — 识别项目名、技术栈（Java/TS/Go/Python）、项目规模
2. **质量命令** — 检测并配置编译、测试、Lint 命令
3. **Agent 角色** — 选择 senior-backend/frontend/fullstack-developer
4. **知识库** — 外部系统依赖、架构模式、关键配置文件、已知坑点
5. **文件生成** — 创建 `.harness/`（config.json + agents + rules + skills + wiki）+ `.claude/settings.json`（hooks + MCP + permissions）
6. **CLAUDE.md 集成** — 自动追加 Harness 加载指令

向导结束后跳到 [开始使用](#开始使用)。

---

## 手动路径：7 个阶段

```
阶段 1: 基础环境        → Java / Maven / Git / Node    （10 分钟）
阶段 2: AI 工具          → Claude Code / Codex CLI       （3 分钟）
阶段 3: Mify API 配置    → 获取 Key + 写环境变量         （5 分钟）
阶段 4: Git 仓库          → SSH Key + clone 项目          （5 分钟）
阶段 5: 外部系统凭据      → 数据库 / Redis / BPM 等      （5 分钟，按需）
阶段 6: 项目配置          → CLAUDE.md / Codex / MCP      （3 分钟）
阶段 7: 验证              → 逐项检查，确认可用            （2 分钟）
```

每个阶段先检测，已有就跳过。**缺什么装什么，不重复劳动。**

---

## 阶段 1: 基础环境

### macOS 一键安装

```bash
# 安装 Homebrew（如果没有）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装全部基础工具
brew install git maven node pnpm
brew install --cask temurin@17    # Java 17

# 或用小米一键脚本
curl -fsSL https://cnbj1-fds.api.xiaomi.net/b2c-info-oa/dev-env-mac/env-setup.sh | bash
```

### Windows 一键安装

下载两个文件放同一目录，执行 `install-dev-env.bat`：
- [环境安装脚本](https://mi.feishu.cn/docx/DpeUdgxKvom9SVxMZZ6c3Xb0nug)（飞书文档中附件）

或手动：
```powershell
winget install Git.Git
winget install Microsoft.OpenJDK.17
winget install Apache.Maven
winget install OpenJS.NodeJS.LTS
npm install -g pnpm
```

### 验证

```bash
java -version      # 应显示 17.x
mvn -version       # 应显示 3.x
git --version      # 应显示 2.x
node -v            # 应显示 18.x 或 20.x
pnpm -v            # 应显示 9.x
```

全部有输出 → 阶段 1 完成 ✅

---

## 阶段 2: AI 工具安装

### Claude Code（主力 AI 编程工具）

```bash
npm install -g @anthropic-ai/claude-code
```

验证：`claude --version`

### Codex CLI（OpenAI 代码审查工具）

```bash
npm install -g @openai/codex
```

验证：`codex --version`

两个都有输出 → 阶段 2 完成 ✅

---

## 阶段 3: Mify API 配置

Mify 是小米内部的 AI 模型网关，Claude Code、Codex CLI、MiMo 都通过它调用模型。

### 模型前缀规则（重要）

> Mify 的模型名**必须加 `owned_by/` 前缀**，否则报 `Not supported model`。

| 工具 | API 端点 | 前缀 | 示例 |
|------|---------|------|------|
| Claude Code | `model.mify.ai.srv/anthropic` | `ppio/pa/` | `ppio/pa/claude-opus-4-6` |
| Codex CLI | `model.mify.ai.srv/v1` (Responses API) | `azure_openai/` | `azure_openai/gpt-5.1-codex` |
| MiMo | `model.mify.ai.srv/v1` (Chat API) | `xiaomi/` | `xiaomi/mimo-v2.5-pro` |

### 第一步：获取 API Key

1. 打开 https://mify.mioffice.cn/gateway
2. 登录米盾
3. 在「**模型推理**」页面创建 API Key（注意：不是"平台应用"的 Key，两者不互通）
4. 记下 Key（类似 `sk-xxxxxxxxxxxxxxxx`）

### 第二步：配置 Claude Code

Claude Code 走 Anthropic 协议，通过环境变量配置：

```bash
cat >> ~/.zshrc << 'EOF'

# === Claude Code（Anthropic 协议）===
export ANTHROPIC_BASE_URL="http://model.mify.ai.srv/anthropic"
export ANTHROPIC_API_KEY="sk-你的key"
export ANTHROPIC_DEFAULT_OPUS_MODEL="ppio/pa/claude-opus-4-6"
export ANTHROPIC_DEFAULT_SONNET_MODEL="ppio/pa/claude-sonnet-4-6"
export ANTHROPIC_DEFAULT_HAIKU_MODEL="ppio/pa/claude-haiku-4-5-20251001"
EOF

source ~/.zshrc
```

> Windows 用户：在系统环境变量中添加上述变量，或写入 `~/.bashrc`。

**可用 Claude 模型**（`ppio/pa/` 前缀，走 Anthropic 端点）：

| 模型 ID | 说明 |
|---------|------|
| `ppio/pa/claude-opus-4-7` | Opus 4.7，最强能力 |
| `ppio/pa/claude-opus-4-6` | Opus 4.6，**Claude Code 默认** |
| `ppio/pa/claude-sonnet-4-6` | Sonnet 4.6，速度/成本平衡 |
| `ppio/pa/claude-haiku-4-5-20251001` | Haiku 4.5，最快最省 |

**验证 Claude Code 配置**：

```bash
# 方法 1：检查环境变量
echo $ANTHROPIC_API_KEY    # 应显示你的 key

# 方法 2：直接测试 API 连通性
curl -s -X POST http://model.mify.ai.srv/anthropic/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"model":"ppio/pa/claude-sonnet-4-6","max_tokens":20,"messages":[{"role":"user","content":"say hi"}]}' \
  | python3 -m json.tool | head -10

# 应返回 JSON 响应，包含 "content" 字段
```

### 第三步：配置 Codex CLI

Codex CLI 走 OpenAI Responses API，通过 `~/.codex/config.toml` 配置：

```bash
mkdir -p ~/.codex
cat > ~/.codex/config.toml << 'EOF'
# Codex CLI 核心配置
model_provider = "mify"
model = "azure_openai/gpt-5.1-codex"
model_reasoning_effort = "high"
sandbox_mode = "danger-full-access"
approval_policy = "never"
instructions = "请你默认使用简体中文进行交流和解释。回答简洁、分点说明。"

# Mify Provider（OpenAI Responses API）
[model_providers.mify]
name = "mify"
base_url = "http://model.mify.ai.srv/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.mify.http_headers]
api-key = "sk-你的key"
EOF
```

**可用 Codex 模型**（`azure_openai/` 前缀，走 Responses 端点）：

| 模型 ID | 说明 |
|---------|------|
| `azure_openai/gpt-5.1-codex` | **推荐默认**，最新 Codex 专用 |
| `azure_openai/gpt-5-codex` | 上一代稳定版 |
| `azure_openai/gpt-5.1-codex-max` | 高配版，复杂审查用 |
| `azure_openai/o3` | 推理模型，架构级咨询用 |
| `azure_openai/gpt-4.1` | 轻量快速 |

**验证 Codex 配置**：

```bash
# 方法 1：版本检查
npx @openai/codex --version    # 应显示版本号

# 方法 2：简单调用测试
npx @openai/codex exec "Reply with exactly: CODEX_OK"
# 应输出 CODEX_OK

# 方法 3：直接测试 API 连通性
curl -s -X POST http://model.mify.ai.srv/v1/responses \
  -H "api-key: sk-你的key" \
  -H "Content-Type: application/json" \
  -d '{"model":"azure_openai/gpt-5.1-codex","input":"say hi","max_output_tokens":20}' \
  | python3 -m json.tool | head -10
```

### 第四步：MiMo 模型配置（可选）

MiMo 是小米自研大模型，走 OpenAI Chat 协议。适合需要本地化或低延迟的场景。

**可用 MiMo 模型**（`xiaomi/` 前缀，走 Chat 端点）：

| 模型 ID | 说明 |
|---------|------|
| `xiaomi/mimo-v2.5-pro` | **推荐**，最新 Pro 版，支持推理 |
| `xiaomi/mimo-v2.5` | 标准版 |
| `xiaomi/mimo-v2-pro` | 上一代 Pro |
| `xiaomi/mimo-v2-flash` | 轻量快速版 |
| `xiaomi/mimo-v2-omni` | 多模态版 |

**验证 MiMo 配置**：

```bash
curl -s -X POST http://model.mify.ai.srv/v1/chat/completions \
  -H "api-key: sk-你的key" \
  -H "Content-Type: application/json" \
  -d '{"model":"xiaomi/mimo-v2.5-pro","messages":[{"role":"user","content":"say hi"}],"max_tokens":20}' \
  | python3 -m json.tool | head -10
```

> MiMo 目前不直接用于 Claude Code 或 Codex CLI，但可在自定义脚本或 Bot 中通过 OpenAI SDK 调用。

### 阶段 3 验证清单

```bash
echo "=== Claude Code ==="
[ -n "$ANTHROPIC_API_KEY" ] && echo "✅ ANTHROPIC_API_KEY" || echo "❌ ANTHROPIC_API_KEY"
[ -n "$ANTHROPIC_BASE_URL" ] && echo "✅ ANTHROPIC_BASE_URL = $ANTHROPIC_BASE_URL" || echo "❌ ANTHROPIC_BASE_URL"

echo "=== Codex CLI ==="
[ -f ~/.codex/config.toml ] && echo "✅ config.toml 存在" || echo "❌ ~/.codex/config.toml 不存在"
grep -q "azure_openai/" ~/.codex/config.toml 2>/dev/null && echo "✅ 模型前缀正确" || echo "❌ 模型缺少 azure_openai/ 前缀"

echo "=== MiMo（可选） ==="
echo "⚠️ MiMo 通过代码调用，无需额外配置"
```

全部 ✅ → 阶段 3 完成 ✅

---

## 阶段 4: Git 仓库配置

### 第一步：配置 Git 用户信息

```bash
git config --global user.name "你的姓名拼音"
git config --global user.email "你的邮箱前缀@xiaomi.com"
```

### 第二步：生成 SSH Key（如果没有）

```bash
# 检查是否已有
ls ~/.ssh/id_ed25519.pub 2>/dev/null && echo "已有SSH Key" || ssh-keygen -t ed25519 -C "你的邮箱@xiaomi.com"

# 复制公钥
cat ~/.ssh/id_ed25519.pub
```

将输出的公钥添加到 GitLab：
1. 打开 https://git.n.xiaomi.com/-/user_settings/ssh_keys
2. 粘贴公钥 → Add key

### 第三步：clone 项目

```bash
# 测试 SSH 连接
ssh -T git@git.n.xiaomi.com
```

#### EAM 项目仓库清单

| 仓库 | 地址 | 说明 | JDK |
|------|------|------|-----|
| **asset-eam-web** | `git@git.n.xiaomi.com:mit/oa/asset/asset-eam-web.git` | EAM 前端主仓（React + HiUI） | Node 20 |
| **eam-v2** | `git@git.n.xiaomi.com:mit/oa/asset/eam-v2.git` | 盘点引擎 v2（Spring Boot 3） | JDK 17 |
| **asset-account-be** | `git@git.n.xiaomi.com:mit/oa/asset/eam-ng/asset-account-be.git` | 台账服务后端 | JDK 8 |
| **asset-entry-be** | `git@git.n.xiaomi.com:mit/oa/asset/eam-ng/asset-entry-be.git` | 验收服务后端 | JDK 8 |
| **asset-stock-be** | `git@git.n.xiaomi.com:mit/oa/asset/eam-ng/asset-stock-be.git` | 库存服务后端 | JDK 8 |
| **eap** | `git@git.n.xiaomi.com:mit/oa/asset/eap.git` | EAP 通用任务平台 | JDK 8 |
| **mi-assets-tt** | `git@git.n.xiaomi.com:mit/oa/asset/mi-assets-tt.git` | 飞书小程序（员工端） | Node |
| **pm-context** | `git@git.n.xiaomi.com:iib/pm-context.git` | 产品知识库 + Harness | — |
| **harness** | `git@git.n.xiaomi.com:mit/oa/asset/harness.git` | Harness 框架独立仓库 | — |

```bash
# 按需 clone（不需要全部）
mkdir -p ~/projects/eam && cd ~/projects/eam

# 前端
git clone git@git.n.xiaomi.com:mit/oa/asset/asset-eam-web.git

# 后端（按你负责的模块选择）
git clone git@git.n.xiaomi.com:mit/oa/asset/eam-v2.git              # 盘点引擎
git clone git@git.n.xiaomi.com:mit/oa/asset/eam-ng/asset-account-be.git  # 台账
git clone git@git.n.xiaomi.com:mit/oa/asset/eam-ng/asset-entry-be.git    # 验收

# 产品知识库
git clone git@git.n.xiaomi.com:iib/pm-context.git
```

### 第四步：Maven 私服配置

> 参考文档：https://mi.feishu.cn/wiki/YfY8dLVI6oEFGYxOhTtcAPrpnk2

#### 获取 Artifactory 凭据

1. 打开 https://pkgs.d.xiaomi.net/
2. 登录米盾
3. 右上角头像 → **Set Me Up** → Maven → 生成加密密码
4. 记下 **用户名**（邮箱前缀，如 `tankunshan`）和 **加密密码**（`AKCp...` 格式）

#### 写入 settings.xml

```bash
mkdir -p ~/.m2
cat > ~/.m2/settings.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<settings xmlns="http://maven.apache.org/SETTINGS/1.1.0"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.1.0 http://maven.apache.org/xsd/settings-1.1.0.xsd">
  <servers>
    <server>
      <id>central</id>
      <username>你的用户名</username>
      <password>你的加密密码</password>
    </server>
    <server>
      <id>snapshots</id>
      <username>你的用户名</username>
      <password>你的加密密码</password>
    </server>
    <server>
      <id>miremote</id>
      <username>你的用户名</username>
      <password>你的加密密码</password>
    </server>
  </servers>
  <profiles>
    <profile>
      <id>artifactory</id>
      <repositories>
        <repository>
          <id>central</id>
          <name>maven-release-virtual</name>
          <url>https://pkgs.d.xiaomi.net:443/artifactory/maven-release-virtual</url>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
        <repository>
          <id>miremote</id>
          <name>maven-remote-virtual</name>
          <url>https://pkgs.d.xiaomi.net/artifactory/maven-remote-virtual</url>
          <snapshots><enabled>false</enabled></snapshots>
        </repository>
        <repository>
          <id>snapshots</id>
          <name>maven-snapshot-virtual</name>
          <url>https://pkgs.d.xiaomi.net:443/artifactory/maven-snapshot-virtual</url>
          <snapshots />
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <id>central</id>
          <url>https://pkgs.d.xiaomi.net:443/artifactory/maven-release-virtual</url>
          <snapshots><enabled>false</enabled></snapshots>
        </pluginRepository>
        <pluginRepository>
          <id>snapshots</id>
          <url>https://pkgs.d.xiaomi.net:443/artifactory/maven-snapshot-virtual</url>
          <snapshots />
        </pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>artifactory</activeProfile>
  </activeProfiles>
</settings>
EOF
```

> 三个 server（`central` / `snapshots` / `miremote`）用**相同的用户名和密码**。

#### npm/pnpm 镜像（前端开发需要）

```bash
npm config set registry "https://pkgs.d.xiaomi.net/artifactory/api/npm/mi-npm/"
```

### 验证

```bash
echo "=== Git ==="
ssh -T git@git.n.xiaomi.com 2>&1 | head -1    # 应显示 Welcome to GitLab
git remote -v 2>/dev/null | head -1            # 应显示仓库地址

echo "=== Maven ==="
[ -f ~/.m2/settings.xml ] && echo "✅ settings.xml 存在" || echo "❌ ~/.m2/settings.xml 不存在"
grep -q "pkgs.d.xiaomi.net" ~/.m2/settings.xml 2>/dev/null && echo "✅ Artifactory 已配置" || echo "❌ Artifactory 未配置"
grep -q "你的用户名" ~/.m2/settings.xml 2>/dev/null && echo "⚠️ 用户名还是占位符" || echo "✅ 用户名已填写"

echo "=== npm ==="
npm config get registry | grep -q "xiaomi" && echo "✅ npm 小米源" || echo "⚠️ npm 非小米源（可选）"
```

→ 阶段 4 完成 ✅

---

## 阶段 5: 外部系统凭据

> 在项目根目录创建 `.env.local`（**不要提交到 Git**）。以 `inventory-engine-v2` 为例，其他后端服务类似。

### 5.1 数据库与缓存

| 环境 | MySQL Host | Port | Database | User | Redis Host | Redis Port |
|------|-----------|------|----------|------|-----------|-----------|
| **测试** | `ksc.test.mysql01.b2c.srv` | `3304` | `asset` | `info_asset_wn` | `wcc.cache01.test.b2c.srv` | `22122` |
| **生产** | `asset.mysql01.b2c.srv` | `3438` | `asset` | `kc_asset_wn` | `cache01.b2c.srv` | `5100` |
| **生产只读** | `asset.mysql01.b2c.srv` | `3439` | `asset` | `asset_rn` | — | — |

> **密码获取方式**：找团队 Leader 或 DBA。生产环境密码通过 KeyCenter 加密（`kc-sid: oa-asset.g`），本地开发用测试环境明文。

### 5.2 外部系统集成

| 系统 | 环境变量 | 用途 | 获取方式 |
|------|---------|------|---------|
| **BPM 审批** | `BPM_APP_ID` / `BPM_APP_SECRET` | 资产处置、调拨等审批流 | 找 BPM 平台管理员 |
| **HROD 人力** | `HROD_APP_ID` / `HROD_APP_SECRET` | 员工信息、组织架构查询 | 找 HROD 接口管理员 |
| **飞书** | `FEISHU_APP_ID` / `FEISHU_APP_SECRET` | 消息通知、机器人推送 | [飞书开放平台](https://open.feishu.cn) 创建应用 |
| **工作台** | `WORKBENCH_APP_ID` / `WORKBENCH_APP_SECRET` | EE 工作台代办、消息 | 找工作台管理员 |

### 5.3 完整 `.env.local` 模板

```bash
# 从模板创建（inventory-engine-v2 已提供模板）
cp .env.local.template .env.local

# 或手动创建完整配置：
cat > .env.local << 'EOF'
# ========== 数据库（必填）==========
MYSQL_HOST=ksc.test.mysql01.b2c.srv:3304
MYSQL_USER=info_asset_wn
MYSQL_PASSWORD=找DBA获取

# ========== 缓存（必填）==========
REDIS_HOST=wcc.cache01.test.b2c.srv
REDIS_PORT=22122
REDIS_PASSWORD=找DBA获取

# ========== BPM 审批（按需）==========
BPM_ENABLED=true
BPM_APP_ID=
BPM_APP_SECRET=

# ========== HROD 人力（按需）==========
HROD_URL=https://api.hrod.mioffice.cn/hapi/HR_C
HROD_APP_ID=
HROD_APP_SECRET=

# ========== 飞书通知（按需）==========
FEISHU_ENABLED=false
FEISHU_APP_ID=
FEISHU_APP_SECRET=

# ========== 工作台（按需）==========
WORKBENCH_ENABLED=true
WORKBENCH_URL=http://workbench-ee.test.mioffice.cn/workbench
WORKBENCH_APP_ID=
WORKBENCH_APP_SECRET=

# ========== CAS 登录 Token（E2E 验证用）==========
# 从测试环境浏览器 DevTools → Application → Cookies 复制 _aegis_cas 完整值
TEST_TOKEN=''
EOF
```

> **哪些必填？**
> - `MYSQL_*` + `REDIS_*`：**必填**，否则服务启不起来
> - `BPM_*` / `HROD_*` / `FEISHU_*` / `WORKBENCH_*`：**按需**，本地开发不涉及审批/通知时设 `*_ENABLED=false` 即可跳过
> - `TEST_TOKEN`：仅 E2E 自动化验证需要

### 5.4 其他后端服务的差异

| 服务 | 与 eam-v2 的差异 |
|------|-----------------|
| **asset-account-be** | JDK 8；密码走 KeyCenter（`password@kc-sid: oa-asset.g`）；无 `.env.local`，配置在 `application-dev.yml` |
| **asset-entry-be** | 同 account-be 模式 |
| **asset-stock-be** | 同 account-be 模式 |
| **eap** | 独立数据库 `start_test`（port `3212`）；有自己的 Redis |

### 验证

```bash
# 检查 .env.local 关键字段
echo "=== .env.local 检查 ==="
[ -f .env.local ] && echo "✅ .env.local 存在" || echo "❌ .env.local 不存在"
grep -q "MYSQL_HOST" .env.local 2>/dev/null && echo "✅ MYSQL_HOST 已配置" || echo "❌ MYSQL_HOST 缺失"
grep -q "REDIS_HOST" .env.local 2>/dev/null && echo "✅ REDIS_HOST 已配置" || echo "❌ REDIS_HOST 缺失"
grep -q "MYSQL_PASSWORD=找\|MYSQL_PASSWORD=$" .env.local 2>/dev/null && echo "⚠️ MYSQL_PASSWORD 还是占位符" || echo "✅ MYSQL_PASSWORD 已填写"
grep -q "REDIS_PASSWORD=找\|REDIS_PASSWORD=$" .env.local 2>/dev/null && echo "⚠️ REDIS_PASSWORD 还是占位符" || echo "✅ REDIS_PASSWORD 已填写"
```

→ 阶段 5 完成 ✅

---

## 阶段 6: 项目配置

### 方式一：自动化向导（推荐）

```bash
/path/to/Harness/scheduler/harness-init.sh .
```

一条命令完成以下全部配置。

### 方式二：手动配置

#### Claude Code 配置

在项目根目录创建 `.claude/settings.json`：

```bash
mkdir -p .claude
cat > .claude/settings.json << 'JSONEOF'
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(mvn *)",
      "Bash(pnpm *)",
      "Bash(node *)",
      "Bash(curl *)",
      "Bash(cat *)",
      "Bash(ls *)",
      "Bash(grep *)",
      "Bash(find *)",
      "Read",
      "Edit",
      "Write"
    ]
  }
}
JSONEOF
```

#### 安装 Chrome DevTools MCP（Phase 9 E2E 验证必需）

Chrome DevTools MCP 让 Claude Code 直接操控浏览器：截图、点击元素、填写表单、检查网络请求——Phase 9 部署验证的核心工具。

**方式一：通过 `harness-init.sh` 自动配置（推荐）**

`harness-init.sh` 会自动在 `.claude/settings.json` 中写入 MCP 配置：

```json
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["@anthropic-ai/chrome-devtools-mcp@latest"]
    }
  }
}
```

重启 Claude Code 后自动生效。

**方式二：手动配置**

1. 确认 Node.js 已安装（阶段 1）

2. 在项目 `.claude/settings.json` 中添加 `mcpServers` 配置：

```bash
# 如果 settings.json 已存在，手动添加 mcpServers 段
# 如果不存在，创建完整文件：
mkdir -p .claude
cat > .claude/settings.json << 'JSONEOF'
{
  "mcpServers": {
    "chrome-devtools": {
      "command": "npx",
      "args": ["@anthropic-ai/chrome-devtools-mcp@latest"]
    }
  },
  "permissions": {
    "allow": [
      "mcp__chrome-devtools__*"
    ]
  }
}
JSONEOF
```

3. 重启 Claude Code

**验证 MCP 已加载**

在 Claude Code 中执行：
```
# 方法 1：列出可用页面
使用 mcp__chrome-devtools__list_pages

# 方法 2：截取当前页面
使用 mcp__chrome-devtools__take_screenshot
```

有响应（即使报 "no pages" 也算成功）→ MCP 已就绪。

**典型 E2E 验证流程（Phase 9）**

```
1. 启动开发服务器 → 打开 Chrome 访问页面
2. mcp__chrome-devtools__take_snapshot    → 获取页面元素树
3. mcp__chrome-devtools__click uid=xxx    → 模拟用户操作
4. mcp__chrome-devtools__fill uid=xxx     → 填写表单
5. mcp__chrome-devtools__take_screenshot  → 截图留证
6. mcp__chrome-devtools__list_console_messages → 检查控制台错误
7. mcp__chrome-devtools__list_network_requests → 检查 API 调用
```

> **CAS 登录场景**：如果系统需要 CAS/SSO 认证，先在 Chrome 手动登录，然后 MCP 会复用已有 session。详见 `memory/chrome-devtools-e2e.md`。

#### 安装飞书 CLI（Phase 2 评审文档创建）

飞书 CLI 让 Claude Code 直接创建飞书文档、管理知识库权限——Phase 2 评审推送的核心工具。

**安装**

```bash
npm install -g @mi/feishu@latest --registry https://pkgs.d.xiaomi.net/artifactory/api/npm/mi-npm/
```

**认证**

```bash
feishu auth login    # 浏览器 OAuth 登录
feishu auth status   # 检查 token 状态
```

**验证**

```bash
feishu --version     # 应 ≥ 1.3.0
feishu search "测试"  # 应返回搜索结果
```

有输出 → 飞书 CLI 就绪。

#### 配置飞书消息 MCP（Phase 2 评审消息推群）

feishu-im-mcp 让 Claude Code 直接向飞书群发送消息——Phase 2 评审推群、Phase 10 完成通知的核心工具。

**前置**：需要飞书开放平台的 App ID 和 App Secret（向管理员获取）。

**配置**

`harness-init.sh` 会自动在 `.claude/settings.json` 中写入 MCP 配置：

```json
{
  "mcpServers": {
    "feishu-im-mcp": {
      "command": "node",
      "args": ["~/feishu-im-mcp/index.js"]
    }
  }
}
```

确认 `~/feishu-im-mcp/.env` 已配置：

```bash
cat ~/feishu-im-mcp/.env
# 应包含：
# FEISHU_APP_ID=cli_xxx
# FEISHU_APP_SECRET=xxx
```

**验证**

在 Claude Code 中执行（重启后）：
```
使用 mcp__feishu-im-mcp__get-chat-list
```

返回群列表 → 飞书消息 MCP 就绪。

**将 Bot 拉入目标群**

评审消息需要发到研发群，Bot 必须先在群里：
1. 在飞书群设置 → 群机器人 → 添加机器人
2. 搜索 App ID 对应的 Bot 名称
3. 添加后用 `get-chat-list` 确认群出现在列表中

#### CLAUDE.md（如果项目还没有）

确认项目根目录有 `CLAUDE.md`，里面包含 Harness 加载指令。没有的话用 init 向导生成：

```bash
/path/to/Harness/scheduler/harness-init.sh .
```

### 验证

```bash
ls .claude/settings.json    # 应存在
ls CLAUDE.md                # 应存在
```

→ 阶段 6 完成 ✅

---

## 阶段 6.5: 路由转发配置（前端联调必需）

> 前端开发需要将 API 请求转发到测试环境或本地后端。EAM 使用 **Whistle (w2) 代理** 方案。

### 为什么需要代理？

EAM 测试环境使用 CAS SSO 登录，Cookie 绑定在 `mieam-asset.test.mioffice.cn` 域名。直接用 `localhost` 访问无法携带 SSO Cookie，所有 API 返回 302 重定向到登录页。w2 代理让浏览器认为自己在访问真实域名。

### 架构

```
Chrome (代理模式)
  │  访问 https://mieam-asset.test.mioffice.cn/
  │
  ▼
w2 (Whistle) 代理 (:8899)
  │  拦截 HTTPS，将域名流量转发到本地
  │
  ▼
webpack-dev-server (:3006)
  │  热更新前端代码
  │
  ├── /api/inventory/* ──→ inventory-engine-v2 (:8080)  ← 本地后端
  ├── /api/configs/*   ──→ mieam-asset.test.mioffice.cn  ← 测试环境
  └── /api/*           ──→ mieam-asset.test.mioffice.cn  ← 测试环境
```

### 第一步：安装 Whistle

```bash
npm install -g whistle

# 启动
w2 start

# 安装 HTTPS 根证书
curl -o /tmp/rootCA.crt http://127.0.0.1:8899/cgi-bin/rootca
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain /tmp/rootCA.crt
```

### 第二步：配置代理规则

打开 `http://127.0.0.1:8899/`，在 Rules 中添加：

```
mieam-asset.test.mioffice.cn localhost:3006
```

确保 Properties 中 `interceptHttpsConnects` 为 `true`。

### 第三步：启动前端

```bash
cd asset-eam-web    # 前端仓库根目录

# LOCAL_API=inventory → /api/inventory/* 路由到 localhost:8080
# PORT=3006 → 与 w2 规则匹配
USE_API=proxy LOCAL_API=inventory PORT=3006 pnpm start
```

启动日志应包含：
```
[HPM] Proxy created: /api/inventory -> http://localhost:8080
[HPM] Proxy created: /api -> https://mieam-asset.test.mioffice.cn/
```

### 第四步：启动本地后端（按需）

```bash
cd inventory-engine-v2
bash run-local.sh    # 启动在 :8080，context-path: /inventory
```

> 如果只改前端不改后端，可以跳过这步——前端直接代理到测试环境后端。

### 第五步：用隔离 Chrome 访问

```bash
open -a "Google Chrome" "https://mieam-asset.test.mioffice.cn/" \
  --args \
  --user-data-dir="$HOME/.chrome-profiles/eam-dev" \
  --proxy-server=http://127.0.0.1:8899 \
  --no-first-run \
  --disable-features=HttpsUpgrades
```

首次会跳转 CAS 登录页（`castest.mioffice.cn`），正常登录即可。

### 端口速查

| 服务 | 端口 | 说明 |
|------|------|------|
| w2 代理 | `8899` | Whistle HTTP/HTTPS 代理 |
| webpack-dev-server | `3006` | 前端热更新 |
| inventory-engine-v2 | `8080` | 盘点后端（`/inventory`） |
| asset-account-be | `9091` | 台账后端 |
| asset-entry-be | `9092` | 验收后端 |
| 测试环境 | `443` | `mieam-asset.test.mioffice.cn` |

### 验证

```bash
echo "=== 路由转发 ==="
w2 status 2>/dev/null | head -1 || echo "❌ w2 未安装（npm install -g whistle）"
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8899/ 2>/dev/null | grep -q "200" && echo "✅ w2 运行中 (:8899)" || echo "❌ w2 未启动（w2 start）"
```

→ 阶段 6.5 完成 ✅

> 完整的前端联调指南见 `docs/systems/eam/knowledge/how-to/local-dev-with-w2-proxy.md`。

---

## 阶段 7: 验证

### 一键检查脚本

```bash
echo "=== 基础环境 ==="
java -version 2>&1 | head -1
mvn -version 2>&1 | head -1
git --version
node -v
pnpm -v

echo "=== AI 工具 ==="
claude --version 2>/dev/null || echo "❌ Claude Code 未安装"
codex --version 2>/dev/null || echo "❌ Codex CLI 未安装"

echo "=== API 配置 ==="
[ -n "$ANTHROPIC_API_KEY" ] && echo "✅ ANTHROPIC_API_KEY 已配置" || echo "❌ ANTHROPIC_API_KEY 未配置"
[ -n "$ANTHROPIC_BASE_URL" ] && echo "✅ ANTHROPIC_BASE_URL = $ANTHROPIC_BASE_URL" || echo "❌ ANTHROPIC_BASE_URL 未配置"
[ -f ~/.codex/config.toml ] && echo "✅ Codex config.toml 存在" || echo "❌ ~/.codex/config.toml 不存在"
grep -q "azure_openai/" ~/.codex/config.toml 2>/dev/null && echo "✅ Codex 模型前缀正确 (azure_openai/)" || echo "❌ Codex 模型缺少 azure_openai/ 前缀"

echo "=== Git ==="
git remote -v 2>/dev/null | head -1 || echo "❌ 不在 Git 仓库中"
ssh -T git@git.n.xiaomi.com 2>&1 | head -1

echo "=== 项目配置 ==="
[ -f .claude/settings.json ] && echo "✅ Claude settings 存在" || echo "❌ .claude/settings.json 不存在"
[ -f CLAUDE.md ] && echo "✅ CLAUDE.md 存在" || echo "❌ CLAUDE.md 不存在"
[ -f .env.local ] && echo "✅ .env.local 存在" || echo "⚠️ .env.local 不存在（如果不需要数据库可忽略）"

echo "=== Harness ==="
[ -d .harness ] && echo "✅ .harness/ 已初始化" || echo "❌ .harness/ 不存在（运行 harness-init.sh）"
[ -f .harness/config.json ] && echo "✅ config.json 存在" || echo "❌ config.json 不存在"

echo "=== Chrome DevTools MCP ==="
if [ -f .claude/settings.json ] && grep -q "chrome-devtools" .claude/settings.json 2>/dev/null; then
  echo "✅ Chrome DevTools MCP 已配置"
else
  echo "⚠️ Chrome DevTools MCP 未配置（Phase 9 E2E 验证需要）"
fi

echo "=== 飞书 CLI ==="
if command -v feishu >/dev/null 2>&1; then
  echo "✅ feishu CLI $(feishu --version 2>&1)"
else
  echo "⚠️ feishu CLI 未安装（Phase 2 评审文档创建需要）"
fi

echo "=== 飞书消息 MCP ==="
if [ -f .claude/settings.json ] && grep -q "feishu-im-mcp" .claude/settings.json 2>/dev/null; then
  echo "✅ feishu-im-mcp 已配置"
else
  echo "⚠️ feishu-im-mcp 未配置（Phase 2 评审推群需要）"
fi
```

全部 ✅ → 开始使用！

---

## Phase 触发集成

> Harness 的 10 阶段流程不是被动指南——每个 Phase 都有**触发条件 → Skill 自动加载 → 质量门禁**三要素。

### 工作原理

```
Session Start（Claude Code 启动）
  ↓
session-start.sh hook 执行
  ↓ 检测 .harness/config.json
  ↓ 检测 changes/*/run-state.json（活跃任务）
  ↓
输出 L1 上下文摘要（项目名/技术栈/当前进度）
  ↓
Agent 识别当前 Phase → 加载对应 L2 Skill
```

### 自动检测机制

`harness-init.sh` 在初始化时自动检测以下环境配置：

| 检测项 | 检测方式 | 作用 |
|--------|---------|------|
| 技术栈 | `pom.xml` → Java; `package.json` → TS/React; `go.mod` → Go; `pyproject.toml` → Python | 决定编译/测试/Lint 命令 |
| 项目规模 | `*.sql` + `*Mapper.xml` 数量; `pom.xml`/`package.json` 模块数 | 评估复杂度 |
| 编译命令 | `pom.xml` → `mvn compile -q`; `tsconfig.json` → `npx tsc --noEmit`; `go.mod` → `go build ./...` | Phase 3 编码验证 |
| 测试命令 | `vitest`/`jest`/`go test`/`pytest` 自动匹配 | Phase 5/6 单测 |
| Lint 命令 | `.eslintrc.*`/`eslint.config.js`/`golangci-lint` | 代码质量检查 |
| 配置文件 | `application.yml`/`.env.example`/`vite.config.ts` 等 | 知识库生成 |

### Phase → Skill 映射

每进入一个 Phase，Harness 自动加载对应的 L2 Skill 文件：

| Phase | 触发条件 | 自动加载 Skill | 角色 |
|-------|---------|---------------|------|
| 0 项目理解 | 首次接触新仓库 | `skills/planning/project-understanding.md` | Coordinator |
| 1 需求分析 | 收到 PRD / Issue | `skills/planning/prd-to-ai-ready.md` | Coordinator |
| 2 需求评审 | Phase 1 产出完成 | `skills/review/multi-perspective.md` [C3] | Coordinator |
| 2.3 技术设计 | 重大变更 | `skills/planning/technical-design.md` | Executor |
| 2.5 任务拆解 | Phase 2 通过 | `skills/planning/auto-decompose.md` [C2] | Coordinator |
| 3 编码 | [H2] 人工确认 | `skills/coding/SKILL.md` + `tdd-loop.md` + `verification.md` | Executor |
| 4 编码评审 | Phase 3 完成 | `skills/review/SKILL.md` + `code-review-template.md` | Verifier |
| 5 单测编写 | [H3] 人工确认 | `skills/testing/SKILL.md` | Executor |
| 6 单测评审 | Phase 5 完成 | `skills/review/SKILL.md` | Verifier |
| 7 推送 | Phase 6 通过 | `skills/shipping/mr-create.md` + `finishing-branch.md` | Executor |
| 8 CI 验证 | 代码推送后 | 等待 CI pipeline | Executor |
| 9 部署验证 | CI 通过 | `skills/testing/playwright.md`（E2E） | Executor |
| 10 用户确认 | 部署完成 | `skills/evolution/extract-and-score.md` [C7] | Coordinator |

### Session Start Hook

`harness-init.sh` 自动配置 `.claude/settings.json` 中的 SessionStart hook：

```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": ".harness/scheduler/hooks/session-start.sh"
      }]
    }]
  }
}
```

每次启动 Claude Code，hook 会输出：

```
=== Harness Active ===
Project: eam-v2
Stack: Java/Spring Boot/MyBatis
Tool: claude
Active: 2026-05-09-vin-duplicate-check [running]
Progress: 3/6 tasks done
======================
```

Agent 据此自动恢复上下文，无需手动告知"上次做到哪了"。

---

## Codex 集成：独立第二视角

> `/codex` 调用 OpenAI Codex CLI 作为独立审查者——"200 IQ 的对抗性 code review"。

### 三种模式

| 命令 | 作用 | 何时用 |
|------|------|--------|
| `/codex review` | 独立审查当前 diff，输出 PASS/FAIL + 逐条问题 | Phase 4（编码评审）、Phase 7（推送前） |
| `/codex challenge` | 对抗模式——尝试找出代码的 bug 和边界情况 | 复杂逻辑编码后、安全敏感变更 |
| `/codex consult` | 向 Codex 咨询任意技术问题，支持会话连续性 | 架构设计、技术选型、疑难排查 |

### 在 Harness 流程中的位置

```
Phase 3 编码 → Phase 4 编码评审
                    ↓
              Claude 自审（skills/review/SKILL.md）
                    ↓
              /codex review ← 独立第二视角
                    ↓
              /codex challenge ← 对抗性破坏测试（可选）
                    ↓
              综合两方意见 → 通过 / 返回 Phase 3 修复
                    ↓
              [H3] 人工确认
```

### 前置条件

1. Codex CLI 已安装：`npm install -g @openai/codex`
2. API 已配置：`~/.codex/config.toml` + `$OPENAI_API_KEY`（阶段 3 已完成）
3. 在 Claude Code 会话中使用 `/codex` 命令

### 审查结果格式

`/codex review` 输出示例：

```
┌─────────────────────────────────────┐
│  Codex Review: FAIL (3 issues)     │
├─────────────────────────────────────┤
│ [CRITICAL] SQL injection risk       │
│   file: UserRepo.java:42           │
│   raw string concat in query        │
│                                     │
│ [WARNING] Missing null check        │
│   file: TaskService.java:108       │
│   param could be null from caller   │
│                                     │
│ [INFO] Dead code                    │
│   file: Utils.java:23-31           │
│   unreachable after early return    │
└─────────────────────────────────────┘
```

### 推荐工作流：Claude + Codex 3 轮循环

```
Round 1: Claude 实现 → /codex review（发现问题）
Round 2: Claude 修复 → /codex review（确认修复，发现遗留）
Round 3: Claude 修复 → /codex review（PASS）→ [H3] 人工确认
```

> 超过 3 轮仍有 CRITICAL → 暂停，人工介入。

---

## 开始使用

### 启动 Claude Code

```bash
cd 你的项目目录
claude
```

进入 Claude Code 后：
- 输入 `/harness` 激活 Harness 流程
- 描述你要做的需求，Claude 会按 10 步流程引导你

### 使用 Codex 做代码审查

在 Claude Code 中输入：
```
/codex review    # 让 Codex 独立审查当前代码变更
/codex challenge # 让 Codex 尝试破坏你的代码
/codex consult   # 向 Codex 咨询问题
```

### 使用 Scheduler 自主执行

```bash
# 创建需求目录
mkdir .harness/changes/$(date +%F)-my-feature
cp .harness/changes/_template/* .harness/changes/$(date +%F)-my-feature/

# 编辑 plan.json 和 requirement.md
# 然后启动 Scheduler
cd .harness/changes/$(date +%F)-my-feature/
../../scheduler/harness-run.sh --tool claude --review
```

### Harness 流程速览

```
第 1 步：搞清楚要做什么 → 你确认
第 2 步：方案评审        → 你确认
第 3 步：写代码
第 4 步：代码评审        → 你确认
  └── /codex review + /codex challenge（可选）
第 5 步：写测试
第 6 步：测试评审+执行
第 7 步：推送代码        → 你确认
  └── /codex review（推送前最终检查）
第 8 步：CI 构建验证
第 9 步：部署验证
第 10步：你验收          → 你确认
```

每一步做什么、看什么、确认什么，详见 [Harness 实操手册](https://mi.feishu.cn/wiki/UvQjwhPagiYlOwk5O9hcKykCn7b)。

---

## 常见问题

| 问题 | 解决 |
|------|------|
| `claude: command not found` | 运行 `npm install -g @anthropic-ai/claude-code` |
| `codex: command not found` | 运行 `npm install -g @openai/codex` |
| `ANTHROPIC_API_KEY 未配置` | 检查 `~/.zshrc` 中是否有 export 语句，运行 `source ~/.zshrc` |
| SSH 连接 GitLab 失败 | 检查 `~/.ssh/id_ed25519.pub` 是否添加到 GitLab |
| Maven 下载依赖失败 | 检查 `~/.m2/settings.xml` 是否配置了小米 Artifactory |
| 数据库连接失败 | 检查 `.env.local` 中的 MYSQL_HOST 是否可达（可能需要 VPN） |
| Codex 调用报错 | 检查 `~/.codex/config.toml`：model 必须有 `azure_openai/` 前缀；base_url 为 `http://model.mify.ai.srv/v1` |
| `Not supported model xxx` | 模型名缺少 provider 前缀。Claude 用 `ppio/pa/`，Codex 用 `azure_openai/`，MiMo 用 `xiaomi/` |
| Claude Code 连不上 Mify | 检查 `$ANTHROPIC_BASE_URL` 是否为 `http://model.mify.ai.srv/anthropic`（注意末尾无 `/`） |
| `/codex review` 无响应 | 确认 `$OPENAI_API_KEY` 已设置；确认 `codex --version` 有输出 |
| Session Start 没有输出 Harness 状态 | 检查 `.claude/settings.json` 中 hooks 配置；确认 `.harness/config.json` 存在 |
| `harness-init.sh` 报错 `jq not found` | 运行 `brew install jq`（macOS）或 `apt install jq`（Linux） |
| Chrome DevTools MCP 未加载 | 确认 `.claude/settings.json` 有 `mcpServers.chrome-devtools` 配置；重启 Claude Code |
| MCP 报 `no pages open` | 先在 Chrome 中打开目标页面，再使用 MCP 工具 |
| MCP 截图/操作超时 | 检查 Chrome 是否以 `--remote-debugging-port=9222` 启动；或用 `npx @anthropic-ai/chrome-devtools-mcp@latest` 自动管理 |
| CAS 登录页面 MCP 无法操作 | 先在 Chrome 手动完成 CAS 登录，MCP 复用已有 session |

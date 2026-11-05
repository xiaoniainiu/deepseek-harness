# 研发环境搭建指南

> Agent 在 Phase 3/9 需要本地构建或部署验证时查阅。来源：飞书文档 DpeUdgxKvom9SVxMZZ6c3Xb0nug

---

## 系统请求链路（全局理解）

```
浏览器 → 代理层(可选) → 米盾(SSO/CAS) → 网关层 → 后端服务 → 数据层
```

| 层级 | 作用 | 本地开发对应 | 常见问题 |
|------|------|-------------|---------|
| 前端页面 | 渲染 UI，发起 API 请求 | `pnpm dev` 启动的 dev server | 端口冲突、依赖缺失、编译报错 |
| 代理层 | 测试环境调试时，转发 API 到指定后端 | SwitchyOmega / Camora / Vite proxy | 仅测试时需要；代理未开启、指向错误环境 |
| 米盾（SSO） | 统一登录鉴权、Token 校验 | 线上必须经过；本地可跳过或 mock | 401 登录态过期、Token 无效、403 权限不足 |
| 网关层 | 域名解析、路由分发、HTTPS 终止 | 本地一般跳过，直接连后端 | 域名未配置、路由 404 |
| 后端服务 | 业务逻辑处理 | `java -jar` 启动的本地服务 | 端口占用、配置错误、依赖服务不通 |
| 数据层 | 持久化、缓存、异步消息 | Docker 本地部署或连测试环境 | 连接超时、账号权限、库表不存在 |

### 代理使用场景

| 场景 | 是否需要代理 | 说明 |
|------|------------|------|
| 访问线上系统 | 否 | 直接走米盾鉴权 |
| 本地前端 + 测试环境后端 | 是 | 代理把 API 请求转发到测试环境 |
| 本地前端 + 本地后端 | 看情况 | 可用 Vite proxy 或代理插件 |
| 纯后端开发 | 否 | 直接 curl/Postman 调本地接口 |

### 本地开发典型链路

**前端联调测试环境：**
```
浏览器 → localhost:3001 → 代理插件 → 米盾(测试) → 测试环境后端
```

**前后端都跑本地：**
```
浏览器 → localhost:3001 → Vite proxy / 代理插件 → localhost:8080 → 本地/远程 DB
```

**纯后端：**
```
Postman / curl → localhost:8080 → 本地/远程 DB
```

### 快速定位问题

```
请求失败？
├── 白屏/编译报错 → dev server 是否启动、依赖是否装全
├── 401/403 → 米盾：登录态过期 / Token 无效 / 权限不足
├── 404 → 代理配置 / 后端路由 / 网关路由
├── 500 → 查后端日志
├── 超时/连接拒绝 → 代理是否开启 / 后端是否启动
├── CORS 跨域 → 代理配置 or 后端 @CrossOrigin
└── 数据异常 → DB 连接配置 / 库表是否存在
```

---

## 后端环境

### Java

| 版本 | 用途 | 推荐 |
|------|------|------|
| JDK 8 | 老项目、Spring Boot 2.x | 必装 |
| JDK 17 | LTS，Spring Boot 3.x | 推荐 |

**macOS 安装：**
```bash
brew install --cask temurin@8
brew install --cask temurin@17
echo 'export PATH="$HOME/.jenv/bin:$PATH"' >> ~/.zshrc
```

**Windows 安装：**
- OpenJDK 下载：Adoptium Temurin / Amazon Corretto
- 路径建议：`C:\Java\jdk-8`、`C:\Java\jdk-17`
- 环境变量：`JAVA_HOME=C:\Java\jdk-17`，PATH 添加 `%JAVA_HOME%\bin`
- 多版本切换修改 JAVA_HOME 即可

**验证：** `java -version && javac -version && echo $JAVA_HOME`

### Maven

**macOS：** `brew install maven`

**Windows：** 下载解压 → `MAVEN_HOME` + PATH 添加 `%MAVEN_HOME%\bin`

**核心配置：** `~/.m2/settings.xml` — 参考 Artifactory Maven 用户配置手册

### Git

**macOS：** `brew install git`
**Windows：** `winget install Git.Git`

**基础配置：**
```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@xiaomi.com"
git config --global core.autocrlf input       # macOS/Linux
git config --global init.defaultBranch main
git config --global core.quotepath false
git config --global pull.rebase true
git config --global push.autoSetupRemote true
```

**SSH 配置：**
```bash
ssh-keygen -t ed25519 -C "your.email@xiaomi.com"
cat ~/.ssh/id_ed25519.pub | pbcopy   # macOS
ssh -T git@git.n.xiaomi.com          # 测试连接
```

---

## 前端环境

### Node.js（EAM 项目统一用 Volta）

EAM 项目**统一使用 Volta** 管理 Node 版本（保证环境及依赖一致），**不用 fnm/nvm**。

**安装 Volta：**
```bash
# macOS
curl https://get.volta.sh | bash

# Windows
winget install Volta.Volta
```

验证：`volta -v`（找不到命令则需手动添加环境变量）

**安装 Node：**
```bash
volta install node@20
node -v  # 应输出 v20.19.5，不能低于 18
```

**镜像配置（小米源）：**
```bash
npm config set registry "https://pkgs.d.xiaomi.net/artifactory/api/npm/mi-npm/"
```

### 包管理器（pnpm）

EAM 项目**统一使用 pnpm**，禁止使用 npm 安装依赖。

```bash
corepack enable pnpm
pnpm -v
```

### VS Code 必装扩展

**基础必备：**
- Prettier - Code formatter（自动格式化）
- ESLint（JS/TS 代码质量检查）
- EditorConfig for VS Code（统一代码风格）

**前端开发：**
- Auto Rename Tag（自动更新闭合标签）
- Path Intellisense / npm Intellisense（路径和包名补全）
- Import Cost（显示引入包大小）
- ES7+ React/Redux/React-Native snippets（React 代码片段）

**样式相关：**
- Tailwind CSS IntelliSense
- PostCSS Language Support
- Color Highlight / CSS Peek

**调试 & 工具：**
- GitLens（Git blame/对比）
- Error Lens（行内显示错误）
- Turbo Console Log（一键插入调试日志）
- Vitest（单测运行扩展）

**效率提升：**
- Bookmarks（标记代码位置）
- TODO Highlight（高亮 TODO/FIXME）
- Code Spell Checker（拼写检查）

### 前端代理配置（EAM 项目用 whistle + ZeroOmega）

EAM 项目使用 whistle 代理 + Chrome ZeroOmega 插件方案：

**第一步：安装 whistle**
```bash
volta install whistle
```

**第二步：安装 Chrome 插件 ZeroOmega**

**第三步：配置 ZeroOmega**
- 新增情景模式，代理服务器指向本地 dev server（localhost:3001）
- auto switch 中增加域名通配符规则，将测试环境域名关联到本地情景模式

**第四步：配置 whistle**
```bash
w2 start                    # 启动 whistle
# 浏览器访问 localhost:8899 打开 whistle 管理界面
# Rules 中写入代理配置：
```
```
http://mieam-asset.test.mioffice.cn localhost:3001
https://mieam-asset.test.mioffice.cn localhost:3001
http://mieam2-asset.test.mioffice.cn localhost:3001
https://mieam2-asset.test.mioffice.cn localhost:3001
```

**第五步：** 直接访问 http://mieam-asset.test.mioffice.cn，ZeroOmega 切换到 auto switch 模式即可代理到本地

**其他代理方案（非 EAM 项目）：**
- **SwitchyOmega**：全局代理切换
- **Camora**：按项目/环境配置代理

**Vite Proxy 方案（前后端都本地时）：**
```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api/, '')
      }
    }
  }
})
```

| 对比项 | whistle + ZeroOmega | Vite Proxy | Camora |
|--------|-------------------|-----------|--------|
| 适用场景 | EAM 前端 + 测试环境后端 | 前后端都本地 | 多环境切换 |
| 配置位置 | whistle Rules | vite.config.ts | Chrome 扩展 |
| 灵活度 | 全局，可切换多环境 | 仅限当前项目 | 全局 |

---

## 辅助工具

### Docker（本地数据层）

**安装：**
- macOS：`brew install --cask docker`
- Windows：Docker Desktop（需 WSL2）

**常用本地服务：**
```bash
# MySQL
docker run -d --name mysql-local -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=root123 -e MYSQL_DATABASE=mydb mysql:8.0

# Redis
docker run -d --name redis-local -p 6379:6379 redis:7-alpine
```

**Docker Compose（推荐）：**
```yaml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    ports: ["3306:3306"]
    environment:
      MYSQL_ROOT_PASSWORD: root123
      MYSQL_DATABASE: mydb
    volumes: [mysql-data:/var/lib/mysql]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
volumes:
  mysql-data:
```

```bash
docker compose up -d    # 启动
docker compose down     # 停止
```

### curl 速查

```bash
curl http://localhost:8080/api/users                                    # GET
curl -X POST http://localhost:8080/api/users \
  -H "Content-Type: application/json" -d '{"name":"张三"}'              # POST JSON
curl http://localhost:8080/api/users -H "Authorization: Bearer <token>" # 带鉴权
```

---

## 自测环境

### 后端自测

```bash
git clone git@git.n.xiaomi.com:group/project.git && cd project
./mvnw clean package -DskipTests
java -jar target/app.jar --spring.profiles.active=local
```

- 数据库连接：配置 `application-local.yml`
- Redis/MySQL：Docker 本地或连测试环境
- 接口文档：`http://localhost:8080/swagger-ui.html`

### 前端自测

```bash
cd project && pnpm install && pnpm dev
```

- 确保代理已配置并开启
- 关闭自动打开的页面，直接访问测试/pre 环境域名

### 注意事项

- 确认代理指向正确环境
- 前后端联调时代理指向后端服务地址
- 404/500 先确认后端是否启动
- 跨域问题检查代理配置

---

## 环境检查清单

```bash
# 后端
java -version          # JDK 17
mvn -version           # Maven 3.9+
git --version          # Git 2.x

# 前端
node -v               # Node 20.x
pnpm -v               # pnpm 9.x

# 验证项目
git clone <project-repo>
cd project && pnpm install && pnpm dev   # 能正常启动
```

---

## 常见问题速查

| 问题 | 解决方案 |
|------|---------|
| npm EACCES 权限错误 | `sudo chown -R $(whoami) ~/.npm` 或用 fnm |
| node-sass 安装失败 | 改用 `pnpm add -D sass` |
| 网络超时 | 切换镜像源 |
| 多项目 Node 版本不同 | `.nvmrc` + fnm 自动切换 |
| Maven 下载慢 | 检查 settings.xml 镜像配置 |
| Git SSH 超时 | 检查防火墙/代理，配置 `~/.ssh/config` |
| Windows 路径过长 | `git config --global core.longpaths true` |

---

## 一键安装脚本

**macOS：**
```bash
curl -fsSL https://cnbj1-fds.api.xiaomi.net/b2c-info-oa/dev-env-mac/env-setup.sh | bash
```

**Windows：** 下载 `install-dev-env.bat` + 配套文件，双击执行

---

## 车位管理项目本地调试（park-server / carport-management-web / park-tt）

> 来源：飞书文档 MJIKwvAK8imIv9kHOtIc2wrnnjb

### 项目总览

| 项目 | 技术栈 | 本地端口 | 调试方式 |
|------|--------|---------|---------|
| park-server | Java 8 + Spring Boot 2.3 + Maven | 8084 | 打 jar 启动 |
| carport-management-web | React 16 + Webpack 4 + HiUI | 3001/3002 | webpack-dev-server |
| park-tt | Taro 3.1 + React 17 + TypeScript | — | 飞书开发者工具 |

### 后端：park-server

**环境要求：**

| 工具 | 版本 | 说明 |
|------|------|------|
| JDK | 1.8 | OpenJDK 或 Eclipse Adoptium |
| Maven | 项目自带 mvnw | 不需要全局安装 |
| mit-starter | 1.1.8 | 需升级，否则 Keycenter 连接失败 |
| keycenter-agent-client | 3.5.3 | 配套 mit-starter 升级 |

**Keycenter 配置（关键）：** 项目使用小米 Keycenter 加密敏感配置，本地启动需升级依赖：

```xml
<dependency>
    <groupId>com.xiaomi.mit</groupId>
    <artifactId>mit-starter</artifactId>
    <version>1.1.8</version>
</dependency>
<dependency>
    <groupId>com.xiaomi</groupId>
    <artifactId>keycenter-agent-client</artifactId>
    <version>3.5.3</version>
</dependency>
```

升级后如遇 Base64 解密错误，需在 `application-biz.yml` 中更新密文配置（找 DBA 获取新密文）。

**启动步骤：**

```bash
cd park-server

# Step 1: 编译依赖模块
MAVEN_OPTS="-Xmx512m" ./mvnw.cmd install -pl park-core,park-dal,park-manager,park-service -Plocal -DskipTests

# Step 2: 打包 park-web
MAVEN_OPTS="-Xmx512m" ./mvnw.cmd package -pl park-web -Plocal -DskipTests

# Step 3: 启动（不能用 spring-boot:run，Windows 路径太长报 error=206）
java -jar park-web/target/park-web-0.0.1-SNAPSHOT.jar --spring.profiles.active=biz,oaucf
```

启动成功标志：`Started ParkServerApplication in xxx seconds`

**本地验证限制：** 后端有 oaucf 鉴权拦截，本地无法通过浏览器直接调用接口（返回 401/403）。验证方式：
- 部署 test 环境验证（推荐）
- 写临时 `/debug/**` 接口（免鉴权路径）

**已知问题：**

| 问题 | 原因 | 解决 |
|------|------|------|
| CreateProcess error=206 | Windows classpath 过长 | 用 `java -jar` 方式启动 |
| Keycenter connect timed out | 旧版 SDK 连接方式不兼容 | 升级 mit-starter 1.1.8 + keycenter 3.5.3 |
| Base64 解密失败 | 旧密文与新 SDK 不兼容 | 更新 yml 中密文配置 |
| ConditionEmployee 编译错误 | 历史遗留，不影响功能 | 忽略 |
| JVM OOM crash | 多模块编译内存不足 | 加 `MAVEN_OPTS="-Xmx512m"` |

### 前端：carport-management-web

**环境要求：** Node.js 18.x（需 `--openssl-legacy-provider`）、nvm-windows、whistle (w2)

**启动前端：**

```powershell
nvm use 18
$env:NODE_OPTIONS = "--openssl-legacy-provider"
cd carport-management-web
npm run start
```

**编译兼容：** 项目有 `?.` 和 `??` 语法，需临时在 `.babelrc` plugins 中加入（不提交）：
```json
"@babel/plugin-proposal-optional-chaining",
"@babel/plugin-proposal-nullish-coalescing-operator"
```

**Whistle 代理配置：**

```bash
nvm use 18 && npm install -g whistle && w2 start
# 管理界面：http://localhost:8899，代理端口：8888
```

启用 HTTPS 拦截：whistle 管理界面 → HTTPS 按钮 → 勾选 Capture TUNNEL CONNECTs → Download RootCA → 安装到「受信任的根证书颁发机构」

Chrome 配置代理（关闭所有 Chrome 后执行）：
```powershell
& "C:\Program Files\Google\Chrome\Application\chrome.exe" --proxy-server="http://localhost:8888" --user-data-dir="C:\Users\mi\chrome-proxy"
```

**Whistle Rules：**

| 场景 | Rules |
|------|-------|
| 只验前端改动（推荐） | `e.test.mioffice.cn/park-web http://localhost:3002` |
| 验后端改动 | `https://e.test.mioffice.cn/park-api replaceStatus://204 resCors://* excludeFilter://m:get excludeFilter://m:post` + `https://e.test.mioffice.cn/park-api http://localhost:8084/park-web resCors://* excludeFilter://m:options` |

**验证代理生效：** whistle Network 标签看 ServerIP 是否变为 `127.0.0.1`；前端可临时加标记文案确认。

### 飞书小程序：park-tt

**环境要求（严格）：**

| 工具 | 版本 | 说明 |
|------|------|------|
| Node.js | **16.x**（推荐 16.20.2） | npm 8 兼容 lockfileVersion@3；node-sass 支持 runtime 93 |
| Python | **2.7.x**（必须） | node-gyp 编译 node-sass 原生模块需要 Python 2 |
| 飞书开发者工具 | 最新版 | 调试/预览小程序 |

**不可使用的版本：** Node 18+（node-sass@4.14.1 不支持 runtime 108+）、Node 14（npm 6 无法解析 lockfileVersion@3）、Python 3.x（node-gyp@3.8.0 使用 Python 2 语法）

**构建步骤：**

```powershell
nvm use 16.20.2
$env:PATH = "C:\Python27;$env:PATH"
$env:NODE_OPTIONS = "--max-old-space-size=4096"
$env:SASS_BINARY_SITE = "https://npmmirror.com/mirrors/node-sass"
cd park-tt
npm install    # 首次约 3-5 分钟
npm run start  # 开发模式（watch），等同 cross-env ENV=test taro build --type tt --watch
```

编译成功后在飞书开发者工具中「导入项目」，目录选 `park-tt\dist`。

**已知坑：**

| 问题 | 解决 |
|------|------|
| node-sass 找不到 Python 2 | `$env:PATH = "C:\Python27;$env:PATH"` + `npm config set python "C:\Python27\python.exe"` |
| node-sass binary 下载 404 | `$env:SASS_BINARY_SITE = "https://npmmirror.com/mirrors/node-sass"` |
| npm install OOM | `$env:NODE_OPTIONS = "--max-old-space-size=4096"` |
| Babel Unknown option: .call$0 | 使用 Node 16（npm 8）+ 正常 `npm install`（不加 --ignore-scripts） |
| dist/app.json 不存在 | 确保编译成功后再导入飞书开发者工具 |

**技术栈速查：**

| 项目 | 值 |
|------|---|
| 框架 | Taro 3.1.5 |
| UI | React 17 + @mi/hiui-mobile-* |
| 语言 | TypeScript 3.9 |
| 样式 | SCSS |
| 编译目标 | 飞书小程序（tt） |
| 后端 API（test） | `http://api.test.mi.com/ams_park_tt/park_server` |
| 后端 API（pro） | `http://cn-gw.zg.mi.com/ams_park/park_server` |
| AppID | cli_a10a4c38de38d063 |

# 工程结构：模块职责 + N 层架构

> 以前靠口头传承的规矩，现在写成文件让 Agent 也能知道。

## 架构分层

按规范一层一层实现，不是凭感觉写。

```
┌─────────────────────────────────┐
│  Presentation Layer             │  ← 接收请求，参数校验，返回响应
│  (Controller / API / Handler)   │
├─────────────────────────────────┤
│  Business Layer                 │  ← 业务编排，事务管理，规则执行
│  (Service / UseCase)            │
├─────────────────────────────────┤
│  Domain Layer                   │  ← 实体定义，枚举，值对象
│  (Entity / Enum / VO)           │
├─────────────────────────────────┤
│  Data Access Layer              │  ← 数据持久化，查询构建
│  (Mapper / Repository / DAO)    │
├─────────────────────────────────┤
│  Integration Layer              │  ← 外部系统调用，消息发送
│  (External / Adapter / Client)  │
└─────────────────────────────────┘
```

## 层间通信规则

| 规则 | 说明 |
|------|------|
| Presentation → Business | 只调 Business，不跨层调 Data Access |
| Business → Data Access | 可以直接调 |
| Business → Integration | 可以直接调 |
| Integration → Business | **禁止**（反向依赖） |
| Data Access → Business | **禁止**（反向依赖） |
| Presentation → Integration | **禁止**（必须经过 Business） |

```
// WHY: Agent 曾直接在 Controller 调用 External 发消息，绕过了 Service 的事务边界，导致发送成功但数据未落库
// 验证: grep -r "External\|Client" controller/ | wc -l == 0
```

## 模块职责

### Presentation（展示层）

| 职责 | 不做 |
|------|------|
| HTTP 路由绑定 | 不写业务逻辑 |
| 参数校验（@Valid） | 不操作数据库 |
| 响应包装（Result<T>） | 不调用外部系统 |
| 权限校验注解 | 不处理事务 |

```
// WHY: 曾在 Controller 里写了 40 行分支判断 + 数据库查询，导致无法复用和测试
// 验证: Controller 方法体 <= 20 行
```

### Business（业务层）

| 职责 | 不做 |
|------|------|
| 业务规则执行 | 不处理 HTTP 细节 |
| 事务管理 | 不拼 SQL |
| 编排多个 Mapper/External | 不解析请求参数 |
| 分布式锁 | — |

### Domain（领域层）

| 职责 | 不做 |
|------|------|
| 实体定义（对应数据库表） | 不包含业务逻辑 |
| 枚举定义 | 不依赖 Spring |
| 值对象 / DTO | — |

### Data Access（数据访问层）

| 职责 | 不做 |
|------|------|
| CRUD 操作 | 不写业务判断 |
| 查询条件构建 | 不调外部系统 |
| 分页/排序 | 不处理事务 |

```
// WHY: 曾在 Mapper XML 里写了复杂的 if-else 业务判断，导致业务规则分散在 Service 和 Mapper 两处
// 验证: Mapper XML 中 <if> 标签仅用于动态查询条件，不用于业务分支
```

### Integration（集成层）

| 职责 | 不做 |
|------|------|
| 封装外部调用 | 不写业务逻辑 |
| 超时/重试/降级 | 不操作本地数据库 |
| 协议转换 | 不暴露外部模型给上层 |
| 日志记录 | — |

```
// WHY: Agent 推测外部接口参数格式，没有查实际文档，导致线上调用全部 400
// 验证: 每个 External 类头部注释标注对应外部系统文档链接
```

## 前端结构（如适用）

```
src/
├── views/           # 页面级组件（路由入口）
├── components/      # 可复用组件
├── api/             # 类型化 API 调用
├── common/          # 工具函数 + 样式
├── hooks/           # 自定义 Hook
└── __tests__/       # 测试文件
```

## 填充指引

使用此模板时，按项目实际情况：
1. 确认你的项目有几层（3 层？5 层？微服务？）
2. 删除不适用的层
3. 在每条规则后补充项目实际的 `// WHY` 和 `// 验证`
4. 层名称和包名替换为项目实际命名

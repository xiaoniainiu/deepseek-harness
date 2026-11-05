# Presentation 层 SOP

## 适用范围

Controller / API Handler / 路由入口

## 实现步骤

1. **确定路由路径**
   - 查现有路由表确认命名风格（RESTful? /api/v1?）
   - 保持与同模块其他接口一致

2. **定义 Request DTO**
   - 独立文件，不复用 Entity
   - 加校验注解（@NotNull, @Size 等）
   - 仅包含该接口需要的字段

3. **定义 Response DTO**
   - 独立文件，不直接返回 Entity
   - 包含前端需要的全部字段
   - 字段命名与前端约定一致（camelCase）

4. **编写 Controller 方法**
   - 方法体 ≤ 20 行
   - 职责：校验 → 调 Service → 包装返回
   - 无业务逻辑

5. **返回值包装**
   - 统一使用 Result<T> 或项目约定的包装类
   - 异常由全局处理器处理，Controller 不 try-catch

## 检查清单

- [ ] 方法体 ≤ 20 行？
- [ ] 无业务逻辑？
- [ ] 参数有校验注解？
- [ ] 返回值是强类型 DTO？
- [ ] 路由风格与同模块一致？

## 反模式

```
// WHY: Agent 在 Controller 做了数据库查询 + 条件判断 + 消息发送
// 正确: 全部下沉到 Service
```

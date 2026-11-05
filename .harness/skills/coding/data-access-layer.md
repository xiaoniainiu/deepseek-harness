# Data Access 层 SOP

## 实现步骤

1. 确认表结构存在（查 `wiki/data-model.md`）
2. 新建/修改 Entity — 字段与数据库列一一对应
3. 新建/修改 Mapper/Repository — 仅 CRUD + 动态查询
4. 不在此层写业务判断

## 检查清单

- [ ] Entity 字段与数据库列一致？
- [ ] 使用类型安全查询构建器（非字符串拼接）？
- [ ] 无业务逻辑混入？

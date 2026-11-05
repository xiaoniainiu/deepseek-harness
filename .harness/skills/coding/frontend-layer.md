# Frontend 层 SOP

## 实现步骤

1. 确认路由和页面位置
2. 定义 Props interface（TypeScript 强类型）
3. API 调用封装在 `api/` 或 `hooks/` 中
4. 组件内不直接 fetch
5. 使用项目 UI 组件库（不自造基础组件）

## 检查清单

- [ ] Props 有 TypeScript interface？
- [ ] API 调用在独立文件？
- [ ] 无内联样式？
- [ ] 使用项目组件库？

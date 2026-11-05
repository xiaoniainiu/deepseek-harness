# Frontend Review Template — 前端代码评审模板

> Phase 4 前端评审的结构化模板。六个维度逐项检查，输出格式化评审结果。

## 评审输出格式

```
[P0/P1/P2] 文件:行号 - 问题描述 - 修复建议
```

- **P0**：必须修复（功能错误/安全漏洞/数据丢失），阻止推进
- **P1**：强烈建议修复（可维护性/性能/规范），最多允许 2 个带理由通过
- **P2**：建议优化（代码风格/可读性），不阻止推进

---

## 维度一：组件架构

### 检查项

| # | 检查点 | 严重级 | 关联 AP |
|---|--------|--------|---------|
| 1.1 | 单文件是否超过 300 行 | P1 | AP-051 |
| 1.2 | Props 是否有 TypeScript interface 定义 | P0 | AP-052 |
| 1.3 | 是否重复造了已有公共组件 | P1 | — |
| 1.4 | 组件职责是否单一（不混合数据获取+渲染+业务逻辑） | P1 | — |
| 1.5 | 组件命名是否 PascalCase + 语义化 | P2 | — |
| 1.6 | 导出方式是否与项目约定一致（default vs named） | P2 | — |

### 评审要点

- 超 300 行的组件：建议按"容器+展示"或"区块"拆分
- Props interface 命名约定：`<ComponentName>Props`
- 复用判断：先查 `docs/reference/components.md`

---

## 维度二：状态管理

### 检查项

| # | 检查点 | 严重级 | 关联 AP |
|---|--------|--------|---------|
| 2.1 | 所有表单元素是否受控（value + onChange 成对） | P0 | AP-010 |
| 2.2 | useMemo/useCallback 依赖数组是否完整 | P0 | AP-009 |
| 2.3 | DatePicker 空值是否守卫为 null（非 Invalid Date） | P0 | AP-011 |
| 2.4 | 异步 Select 是否使用受控 value | P0 | AP-010 |
| 2.5 | useEffect 内是否有 stale closure 风险 | P1 | — |
| 2.6 | Redux store 更新是否有副作用泄露 | P1 | — |
| 2.7 | 状态范围是否合理（不过度全局化） | P2 | — |

### 评审要点

- 受控检查模式：搜索 `<Select` / `<DatePicker` / `<Input`，验证对应 value prop 存在
- stale closure 检测：useEffect 依赖数组缺少引用的外部变量
- 异步数据加载后状态是否正确更新（竞态条件）

---

## 维度三：路由与安全

### 检查项

| # | 检查点 | 严重级 | 关联 AP |
|---|--------|--------|---------|
| 3.1 | 新路由是否注册到 routes/ 目录 | P0 | — |
| 3.2 | admin/employee 路径是否正确隔离 | P0 | — |
| 3.3 | iframe src 是否包含正确 base path | P0 | AP-029 |
| 3.4 | fetch 是否设置 credentials: 'include' | P0 | AP-024 |
| 3.5 | linkNavigate 前缀匹配是否过宽 | P1 | AP-032 |
| 3.6 | 跨 SPA 导航是否使用 window.location.href | P0 | — |
| 3.7 | 敏感数据是否暴露在 URL query 中 | P1 | — |

### 评审要点

- iframe 场景三查：base path + embed-inject + postMessage
- 路由隔离：admin 页面不能通过 employee 路由访问
- CORS：跨域 fetch 必须配置 credentials + 服务端 Access-Control-Allow-Credentials

---

## 维度四：API 对齐

### 检查项

| # | 检查点 | 严重级 | 关联 AP |
|---|--------|--------|---------|
| 4.1 | 请求字段名与后端 DTO 是否严格对齐 | P0 | — |
| 4.2 | 响应空值/null 是否有防御处理 | P0 | — |
| 4.3 | 加载状态是否展示（loading/empty/error 三态） | P1 | — |
| 4.4 | API 调用是否在独立 hooks/api 文件 | P1 | — |
| 4.5 | 多值字段（分号/逗号分隔）是否有 split 处理 | P1 | — |
| 4.6 | 分页/排序参数是否正确传递 | P1 | — |
| 4.7 | 错误响应是否有用户友好提示 | P2 | — |

### 评审要点

- 字段对齐检查：对照后端 DTO 类定义，逐字段验证命名和类型
- null 防御模式：`data?.field ?? defaultValue`
- 三态验证：搜索组件是否有 loading/data.length===0/error 三个分支

---

## 维度五：HiUI 合规

### 检查项

| # | 检查点 | 严重级 | 关联 AP |
|---|--------|--------|---------|
| 5.1 | HiUI 组件 API 是否与当前版本一致 | P0 | — |
| 5.2 | Select/DatePicker/Form 是否用官方受控模式 | P0 | AP-010 |
| 5.3 | Portal 组件（Select dropdown/Modal）挂载点是否处理 | P1 | — |
| 5.4 | 是否自造了 HiUI 已有的组件 | P1 | — |
| 5.5 | 主题变量是否正确使用（非硬编码颜色值） | P2 | — |
| 5.6 | 图标是否使用 HiUI Icon 组件 | P2 | — |

### 评审要点

- 版本确认：查 package.json 中 @hi-ui 版本，对照该版本文档
- Portal 处理：Select dropdown 在 overflow:hidden 容器内需要 `zIndex` 或 `getPopupContainer`
- 受控模式：HiUI v4 部分组件从 v3 的非受控改为受控，注意迁移

---

## 维度六：性能

### 检查项

| # | 检查点 | 严重级 | 关联 AP |
|---|--------|--------|---------|
| 6.1 | 是否有不必要的重渲染（缺少 memo/useMemo） | P1 | — |
| 6.2 | 大列表是否使用虚拟滚动 | P2 | — |
| 6.3 | 图片/资源是否懒加载 | P2 | — |
| 6.4 | 新增包的 bundle 贡献是否合理 | P1 | — |
| 6.5 | 路由级组件是否使用 React.lazy | P2 | — |
| 6.6 | 频繁触发的事件是否有防抖/节流 | P1 | — |

### 评审要点

- 重渲染检测：父组件 state 变更是否导致无关子组件渲染
- bundle 贡献：新增 npm 包应查其 bundlephobia 大小
- 防抖/节流：搜索框输入、窗口 resize、滚动事件

---

## 评审总结模板

```markdown
## 前端评审结果 — v{N}

### 统计
- P0: {count} 项
- P1: {count} 项
- P2: {count} 项

### P0 问题（必须修复）
| # | 文件:行号 | 问题 | 建议 |
|---|-----------|------|------|
| 1 | ... | ... | ... |

### P1 问题（强烈建议修复）
| # | 文件:行号 | 问题 | 建议 |
|---|-----------|------|------|
| 1 | ... | ... | ... |

### P2 建议
| # | 文件:行号 | 建议 |
|---|-----------|------|
| 1 | ... | ... |

### 通过判定
- [ ] P0 = 0
- [ ] P1 ≤ 2（附理由：...）
- **结论**: PASS / FAIL（回退到 Phase {X}）
```

---

## 与设计意图对照

评审时必须读取 `phase-state.json → expertContext.frontend.designDecisions`，检查：
1. 实现的组件树是否与设计一致
2. 状态管理方案是否与设计一致
3. 如有偏离，是否有合理原因记录在 crossPhaseNotes 中

偏离无理由 → P1（设计-实现不一致，需补充说明或回退修改）

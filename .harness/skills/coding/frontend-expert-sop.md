# Frontend Expert SOP — 六步实现协议

> Phase 3 前端编码的完整执行流程。每一步有明确的输入/输出/检查点。

## 前置条件

- Phase 2.3 已完成前端架构设计（组件树、状态方案、API 契约）
- expertContext.frontend.designDecisions 已填充
- requirement.md 验收标准已确认

## Step 0: 经验注入与上下文加载

```
输入: phase-state.json → expertContext.frontend
动作:
  1. 读取 designDecisions → 不重新推导已确定的架构
  2. 从 cases-index.yaml 按 agent_role IN (frontend, shared) + module 匹配注入 CASE
  3. 从 genes/ 按 agent_role IN (frontend, shared) 注入活跃 Gene
  4. 匹配 AP（AP-009~AP-052 前端相关反模式）
输出: 注入摘要（三段格式）
检查: 注入摘要非空才能进入 Step 1
```

## Step 1: 路由与布局确认

```
输入: docs/reference/routes.md + 设计稿/需求文档
动作:
  1. 确认新页面的路由路径（/admin/xxx or /employee/xxx）
  2. 确认所在 SPA 边界（是 asset-eam-web 还是独立微前端）
  3. 确认是独立页面 or iframe 嵌入（嵌入需检查 base path + embed-inject）
  4. 确认布局模板（侧边栏/顶部导航/纯内容区）
  5. 注册路由到对应的 routes/ 目录
输出: 路由注册代码 + 布局确认
检查:
  - [ ] 路由路径与项目约定一致（admin/employee 前缀正确）
  - [ ] SPA 边界明确（不跨 SPA 使用 React Router）
  - [ ] iframe 场景已设置 base path（AP-029）
```

## Step 2: 组件架构

```
输入: Phase 2.3 组件树设计
动作:
  1. 创建页面主组件（pages/<module>/<PageName>/index.tsx）
  2. 定义所有 Props interface（TypeScript 强类型）
  3. 拆分子组件（>300 行必拆）
  4. 识别可复用组件 → 先查 docs/reference/components.md 是否已有
  5. 共享组件放 components/，页面专属组件放 pages/<module>/<PageName>/components/
输出: 组件文件骨架（interface + 空实现 + 导出）
检查:
  - [ ] 所有 Props 有 TypeScript interface
  - [ ] 无单文件超 300 行
  - [ ] 不重复造已有组件
  - [ ] 组件命名 PascalCase
```

## Step 3: 状态管理

```
输入: Phase 2.3 状态管理方案
动作:
  1. 确定状态范围：
     - 页面私有状态 → useState
     - 跨组件共享但不持久化 → useContext / 组件提升
     - 全局持久化 → Redux Store（新建 slice）
  2. 所有表单元素使用受控组件：
     - Select: value + onChange（异步 Select 必须受控 value）
     - DatePicker: value 为 null 时不传 Invalid Date
     - Form: 使用 Form.useForm() 或受控 state
  3. useMemo/useCallback 依赖数组完整（AP-009）
  4. 异步状态使用 loading/data/error 三元组
输出: 状态逻辑代码（hooks/ 或 stores/ 下）
检查:
  - [ ] 所有表单元素受控（value + onChange 成对出现）
  - [ ] DatePicker 空值守卫为 null（AP-011）
  - [ ] useMemo/useCallback deps 完整（AP-009）
  - [ ] 异步 Select 受控 value（AP-010）
  - [ ] 无 stale closure（useEffect 内引用最新值）
```

## Step 4: API 集成

```
输入: Phase 2.3 API 契约 + 后端接口文档
动作:
  1. API 调用封装在 api/<module>.ts 或 hooks/use<Feature>.ts
  2. 组件内不直接 fetch — 通过 hook 或 api 调用
  3. 请求字段名与后端 DTO 严格对齐（命名大小写、类型）
  4. 响应处理：
     - loading 态展示 Spinner/Skeleton
     - empty 态展示空状态提示
     - error 态展示错误信息 + 重试
  5. 所有 fetch 设置 credentials: 'include'（AP-024）
  6. 多值字段（分号/逗号分隔）防御性 split
输出: API 调用代码 + 三态渲染
检查:
  - [ ] fetch 有 credentials: 'include'
  - [ ] 字段名与后端 DTO 一致
  - [ ] loading / empty / error 三态覆盖
  - [ ] 空值/null 响应有防御处理
  - [ ] 多值字段有 split 处理
```

## Step 5: 样式

```
输入: 设计稿 / HiUI 设计规范
动作:
  1. 使用 SCSS/CSS Modules（跟随项目约定）
  2. HiUI 设计 tokens 用于颜色/间距/字体
  3. 无内联样式（style={{}} 禁止）
  4. 响应式断点覆盖（如项目要求）
  5. CSS 类命名与项目约定一致（BEM / module 命名）
输出: 样式文件
检查:
  - [ ] 无 style={{}} 内联样式
  - [ ] 使用项目 CSS 变量 / HiUI tokens
  - [ ] 类名命名一致
  - [ ] 无 !important（除非覆盖第三方）
```

## Step 6: 跨系统集成

```
输入: 系统架构图 / iframe 配置
动作:
  1. iframe 嵌入场景：
     - src 含正确 base path（与 Spring Boot context-path 对齐）
     - embed-inject.js 逻辑执行（不跳过 DOM 操作）
     - postMessage 通信协议确认
  2. Portal 组件处理：
     - Select dropdown → document.body（不被父容器 overflow:hidden 裁剪）
     - Modal/Drawer 挂载点确认
  3. 认证上下文传播：
     - iframe 内 cookie 传递
     - 跨域场景 CORS 配置
  4. 跨 SPA 导航：
     - 使用 window.location.href（不用 React Router navigate）
     - linkNavigate 前缀匹配精确（AP-032）
输出: 集成代码 + 配置
检查:
  - [ ] iframe base path 正确
  - [ ] Portal 组件挂载正确
  - [ ] 跨域认证配置完整
  - [ ] 跨 SPA 导航使用 location.href
  - [ ] linkNavigate 前缀不过宽
```

## Checkpoint 协议（贯穿所有 Step）

每完成一个 Step：
1. 对照验收标准自检
2. 对照 designDecisions 检查是否偏离
3. 发现偏离或新约束 → 写入 expertContext.frontend.crossPhaseNotes
4. 编译验证（tsc --noEmit 或 vite build --mode check）

## 完成门禁

全部 Step 完成后，必须满足：
- [ ] TypeScript 编译无 error
- [ ] Lint 无 error（warning ≤ 项目基线）
- [ ] 所有组件 Props 有 interface
- [ ] 所有表单元素受控
- [ ] 所有 API 调用在独立文件
- [ ] 三态渲染完整
- [ ] 无内联样式
- [ ] 路由已注册
- [ ] 原子提交（每个 Step 一个 commit 或按功能聚合）

# Playwright E2E 测试

> Phase 5（前端E2E测试编写）+ Phase 6（E2E测试评审）+ Phase 9（部署验证自动化）
> 框架文档：https://github.com/microsoft/playwright
> 安装：`npm init playwright@latest` 或 `pnpm add -D @playwright/test`

---

## 何时使用 Playwright

| 场景 | 用 Playwright | 用单元测试 | 用 Chrome DevTools MCP |
|------|:---:|:---:|:---:|
| API 接口逻辑验证 | | V | |
| 前端组件渲染 | V | | |
| 表单提交端到端流程 | V | | |
| 跨页面业务流程 | V | | |
| 需要 CAS 认证的页面 | V | | V |
| 一次性手动验证 | | | V |
| CI 持续回归 | V | | |
| 可视化回归（截图对比） | V | | |

**原则**：Playwright 用于可重复执行的自动化 E2E 测试；Chrome DevTools MCP 用于一次性交互式验证。两者互补，不替代。

---

## 项目配置

### 初始化

```bash
# 新项目
npm init playwright@latest

# 已有项目
pnpm add -D @playwright/test
npx playwright install    # 安装浏览器
```

### playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,
  reporter: process.env.CI ? 'html' : 'list',

  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  ],

  webServer: process.env.CI ? undefined : {
    command: 'pnpm dev',
    port: 3000,
    reuseExistingServer: true,
  },
});
```

### 目录结构

```
e2e/
├── fixtures/           # 共享 fixtures（认证状态、测试数据）
│   └── auth.ts         # CAS 认证 fixture
├── pages/              # Page Object Model
│   ├── login.page.ts
│   └── plan-list.page.ts
├── tests/              # 测试文件
│   ├── plan-crud.spec.ts
│   └── task-assign.spec.ts
└── playwright.config.ts
```

---

## CAS/米盾认证处理

> 公司系统统一走米盾 CAS 认证。Playwright 需要在测试前完成登录并保存认证状态。

### 方式 A：storageState 持久化（推荐）

```typescript
// e2e/fixtures/auth.ts
import { test as base, expect } from '@playwright/test';
import path from 'path';

const AUTH_FILE = path.join(__dirname, '../.auth/user.json');

// 全局 setup：登录一次，后续所有测试复用
export async function globalSetup() {
  const { chromium } = await import('@playwright/test');
  const browser = await chromium.launch({ headless: false });
  const page = await browser.newPage();

  // 访问目标系统，自动跳转 CAS 登录页
  await page.goto(process.env.BASE_URL!);

  // 等待手动登录（米盾扫码/密码）
  // 或自动填写（如果测试环境支持密码登录）
  await page.waitForURL('**/inventory/**', { timeout: 120_000 });

  // 保存认证状态（cookies + localStorage）
  await page.context().storageState({ path: AUTH_FILE });
  await browser.close();
}
```

```typescript
// playwright.config.ts
export default defineConfig({
  globalSetup: './e2e/fixtures/auth.ts',
  use: {
    storageState: './e2e/.auth/user.json',
  },
});
```

### 方式 B：API Token 注入

```typescript
// 如果有可用的 API token（从米盾获取）
test.use({
  extraHTTPHeaders: {
    'X-Proxy-UserDetail': process.env.CAS_USER_DETAIL || '',
  },
});
```

### .gitignore

```
e2e/.auth/
test-results/
playwright-report/
```

---

## Page Object Model

### 示例：盘点计划列表页

```typescript
// e2e/pages/plan-list.page.ts
import { Page, Locator, expect } from '@playwright/test';

export class PlanListPage {
  readonly page: Page;
  readonly createButton: Locator;
  readonly searchInput: Locator;
  readonly table: Locator;

  constructor(page: Page) {
    this.page = page;
    this.createButton = page.getByRole('button', { name: '新建计划' });
    this.searchInput = page.getByPlaceholder('搜索');
    this.table = page.locator('.hi-v4-table');
  }

  async goto() {
    await this.page.goto('/inventory/asset/inventory/plan');
    await this.page.waitForLoadState('networkidle');
  }

  async search(keyword: string) {
    await this.searchInput.fill(keyword);
    await this.searchInput.press('Enter');
    await this.page.waitForResponse(resp =>
      resp.url().includes('/api/plan/list') && resp.status() === 200
    );
  }

  async getRowCount(): Promise<number> {
    return this.table.locator('tbody tr').count();
  }

  async clickCreate() {
    await this.createButton.click();
    await this.page.waitForURL('**/plan/create');
  }
}
```

---

## 测试编写规范

### 基本结构

```typescript
import { test, expect } from '@playwright/test';
import { PlanListPage } from '../pages/plan-list.page';

test.describe('盘点计划列表', () => {
  let planList: PlanListPage;

  test.beforeEach(async ({ page }) => {
    planList = new PlanListPage(page);
    await planList.goto();
  });

  test('加载计划列表', async ({ page }) => {
    // 验证页面标题
    await expect(page.getByText('盘点计划')).toBeVisible();

    // 验证表格有数据
    const rowCount = await planList.getRowCount();
    expect(rowCount).toBeGreaterThan(0);
  });

  test('搜索过滤', async ({ page }) => {
    await planList.search('2026年');

    // 验证搜索结果
    const rows = page.locator('.hi-v4-table tbody tr');
    const count = await rows.count();
    for (let i = 0; i < count; i++) {
      await expect(rows.nth(i)).toContainText('2026');
    }
  });
});
```

### API 响应拦截验证

```typescript
test('创建计划 - API 参数正确', async ({ page }) => {
  const planList = new PlanListPage(page);
  await planList.goto();

  // 拦截 API 请求
  const [request] = await Promise.all([
    page.waitForRequest(req =>
      req.url().includes('/api/plan/create') && req.method() === 'POST'
    ),
    planList.clickCreate(),
    // ... 填写表单并提交
  ]);

  // 验证请求参数
  const body = request.postDataJSON();
  expect(body.planName).toBeDefined();
  expect(body.planType).toMatch(/^(full|sample|spot)$/);
});
```

### 网络请求断言

```typescript
test('API 返回业务正确数据', async ({ page }) => {
  const response = await page.waitForResponse(
    resp => resp.url().includes('/api/plan/list') && resp.ok()
  );
  const data = await response.json();

  // 不只检查 code=0，检查业务数据
  expect(data.code).toBe(200);
  expect(data.data.list).toBeInstanceOf(Array);
  expect(data.data.list.length).toBeGreaterThan(0);
  expect(data.data.list[0]).toHaveProperty('planId');
});
```

### 截图对比（视觉回归）

```typescript
test('页面视觉一致', async ({ page }) => {
  const planList = new PlanListPage(page);
  await planList.goto();

  await expect(page).toHaveScreenshot('plan-list.png', {
    maxDiffPixelRatio: 0.01,
  });
});
```

---

## HiUI 组件交互模式

> EAM 系统使用 HiUI v4，部分组件有 portal 挂载行为。

| HiUI 组件 | Playwright 选择器 | 注意事项 |
|-----------|------------------|---------|
| Select 下拉 | `page.locator('.hi-v4-select__option')` | 选项挂载在 `body` 上，不在组件 DOM 内 |
| DatePicker | `page.getByRole('textbox')` → click → 选日期 | range 模式需两次选择+确认 |
| Table 行 | `page.locator('.hi-v4-table tbody tr').nth(N)` | 虚拟滚动时需先滚到可见区域 |
| Modal 弹窗 | `page.locator('.hi-v4-modal')` | 挂载在 body portal |
| Drawer 抽屉 | `page.locator('.hi-v4-drawer')` | 同 modal，body portal |
| Tree 节点 | `page.getByRole('treeitem', { name: '节点名' })` | 展开需先 click 箭头 |

### Select 选择示例

```typescript
// HiUI Select 选项在 body portal 中
async function selectOption(page: Page, triggerSelector: string, optionText: string) {
  await page.locator(triggerSelector).click();
  await page.locator('.hi-v4-select__option').filter({ hasText: optionText }).click();
}
```

---

## CI 集成

### GitHub Actions / GitLab CI

```yaml
# .gitlab-ci.yml
e2e-test:
  image: mcr.microsoft.com/playwright:v1.49.0-noble
  stage: test
  script:
    - npm ci
    - npx playwright install --with-deps
    - npx playwright test
  artifacts:
    when: always
    paths:
      - playwright-report/
      - test-results/
    expire_in: 7 days
```

### 与 Harness Phase 集成

| Phase | Playwright 用法 |
|-------|----------------|
| Phase 5 单测编写 | 编写 E2E 测试（`e2e/tests/*.spec.ts`） |
| Phase 6 单测评审 | 评审 E2E 测试覆盖度 |
| Phase 6.5 单测执行 | `npx playwright test` 全量执行 |
| Phase 8 CI 验证 | CI 中自动运行 + 生成报告 |
| Phase 9 部署验证 | 指向测试/生产 URL 运行冒烟测试 |

### plan.json 中的 E2E 任务

```json
{
  "id": "T-010",
  "title": "编写盘点计划 E2E 测试",
  "type": "ui",
  "testFile": "e2e/tests/plan-crud.spec.ts",
  "verificationCommands": [
    {"command": "npx playwright test e2e/tests/plan-crud.spec.ts", "concurrencyType": "readonly"},
    {"command": "npx tsc --noEmit", "concurrencyType": "readonly"}
  ]
}
```

---

## 命令速查

```bash
# 运行所有测试
npx playwright test

# 运行指定文件
npx playwright test e2e/tests/plan-crud.spec.ts

# UI 模式（交互式调试）
npx playwright test --ui

# 带浏览器窗口运行（调试用）
npx playwright test --headed

# 只运行失败的测试
npx playwright test --last-failed

# 生成测试代码（录制模式）
npx playwright codegen http://localhost:3000

# 查看最近一次报告
npx playwright show-report

# 更新截图基线
npx playwright test --update-snapshots
```

---

## 与 Chrome DevTools MCP 的分工

| 维度 | Playwright | Chrome DevTools MCP |
|------|-----------|-------------------|
| 执行方式 | 脚本自动化，CI 可重复 | Agent 交互式，单次验证 |
| 认证 | storageState 持久化 | 手动 CAS 登录 |
| 适用阶段 | Phase 5/6/6.5/8（自动化回归） | Phase 9（一次性部署验证） |
| 报告 | HTML 报告 + 截图 + trace | Agent 对话中的文字描述 |
| 维护成本 | 需要维护测试代码 | 零维护（每次手动执行） |
| 覆盖范围 | 预定义场景，可扩展 | 即兴验证，灵活 |

**推荐组合**：
1. Phase 5：用 Playwright 写核心业务流程 E2E 测试
2. Phase 8：CI 中跑 Playwright 回归
3. Phase 9：Chrome DevTools MCP 做部署后即兴冒烟验证

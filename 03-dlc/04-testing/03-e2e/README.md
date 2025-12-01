# 端到端测试 (E2E) 最佳实践

## 角色设定

你是一位精通端到端测试的自动化测试专家，擅长 Cypress、Playwright、Selenium 等工具，注重用户体验和业务流程验证。

## 提示词模板

### E2E 测试编写

```
请帮我编写 E2E 测试：
- 用户场景：[描述用户操作流程]
- 测试工具：[Cypress/Playwright/Selenium]
- 涉及页面：[列出页面]
- 验证要点：[期望的结果]
- 测试数据：[需要准备的数据]

请提供完整的测试代码。
```

## 核心代码示例

### Cypress E2E 测试

```typescript
// cypress/e2e/checkout.cy.ts
describe('购物流程', () => {
  beforeEach(() => {
    cy.login('testuser', 'password123');
    cy.clearCart();
  });

  it('完整购物流程 - 从浏览到下单', () => {
    // 1. 浏览商品
    cy.visit('/products');
    cy.getByTestId('product-list').should('be.visible');

    // 2. 搜索商品
    cy.getByTestId('search-input').type('手机');
    cy.getByTestId('search-button').click();
    cy.url().should('include', 'keyword=手机');
    cy.getByTestId('product-card').should('have.length.greaterThan', 0);

    // 3. 查看商品详情
    cy.getByTestId('product-card').first().click();
    cy.url().should('match', /\/products\/\d+/);
    cy.getByTestId('product-name').should('be.visible');
    cy.getByTestId('product-price').should('be.visible');

    // 4. 加入购物车
    cy.getByTestId('quantity-input').clear().type('2');
    cy.getByTestId('add-to-cart').click();
    cy.getByTestId('toast-success').should('contain', '已加入购物车');
    cy.getByTestId('cart-count').should('contain', '2');

    // 5. 查看购物车
    cy.getByTestId('cart-icon').click();
    cy.url().should('include', '/cart');
    cy.getByTestId('cart-item').should('have.length', 1);
    cy.getByTestId('cart-total').should('not.be.empty');

    // 6. 结算
    cy.getByTestId('checkout-button').click();
    cy.url().should('include', '/checkout');

    // 7. 填写收货地址
    cy.getByTestId('address-form').within(() => {
      cy.get('input[name="name"]').type('张三');
      cy.get('input[name="phone"]').type('13800138000');
      cy.get('input[name="address"]').type('北京市朝阳区xxx路xxx号');
    });

    // 8. 选择支付方式
    cy.getByTestId('payment-alipay').click();

    // 9. 提交订单
    cy.intercept('POST', '/api/orders').as('createOrder');
    cy.getByTestId('submit-order').click();

    // 10. 验证订单创建成功
    cy.wait('@createOrder').its('response.statusCode').should('eq', 200);
    cy.url().should('include', '/order/success');
    cy.getByTestId('order-number').should('be.visible');
    cy.getByTestId('order-status').should('contain', '待支付');
  });

  it('库存不足时显示提示', () => {
    cy.intercept('POST', '/api/cart/add', {
      statusCode: 400,
      body: { code: 'INSUFFICIENT_STOCK', message: '库存不足' },
    });

    cy.visit('/products/1');
    cy.getByTestId('quantity-input').clear().type('9999');
    cy.getByTestId('add-to-cart').click();

    cy.getByTestId('toast-error').should('contain', '库存不足');
  });
});

// 用户认证流程
describe('用户认证', () => {
  it('注册新用户', () => {
    const email = `test${Date.now()}@example.com`;

    cy.visit('/register');

    cy.getByTestId('register-form').within(() => {
      cy.get('input[name="email"]').type(email);
      cy.get('input[name="username"]').type('newuser');
      cy.get('input[name="password"]').type('Password123!');
      cy.get('input[name="confirmPassword"]').type('Password123!');
    });

    cy.getByTestId('agree-terms').check();
    cy.getByTestId('submit-register').click();

    cy.url().should('include', '/login');
    cy.getByTestId('toast-success').should('contain', '注册成功');
  });

  it('登录失败 - 密码错误', () => {
    cy.visit('/login');

    cy.getByTestId('login-form').within(() => {
      cy.get('input[name="username"]').type('testuser');
      cy.get('input[name="password"]').type('wrongpassword');
    });

    cy.getByTestId('submit-login').click();

    cy.getByTestId('error-message').should('contain', '用户名或密码错误');
    cy.url().should('include', '/login');
  });
});
```

### Playwright E2E 测试

```typescript
// tests/e2e/order.spec.ts
import { test, expect } from '@playwright/test';

test.describe('订单管理', () => {
  test.beforeEach(async ({ page }) => {
    // 登录
    await page.goto('/login');
    await page.fill('[data-testid="username"]', 'testuser');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="submit-login"]');
    await expect(page).toHaveURL('/dashboard');
  });

  test('查看订单列表', async ({ page }) => {
    await page.goto('/orders');

    // 等待订单列表加载
    await expect(page.locator('[data-testid="order-list"]')).toBeVisible();

    // 验证订单项存在
    const orderItems = page.locator('[data-testid="order-item"]');
    await expect(orderItems).toHaveCount.greaterThan(0);

    // 验证订单信息
    const firstOrder = orderItems.first();
    await expect(firstOrder.locator('[data-testid="order-id"]')).toBeVisible();
    await expect(firstOrder.locator('[data-testid="order-status"]')).toBeVisible();
    await expect(firstOrder.locator('[data-testid="order-amount"]')).toBeVisible();
  });

  test('筛选订单', async ({ page }) => {
    await page.goto('/orders');

    // 按状态筛选
    await page.selectOption('[data-testid="status-filter"]', 'COMPLETED');
    await page.waitForResponse(resp =>
      resp.url().includes('/api/orders') && resp.status() === 200
    );

    // 验证筛选结果
    const orders = page.locator('[data-testid="order-item"]');
    for (const order of await orders.all()) {
      await expect(order.locator('[data-testid="order-status"]')).toContainText('已完成');
    }
  });

  test('取消订单', async ({ page }) => {
    await page.goto('/orders');

    // 找到待支付订单
    const pendingOrder = page.locator('[data-testid="order-item"]')
      .filter({ has: page.locator('text=待支付') })
      .first();

    await pendingOrder.locator('[data-testid="cancel-button"]').click();

    // 确认对话框
    await expect(page.locator('[data-testid="confirm-dialog"]')).toBeVisible();
    await page.click('[data-testid="confirm-yes"]');

    // 验证取消成功
    await expect(page.locator('[data-testid="toast-success"]')).toContainText('订单已取消');
  });

  test('订单详情页', async ({ page }) => {
    await page.goto('/orders');

    // 点击第一个订单
    await page.locator('[data-testid="order-item"]').first().click();

    // 验证详情页
    await expect(page).toHaveURL(/\/orders\/\d+/);
    await expect(page.locator('[data-testid="order-detail"]')).toBeVisible();
    await expect(page.locator('[data-testid="order-items"]')).toBeVisible();
    await expect(page.locator('[data-testid="shipping-info"]')).toBeVisible();
    await expect(page.locator('[data-testid="payment-info"]')).toBeVisible();
  });
});

// 多浏览器测试
test.describe('跨浏览器兼容性', () => {
  test('页面在不同视口正确显示', async ({ page }) => {
    // 桌面视口
    await page.setViewportSize({ width: 1920, height: 1080 });
    await page.goto('/');
    await expect(page.locator('[data-testid="desktop-nav"]')).toBeVisible();
    await expect(page.locator('[data-testid="mobile-nav"]')).not.toBeVisible();

    // 平板视口
    await page.setViewportSize({ width: 768, height: 1024 });
    await page.goto('/');

    // 移动端视口
    await page.setViewportSize({ width: 375, height: 667 });
    await page.goto('/');
    await expect(page.locator('[data-testid="mobile-nav"]')).toBeVisible();
  });
});

// 截图和视觉测试
test('视觉回归测试', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png', {
    maxDiffPixels: 100,
  });
});
```

### 页面对象模式 (POM)

```typescript
// pages/LoginPage.ts
import { Page, Locator, expect } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly usernameInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.usernameInput = page.locator('[data-testid="username"]');
    this.passwordInput = page.locator('[data-testid="password"]');
    this.submitButton = page.locator('[data-testid="submit-login"]');
    this.errorMessage = page.locator('[data-testid="error-message"]');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(username: string, password: string) {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string) {
    await expect(this.errorMessage).toContainText(message);
  }
}

// pages/OrderPage.ts
export class OrderPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  async goto() {
    await this.page.goto('/orders');
  }

  async filterByStatus(status: string) {
    await this.page.selectOption('[data-testid="status-filter"]', status);
    await this.page.waitForLoadState('networkidle');
  }

  async getOrderCount(): Promise<number> {
    return await this.page.locator('[data-testid="order-item"]').count();
  }

  async cancelOrder(orderId: string) {
    const order = this.page.locator(`[data-testid="order-${orderId}"]`);
    await order.locator('[data-testid="cancel-button"]').click();
    await this.page.locator('[data-testid="confirm-yes"]').click();
  }
}

// 使用页面对象
test('使用 POM 进行登录测试', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login('testuser', 'password123');
  await expect(page).toHaveURL('/dashboard');
});
```

### CI 配置

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Start application
        run: npm run start &
        env:
          NODE_ENV: test

      - name: Wait for app
        run: npx wait-on http://localhost:3000

      - name: Run E2E tests
        run: npx playwright test

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
```

## 最佳实践清单

- [ ] 使用 data-testid 选择元素
- [ ] 采用页面对象模式 (POM)
- [ ] 测试核心用户流程
- [ ] 模拟网络请求
- [ ] 处理异步操作
- [ ] 截图记录失败场景
- [ ] CI/CD 集成
- [ ] 并行执行测试

# Cypress E2E 测试最佳实践

## 角色设定

你是一位精通 Cypress 的前端测试专家，擅长 E2E 测试、组件测试和测试自动化。

## 提示词模板

### Cypress 测试

```
请帮我编写 Cypress 测试：
- 测试场景：[描述业务场景]
- 页面路径：[页面 URL]
- 关键操作：[列出操作步骤]
- 断言要点：[期望结果]

请提供完整的测试代码。
```

## 核心配置示例

### Cypress 配置

```typescript
// cypress.config.ts
import { defineConfig } from 'cypress';

export default defineConfig({
  e2e: {
    baseUrl: 'http://localhost:3000',
    supportFile: 'cypress/support/e2e.ts',
    specPattern: 'cypress/e2e/**/*.cy.{js,jsx,ts,tsx}',
    viewportWidth: 1280,
    viewportHeight: 720,
    video: true,
    screenshotOnRunFailure: true,
    defaultCommandTimeout: 10000,
    requestTimeout: 10000,
    responseTimeout: 30000,
    retries: {
      runMode: 2,
      openMode: 0,
    },
    env: {
      apiUrl: 'http://localhost:8080/api',
    },
    setupNodeEvents(on, config) {
      // 任务和插件
      on('task', {
        log(message) {
          console.log(message);
          return null;
        },
        clearDatabase() {
          // 清理测试数据
          return null;
        },
      });

      return config;
    },
  },
  component: {
    devServer: {
      framework: 'react',
      bundler: 'vite',
    },
    specPattern: 'src/**/*.cy.{js,jsx,ts,tsx}',
  },
});
```

### 支持文件

```typescript
// cypress/support/e2e.ts
import './commands';

// 全局钩子
beforeEach(() => {
  // 清除 localStorage
  cy.clearLocalStorage();
  // 清除 cookies
  cy.clearCookies();
});

// 忽略未捕获的异常
Cypress.on('uncaught:exception', (err) => {
  // 返回 false 阻止测试失败
  if (err.message.includes('ResizeObserver')) {
    return false;
  }
  return true;
});
```

### 自定义命令

```typescript
// cypress/support/commands.ts
declare global {
  namespace Cypress {
    interface Chainable {
      login(username: string, password: string): Chainable<void>;
      logout(): Chainable<void>;
      getByTestId(testId: string): Chainable<JQuery<HTMLElement>>;
      mockApi(fixture: string): Chainable<void>;
    }
  }
}

// 登录命令
Cypress.Commands.add('login', (username: string, password: string) => {
  cy.session([username, password], () => {
    cy.visit('/login');
    cy.get('[data-testid="username"]').type(username);
    cy.get('[data-testid="password"]').type(password);
    cy.get('[data-testid="submit"]').click();
    cy.url().should('not.include', '/login');
    cy.window().its('localStorage.token').should('exist');
  });
});

// 登出命令
Cypress.Commands.add('logout', () => {
  cy.clearLocalStorage();
  cy.clearCookies();
  cy.visit('/login');
});

// 按 data-testid 获取元素
Cypress.Commands.add('getByTestId', (testId: string) => {
  return cy.get(`[data-testid="${testId}"]`);
});

// Mock API
Cypress.Commands.add('mockApi', (fixture: string) => {
  cy.intercept('GET', '/api/**', { fixture }).as('api');
});

export {};
```

### E2E 测试示例

```typescript
// cypress/e2e/auth/login.cy.ts
describe('Login', () => {
  beforeEach(() => {
    cy.visit('/login');
  });

  it('should display login form', () => {
    cy.getByTestId('login-form').should('be.visible');
    cy.getByTestId('username').should('be.visible');
    cy.getByTestId('password').should('be.visible');
    cy.getByTestId('submit').should('be.visible').and('be.disabled');
  });

  it('should enable submit button when form is valid', () => {
    cy.getByTestId('username').type('testuser');
    cy.getByTestId('password').type('password123');
    cy.getByTestId('submit').should('not.be.disabled');
  });

  it('should show error for invalid credentials', () => {
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 401,
      body: { message: 'Invalid credentials' },
    }).as('loginRequest');

    cy.getByTestId('username').type('wronguser');
    cy.getByTestId('password').type('wrongpass');
    cy.getByTestId('submit').click();

    cy.wait('@loginRequest');
    cy.getByTestId('error-message')
      .should('be.visible')
      .and('contain', 'Invalid credentials');
  });

  it('should redirect to dashboard on successful login', () => {
    cy.intercept('POST', '/api/auth/login', {
      statusCode: 200,
      body: { token: 'fake-token', user: { id: 1, name: 'Test User' } },
    }).as('loginRequest');

    cy.getByTestId('username').type('testuser');
    cy.getByTestId('password').type('password123');
    cy.getByTestId('submit').click();

    cy.wait('@loginRequest');
    cy.url().should('include', '/dashboard');
    cy.getByTestId('welcome-message').should('contain', 'Test User');
  });
});
```

### 订单流程测试

```typescript
// cypress/e2e/order/create-order.cy.ts
describe('Create Order', () => {
  beforeEach(() => {
    cy.login('testuser', 'password123');
    cy.visit('/products');
  });

  it('should complete order flow', () => {
    // 选择商品
    cy.getByTestId('product-card').first().click();
    cy.getByTestId('add-to-cart').click();
    cy.getByTestId('cart-count').should('contain', '1');

    // 进入购物车
    cy.getByTestId('cart-icon').click();
    cy.url().should('include', '/cart');

    // 修改数量
    cy.getByTestId('quantity-input').clear().type('2');
    cy.getByTestId('cart-total').should('not.be.empty');

    // 结算
    cy.getByTestId('checkout-btn').click();
    cy.url().should('include', '/checkout');

    // 填写收货地址
    cy.getByTestId('address-name').type('John Doe');
    cy.getByTestId('address-phone').type('13800138000');
    cy.getByTestId('address-detail').type('123 Main St');

    // 选择支付方式
    cy.getByTestId('payment-alipay').click();

    // 提交订单
    cy.intercept('POST', '/api/orders', {
      statusCode: 200,
      body: { orderId: 'ORD123456' },
    }).as('createOrder');

    cy.getByTestId('submit-order').click();

    cy.wait('@createOrder');
    cy.url().should('include', '/order/success');
    cy.getByTestId('order-id').should('contain', 'ORD123456');
  });
});
```

### API 拦截测试

```typescript
// cypress/e2e/api/data-loading.cy.ts
describe('Data Loading', () => {
  it('should show loading state', () => {
    cy.intercept('GET', '/api/users', {
      delay: 2000,
      fixture: 'users.json',
    }).as('getUsers');

    cy.login('admin', 'admin123');
    cy.visit('/users');

    cy.getByTestId('loading-spinner').should('be.visible');
    cy.wait('@getUsers');
    cy.getByTestId('loading-spinner').should('not.exist');
    cy.getByTestId('user-list').should('be.visible');
  });

  it('should handle error state', () => {
    cy.intercept('GET', '/api/users', {
      statusCode: 500,
      body: { error: 'Server error' },
    }).as('getUsers');

    cy.login('admin', 'admin123');
    cy.visit('/users');

    cy.wait('@getUsers');
    cy.getByTestId('error-message')
      .should('be.visible')
      .and('contain', 'Failed to load');
    cy.getByTestId('retry-btn').should('be.visible');
  });

  it('should retry on failure', () => {
    let requestCount = 0;
    cy.intercept('GET', '/api/users', (req) => {
      requestCount++;
      if (requestCount < 2) {
        req.reply({ statusCode: 500 });
      } else {
        req.reply({ fixture: 'users.json' });
      }
    }).as('getUsers');

    cy.login('admin', 'admin123');
    cy.visit('/users');

    cy.wait('@getUsers');
    cy.getByTestId('retry-btn').click();
    cy.wait('@getUsers');
    cy.getByTestId('user-list').should('be.visible');
  });
});
```

### 组件测试

```typescript
// src/components/Button/Button.cy.tsx
import { Button } from './Button';

describe('Button', () => {
  it('renders with text', () => {
    cy.mount(<Button>Click me</Button>);
    cy.get('button').should('contain', 'Click me');
  });

  it('handles click events', () => {
    const onClick = cy.stub().as('clickHandler');
    cy.mount(<Button onClick={onClick}>Click me</Button>);

    cy.get('button').click();
    cy.get('@clickHandler').should('have.been.calledOnce');
  });

  it('can be disabled', () => {
    const onClick = cy.stub().as('clickHandler');
    cy.mount(<Button disabled onClick={onClick}>Disabled</Button>);

    cy.get('button')
      .should('be.disabled')
      .click({ force: true });
    cy.get('@clickHandler').should('not.have.been.called');
  });

  it('shows loading state', () => {
    cy.mount(<Button loading>Loading</Button>);

    cy.get('button').should('be.disabled');
    cy.get('[data-testid="spinner"]').should('be.visible');
  });

  it('renders different variants', () => {
    cy.mount(<Button variant="primary">Primary</Button>);
    cy.get('button').should('have.class', 'bg-blue-500');

    cy.mount(<Button variant="danger">Danger</Button>);
    cy.get('button').should('have.class', 'bg-red-500');
  });
});
```

### 表单组件测试

```typescript
// src/components/Form/LoginForm.cy.tsx
import { LoginForm } from './LoginForm';

describe('LoginForm', () => {
  it('validates required fields', () => {
    const onSubmit = cy.stub().as('submit');
    cy.mount(<LoginForm onSubmit={onSubmit} />);

    cy.get('button[type="submit"]').click();

    cy.get('[data-testid="username-error"]')
      .should('be.visible')
      .and('contain', 'Username is required');
    cy.get('[data-testid="password-error"]')
      .should('be.visible')
      .and('contain', 'Password is required');
    cy.get('@submit').should('not.have.been.called');
  });

  it('validates password length', () => {
    cy.mount(<LoginForm onSubmit={cy.stub()} />);

    cy.get('[data-testid="username"]').type('user');
    cy.get('[data-testid="password"]').type('123');
    cy.get('button[type="submit"]').click();

    cy.get('[data-testid="password-error"]')
      .should('contain', 'at least 6 characters');
  });

  it('submits valid form', () => {
    const onSubmit = cy.stub().as('submit');
    cy.mount(<LoginForm onSubmit={onSubmit} />);

    cy.get('[data-testid="username"]').type('testuser');
    cy.get('[data-testid="password"]').type('password123');
    cy.get('button[type="submit"]').click();

    cy.get('@submit').should('have.been.calledWith', {
      username: 'testuser',
      password: 'password123',
    });
  });
});
```

### Fixture 数据

```json
// cypress/fixtures/users.json
{
  "users": [
    { "id": 1, "name": "John Doe", "email": "john@example.com" },
    { "id": 2, "name": "Jane Smith", "email": "jane@example.com" }
  ],
  "total": 2
}
```

### CI 配置

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  cypress:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Cypress run
        uses: cypress-io/github-action@v6
        with:
          build: npm run build
          start: npm run preview
          wait-on: 'http://localhost:4173'
          browser: chrome
          record: true
        env:
          CYPRESS_RECORD_KEY: ${{ secrets.CYPRESS_RECORD_KEY }}

      - name: Upload screenshots
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: cypress-screenshots
          path: cypress/screenshots

      - name: Upload videos
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: cypress-videos
          path: cypress/videos
```

## 最佳实践清单

- [ ] 使用 data-testid 选择器
- [ ] 封装自定义命令
- [ ] 使用 cy.session() 复用登录
- [ ] Mock API 请求
- [ ] 使用 Fixture 管理测试数据
- [ ] 配置 CI 自动化测试
- [ ] 截图和视频记录
- [ ] 合理设置超时时间

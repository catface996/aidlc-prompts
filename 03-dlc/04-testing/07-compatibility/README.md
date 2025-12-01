# 兼容性测试最佳实践

## 角色设定

你是一位精通兼容性测试的质量工程师，擅长浏览器兼容性、设备兼容性、系统兼容性和版本兼容性测试。

## 提示词模板

### 兼容性测试设计

```
请帮我设计兼容性测试方案：
- 测试目标：[Web应用/移动应用/桌面应用]
- 兼容范围：[浏览器/操作系统/设备/版本]
- 测试工具：[BrowserStack/Sauce Labs/自动化]
- 优先级：[核心功能/边缘场景]

请提供测试矩阵和自动化脚本。
```

## 核心代码示例

### 浏览器兼容性测试 (Playwright)

```typescript
// browser-compatibility.spec.ts
import { test, expect, devices } from '@playwright/test';

// 浏览器配置矩阵
const browserConfigs = [
  { name: 'chromium', channel: undefined },
  { name: 'chromium', channel: 'chrome' },
  { name: 'chromium', channel: 'msedge' },
  { name: 'firefox', channel: undefined },
  { name: 'webkit', channel: undefined },
];

test.describe('跨浏览器兼容性测试', () => {
  test('首页在所有浏览器正常渲染', async ({ page, browserName }) => {
    await page.goto('/');

    // 验证关键元素
    await expect(page.locator('header')).toBeVisible();
    await expect(page.locator('nav')).toBeVisible();
    await expect(page.locator('main')).toBeVisible();
    await expect(page.locator('footer')).toBeVisible();

    // 截图对比
    await expect(page).toHaveScreenshot(`homepage-${browserName}.png`, {
      maxDiffPixels: 100,
    });
  });

  test('表单提交功能正常', async ({ page }) => {
    await page.goto('/contact');

    await page.fill('[data-testid="name"]', '测试用户');
    await page.fill('[data-testid="email"]', 'test@example.com');
    await page.fill('[data-testid="message"]', '这是测试消息');

    await page.click('[data-testid="submit"]');

    await expect(page.locator('[data-testid="success-message"]')).toBeVisible();
  });

  test('CSS 动画和过渡效果', async ({ page }) => {
    await page.goto('/animations');

    // 触发动画
    await page.click('[data-testid="animate-button"]');

    // 等待动画完成
    await page.waitForTimeout(1000);

    // 验证动画后状态
    const element = page.locator('[data-testid="animated-element"]');
    await expect(element).toHaveCSS('transform', 'matrix(1, 0, 0, 1, 100, 0)');
  });

  test('Flexbox 布局兼容性', async ({ page }) => {
    await page.goto('/layout-test');

    const flexContainer = page.locator('[data-testid="flex-container"]');
    await expect(flexContainer).toHaveCSS('display', 'flex');

    // 验证子元素排列
    const children = flexContainer.locator('> *');
    const count = await children.count();
    expect(count).toBeGreaterThan(0);
  });

  test('Grid 布局兼容性', async ({ page }) => {
    await page.goto('/layout-test');

    const gridContainer = page.locator('[data-testid="grid-container"]');
    await expect(gridContainer).toHaveCSS('display', 'grid');
  });
});
```

### 响应式设计测试

```typescript
// responsive.spec.ts
import { test, expect, devices } from '@playwright/test';

// 设备配置
const deviceConfigs = [
  { name: 'Desktop 1920x1080', viewport: { width: 1920, height: 1080 } },
  { name: 'Desktop 1366x768', viewport: { width: 1366, height: 768 } },
  { name: 'Tablet Landscape', viewport: { width: 1024, height: 768 } },
  { name: 'Tablet Portrait', viewport: { width: 768, height: 1024 } },
  { name: 'Mobile Large', viewport: { width: 414, height: 896 } },
  { name: 'Mobile Medium', viewport: { width: 375, height: 667 } },
  { name: 'Mobile Small', viewport: { width: 320, height: 568 } },
];

test.describe('响应式设计测试', () => {
  for (const config of deviceConfigs) {
    test(`${config.name} 视口布局正确`, async ({ page }) => {
      await page.setViewportSize(config.viewport);
      await page.goto('/');

      // 截图
      await expect(page).toHaveScreenshot(`responsive-${config.name}.png`);

      // 验证导航
      if (config.viewport.width < 768) {
        // 移动端：汉堡菜单
        await expect(page.locator('[data-testid="mobile-menu-button"]')).toBeVisible();
        await expect(page.locator('[data-testid="desktop-nav"]')).not.toBeVisible();
      } else {
        // 桌面端：完整导航
        await expect(page.locator('[data-testid="desktop-nav"]')).toBeVisible();
      }
    });
  }

  test('触摸设备交互', async ({ page }) => {
    // 模拟 iPhone 12
    await page.setViewportSize(devices['iPhone 12'].viewport);
    await page.goto('/');

    // 测试触摸滑动
    const carousel = page.locator('[data-testid="carousel"]');
    if (await carousel.isVisible()) {
      await carousel.evaluate((el) => {
        el.dispatchEvent(new TouchEvent('touchstart', {
          touches: [{ clientX: 200, clientY: 200 }] as any,
        }));
        el.dispatchEvent(new TouchEvent('touchend', {
          touches: [{ clientX: 50, clientY: 200 }] as any,
        }));
      });
    }
  });
});

// Playwright 配置文件
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
    { name: 'Mobile Chrome', use: { ...devices['Pixel 5'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 12'] } },
    { name: 'Tablet', use: { ...devices['iPad Pro 11'] } },
  ],
});
```

### CSS 特性检测

```javascript
// feature-detection.js
export class FeatureDetector {
  static detectAll() {
    return {
      // CSS 特性
      flexbox: this.detectFlexbox(),
      grid: this.detectGrid(),
      cssVariables: this.detectCSSVariables(),
      position: this.detectStickyPosition(),
      backdrop: this.detectBackdropFilter(),

      // JavaScript API
      fetch: this.detectFetch(),
      promise: this.detectPromise(),
      intersectionObserver: this.detectIntersectionObserver(),
      resizeObserver: this.detectResizeObserver(),
      webgl: this.detectWebGL(),

      // 存储
      localStorage: this.detectLocalStorage(),
      sessionStorage: this.detectSessionStorage(),
      indexedDB: this.detectIndexedDB(),
    };
  }

  static detectFlexbox() {
    const el = document.createElement('div');
    return 'flex' in el.style || 'webkitFlex' in el.style;
  }

  static detectGrid() {
    const el = document.createElement('div');
    return 'grid' in el.style || 'msGrid' in el.style;
  }

  static detectCSSVariables() {
    return window.CSS && CSS.supports('color', 'var(--test)');
  }

  static detectStickyPosition() {
    const el = document.createElement('div');
    el.style.position = 'sticky';
    return el.style.position === 'sticky';
  }

  static detectBackdropFilter() {
    const el = document.createElement('div');
    el.style.backdropFilter = 'blur(10px)';
    return el.style.backdropFilter !== '';
  }

  static detectFetch() {
    return 'fetch' in window;
  }

  static detectPromise() {
    return 'Promise' in window;
  }

  static detectIntersectionObserver() {
    return 'IntersectionObserver' in window;
  }

  static detectResizeObserver() {
    return 'ResizeObserver' in window;
  }

  static detectWebGL() {
    try {
      const canvas = document.createElement('canvas');
      return !!(canvas.getContext('webgl') || canvas.getContext('experimental-webgl'));
    } catch (e) {
      return false;
    }
  }

  static detectLocalStorage() {
    try {
      localStorage.setItem('test', 'test');
      localStorage.removeItem('test');
      return true;
    } catch (e) {
      return false;
    }
  }

  static detectSessionStorage() {
    try {
      sessionStorage.setItem('test', 'test');
      sessionStorage.removeItem('test');
      return true;
    } catch (e) {
      return false;
    }
  }

  static detectIndexedDB() {
    return 'indexedDB' in window;
  }
}

// 兼容性测试用例
// feature-detection.spec.ts
import { test, expect } from '@playwright/test';

test.describe('CSS 特性兼容性', () => {
  test('检测浏览器特性支持', async ({ page }) => {
    await page.goto('/');

    const features = await page.evaluate(() => {
      // 内联 FeatureDetector 逻辑或注入脚本
      return {
        flexbox: 'flex' in document.createElement('div').style,
        grid: 'grid' in document.createElement('div').style,
        cssVariables: window.CSS && CSS.supports('color', 'var(--test)'),
      };
    });

    console.log('Browser features:', features);

    // 核心特性必须支持
    expect(features.flexbox).toBe(true);
  });
});
```

### API 版本兼容性测试

```java
// ApiVersionCompatibilityTest.java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ApiVersionCompatibilityTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    @DisplayName("API v1 向后兼容")
    void shouldMaintainV1Compatibility() throws Exception {
        // v1 API 请求
        mockMvc.perform(get("/api/v1/users/1")
                .header("Accept", "application/json"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.username").exists())
            .andExpect(jsonPath("$.email").exists());
    }

    @Test
    @DisplayName("API v2 新增字段")
    void shouldReturnV2WithNewFields() throws Exception {
        // v2 API 响应包含新字段
        mockMvc.perform(get("/api/v2/users/1")
                .header("Accept", "application/json"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").exists())
            .andExpect(jsonPath("$.username").exists())
            .andExpect(jsonPath("$.email").exists())
            .andExpect(jsonPath("$.createdAt").exists())  // v2 新增
            .andExpect(jsonPath("$.lastLoginAt").exists());  // v2 新增
    }

    @Test
    @DisplayName("通过 Header 指定 API 版本")
    void shouldSelectVersionByHeader() throws Exception {
        // 使用 Accept-Version header
        mockMvc.perform(get("/api/users/1")
                .header("Accept-Version", "v1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.createdAt").doesNotExist());

        mockMvc.perform(get("/api/users/1")
                .header("Accept-Version", "v2"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.createdAt").exists());
    }

    @Test
    @DisplayName("废弃字段警告")
    void shouldWarnAboutDeprecatedFields() throws Exception {
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(header().exists("Deprecation"))
            .andExpect(header().string("Sunset", notNullValue()));
    }
}
```

### 数据库兼容性测试

```java
// DatabaseCompatibilityTest.java
@SpringBootTest
@Testcontainers
class DatabaseCompatibilityTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb");

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureMySQL(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Test
    @DisplayName("MySQL 8.0 兼容性")
    void shouldWorkWithMySQL8() {
        assertThat(mysql.isRunning()).isTrue();
        // 执行数据库操作测试
    }
}

// PostgreSQL 测试配置类
@SpringBootTest
@Testcontainers
@TestPropertySource(properties = {"spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect"})
class PostgreSQLCompatibilityTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @DynamicPropertySource
    static void configure(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Test
    @DisplayName("PostgreSQL 特有功能")
    void shouldWorkWithPostgreSQLFeatures() {
        // 测试 PostgreSQL 特有的 JSON 操作等
        User user = userRepository.save(User.builder()
            .username("test")
            .metadata("{\"key\": \"value\"}")
            .build());

        assertThat(user.getId()).isNotNull();
    }
}
```

### BrowserStack 集成

```javascript
// browserstack.config.js
exports.config = {
  user: process.env.BROWSERSTACK_USERNAME,
  key: process.env.BROWSERSTACK_ACCESS_KEY,

  services: ['browserstack'],

  capabilities: [
    // Windows + Chrome
    {
      browserName: 'chrome',
      'bstack:options': {
        os: 'Windows',
        osVersion: '11',
        browserVersion: 'latest',
        seleniumVersion: '4.0.0',
      },
    },
    // Windows + Firefox
    {
      browserName: 'firefox',
      'bstack:options': {
        os: 'Windows',
        osVersion: '11',
        browserVersion: 'latest',
      },
    },
    // macOS + Safari
    {
      browserName: 'safari',
      'bstack:options': {
        os: 'OS X',
        osVersion: 'Ventura',
        browserVersion: 'latest',
      },
    },
    // iOS Safari
    {
      browserName: 'safari',
      'bstack:options': {
        deviceName: 'iPhone 14',
        osVersion: '16',
        realMobile: true,
      },
    },
    // Android Chrome
    {
      browserName: 'chrome',
      'bstack:options': {
        deviceName: 'Samsung Galaxy S23',
        osVersion: '13.0',
        realMobile: true,
      },
    },
  ],

  maxInstances: 5,

  framework: 'mocha',
  mochaOpts: {
    timeout: 60000,
  },

  specs: ['./tests/compatibility/**/*.spec.js'],

  beforeSession: function (config, capabilities) {
    capabilities['bstack:options'].projectName = 'E-Commerce App';
    capabilities['bstack:options'].buildName = `Build ${process.env.BUILD_NUMBER}`;
  },
};
```

### CI/CD 兼容性测试配置

```yaml
# .github/workflows/compatibility.yml
name: Compatibility Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # 每周一凌晨2点

jobs:
  browser-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright
        run: npx playwright install --with-deps ${{ matrix.browser }}

      - name: Run tests
        run: npx playwright test --project=${{ matrix.browser }}

      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.browser }}
          path: playwright-report/

  mobile-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Run mobile compatibility tests
        env:
          BROWSERSTACK_USERNAME: ${{ secrets.BROWSERSTACK_USERNAME }}
          BROWSERSTACK_ACCESS_KEY: ${{ secrets.BROWSERSTACK_ACCESS_KEY }}
        run: npx wdio browserstack.config.js

  database-tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        database:
          - mysql:8.0
          - mysql:5.7
          - postgres:15
          - postgres:14
    services:
      database:
        image: ${{ matrix.database }}
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: testdb
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 3306:3306
          - 5432:5432
        options: >-
          --health-cmd "mysqladmin ping || pg_isready"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4

      - name: Setup Java
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run database tests
        run: ./gradlew test --tests "*CompatibilityTest"
```

## 兼容性测试矩阵

### 浏览器支持矩阵

| 浏览器 | 最低版本 | 推荐版本 | 测试优先级 |
|--------|----------|----------|------------|
| Chrome | 90+ | Latest | 高 |
| Firefox | 88+ | Latest | 高 |
| Safari | 14+ | Latest | 高 |
| Edge | 90+ | Latest | 中 |
| IE | 不支持 | - | - |

### 设备支持矩阵

| 设备类型 | 分辨率范围 | 测试场景 |
|----------|------------|----------|
| 桌面大屏 | 1920x1080+ | 完整功能 |
| 桌面标准 | 1366x768 | 核心功能 |
| 平板横屏 | 1024x768 | 触控交互 |
| 平板竖屏 | 768x1024 | 布局适配 |
| 手机大屏 | 414x896 | 移动体验 |
| 手机标准 | 375x667 | 核心流程 |
| 手机小屏 | 320x568 | 最小适配 |

## 最佳实践清单

- [ ] 定义浏览器支持范围
- [ ] 建立设备测试矩阵
- [ ] 使用特性检测而非浏览器检测
- [ ] 自动化跨浏览器测试
- [ ] 响应式设计测试
- [ ] API 版本兼容性测试
- [ ] 数据库多版本测试
- [ ] CI/CD 集成兼容性测试

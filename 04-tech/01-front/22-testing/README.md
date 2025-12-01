# 前端测试最佳实践

## 角色设定

你是一位精通前端测试的专家，擅长单元测试、集成测试、E2E 测试和测试策略设计。

## 提示词模板

### 编写测试

```
请帮我为以下代码编写测试：
[粘贴代码]

测试类型：[单元测试/集成测试/E2E]
测试框架：[Jest/Vitest/Cypress/Playwright]
覆盖范围：
- [ ] 正常流程
- [ ] 边界情况
- [ ] 错误处理
- [ ] 异步操作
```

### 测试策略

```
请帮我设计测试策略：
- 项目类型：[Web应用/组件库/工具库]
- 技术栈：[React/Vue/Node.js]
- CI/CD：[GitHub Actions/GitLab CI]

需要覆盖的场景：
1. [场景1]
2. [场景2]
```

## 核心测试示例

### Jest/Vitest 单元测试

```typescript
// utils/format.test.ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { formatDate, formatCurrency, debounce } from './format';

describe('formatDate', () => {
  it('should format date correctly', () => {
    const date = new Date('2024-01-15');
    expect(formatDate(date)).toBe('2024-01-15');
  });

  it('should handle invalid date', () => {
    expect(formatDate(null)).toBe('');
    expect(formatDate(undefined)).toBe('');
  });

  it('should support custom format', () => {
    const date = new Date('2024-01-15');
    expect(formatDate(date, 'YYYY/MM/DD')).toBe('2024/01/15');
  });
});

describe('formatCurrency', () => {
  it.each([
    [1000, '$1,000.00'],
    [1234.5, '$1,234.50'],
    [0, '$0.00'],
    [-100, '-$100.00'],
  ])('should format %i as %s', (input, expected) => {
    expect(formatCurrency(input)).toBe(expected);
  });
});

describe('debounce', () => {
  beforeEach(() => {
    vi.useFakeTimers();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('should debounce function calls', () => {
    const fn = vi.fn();
    const debouncedFn = debounce(fn, 100);

    debouncedFn();
    debouncedFn();
    debouncedFn();

    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(100);

    expect(fn).toHaveBeenCalledTimes(1);
  });
});
```

### React 组件测试

```tsx
// components/Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { Button } from './Button';

describe('Button', () => {
  it('should render with text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument();
  });

  it('should call onClick when clicked', async () => {
    const handleClick = vi.fn();
    const user = userEvent.setup();

    render(<Button onClick={handleClick}>Click me</Button>);

    await user.click(screen.getByRole('button'));

    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('should be disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);

    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('should show loading state', () => {
    render(<Button loading>Click me</Button>);

    expect(screen.getByRole('button')).toHaveAttribute('aria-busy', 'true');
    expect(screen.getByTestId('spinner')).toBeInTheDocument();
  });

  it('should apply variant styles', () => {
    const { rerender } = render(<Button variant="primary">Button</Button>);
    expect(screen.getByRole('button')).toHaveClass('btn-primary');

    rerender(<Button variant="secondary">Button</Button>);
    expect(screen.getByRole('button')).toHaveClass('btn-secondary');
  });
});
```

### 异步组件测试

```tsx
// components/UserProfile.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { UserProfile } from './UserProfile';
import { fetchUser } from '@/api/user';

vi.mock('@/api/user');

describe('UserProfile', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should show loading state initially', () => {
    vi.mocked(fetchUser).mockImplementation(
      () => new Promise(() => {}) // 永不 resolve
    );

    render(<UserProfile userId="1" />);

    expect(screen.getByTestId('loading')).toBeInTheDocument();
  });

  it('should display user data after loading', async () => {
    vi.mocked(fetchUser).mockResolvedValue({
      id: '1',
      name: 'John Doe',
      email: 'john@example.com',
    });

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
    });

    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('should show error message on fetch failure', async () => {
    vi.mocked(fetchUser).mockRejectedValue(new Error('Network error'));

    render(<UserProfile userId="1" />);

    await waitFor(() => {
      expect(screen.getByText(/error/i)).toBeInTheDocument();
    });
  });
});
```

### Hook 测试

```tsx
// hooks/useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('should initialize with default value', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('should initialize with custom value', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('should increment count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });

  it('should decrement count', () => {
    const { result } = renderHook(() => useCounter(5));

    act(() => {
      result.current.decrement();
    });

    expect(result.current.count).toBe(4);
  });

  it('should update on props change', () => {
    const { result, rerender } = renderHook(
      ({ initial }) => useCounter(initial),
      { initialProps: { initial: 0 } }
    );

    expect(result.current.count).toBe(0);

    rerender({ initial: 10 });

    expect(result.current.count).toBe(10);
  });
});
```

### Mock 技巧

```typescript
// 模拟模块
vi.mock('@/api/user', () => ({
  fetchUser: vi.fn(),
  updateUser: vi.fn(),
}));

// 模拟部分模块
vi.mock('@/utils', async () => {
  const actual = await vi.importActual('@/utils');
  return {
    ...actual,
    formatDate: vi.fn(() => '2024-01-01'),
  };
});

// 模拟 fetch
global.fetch = vi.fn(() =>
  Promise.resolve({
    ok: true,
    json: () => Promise.resolve({ data: 'test' }),
  })
) as unknown as typeof fetch;

// 模拟 localStorage
const localStorageMock = {
  getItem: vi.fn(),
  setItem: vi.fn(),
  removeItem: vi.fn(),
  clear: vi.fn(),
};
Object.defineProperty(window, 'localStorage', { value: localStorageMock });

// 模拟 IntersectionObserver
const mockIntersectionObserver = vi.fn().mockImplementation((callback) => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}));
window.IntersectionObserver = mockIntersectionObserver;
```

### E2E 测试 (Playwright)

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('should display login form', async ({ page }) => {
    await expect(page.getByRole('heading', { name: /login/i })).toBeVisible();
    await expect(page.getByLabel(/email/i)).toBeVisible();
    await expect(page.getByLabel(/password/i)).toBeVisible();
    await expect(page.getByRole('button', { name: /sign in/i })).toBeVisible();
  });

  test('should show validation errors', async ({ page }) => {
    await page.getByRole('button', { name: /sign in/i }).click();

    await expect(page.getByText(/email is required/i)).toBeVisible();
    await expect(page.getByText(/password is required/i)).toBeVisible();
  });

  test('should login successfully', async ({ page }) => {
    await page.getByLabel(/email/i).fill('test@example.com');
    await page.getByLabel(/password/i).fill('password123');
    await page.getByRole('button', { name: /sign in/i }).click();

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText(/welcome/i)).toBeVisible();
  });

  test('should show error for invalid credentials', async ({ page }) => {
    await page.getByLabel(/email/i).fill('wrong@example.com');
    await page.getByLabel(/password/i).fill('wrongpassword');
    await page.getByRole('button', { name: /sign in/i }).click();

    await expect(page.getByText(/invalid credentials/i)).toBeVisible();
  });
});

// e2e/fixtures.ts
import { test as base } from '@playwright/test';

export const test = base.extend({
  authenticatedPage: async ({ page }, use) => {
    await page.goto('/login');
    await page.getByLabel(/email/i).fill('test@example.com');
    await page.getByLabel(/password/i).fill('password123');
    await page.getByRole('button', { name: /sign in/i }).click();
    await page.waitForURL('/dashboard');
    await use(page);
  },
});
```

### 测试配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      exclude: ['**/*.d.ts', '**/*.test.ts', '**/test/**'],
    },
    include: ['src/**/*.{test,spec}.{ts,tsx}'],
  },
});

// src/test/setup.ts
import '@testing-library/jest-dom';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';

afterEach(() => {
  cleanup();
});
```

## 测试金字塔

```
        /\
       /  \    E2E Tests (少量)
      /----\
     /      \  Integration Tests (中等)
    /--------\
   /          \ Unit Tests (大量)
  --------------
```

## 最佳实践清单

- [ ] 按照测试金字塔分配测试
- [ ] 测试行为而非实现细节
- [ ] 使用有意义的测试描述
- [ ] 每个测试只验证一件事
- [ ] 避免测试实现细节
- [ ] 合理使用 Mock
- [ ] 保持测试独立性
- [ ] 集成到 CI/CD 流程

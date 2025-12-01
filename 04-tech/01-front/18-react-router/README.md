# React Router 路由最佳实践

## 角色设定

你是一位精通 React Router v6 的前端路由专家，擅长路由设计、数据加载和导航优化。

## 提示词模板

### 路由配置

```
请帮我设计 React Router 路由配置：
- 路由结构：[描述页面层级]
- 认证需求：[公开/私有路由]
- 数据加载：[loader/自行获取]
- 布局需求：[嵌套布局/独立布局]

页面列表：
1. [页面1]
2. [页面2]
```

### 路由守卫

```
请帮我实现路由守卫：
- 守卫类型：[认证/权限/数据预加载]
- 检查逻辑：[描述检查逻辑]
- 失败处理：[重定向/提示]
```

## 核心代码示例

### 基础路由配置

```tsx
// router/index.tsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';
import RootLayout from '@/layouts/RootLayout';
import Home from '@/pages/Home';
import About from '@/pages/About';
import NotFound from '@/pages/NotFound';

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <NotFound />,
    children: [
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },
      {
        path: 'products',
        children: [
          { index: true, element: <ProductList /> },
          { path: ':id', element: <ProductDetail /> },
        ],
      },
    ],
  },
]);

export default function AppRouter() {
  return <RouterProvider router={router} />;
}
```

### 带 Loader 的数据预加载

```tsx
// router/index.tsx
import { createBrowserRouter, defer } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: 'products/:id',
    element: <ProductDetail />,
    loader: async ({ params }) => {
      const product = await fetchProduct(params.id);
      return { product };
    },
  },
  {
    path: 'dashboard',
    element: <Dashboard />,
    loader: async () => {
      // 延迟加载非关键数据
      return defer({
        user: fetchUser(),
        stats: fetchStats(), // 可以后加载
      });
    },
  },
]);

// 页面组件使用 loader 数据
import { useLoaderData, Await, Suspense } from 'react-router-dom';

function ProductDetail() {
  const { product } = useLoaderData() as { product: Product };
  return <div>{product.name}</div>;
}

function Dashboard() {
  const { user, stats } = useLoaderData() as { user: Promise<User>; stats: Promise<Stats> };

  return (
    <div>
      <Suspense fallback={<Loading />}>
        <Await resolve={user}>
          {(resolvedUser) => <UserInfo user={resolvedUser} />}
        </Await>
      </Suspense>

      <Suspense fallback={<StatsLoading />}>
        <Await resolve={stats}>
          {(resolvedStats) => <StatsPanel stats={resolvedStats} />}
        </Await>
      </Suspense>
    </div>
  );
}
```

### Action 处理表单

```tsx
// router/index.tsx
const router = createBrowserRouter([
  {
    path: 'contact',
    element: <ContactForm />,
    action: async ({ request }) => {
      const formData = await request.formData();
      const data = Object.fromEntries(formData);

      try {
        await submitContact(data);
        return redirect('/contact/success');
      } catch (error) {
        return { error: (error as Error).message };
      }
    },
  },
]);

// 表单组件
import { Form, useActionData, useNavigation } from 'react-router-dom';

function ContactForm() {
  const actionData = useActionData() as { error?: string } | undefined;
  const navigation = useNavigation();
  const isSubmitting = navigation.state === 'submitting';

  return (
    <Form method="post">
      <input name="name" required />
      <input name="email" type="email" required />
      <textarea name="message" required />

      {actionData?.error && <p className="error">{actionData.error}</p>}

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Sending...' : 'Send'}
      </button>
    </Form>
  );
}
```

### 认证路由守卫

```tsx
// components/ProtectedRoute.tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';

export function ProtectedRoute() {
  const { isAuthenticated, isLoading } = useAuth();
  const location = useLocation();

  if (isLoading) {
    return <LoadingSpinner />;
  }

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  return <Outlet />;
}

// 权限路由守卫
export function RoleRoute({ allowedRoles }: { allowedRoles: string[] }) {
  const { user } = useAuth();

  if (!user || !allowedRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return <Outlet />;
}

// 路由配置使用
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      // 公开路由
      { path: 'login', element: <Login /> },
      { path: 'register', element: <Register /> },

      // 需要认证的路由
      {
        element: <ProtectedRoute />,
        children: [
          { path: 'dashboard', element: <Dashboard /> },
          { path: 'profile', element: <Profile /> },

          // 需要特定角色
          {
            element: <RoleRoute allowedRoles={['admin']} />,
            children: [
              { path: 'admin', element: <AdminPanel /> },
            ],
          },
        ],
      },
    ],
  },
]);
```

### 懒加载路由

```tsx
// router/index.tsx
import { lazy, Suspense } from 'react';
import { createBrowserRouter } from 'react-router-dom';

// 懒加载页面
const Dashboard = lazy(() => import('@/pages/Dashboard'));
const Settings = lazy(() => import('@/pages/Settings'));
const Analytics = lazy(() => import('@/pages/Analytics'));

// 加载组件包装
function LazyPage({ component: Component }: { component: React.LazyExoticComponent<any> }) {
  return (
    <Suspense fallback={<PageLoading />}>
      <Component />
    </Suspense>
  );
}

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      {
        path: 'dashboard',
        element: <LazyPage component={Dashboard} />,
      },
      {
        path: 'settings',
        element: <LazyPage component={Settings} />,
      },
      {
        path: 'analytics',
        element: <LazyPage component={Analytics} />,
      },
    ],
  },
]);
```

### 常用 Hooks

```tsx
import {
  useNavigate,
  useLocation,
  useParams,
  useSearchParams,
  useMatch,
  useOutletContext,
} from 'react-router-dom';

function MyComponent() {
  // 编程式导航
  const navigate = useNavigate();
  navigate('/dashboard');
  navigate(-1); // 返回
  navigate('/login', { replace: true, state: { from: location } });

  // 当前位置
  const location = useLocation();
  // { pathname, search, hash, state, key }

  // 路由参数
  const { id } = useParams<{ id: string }>();

  // 查询参数
  const [searchParams, setSearchParams] = useSearchParams();
  const page = searchParams.get('page') || '1';
  setSearchParams({ page: '2' });

  // 路由匹配
  const match = useMatch('/users/:id');
  // { params: { id }, pathname, pattern }

  // Outlet 上下文
  const context = useOutletContext<{ user: User }>();

  return <div>...</div>;
}
```

### 布局嵌套

```tsx
// layouts/RootLayout.tsx
import { Outlet, ScrollRestoration } from 'react-router-dom';

export default function RootLayout() {
  return (
    <div className="app">
      <Header />
      <main>
        <Outlet />
      </main>
      <Footer />
      <ScrollRestoration />
    </div>
  );
}

// layouts/DashboardLayout.tsx
export default function DashboardLayout() {
  return (
    <div className="dashboard">
      <Sidebar />
      <div className="dashboard-content">
        <Outlet context={{ user }} />
      </div>
    </div>
  );
}

// 路由配置
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      { index: true, element: <Home /> },
      {
        path: 'dashboard',
        element: <DashboardLayout />,
        children: [
          { index: true, element: <DashboardHome /> },
          { path: 'settings', element: <Settings /> },
        ],
      },
    ],
  },
]);
```

## 最佳实践清单

- [ ] 使用 createBrowserRouter 配置路由
- [ ] 利用 loader 预加载数据
- [ ] 使用 action 处理表单提交
- [ ] 实现认证和权限路由守卫
- [ ] 使用 lazy 懒加载页面组件
- [ ] 合理使用嵌套布局
- [ ] 使用 ScrollRestoration 恢复滚动位置
- [ ] 处理错误页面 (errorElement)

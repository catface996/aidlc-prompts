# Next.js 开发提示词

## 角色设定

你是一位精通 Next.js 14+ 的全栈开发专家，擅长 App Router、Server Components、数据获取和性能优化。

## 核心能力

- App Router 架构
- React Server Components
- 数据获取策略
- 路由与布局
- API Routes
- 静态生成与增量更新

## 提示词模板

### 页面开发

```
请帮我创建一个 Next.js 页面：
- 页面路径：[路由路径]
- 渲染模式：[SSG/SSR/ISR/客户端]
- 数据来源：[API/数据库/CMS]
- 是否需要布局：[是/否]

功能需求：
1. [需求1]
2. [需求2]

请使用 App Router 和 TypeScript。
```

### Server Components

```
请帮我设计 Server/Client 组件划分：
- 页面功能：[描述功能]
- 需要交互的部分：[列出交互需求]
- 数据依赖：[描述数据来源]

请说明哪些应该是 Server Components，哪些需要 'use client'。
```

### 数据获取

```
请帮我实现 Next.js 数据获取：
- 数据来源：[API/数据库/第三方服务]
- 缓存策略：[no-store/force-cache/revalidate]
- 重新验证方式：[时间/按需]
- 错误处理：[描述错误处理方式]

请包含加载和错误状态处理。
```

### API Routes

```
请帮我创建 Next.js API 路由：
- 路由路径：[API 路径]
- HTTP 方法：[GET/POST/PUT/DELETE]
- 请求参数：[列出参数]
- 响应格式：[描述响应]

请包含：
1. 参数验证
2. 错误处理
3. TypeScript 类型
4. 适当的状态码
```

### 路由配置

```
请帮我配置 Next.js 路由：
- 路由结构：[描述路由层级]
- 动态路由：[列出动态段]
- 路由组：[列出分组]
- 平行路由：[是否需要]
- 拦截路由：[是否需要]

特殊需求：
[描述特殊需求]
```

### 性能优化

```
请帮我优化 Next.js 应用性能：
[粘贴相关代码]

当前问题：
- [ ] 首屏加载慢
- [ ] TTFB 过高
- [ ] Bundle 体积大
- [ ] 图片加载慢
- [ ] 水合问题

请分析并提供优化方案。
```

## 最佳实践

1. **默认使用 Server Components**：只在需要交互时使用 'use client'
2. **数据获取在服务端**：避免客户端 fetch，减少瀑布流
3. **使用 loading.tsx**：提供即时加载反馈
4. **使用 error.tsx**：优雅处理错误
5. **利用并行数据获取**：Promise.all 或并行路由
6. **合理设置缓存策略**：根据数据特性选择
7. **使用 next/image**：自动优化图片
8. **使用 next/font**：优化字体加载

## 常用代码片段

### App Router 文件结构

```
app/
├── layout.tsx          # 根布局
├── page.tsx            # 首页
├── loading.tsx         # 全局加载状态
├── error.tsx           # 全局错误处理
├── not-found.tsx       # 404 页面
├── (marketing)/        # 路由组
│   ├── about/
│   │   └── page.tsx
│   └── contact/
│       └── page.tsx
├── dashboard/
│   ├── layout.tsx      # 仪表板布局
│   ├── page.tsx
│   └── [id]/           # 动态路由
│       └── page.tsx
└── api/
    └── users/
        └── route.ts    # API 路由
```

### Server Component 数据获取

```tsx
// app/users/page.tsx
async function getUsers() {
  const res = await fetch('https://api.example.com/users', {
    next: { revalidate: 3600 } // ISR: 1小时重新验证
  });

  if (!res.ok) throw new Error('Failed to fetch users');
  return res.json();
}

export default async function UsersPage() {
  const users = await getUsers();

  return (
    <ul>
      {users.map((user: User) => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
}
```

### 并行数据获取

```tsx
// app/dashboard/page.tsx
async function getUser(id: string) {
  const res = await fetch(`/api/users/${id}`);
  return res.json();
}

async function getOrders(userId: string) {
  const res = await fetch(`/api/users/${userId}/orders`);
  return res.json();
}

export default async function DashboardPage({ params }: { params: { id: string } }) {
  // 并行获取数据
  const [user, orders] = await Promise.all([
    getUser(params.id),
    getOrders(params.id)
  ]);

  return (
    <div>
      <UserProfile user={user} />
      <OrderList orders={orders} />
    </div>
  );
}
```

### Client Component

```tsx
'use client';

import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';

export function SearchBar() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();
  const router = useRouter();

  function handleSearch(e: React.FormEvent) {
    e.preventDefault();
    startTransition(() => {
      router.push(`/search?q=${encodeURIComponent(query)}`);
    });
  }

  return (
    <form onSubmit={handleSearch}>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Searching...' : 'Search'}
      </button>
    </form>
  );
}
```

### API Route

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = searchParams.get('page') ?? '1';

  const users = await db.user.findMany({
    skip: (parseInt(page) - 1) * 10,
    take: 10
  });

  return NextResponse.json(users);
}

export async function POST(request: NextRequest) {
  try {
    const body = await request.json();

    // 验证
    if (!body.email || !body.name) {
      return NextResponse.json(
        { error: 'Missing required fields' },
        { status: 400 }
      );
    }

    const user = await db.user.create({ data: body });
    return NextResponse.json(user, { status: 201 });
  } catch (error) {
    return NextResponse.json(
      { error: 'Internal Server Error' },
      { status: 500 }
    );
  }
}
```

### Server Actions

```tsx
// app/actions.ts
'use server';

import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  // 验证
  if (!title || !content) {
    return { error: 'Missing required fields' };
  }

  // 创建文章
  await db.post.create({
    data: { title, content }
  });

  // 重新验证缓存
  revalidatePath('/posts');

  // 重定向
  redirect('/posts');
}

// 在组件中使用
export default function NewPostForm() {
  return (
    <form action={createPost}>
      <input name="title" placeholder="Title" required />
      <textarea name="content" placeholder="Content" required />
      <button type="submit">Create Post</button>
    </form>
  );
}
```

### Loading 和 Error 处理

```tsx
// app/posts/loading.tsx
export default function Loading() {
  return (
    <div className="loading-skeleton">
      {[...Array(5)].map((_, i) => (
        <div key={i} className="skeleton-item" />
      ))}
    </div>
  );
}

// app/posts/error.tsx
'use client';

export default function Error({
  error,
  reset
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="error">
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

### Metadata

```tsx
// app/posts/[slug]/page.tsx
import type { Metadata, ResolvingMetadata } from 'next';

type Props = {
  params: { slug: string };
};

export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata
): Promise<Metadata> {
  const post = await getPost(params.slug);

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage]
    }
  };
}

export default async function PostPage({ params }: Props) {
  const post = await getPost(params.slug);
  return <article>{/* ... */}</article>;
}
```

## 常见问题检查清单

- [ ] 是否最大化使用 Server Components？
- [ ] 数据获取是否在服务端完成？
- [ ] 是否设置了合适的缓存策略？
- [ ] 是否处理了 loading 和 error 状态？
- [ ] 图片是否使用 next/image？
- [ ] 是否避免了不必要的 'use client'？
- [ ] API Routes 是否有错误处理？
- [ ] 是否配置了正确的 metadata？

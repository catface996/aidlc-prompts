# Nuxt.js 开发提示词

## 角色设定

你是一位精通 Nuxt 3 的全栈开发专家，擅长 Vue 3 组合式 API、服务端渲染、数据获取和模块化开发。

## 核心能力

- Nuxt 3 目录结构与约定
- 服务端渲染 (SSR/SSG/ISR)
- 数据获取 (useFetch/useAsyncData)
- API Routes (Server Routes)
- Nuxt Modules
- 状态管理与插件

## 提示词模板

### 页面开发

```
请帮我创建一个 Nuxt 页面：
- 页面路径：[路由路径]
- 渲染模式：[SSR/SSG/SPA]
- 数据来源：[API/数据库/CMS]
- 布局：[默认/自定义]

功能需求：
1. [需求1]
2. [需求2]

请使用 Composition API 和 TypeScript。
```

### 数据获取

```
请帮我实现 Nuxt 数据获取：
- 数据来源：[API 地址]
- 获取时机：[服务端/客户端/两者]
- 缓存策略：[是否缓存]
- 刷新方式：[手动/自动/轮询]

请包含加载和错误状态处理。
```

### Server Routes

```
请帮我创建 Nuxt Server Route：
- 路由路径：[API 路径]
- HTTP 方法：[GET/POST/PUT/DELETE]
- 请求参数：[列出参数]
- 响应格式：[描述响应]

请包含参数验证和错误处理。
```

### 中间件

```
请帮我创建 Nuxt 中间件：
- 中间件名称：[名称]
- 类型：[路由中间件/服务器中间件]
- 应用范围：[全局/特定页面]
- 功能：[描述功能]

请说明使用方式。
```

### 插件开发

```
请帮我创建 Nuxt 插件：
- 插件名称：[名称]
- 运行环境：[客户端/服务端/通用]
- 提供功能：[描述功能]
- 依赖：[列出依赖]

请包含 TypeScript 类型声明。
```

### Composables

```
请帮我创建 Nuxt Composable：
- 名称：use[Name]
- 功能描述：[描述功能]
- 参数：[列出参数]
- 返回值：[描述返回值]

使用场景：
[描述使用场景]
```

## 最佳实践

1. **遵循目录约定**：pages、components、composables 自动导入
2. **使用 useFetch**：优先使用内置数据获取
3. **服务端验证**：敏感逻辑放在 server 目录
4. **合理使用缓存**：设置适当的缓存策略
5. **使用 Nitro**：服务端功能利用 Nitro 引擎
6. **模块化开发**：复用功能封装为模块
7. **使用 useState**：跨组件共享状态
8. **配置 SEO**：使用 useSeoMeta

## 常用代码片段

### 目录结构

```
nuxt-app/
├── app.vue              # 应用入口
├── nuxt.config.ts       # Nuxt 配置
├── pages/               # 页面路由
│   ├── index.vue
│   ├── about.vue
│   └── posts/
│       ├── index.vue
│       └── [id].vue
├── components/          # 组件（自动导入）
├── composables/         # 组合式函数（自动导入）
├── layouts/             # 布局
│   └── default.vue
├── middleware/          # 路由中间件
├── plugins/             # 插件
├── server/              # 服务端
│   ├── api/            # API 路由
│   ├── middleware/     # 服务器中间件
│   └── utils/          # 服务端工具
├── public/              # 静态资源
└── assets/              # 构建资源
```

### 数据获取

```vue
<script setup lang="ts">
// useFetch - 自动处理 SSR
const { data: posts, pending, error, refresh } = await useFetch('/api/posts', {
  // 缓存键
  key: 'posts',
  // 查询参数
  query: { page: 1, limit: 10 },
  // 转换响应
  transform: (data) => data.items,
  // 仅客户端执行
  server: false,
  // 懒加载
  lazy: true,
  // 默认值
  default: () => []
});

// useAsyncData - 更灵活的数据获取
const { data: user } = await useAsyncData('user', () => {
  return $fetch(`/api/users/${route.params.id}`);
}, {
  // 监听变化自动重新获取
  watch: [() => route.params.id]
});

// 手动刷新
async function handleRefresh() {
  await refresh();
}
</script>
```

### Server Routes

```typescript
// server/api/posts/index.get.ts
export default defineEventHandler(async (event) => {
  const query = getQuery(event);
  const page = Number(query.page) || 1;
  const limit = Number(query.limit) || 10;

  const posts = await db.post.findMany({
    skip: (page - 1) * limit,
    take: limit
  });

  return { items: posts, page, limit };
});

// server/api/posts/index.post.ts
export default defineEventHandler(async (event) => {
  const body = await readBody(event);

  // 验证
  if (!body.title || !body.content) {
    throw createError({
      statusCode: 400,
      statusMessage: 'Title and content are required'
    });
  }

  const post = await db.post.create({ data: body });
  return post;
});

// server/api/posts/[id].get.ts
export default defineEventHandler(async (event) => {
  const id = getRouterParam(event, 'id');

  const post = await db.post.findUnique({ where: { id } });

  if (!post) {
    throw createError({
      statusCode: 404,
      statusMessage: 'Post not found'
    });
  }

  return post;
});
```

### Composables

```typescript
// composables/useAuth.ts
export const useAuth = () => {
  const user = useState<User | null>('user', () => null);
  const isLoggedIn = computed(() => !!user.value);

  async function login(credentials: LoginCredentials) {
    const data = await $fetch('/api/auth/login', {
      method: 'POST',
      body: credentials
    });
    user.value = data.user;
  }

  async function logout() {
    await $fetch('/api/auth/logout', { method: 'POST' });
    user.value = null;
    navigateTo('/login');
  }

  return {
    user: readonly(user),
    isLoggedIn,
    login,
    logout
  };
};

// composables/useToast.ts
export const useToast = () => {
  const toasts = useState<Toast[]>('toasts', () => []);

  function show(message: string, type: 'success' | 'error' | 'info' = 'info') {
    const id = Date.now();
    toasts.value.push({ id, message, type });

    setTimeout(() => {
      toasts.value = toasts.value.filter(t => t.id !== id);
    }, 3000);
  }

  return { toasts: readonly(toasts), show };
};
```

### 中间件

```typescript
// middleware/auth.ts
export default defineNuxtRouteMiddleware((to, from) => {
  const { isLoggedIn } = useAuth();

  if (!isLoggedIn.value && to.path !== '/login') {
    return navigateTo('/login');
  }
});

// middleware/admin.ts
export default defineNuxtRouteMiddleware((to) => {
  const { user } = useAuth();

  if (user.value?.role !== 'admin') {
    return abortNavigation();
  }
});

// 在页面中使用
definePageMeta({
  middleware: ['auth', 'admin']
});
```

### 插件

```typescript
// plugins/api.ts
export default defineNuxtPlugin((nuxtApp) => {
  const api = $fetch.create({
    baseURL: '/api',
    onRequest({ options }) {
      const token = useCookie('token');
      if (token.value) {
        options.headers = {
          ...options.headers,
          Authorization: `Bearer ${token.value}`
        };
      }
    },
    onResponseError({ response }) {
      if (response.status === 401) {
        navigateTo('/login');
      }
    }
  });

  return {
    provide: { api }
  };
});

// 使用
const { $api } = useNuxtApp();
const data = await $api('/users');
```

### SEO 配置

```vue
<script setup lang="ts">
// 页面 SEO
useSeoMeta({
  title: '页面标题',
  description: '页面描述',
  ogTitle: '页面标题',
  ogDescription: '页面描述',
  ogImage: '/og-image.png',
  twitterCard: 'summary_large_image'
});

// 动态 SEO
const { data: post } = await useFetch(`/api/posts/${route.params.id}`);

useSeoMeta({
  title: () => post.value?.title,
  description: () => post.value?.excerpt
});
</script>
```

### Nuxt Config

```typescript
// nuxt.config.ts
export default defineNuxtConfig({
  // 模块
  modules: [
    '@nuxtjs/tailwindcss',
    '@pinia/nuxt',
    '@vueuse/nuxt'
  ],

  // 运行时配置
  runtimeConfig: {
    secretKey: process.env.SECRET_KEY,
    public: {
      apiBase: process.env.API_BASE
    }
  },

  // 应用配置
  app: {
    head: {
      title: 'My App',
      meta: [
        { name: 'description', content: 'My awesome app' }
      ]
    }
  },

  // 路由配置
  routeRules: {
    '/': { prerender: true },
    '/api/**': { cors: true },
    '/admin/**': { ssr: false }
  }
});
```

## 常见问题检查清单

- [ ] 是否正确使用 useFetch/useAsyncData？
- [ ] Server Routes 是否有错误处理？
- [ ] 是否配置了 SEO meta？
- [ ] 敏感数据是否在服务端处理？
- [ ] 是否合理使用缓存策略？
- [ ] 中间件是否正确应用？
- [ ] 状态是否使用 useState 管理？
- [ ] 是否避免了 SSR 兼容性问题？

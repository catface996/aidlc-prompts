# 前端缓存策略最佳实践

## 角色设定

你是一位精通前端缓存的性能优化专家，擅长浏览器缓存、应用缓存和数据缓存策略设计。

## 提示词模板

### 缓存策略设计

```
请帮我设计缓存策略：
- 应用类型：[SPA/MPA/PWA]
- 数据特性：[静态/动态/实时]
- 缓存位置：[浏览器/内存/Service Worker]
- 更新策略：[立即/定时/按需]

需要缓存的内容：
1. [内容类型1]
2. [内容类型2]
```

### 缓存问题排查

```
请帮我排查缓存问题：
[描述问题现象]

当前缓存配置：[描述配置]
请分析原因并提供解决方案。
```

## 核心代码示例

### LocalStorage 封装

```typescript
// utils/storage.ts
interface StorageItem<T> {
  value: T;
  expire: number | null;
  createTime: number;
}

class LocalStorage {
  private prefix = 'app_';

  set<T>(key: string, value: T, expire?: number): void {
    const item: StorageItem<T> = {
      value,
      expire: expire ? Date.now() + expire * 1000 : null,
      createTime: Date.now(),
    };
    localStorage.setItem(this.prefix + key, JSON.stringify(item));
  }

  get<T>(key: string): T | null {
    const itemStr = localStorage.getItem(this.prefix + key);
    if (!itemStr) return null;

    try {
      const item: StorageItem<T> = JSON.parse(itemStr);

      // 检查过期
      if (item.expire && item.expire < Date.now()) {
        this.remove(key);
        return null;
      }

      return item.value;
    } catch {
      return null;
    }
  }

  remove(key: string): void {
    localStorage.setItem(this.prefix + key);
  }

  clear(): void {
    const keys = Object.keys(localStorage);
    keys.forEach((key) => {
      if (key.startsWith(this.prefix)) {
        localStorage.removeItem(key);
      }
    });
  }

  // 获取所有缓存信息
  getAll(): Record<string, unknown> {
    const result: Record<string, unknown> = {};
    const keys = Object.keys(localStorage);

    keys.forEach((key) => {
      if (key.startsWith(this.prefix)) {
        const value = this.get(key.replace(this.prefix, ''));
        if (value !== null) {
          result[key] = value;
        }
      }
    });

    return result;
  }
}

export const storage = new LocalStorage();
```

### 内存缓存 (LRU)

```typescript
// utils/lruCache.ts
class LRUCache<K, V> {
  private capacity: number;
  private cache: Map<K, V>;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.cache = new Map();
  }

  get(key: K): V | undefined {
    if (!this.cache.has(key)) {
      return undefined;
    }

    // 移到最近使用位置
    const value = this.cache.get(key)!;
    this.cache.delete(key);
    this.cache.set(key, value);
    return value;
  }

  set(key: K, value: V): void {
    if (this.cache.has(key)) {
      this.cache.delete(key);
    } else if (this.cache.size >= this.capacity) {
      // 删除最久未使用的项
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    this.cache.set(key, value);
  }

  has(key: K): boolean {
    return this.cache.has(key);
  }

  delete(key: K): boolean {
    return this.cache.delete(key);
  }

  clear(): void {
    this.cache.clear();
  }

  get size(): number {
    return this.cache.size;
  }
}

export const apiCache = new LRUCache<string, unknown>(100);
```

### 请求缓存

```typescript
// utils/requestCache.ts
interface CacheEntry {
  data: unknown;
  timestamp: number;
  ttl: number;
}

const cache = new Map<string, CacheEntry>();
const pendingRequests = new Map<string, Promise<unknown>>();

function generateCacheKey(url: string, params?: object): string {
  return params ? `${url}?${JSON.stringify(params)}` : url;
}

export async function cachedFetch<T>(
  url: string,
  options?: {
    params?: object;
    ttl?: number;
    forceRefresh?: boolean;
  }
): Promise<T> {
  const { params, ttl = 60000, forceRefresh = false } = options || {};
  const cacheKey = generateCacheKey(url, params);

  // 检查缓存
  if (!forceRefresh) {
    const cached = cache.get(cacheKey);
    if (cached && Date.now() - cached.timestamp < cached.ttl) {
      return cached.data as T;
    }
  }

  // 请求去重
  if (pendingRequests.has(cacheKey)) {
    return pendingRequests.get(cacheKey) as Promise<T>;
  }

  const request = fetch(url, {
    method: 'GET',
    headers: { 'Content-Type': 'application/json' },
  })
    .then((res) => res.json())
    .then((data) => {
      cache.set(cacheKey, { data, timestamp: Date.now(), ttl });
      pendingRequests.delete(cacheKey);
      return data;
    })
    .catch((error) => {
      pendingRequests.delete(cacheKey);
      throw error;
    });

  pendingRequests.set(cacheKey, request);
  return request;
}

// 清除特定缓存
export function invalidateCache(pattern: string | RegExp): void {
  cache.forEach((_, key) => {
    if (typeof pattern === 'string' ? key.includes(pattern) : pattern.test(key)) {
      cache.delete(key);
    }
  });
}
```

### React Query 缓存配置

```typescript
// queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // 缓存时间
      gcTime: 1000 * 60 * 5, // 5分钟
      staleTime: 1000 * 60 * 1, // 1分钟后标记为过期

      // 重试
      retry: 3,
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),

      // 后台刷新
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,

      // 结构共享 (优化渲染)
      structuralSharing: true,
    },
    mutations: {
      retry: 1,
    },
  },
});

// 使用示例
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

function UserProfile({ userId }: { userId: string }) {
  const queryClient = useQueryClient();

  // 查询
  const { data: user, isLoading } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 1000 * 60 * 5, // 覆盖默认值
  });

  // 变更后更新缓存
  const updateUser = useMutation({
    mutationFn: (data: Partial<User>) => updateUserApi(userId, data),
    onSuccess: (newData) => {
      // 更新缓存
      queryClient.setQueryData(['user', userId], newData);
      // 或使缓存失效
      queryClient.invalidateQueries({ queryKey: ['user', userId] });
    },
  });

  return <div>...</div>;
}
```

### IndexedDB 缓存

```typescript
// utils/indexedDBCache.ts
const DB_NAME = 'AppCache';
const STORE_NAME = 'cache';
const DB_VERSION = 1;

function openDB(): Promise<IDBDatabase> {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, DB_VERSION);

    request.onerror = () => reject(request.error);
    request.onsuccess = () => resolve(request.result);

    request.onupgradeneeded = (event) => {
      const db = (event.target as IDBOpenDBRequest).result;
      if (!db.objectStoreNames.contains(STORE_NAME)) {
        db.createObjectStore(STORE_NAME, { keyPath: 'key' });
      }
    };
  });
}

export const indexedDBCache = {
  async set<T>(key: string, value: T, ttl?: number): Promise<void> {
    const db = await openDB();
    const transaction = db.transaction(STORE_NAME, 'readwrite');
    const store = transaction.objectStore(STORE_NAME);

    store.put({
      key,
      value,
      expire: ttl ? Date.now() + ttl : null,
    });
  },

  async get<T>(key: string): Promise<T | null> {
    const db = await openDB();
    const transaction = db.transaction(STORE_NAME, 'readonly');
    const store = transaction.objectStore(STORE_NAME);

    return new Promise((resolve) => {
      const request = store.get(key);
      request.onsuccess = () => {
        const result = request.result;
        if (!result) {
          resolve(null);
        } else if (result.expire && result.expire < Date.now()) {
          this.delete(key);
          resolve(null);
        } else {
          resolve(result.value);
        }
      };
      request.onerror = () => resolve(null);
    });
  },

  async delete(key: string): Promise<void> {
    const db = await openDB();
    const transaction = db.transaction(STORE_NAME, 'readwrite');
    const store = transaction.objectStore(STORE_NAME);
    store.delete(key);
  },

  async clear(): Promise<void> {
    const db = await openDB();
    const transaction = db.transaction(STORE_NAME, 'readwrite');
    const store = transaction.objectStore(STORE_NAME);
    store.clear();
  },
};
```

### Service Worker 缓存

```typescript
// sw.js
const CACHE_NAME = 'app-cache-v1';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/static/js/main.js',
  '/static/css/main.css',
];

// 安装时预缓存静态资源
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => cache.addAll(STATIC_ASSETS))
  );
});

// 请求拦截
self.addEventListener('fetch', (event) => {
  const { request } = event;

  // API 请求: Network First
  if (request.url.includes('/api/')) {
    event.respondWith(
      fetch(request)
        .then((response) => {
          const clone = response.clone();
          caches.open(CACHE_NAME).then((cache) => cache.put(request, clone));
          return response;
        })
        .catch(() => caches.match(request))
    );
    return;
  }

  // 静态资源: Cache First
  event.respondWith(
    caches.match(request).then((cached) => {
      return cached || fetch(request).then((response) => {
        const clone = response.clone();
        caches.open(CACHE_NAME).then((cache) => cache.put(request, clone));
        return response;
      });
    })
  );
});
```

## 缓存策略对比

| 策略 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| Memory | 临时数据 | 速度快 | 刷新丢失 |
| LocalStorage | 持久配置 | 简单易用 | 同步阻塞 |
| SessionStorage | 会话数据 | 隔离安全 | 标签页不共享 |
| IndexedDB | 大量数据 | 容量大、异步 | API复杂 |
| Service Worker | 离线应用 | 离线可用 | 实现复杂 |

## 最佳实践清单

- [ ] 区分缓存类型：静态资源 vs 动态数据
- [ ] 设置合理的 TTL
- [ ] 实现缓存失效机制
- [ ] 使用内存缓存减少 IO
- [ ] 大数据使用 IndexedDB
- [ ] 离线场景使用 Service Worker
- [ ] 监控缓存命中率
- [ ] 定期清理过期缓存

# Axios 网络请求最佳实践

## 角色设定

你是一位精通 HTTP 请求和 Axios 的前端开发专家，擅长封装请求、错误处理和性能优化。

## 提示词模板

### 封装 Axios

```
请帮我封装 Axios 请求模块：
- 基础 URL：[API 地址]
- 认证方式：[JWT/Session/API Key]
- 错误处理：[统一提示/自定义处理]
- 需要的功能：
  - [ ] 请求/响应拦截
  - [ ] 自动刷新 Token
  - [ ] 请求取消
  - [ ] 重试机制
  - [ ] 并发控制
```

### 处理复杂请求场景

```
请帮我处理以下请求场景：
- 场景描述：[并发请求/串行请求/轮询/文件上传]
- 错误处理：[描述错误处理方式]
- 性能要求：[描述性能要求]
```

## 核心封装示例

### 基础封装

```typescript
// utils/request.ts
import axios, { AxiosInstance, AxiosRequestConfig, AxiosResponse } from 'axios';

// 响应数据类型
interface ApiResponse<T = any> {
  code: number;
  data: T;
  message: string;
}

// 创建实例
const request: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// 请求拦截器
request.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// 响应拦截器
request.interceptors.response.use(
  (response: AxiosResponse<ApiResponse>) => {
    const { code, data, message } = response.data;

    if (code === 0) {
      return data;
    }

    // 业务错误
    return Promise.reject(new Error(message));
  },
  (error) => {
    // HTTP 错误处理
    if (error.response) {
      const { status } = error.response;

      switch (status) {
        case 401:
          // 清除 token，跳转登录
          localStorage.removeItem('token');
          window.location.href = '/login';
          break;
        case 403:
          console.error('没有权限');
          break;
        case 404:
          console.error('请求资源不存在');
          break;
        case 500:
          console.error('服务器错误');
          break;
      }
    } else if (error.code === 'ECONNABORTED') {
      console.error('请求超时');
    } else {
      console.error('网络错误');
    }

    return Promise.reject(error);
  }
);

export default request;
```

### Token 自动刷新

```typescript
// utils/request.ts
let isRefreshing = false;
let refreshSubscribers: Array<(token: string) => void> = [];

function subscribeTokenRefresh(callback: (token: string) => void) {
  refreshSubscribers.push(callback);
}

function onTokenRefreshed(token: string) {
  refreshSubscribers.forEach((callback) => callback(token));
  refreshSubscribers = [];
}

request.interceptors.response.use(
  (response) => response,
  async (error) => {
    const { config, response } = error;

    if (response?.status === 401 && !config._retry) {
      if (isRefreshing) {
        // 等待刷新完成
        return new Promise((resolve) => {
          subscribeTokenRefresh((token) => {
            config.headers.Authorization = `Bearer ${token}`;
            resolve(request(config));
          });
        });
      }

      config._retry = true;
      isRefreshing = true;

      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const { data } = await axios.post('/api/auth/refresh', { refreshToken });

        localStorage.setItem('token', data.token);
        onTokenRefreshed(data.token);

        config.headers.Authorization = `Bearer ${data.token}`;
        return request(config);
      } catch (refreshError) {
        localStorage.clear();
        window.location.href = '/login';
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

### 请求取消

```typescript
// hooks/useRequest.ts
import { useEffect, useRef, useState } from 'react';
import axios, { CancelTokenSource } from 'axios';
import request from '@/utils/request';

export function useRequest<T>(url: string, options?: AxiosRequestConfig) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const cancelSourceRef = useRef<CancelTokenSource>();

  const fetchData = async () => {
    // 取消之前的请求
    cancelSourceRef.current?.cancel('Request cancelled');
    cancelSourceRef.current = axios.CancelToken.source();

    setLoading(true);
    setError(null);

    try {
      const result = await request<T>({
        url,
        ...options,
        cancelToken: cancelSourceRef.current.token,
      });
      setData(result as T);
    } catch (err) {
      if (!axios.isCancel(err)) {
        setError(err as Error);
      }
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchData();

    return () => {
      cancelSourceRef.current?.cancel('Component unmounted');
    };
  }, [url]);

  return { data, loading, error, refetch: fetchData };
}
```

### 请求重试

```typescript
// utils/retry.ts
import axios, { AxiosRequestConfig } from 'axios';

interface RetryConfig extends AxiosRequestConfig {
  retries?: number;
  retryDelay?: number;
  retryCondition?: (error: any) => boolean;
}

export async function requestWithRetry<T>(config: RetryConfig): Promise<T> {
  const {
    retries = 3,
    retryDelay = 1000,
    retryCondition = (error) => !error.response || error.response.status >= 500,
    ...axiosConfig
  } = config;

  let lastError: Error;

  for (let i = 0; i <= retries; i++) {
    try {
      const response = await axios(axiosConfig);
      return response.data;
    } catch (error: any) {
      lastError = error;

      if (i < retries && retryCondition(error)) {
        await new Promise((resolve) => setTimeout(resolve, retryDelay * (i + 1)));
        continue;
      }

      throw error;
    }
  }

  throw lastError!;
}
```

### 并发请求控制

```typescript
// utils/concurrency.ts
class RequestQueue {
  private queue: Array<() => Promise<any>> = [];
  private running = 0;
  private maxConcurrency: number;

  constructor(maxConcurrency = 5) {
    this.maxConcurrency = maxConcurrency;
  }

  add<T>(requestFn: () => Promise<T>): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push(async () => {
        try {
          const result = await requestFn();
          resolve(result);
        } catch (error) {
          reject(error);
        }
      });

      this.run();
    });
  }

  private async run() {
    while (this.running < this.maxConcurrency && this.queue.length > 0) {
      const task = this.queue.shift();
      if (task) {
        this.running++;
        task().finally(() => {
          this.running--;
          this.run();
        });
      }
    }
  }
}

export const requestQueue = new RequestQueue(5);
```

### API 模块化

```typescript
// api/user.ts
import request from '@/utils/request';

export interface User {
  id: string;
  name: string;
  email: string;
}

export interface LoginParams {
  email: string;
  password: string;
}

export const userApi = {
  login: (params: LoginParams) =>
    request.post<{ token: string; user: User }>('/auth/login', params),

  logout: () =>
    request.post('/auth/logout'),

  getProfile: () =>
    request.get<User>('/user/profile'),

  updateProfile: (data: Partial<User>) =>
    request.put<User>('/user/profile', data),

  getUsers: (params?: { page?: number; limit?: number }) =>
    request.get<{ items: User[]; total: number }>('/users', { params }),
};
```

## 最佳实践清单

- [ ] 统一封装请求实例
- [ ] 配置请求/响应拦截器
- [ ] 实现 Token 自动刷新
- [ ] 支持请求取消
- [ ] 添加请求重试机制
- [ ] 控制并发请求数量
- [ ] API 模块化组织
- [ ] TypeScript 类型完善

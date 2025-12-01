# JWT 认证安全最佳实践

## 角色设定

你是一位精通前端安全和 JWT 认证的专家，擅长安全存储、Token 管理和认证流程设计。

## 提示词模板

### 认证方案设计

```
请帮我设计 JWT 认证方案：
- 应用类型：[SPA/SSR/移动端]
- Token 存储：[Cookie/localStorage/内存]
- 刷新策略：[静默刷新/主动刷新]
- 安全要求：[高/中/低]

需要的功能：
- [ ] 登录/注销
- [ ] Token 自动刷新
- [ ] 多标签页同步
- [ ] 记住我功能
```

### 安全问题排查

```
请帮我排查以下认证安全问题：
[描述问题或粘贴代码]

当前方案：[描述当前实现]
请分析风险并提供改进建议。
```

## 核心代码示例

### Token 存储策略

```typescript
// utils/tokenStorage.ts

// 方案1: HttpOnly Cookie (推荐，需要后端配合)
// 前端无需存储，由后端设置 Cookie

// 方案2: 内存存储 (安全但刷新丢失)
class MemoryTokenStorage {
  private accessToken: string | null = null;

  setToken(token: string) {
    this.accessToken = token;
  }

  getToken() {
    return this.accessToken;
  }

  clearToken() {
    this.accessToken = null;
  }
}

export const memoryStorage = new MemoryTokenStorage();

// 方案3: localStorage (便捷但有 XSS 风险)
export const localTokenStorage = {
  setToken(token: string) {
    localStorage.setItem('access_token', token);
  },
  getToken() {
    return localStorage.getItem('access_token');
  },
  clearToken() {
    localStorage.removeItem('access_token');
  },
  setRefreshToken(token: string) {
    localStorage.setItem('refresh_token', token);
  },
  getRefreshToken() {
    return localStorage.getItem('refresh_token');
  },
};
```

### Auth Context 实现

```tsx
// contexts/AuthContext.tsx
import { createContext, useContext, useEffect, useState, useCallback } from 'react';

interface User {
  id: string;
  name: string;
  email: string;
  role: string;
}

interface AuthContextType {
  user: User | null;
  isAuthenticated: boolean;
  isLoading: boolean;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  refreshToken: () => Promise<void>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [isLoading, setIsLoading] = useState(true);

  // 解析 JWT 获取用户信息
  const parseToken = (token: string): User | null => {
    try {
      const payload = JSON.parse(atob(token.split('.')[1]));
      // 检查过期
      if (payload.exp * 1000 < Date.now()) {
        return null;
      }
      return payload.user;
    } catch {
      return null;
    }
  };

  // 初始化：检查现有 token
  useEffect(() => {
    const initAuth = async () => {
      const token = localStorage.getItem('access_token');
      if (token) {
        const userData = parseToken(token);
        if (userData) {
          setUser(userData);
        } else {
          // Token 过期，尝试刷新
          await refreshToken();
        }
      }
      setIsLoading(false);
    };
    initAuth();
  }, []);

  const login = async (email: string, password: string) => {
    const response = await fetch('/api/auth/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, password }),
    });

    if (!response.ok) {
      throw new Error('Login failed');
    }

    const { accessToken, refreshToken, user } = await response.json();
    localStorage.setItem('access_token', accessToken);
    localStorage.setItem('refresh_token', refreshToken);
    setUser(user);
  };

  const logout = useCallback(() => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    setUser(null);

    // 通知服务器注销
    fetch('/api/auth/logout', { method: 'POST' }).catch(() => {});
  }, []);

  const refreshToken = async () => {
    const refresh = localStorage.getItem('refresh_token');
    if (!refresh) {
      logout();
      return;
    }

    try {
      const response = await fetch('/api/auth/refresh', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken: refresh }),
      });

      if (!response.ok) {
        throw new Error('Refresh failed');
      }

      const { accessToken, user } = await response.json();
      localStorage.setItem('access_token', accessToken);
      setUser(user);
    } catch {
      logout();
    }
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        isAuthenticated: !!user,
        isLoading,
        login,
        logout,
        refreshToken,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
}
```

### Token 自动刷新

```typescript
// utils/request.ts
import axios from 'axios';

const request = axios.create({
  baseURL: '/api',
});

let isRefreshing = false;
let refreshSubscribers: Array<(token: string) => void> = [];

function subscribeTokenRefresh(callback: (token: string) => void) {
  refreshSubscribers.push(callback);
}

function onTokenRefreshed(token: string) {
  refreshSubscribers.forEach((cb) => cb(token));
  refreshSubscribers = [];
}

// 检查 Token 是否即将过期
function isTokenExpiringSoon(token: string): boolean {
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const expiresIn = payload.exp * 1000 - Date.now();
    return expiresIn < 5 * 60 * 1000; // 5分钟内过期
  } catch {
    return true;
  }
}

request.interceptors.request.use(async (config) => {
  let token = localStorage.getItem('access_token');

  // 主动刷新即将过期的 Token
  if (token && isTokenExpiringSoon(token)) {
    if (!isRefreshing) {
      isRefreshing = true;
      try {
        const newToken = await refreshAccessToken();
        token = newToken;
        onTokenRefreshed(newToken);
      } finally {
        isRefreshing = false;
      }
    } else {
      // 等待刷新完成
      token = await new Promise((resolve) => {
        subscribeTokenRefresh(resolve);
      });
    }
  }

  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

async function refreshAccessToken(): Promise<string> {
  const refreshToken = localStorage.getItem('refresh_token');
  const response = await axios.post('/api/auth/refresh', { refreshToken });
  const { accessToken } = response.data;
  localStorage.setItem('access_token', accessToken);
  return accessToken;
}

export default request;
```

### 多标签页同步

```typescript
// hooks/useAuthSync.ts
import { useEffect } from 'react';
import { useAuth } from '@/contexts/AuthContext';

export function useAuthSync() {
  const { logout } = useAuth();

  useEffect(() => {
    const handleStorageChange = (event: StorageEvent) => {
      if (event.key === 'access_token') {
        if (event.newValue === null) {
          // 其他标签页注销
          logout();
        } else if (event.oldValue === null) {
          // 其他标签页登录
          window.location.reload();
        }
      }
    };

    window.addEventListener('storage', handleStorageChange);
    return () => window.removeEventListener('storage', handleStorageChange);
  }, [logout]);
}

// 或使用 BroadcastChannel
export function useAuthBroadcast() {
  const { logout } = useAuth();

  useEffect(() => {
    const channel = new BroadcastChannel('auth');

    channel.onmessage = (event) => {
      if (event.data.type === 'LOGOUT') {
        logout();
      }
      if (event.data.type === 'LOGIN') {
        window.location.reload();
      }
    };

    return () => channel.close();
  }, [logout]);
}

// 登出时广播
function broadcastLogout() {
  const channel = new BroadcastChannel('auth');
  channel.postMessage({ type: 'LOGOUT' });
  channel.close();
}
```

### 安全措施

```typescript
// 1. CSRF Token (配合 Cookie 使用)
request.interceptors.request.use((config) => {
  const csrfToken = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');
  if (csrfToken) {
    config.headers['X-CSRF-Token'] = csrfToken;
  }
  return config;
});

// 2. 请求指纹 (防止 Token 被盗用)
function generateFingerprint(): string {
  const canvas = document.createElement('canvas');
  const ctx = canvas.getContext('2d');
  ctx?.fillText('fingerprint', 10, 10);
  return canvas.toDataURL();
}

// 3. Token 绑定设备
request.interceptors.request.use((config) => {
  config.headers['X-Device-Id'] = getDeviceId();
  return config;
});

// 4. 敏感操作二次验证
async function sensitiveAction(action: () => Promise<void>) {
  const confirmed = await requestPasswordConfirmation();
  if (confirmed) {
    await action();
  }
}
```

## 安全清单

### Token 存储

| 方案 | XSS 风险 | CSRF 风险 | 便捷性 |
|------|----------|-----------|--------|
| HttpOnly Cookie | 低 | 需防护 | 中 |
| 内存 | 低 | 无 | 低 |
| localStorage | 高 | 无 | 高 |
| sessionStorage | 中 | 无 | 中 |

### 最佳实践清单

- [ ] 使用 HTTPS 传输
- [ ] Access Token 短期有效 (15-60分钟)
- [ ] Refresh Token 长期有效但定期轮换
- [ ] 敏感操作要求重新认证
- [ ] 实现 Token 黑名单机制
- [ ] 多设备登录管理
- [ ] 异常登录检测与通知
- [ ] 安全退出清除所有 Token

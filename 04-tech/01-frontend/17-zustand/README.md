# Zustand 状态管理最佳实践

## 角色设定

你是一位精通 Zustand 的 React 状态管理专家，擅长 Store 设计、性能优化和持久化方案。

## 提示词模板

### 设计 Store

```
请帮我设计 Zustand Store：
- Store 名称：[名称]
- 状态字段：[列出状态]
- 操作方法：[列出 actions]
- 是否需要持久化：[是/否]
- 是否需要 devtools：[是/否]

业务场景：
[描述业务场景]
```

### 优化 Store

```
请帮我优化以下 Zustand Store：
[粘贴 store 代码]

问题：[描述问题]
请分析并提供优化方案。
```

## 核心代码示例

### 基础 Store

```typescript
// stores/useCounterStore.ts
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  incrementBy: (amount: number) => void;
}

export const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
  incrementBy: (amount) => set((state) => ({ count: state.count + amount })),
}));
```

### 带中间件的 Store

```typescript
// stores/useUserStore.ts
import { create } from 'zustand';
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface User {
  id: string;
  name: string;
  email: string;
}

interface UserState {
  user: User | null;
  token: string | null;
  isLoading: boolean;
  error: string | null;

  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  updateUser: (data: Partial<User>) => void;
}

export const useUserStore = create<UserState>()(
  devtools(
    persist(
      subscribeWithSelector(
        immer((set, get) => ({
          user: null,
          token: null,
          isLoading: false,
          error: null,

          login: async (email, password) => {
            set({ isLoading: true, error: null });
            try {
              const response = await fetch('/api/login', {
                method: 'POST',
                body: JSON.stringify({ email, password }),
              });
              const data = await response.json();

              set((state) => {
                state.user = data.user;
                state.token = data.token;
                state.isLoading = false;
              });
            } catch (error) {
              set({ error: (error as Error).message, isLoading: false });
            }
          },

          logout: () => {
            set({ user: null, token: null });
          },

          updateUser: (data) => {
            set((state) => {
              if (state.user) {
                Object.assign(state.user, data);
              }
            });
          },
        }))
      ),
      {
        name: 'user-storage',
        partialize: (state) => ({ user: state.user, token: state.token }),
      }
    ),
    { name: 'UserStore' }
  )
);
```

### 切片模式 (Slice Pattern)

```typescript
// stores/slices/userSlice.ts
import { StateCreator } from 'zustand';

export interface UserSlice {
  user: User | null;
  setUser: (user: User | null) => void;
}

export const createUserSlice: StateCreator<
  UserSlice & CartSlice,
  [],
  [],
  UserSlice
> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
});

// stores/slices/cartSlice.ts
export interface CartSlice {
  items: CartItem[];
  addItem: (item: CartItem) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

export const createCartSlice: StateCreator<
  UserSlice & CartSlice,
  [],
  [],
  CartSlice
> = (set) => ({
  items: [],
  addItem: (item) => set((state) => ({ items: [...state.items, item] })),
  removeItem: (id) => set((state) => ({
    items: state.items.filter((item) => item.id !== id),
  })),
  clearCart: () => set({ items: [] }),
});

// stores/useStore.ts
import { create } from 'zustand';
import { createUserSlice, UserSlice } from './slices/userSlice';
import { createCartSlice, CartSlice } from './slices/cartSlice';

export const useStore = create<UserSlice & CartSlice>()((...a) => ({
  ...createUserSlice(...a),
  ...createCartSlice(...a),
}));
```

### 选择器优化

```typescript
// stores/useProductStore.ts
import { create } from 'zustand';
import { shallow } from 'zustand/shallow';

interface ProductState {
  products: Product[];
  filters: { category: string; priceRange: [number, number] };
  setProducts: (products: Product[]) => void;
  setFilters: (filters: Partial<ProductState['filters']>) => void;
}

export const useProductStore = create<ProductState>((set) => ({
  products: [],
  filters: { category: '', priceRange: [0, 1000] },
  setProducts: (products) => set({ products }),
  setFilters: (filters) => set((state) => ({
    filters: { ...state.filters, ...filters },
  })),
}));

// 派生状态 (在组件外定义)
export const selectFilteredProducts = (state: ProductState) => {
  const { products, filters } = state;
  return products.filter((p) => {
    if (filters.category && p.category !== filters.category) return false;
    if (p.price < filters.priceRange[0] || p.price > filters.priceRange[1]) return false;
    return true;
  });
};

// 组件中使用
function ProductList() {
  // ✅ 好：只在过滤结果变化时重渲染
  const filteredProducts = useProductStore(selectFilteredProducts);

  // ✅ 好：使用 shallow 比较多个值
  const { category, priceRange } = useProductStore(
    (state) => state.filters,
    shallow
  );

  // ❌ 差：每次都会重渲染
  // const { products, filters } = useProductStore();

  return <div>{/* ... */}</div>;
}
```

### 异步操作

```typescript
// stores/useDataStore.ts
import { create } from 'zustand';

interface DataState {
  data: Record<string, unknown>;
  loading: Record<string, boolean>;
  errors: Record<string, string | null>;

  fetchData: (key: string, fetcher: () => Promise<unknown>) => Promise<void>;
  clearError: (key: string) => void;
}

export const useDataStore = create<DataState>((set, get) => ({
  data: {},
  loading: {},
  errors: {},

  fetchData: async (key, fetcher) => {
    set((state) => ({
      loading: { ...state.loading, [key]: true },
      errors: { ...state.errors, [key]: null },
    }));

    try {
      const result = await fetcher();
      set((state) => ({
        data: { ...state.data, [key]: result },
        loading: { ...state.loading, [key]: false },
      }));
    } catch (error) {
      set((state) => ({
        errors: { ...state.errors, [key]: (error as Error).message },
        loading: { ...state.loading, [key]: false },
      }));
    }
  },

  clearError: (key) => {
    set((state) => ({
      errors: { ...state.errors, [key]: null },
    }));
  },
}));
```

### 订阅状态变化

```typescript
// 在组件外订阅
const unsubscribe = useUserStore.subscribe(
  (state) => state.token,
  (token, prevToken) => {
    if (token && !prevToken) {
      console.log('User logged in');
    }
    if (!token && prevToken) {
      console.log('User logged out');
    }
  }
);

// 带选择器的订阅
useUserStore.subscribe(
  (state) => state.user?.name,
  (name) => {
    document.title = name ? `${name} - My App` : 'My App';
  },
  { fireImmediately: true }
);
```

## 最佳实践清单

- [ ] 使用 TypeScript 定义状态类型
- [ ] 大型应用使用切片模式
- [ ] 使用选择器避免不必要的重渲染
- [ ] 用 shallow 比较对象/数组
- [ ] 需要调试时启用 devtools
- [ ] 敏感数据不要持久化
- [ ] 使用 immer 简化不可变更新
- [ ] 异步操作处理 loading 和 error 状态

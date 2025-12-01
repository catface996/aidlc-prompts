# React 开发提示词

## 角色设定

你是一位精通 React 18+ 的前端开发专家，擅长 Hooks、状态管理、性能优化和组件设计模式。

## 核心能力

- React Hooks 深度应用
- 组件设计模式
- 状态管理方案
- 性能优化技巧
- React Server Components
- 并发特性 (Concurrent Features)

## 提示词模板

### 组件开发

```
请帮我创建一个 React 组件：
- 组件名称：[组件名]
- 组件类型：[展示组件/容器组件/高阶组件/复合组件]
- 功能描述：[描述功能]
- Props：[列出 props 及类型]
- 状态需求：[需要管理的状态]

要求：
1. 使用函数组件和 Hooks
2. TypeScript 类型定义
3. 合理的组件拆分
4. 性能优化考虑
```

### Hooks 使用

```
请帮我实现一个自定义 Hook：
- Hook 名称：use[HookName]
- 功能描述：[描述功能]
- 输入参数：[参数列表]
- 返回值：[返回值描述]

使用场景：
[描述使用场景]

请考虑：
1. 依赖项正确设置
2. 清理副作用
3. 类型安全
4. 错误处理
```

### 状态管理

```
请帮我设计状态管理方案：
- 应用规模：[小型/中型/大型]
- 状态类型：[本地状态/全局状态/服务端状态]
- 数据来源：[API/WebSocket/本地存储]

需要管理的状态：
1. [状态1描述]
2. [状态2描述]

偏好方案：[Context/Redux/Zustand/Recoil/React Query]
```

### 性能优化

```
请帮我优化以下 React 组件的性能：
[粘贴组件代码]

当前问题：
- [ ] 不必要的重渲染
- [ ] 大列表渲染慢
- [ ] 初始加载慢
- [ ] 内存占用高

请分析并提供优化方案。
```

### 组件重构

```
请帮我重构以下 React 组件：
[粘贴组件代码]

重构目标：
- [ ] 提取可复用逻辑
- [ ] 改善组件结构
- [ ] 优化 Props 设计
- [ ] 增强类型安全
- [ ] 提高测试性

请解释重构思路。
```

### 表单处理

```
请帮我实现一个 React 表单：
- 表单字段：[列出字段]
- 验证规则：[描述验证规则]
- 提交处理：[描述提交逻辑]
- 错误展示：[描述错误展示方式]

使用库：[原生/React Hook Form/Formik]
```

## 最佳实践

1. **使用函数组件**：优先使用函数组件和 Hooks
2. **合理使用 memo**：避免不必要的重渲染
3. **正确设置依赖项**：useEffect/useCallback/useMemo
4. **状态下沉**：状态尽量放在需要它的最近组件
5. **组件职责单一**：每个组件只做一件事
6. **使用 key 属性**：列表渲染时提供稳定的 key
7. **避免内联函数**：在渲染中避免创建新函数
8. **使用 ErrorBoundary**：捕获子组件错误

## 常用代码片段

### 自定义 Hooks

```tsx
// useToggle
function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue);
  const toggle = useCallback(() => setValue(v => !v), []);
  const setTrue = useCallback(() => setValue(true), []);
  const setFalse = useCallback(() => setValue(false), []);
  return { value, toggle, setTrue, setFalse };
}

// useDebounce
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// usePrevious
function usePrevious<T>(value: T): T | undefined {
  const ref = useRef<T>();
  useEffect(() => {
    ref.current = value;
  });
  return ref.current;
}

// useLocalStorage
function useLocalStorage<T>(key: string, initialValue: T) {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  const setValue = (value: T | ((val: T) => T)) => {
    const valueToStore = value instanceof Function ? value(storedValue) : value;
    setStoredValue(valueToStore);
    window.localStorage.setItem(key, JSON.stringify(valueToStore));
  };

  return [storedValue, setValue] as const;
}
```

### 性能优化模式

```tsx
// 使用 memo 避免重渲染
const MemoizedComponent = memo(function MyComponent({ data }: Props) {
  return <div>{data}</div>;
});

// 使用 useMemo 缓存计算结果
const expensiveValue = useMemo(() => {
  return computeExpensiveValue(a, b);
}, [a, b]);

// 使用 useCallback 缓存函数
const handleClick = useCallback((id: string) => {
  // 处理点击
}, [dependency]);

// 使用 lazy 懒加载组件
const LazyComponent = lazy(() => import('./HeavyComponent'));

// 使用 startTransition 标记非紧急更新
function handleChange(value: string) {
  setInputValue(value); // 紧急更新
  startTransition(() => {
    setSearchQuery(value); // 非紧急更新
  });
}
```

### 组件模式

```tsx
// 复合组件模式
const Tabs = ({ children }: TabsProps) => {
  const [activeIndex, setActiveIndex] = useState(0);
  return (
    <TabsContext.Provider value={{ activeIndex, setActiveIndex }}>
      {children}
    </TabsContext.Provider>
  );
};

Tabs.List = TabList;
Tabs.Tab = Tab;
Tabs.Panels = TabPanels;
Tabs.Panel = TabPanel;

// 渲染属性模式
function DataFetcher<T>({ url, render }: DataFetcherProps<T>) {
  const { data, loading, error } = useFetch<T>(url);
  return render({ data, loading, error });
}

// 高阶组件模式
function withAuth<P>(Component: ComponentType<P>) {
  return function AuthenticatedComponent(props: P) {
    const { isAuthenticated } = useAuth();
    if (!isAuthenticated) return <Navigate to="/login" />;
    return <Component {...props} />;
  };
}
```

## 常见问题检查清单

- [ ] useEffect 依赖项是否完整？
- [ ] 是否存在闭包陷阱？
- [ ] key 是否稳定且唯一？
- [ ] 是否有内存泄漏？
- [ ] 组件是否过于庞大？
- [ ] 是否滥用 Context？
- [ ] 是否有不必要的重渲染？
- [ ] 错误边界是否覆盖？

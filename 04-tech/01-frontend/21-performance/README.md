# 前端性能优化最佳实践

## 角色设定

你是一位精通前端性能优化的专家，擅长加载性能、运行时性能和用户体验优化。

## 提示词模板

### 性能诊断

```
请帮我诊断以下性能问题：
- 问题描述：[首屏慢/交互卡顿/内存泄漏]
- 页面类型：[SPA/SSR/静态页面]
- 技术栈：[框架/构建工具]
- Lighthouse 分数：[LCP/FID/CLS]

当前情况：
[描述现状或粘贴性能数据]
```

### 优化方案

```
请帮我优化以下场景的性能：
- 场景描述：[大列表/复杂表单/实时数据]
- 性能指标目标：[具体指标]
- 当前瓶颈：[描述瓶颈]

请提供优化方案和代码示例。
```

## 核心优化示例

### 加载性能优化

```typescript
// 1. 路由懒加载
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

// 2. 预加载关键资源
<link rel="preload" href="/fonts/main.woff2" as="font" crossorigin />
<link rel="preload" href="/api/critical-data" as="fetch" crossorigin />

// 3. 预连接
<link rel="preconnect" href="https://api.example.com" />
<link rel="dns-prefetch" href="https://cdn.example.com" />

// 4. 动态导入非关键模块
async function loadAnalytics() {
  const { init } = await import('./analytics');
  init();
}

// 在空闲时加载
if ('requestIdleCallback' in window) {
  requestIdleCallback(loadAnalytics);
} else {
  setTimeout(loadAnalytics, 2000);
}
```

### 图片优化

```tsx
// 1. 响应式图片
function ResponsiveImage({ src, alt }: { src: string; alt: string }) {
  return (
    <picture>
      <source
        media="(max-width: 640px)"
        srcSet={`${src}?w=640 1x, ${src}?w=1280 2x`}
        type="image/webp"
      />
      <source
        media="(max-width: 1024px)"
        srcSet={`${src}?w=1024 1x, ${src}?w=2048 2x`}
        type="image/webp"
      />
      <img
        src={`${src}?w=1920`}
        alt={alt}
        loading="lazy"
        decoding="async"
      />
    </picture>
  );
}

// 2. 懒加载图片组件
function LazyImage({ src, alt, placeholder }: LazyImageProps) {
  const [loaded, setLoaded] = useState(false);
  const [inView, setInView] = useState(false);
  const imgRef = useRef<HTMLImageElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setInView(true);
          observer.disconnect();
        }
      },
      { rootMargin: '200px' }
    );

    if (imgRef.current) {
      observer.observe(imgRef.current);
    }

    return () => observer.disconnect();
  }, []);

  return (
    <div className="lazy-image" ref={imgRef}>
      {!loaded && <div className="placeholder">{placeholder}</div>}
      {inView && (
        <img
          src={src}
          alt={alt}
          onLoad={() => setLoaded(true)}
          style={{ opacity: loaded ? 1 : 0 }}
        />
      )}
    </div>
  );
}
```

### 列表虚拟化

```tsx
// 使用 react-window
import { FixedSizeList, VariableSizeList } from 'react-window';

// 固定高度列表
function VirtualList({ items }: { items: Item[] }) {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <FixedSizeList
      height={600}
      width="100%"
      itemCount={items.length}
      itemSize={50}
    >
      {Row}
    </FixedSizeList>
  );
}

// 动态高度列表
function DynamicList({ items }: { items: Item[] }) {
  const listRef = useRef<VariableSizeList>(null);
  const sizeMap = useRef<Record<number, number>>({});

  const getSize = (index: number) => sizeMap.current[index] || 50;

  const setSize = (index: number, size: number) => {
    sizeMap.current[index] = size;
    listRef.current?.resetAfterIndex(index);
  };

  return (
    <VariableSizeList
      ref={listRef}
      height={600}
      width="100%"
      itemCount={items.length}
      itemSize={getSize}
    >
      {({ index, style }) => (
        <DynamicRow
          index={index}
          style={style}
          item={items[index]}
          setSize={setSize}
        />
      )}
    </VariableSizeList>
  );
}
```

### React 渲染优化

```tsx
// 1. 使用 memo 避免重渲染
const ExpensiveComponent = memo(function ExpensiveComponent({ data }: Props) {
  return <div>{/* 复杂渲染 */}</div>;
}, (prevProps, nextProps) => {
  // 自定义比较函数
  return prevProps.data.id === nextProps.data.id;
});

// 2. useMemo 缓存计算
function DataTable({ items, filter }: Props) {
  const filteredItems = useMemo(() => {
    return items.filter(item => item.name.includes(filter));
  }, [items, filter]);

  const sortedItems = useMemo(() => {
    return [...filteredItems].sort((a, b) => a.name.localeCompare(b.name));
  }, [filteredItems]);

  return <Table data={sortedItems} />;
}

// 3. useCallback 缓存函数
function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback((id: string) => {
    console.log('clicked', id);
  }, []);

  return <Child onClick={handleClick} />;
}

// 4. 状态下沉
function App() {
  // ❌ 状态提升过高导致整体重渲染
  // const [inputValue, setInputValue] = useState('');

  return (
    <div>
      <Header />
      <SearchInput /> {/* ✅ 状态封装在组件内部 */}
      <Content />
    </div>
  );
}
```

### 防抖和节流

```typescript
// hooks/useDebounce.ts
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// hooks/useThrottle.ts
function useThrottle<T>(value: T, interval: number): T {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastUpdated = useRef(Date.now());

  useEffect(() => {
    const now = Date.now();
    if (now - lastUpdated.current >= interval) {
      lastUpdated.current = now;
      setThrottledValue(value);
    } else {
      const timer = setTimeout(() => {
        lastUpdated.current = Date.now();
        setThrottledValue(value);
      }, interval - (now - lastUpdated.current));
      return () => clearTimeout(timer);
    }
  }, [value, interval]);

  return throttledValue;
}

// 使用示例
function SearchInput() {
  const [input, setInput] = useState('');
  const debouncedInput = useDebounce(input, 300);

  useEffect(() => {
    if (debouncedInput) {
      search(debouncedInput);
    }
  }, [debouncedInput]);

  return <input value={input} onChange={(e) => setInput(e.target.value)} />;
}
```

### Web Worker

```typescript
// worker.ts
self.onmessage = function (e) {
  const { type, data } = e.data;

  if (type === 'HEAVY_COMPUTATION') {
    const result = heavyComputation(data);
    self.postMessage({ type: 'RESULT', data: result });
  }
};

function heavyComputation(data: number[]) {
  // CPU 密集型计算
  return data.reduce((acc, val) => acc + Math.sqrt(val), 0);
}

// 主线程使用
const worker = new Worker(new URL('./worker.ts', import.meta.url));

function useWorker() {
  const [result, setResult] = useState<number | null>(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    worker.onmessage = (e) => {
      if (e.data.type === 'RESULT') {
        setResult(e.data.data);
        setLoading(false);
      }
    };
  }, []);

  const compute = (data: number[]) => {
    setLoading(true);
    worker.postMessage({ type: 'HEAVY_COMPUTATION', data });
  };

  return { result, loading, compute };
}
```

### 性能监控

```typescript
// utils/performance.ts
export function measurePerformance() {
  // Core Web Vitals
  if ('web-vital' in window) {
    import('web-vitals').then(({ onCLS, onFID, onLCP, onFCP, onTTFB }) => {
      onCLS(console.log);
      onFID(console.log);
      onLCP(console.log);
      onFCP(console.log);
      onTTFB(console.log);
    });
  }

  // 自定义性能标记
  performance.mark('app-init-start');
  // ... 初始化代码
  performance.mark('app-init-end');
  performance.measure('app-init', 'app-init-start', 'app-init-end');

  // 资源加载性能
  const resources = performance.getEntriesByType('resource');
  resources.forEach((resource) => {
    console.log(`${resource.name}: ${resource.duration}ms`);
  });

  // 长任务监控
  const observer = new PerformanceObserver((list) => {
    list.getEntries().forEach((entry) => {
      console.warn('Long Task:', entry.duration);
    });
  });
  observer.observe({ entryTypes: ['longtask'] });
}
```

## 性能指标说明

| 指标 | 说明 | 目标值 |
|------|------|--------|
| LCP | 最大内容绘制 | < 2.5s |
| FID | 首次输入延迟 | < 100ms |
| CLS | 累积布局偏移 | < 0.1 |
| FCP | 首次内容绘制 | < 1.8s |
| TTFB | 首字节时间 | < 800ms |
| TTI | 可交互时间 | < 3.8s |

## 最佳实践清单

### 加载性能
- [ ] 代码分割和懒加载
- [ ] 预加载关键资源
- [ ] 压缩和缓存静态资源
- [ ] 使用 CDN 加速
- [ ] 优化图片 (WebP, 懒加载, 响应式)

### 运行时性能
- [ ] 避免不必要的重渲染
- [ ] 虚拟化长列表
- [ ] 防抖/节流高频事件
- [ ] Web Worker 处理复杂计算
- [ ] 及时清理事件监听和定时器

### 用户体验
- [ ] 骨架屏和加载提示
- [ ] 渐进式加载内容
- [ ] 预留图片空间避免 CLS
- [ ] 优化交互反馈

# Tailwind CSS 样式最佳实践

## 角色设定

你是一位精通 Tailwind CSS 的前端样式专家，擅长设计系统、响应式布局和组件样式封装。

## 提示词模板

### 样式实现

```
请用 Tailwind CSS 实现以下样式：
- 组件类型：[按钮/卡片/表单/导航/...]
- 设计要求：[描述视觉效果]
- 响应式需求：[移动端/平板/桌面]
- 状态变体：[hover/focus/active/disabled]
- 暗色模式：[是/否]
```

### 配置优化

```
请帮我优化 Tailwind CSS 配置：
- 项目类型：[React/Vue/Next.js]
- 设计系统：[是否有设计稿]
- 需要的功能：
  - [ ] 自定义颜色
  - [ ] 自定义字体
  - [ ] 自定义间距
  - [ ] 自定义动画
```

## 核心配置示例

### tailwind.config.js

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: [
    './index.html',
    './src/**/*.{js,ts,jsx,tsx}',
  ],
  darkMode: 'class',
  theme: {
    extend: {
      // 自定义颜色
      colors: {
        primary: {
          50: '#f0f9ff',
          100: '#e0f2fe',
          200: '#bae6fd',
          300: '#7dd3fc',
          400: '#38bdf8',
          500: '#0ea5e9',
          600: '#0284c7',
          700: '#0369a1',
          800: '#075985',
          900: '#0c4a6e',
          950: '#082f49',
        },
        gray: {
          // 覆盖默认灰色
        },
      },

      // 字体
      fontFamily: {
        sans: ['Inter', 'system-ui', 'sans-serif'],
        mono: ['Fira Code', 'monospace'],
      },

      // 间距
      spacing: {
        '18': '4.5rem',
        '112': '28rem',
        '128': '32rem',
      },

      // 动画
      animation: {
        'fade-in': 'fadeIn 0.3s ease-in-out',
        'slide-up': 'slideUp 0.3s ease-out',
        'spin-slow': 'spin 3s linear infinite',
      },
      keyframes: {
        fadeIn: {
          '0%': { opacity: '0' },
          '100%': { opacity: '1' },
        },
        slideUp: {
          '0%': { transform: 'translateY(10px)', opacity: '0' },
          '100%': { transform: 'translateY(0)', opacity: '1' },
        },
      },

      // 其他
      borderRadius: {
        '4xl': '2rem',
      },
      boxShadow: {
        'inner-lg': 'inset 0 2px 4px 0 rgb(0 0 0 / 0.1)',
      },
    },
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
    require('@tailwindcss/aspect-ratio'),
  ],
}
```

### 组件样式封装

```tsx
// 方式1: CVA (class-variance-authority)
import { cva, type VariantProps } from 'class-variance-authority';

const buttonVariants = cva(
  // 基础样式
  'inline-flex items-center justify-center rounded-md font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary-600 text-white hover:bg-primary-700',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200',
        outline: 'border border-gray-300 bg-transparent hover:bg-gray-100',
        ghost: 'hover:bg-gray-100',
        destructive: 'bg-red-600 text-white hover:bg-red-700',
      },
      size: {
        sm: 'h-8 px-3 text-sm',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'md',
    },
  }
);

interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  loading?: boolean;
}

function Button({ className, variant, size, loading, children, ...props }: ButtonProps) {
  return (
    <button
      className={buttonVariants({ variant, size, className })}
      disabled={loading}
      {...props}
    >
      {loading && <Spinner className="mr-2 h-4 w-4" />}
      {children}
    </button>
  );
}

// 方式2: Tailwind Merge
import { twMerge } from 'tailwind-merge';
import { clsx, type ClassValue } from 'clsx';

function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

function Card({ className, ...props }: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={cn(
        'rounded-lg border bg-white p-6 shadow-sm',
        'dark:border-gray-800 dark:bg-gray-950',
        className
      )}
      {...props}
    />
  );
}
```

### 响应式设计

```tsx
// 移动优先响应式
function ResponsiveLayout() {
  return (
    <div className="
      grid grid-cols-1 gap-4
      sm:grid-cols-2 sm:gap-6
      lg:grid-cols-3 lg:gap-8
      xl:grid-cols-4
    ">
      {items.map(item => <Card key={item.id} />)}
    </div>
  );
}

// 容器查询 (Tailwind v3.2+)
function ContainerQueryCard() {
  return (
    <div className="@container">
      <div className="
        flex flex-col
        @md:flex-row @md:items-center
        @lg:gap-8
      ">
        <img className="w-full @md:w-48" />
        <div className="flex-1" />
      </div>
    </div>
  );
}
```

### 暗色模式

```tsx
// 使用 class 策略
function ThemeToggle() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');

  useEffect(() => {
    document.documentElement.classList.toggle('dark', theme === 'dark');
  }, [theme]);

  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      {theme === 'light' ? <MoonIcon /> : <SunIcon />}
    </button>
  );
}

// 组件示例
function Card() {
  return (
    <div className="
      bg-white text-gray-900
      dark:bg-gray-900 dark:text-gray-100
      border border-gray-200
      dark:border-gray-800
    ">
      {/* 内容 */}
    </div>
  );
}
```

### 常用组件样式

```tsx
// 输入框
<input className="
  w-full rounded-md border border-gray-300 px-3 py-2
  text-sm placeholder:text-gray-400
  focus:border-primary-500 focus:outline-none focus:ring-2 focus:ring-primary-500/20
  disabled:cursor-not-allowed disabled:bg-gray-50 disabled:opacity-50
  dark:border-gray-700 dark:bg-gray-900 dark:text-white
" />

// 卡片
<div className="
  rounded-lg border border-gray-200 bg-white p-6 shadow-sm
  hover:shadow-md transition-shadow
  dark:border-gray-800 dark:bg-gray-950
">

// 徽章
<span className="
  inline-flex items-center rounded-full px-2.5 py-0.5
  text-xs font-medium
  bg-primary-100 text-primary-800
  dark:bg-primary-900 dark:text-primary-200
">

// 头像
<img className="
  h-10 w-10 rounded-full object-cover
  ring-2 ring-white
  dark:ring-gray-900
" />

// 骨架屏
<div className="animate-pulse">
  <div className="h-4 w-3/4 rounded bg-gray-200 dark:bg-gray-800" />
  <div className="mt-2 h-4 w-1/2 rounded bg-gray-200 dark:bg-gray-800" />
</div>
```

### 动画效果

```tsx
// 过渡动画
<button className="
  transition-all duration-200
  hover:scale-105 hover:shadow-lg
  active:scale-95
">

// 进入动画
<div className="
  animate-fade-in
  motion-safe:animate-slide-up
  motion-reduce:animate-none
">

// 加载动画
<svg className="animate-spin h-5 w-5 text-primary-600">
  {/* spinner SVG */}
</svg>

// 悬停效果
<div className="
  group relative overflow-hidden
">
  <img className="
    transition-transform duration-300
    group-hover:scale-110
  " />
  <div className="
    absolute inset-0 bg-black/50 opacity-0
    transition-opacity group-hover:opacity-100
  ">
</div>
```

### CSS 变量与主题

```css
/* globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --muted: 210 40% 96%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --border: 214.3 31.8% 91.4%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --border: 217.2 32.6% 17.5%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        border: 'hsl(var(--border))',
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
    },
  },
}
```

## 最佳实践清单

- [ ] 使用移动优先的响应式设计
- [ ] 使用 CVA 或 cn 函数封装变体
- [ ] 配置设计系统的颜色和间距
- [ ] 实现暗色模式支持
- [ ] 使用 @apply 提取重复样式（谨慎使用）
- [ ] 配置 PurgeCSS 减少打包体积
- [ ] 使用插件扩展功能
- [ ] 保持类名顺序一致

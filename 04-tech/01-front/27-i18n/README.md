# i18n 国际化最佳实践

## 角色设定

你是一位精通前端国际化的专家，擅长多语言支持、日期时间格式化和 RTL 布局。

## 提示词模板

### 国际化配置

```
请帮我实现国际化功能：
- 框架：[React/Vue/Next.js]
- 语言列表：[支持的语言]
- 翻译方案：[本地文件/远程加载]
- 特殊需求：[复数/日期/货币]

请提供完整的配置和代码。
```

## 核心代码示例

### react-i18next 配置

```typescript
// i18n/index.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';
import Backend from 'i18next-http-backend';

i18n
  .use(Backend)
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    supportedLngs: ['en', 'zh', 'ja', 'ko'],
    debug: import.meta.env.DEV,

    interpolation: {
      escapeValue: false,
    },

    backend: {
      loadPath: '/locales/{{lng}}/{{ns}}.json',
    },

    detection: {
      order: ['localStorage', 'navigator', 'htmlTag'],
      caches: ['localStorage'],
    },

    ns: ['common', 'home', 'user', 'order'],
    defaultNS: 'common',

    react: {
      useSuspense: true,
    },
  });

export default i18n;
```

### 翻译文件结构

```json
// locales/en/common.json
{
  "app": {
    "name": "My Application",
    "welcome": "Welcome, {{name}}!"
  },
  "nav": {
    "home": "Home",
    "products": "Products",
    "cart": "Cart",
    "profile": "Profile"
  },
  "actions": {
    "save": "Save",
    "cancel": "Cancel",
    "delete": "Delete",
    "confirm": "Confirm",
    "submit": "Submit",
    "loading": "Loading..."
  },
  "messages": {
    "success": "Operation successful",
    "error": "An error occurred",
    "confirm_delete": "Are you sure you want to delete?"
  },
  "validation": {
    "required": "This field is required",
    "email": "Please enter a valid email",
    "min_length": "Minimum {{count}} characters required"
  },
  "time": {
    "just_now": "Just now",
    "minutes_ago": "{{count}} minute ago",
    "minutes_ago_plural": "{{count}} minutes ago",
    "hours_ago": "{{count}} hour ago",
    "hours_ago_plural": "{{count}} hours ago",
    "days_ago": "{{count}} day ago",
    "days_ago_plural": "{{count}} days ago"
  }
}
```

```json
// locales/zh/common.json
{
  "app": {
    "name": "我的应用",
    "welcome": "欢迎，{{name}}！"
  },
  "nav": {
    "home": "首页",
    "products": "商品",
    "cart": "购物车",
    "profile": "个人中心"
  },
  "actions": {
    "save": "保存",
    "cancel": "取消",
    "delete": "删除",
    "confirm": "确认",
    "submit": "提交",
    "loading": "加载中..."
  },
  "messages": {
    "success": "操作成功",
    "error": "发生错误",
    "confirm_delete": "确定要删除吗？"
  },
  "validation": {
    "required": "此字段必填",
    "email": "请输入有效的邮箱地址",
    "min_length": "至少需要 {{count}} 个字符"
  },
  "time": {
    "just_now": "刚刚",
    "minutes_ago": "{{count}} 分钟前",
    "minutes_ago_plural": "{{count}} 分钟前",
    "hours_ago": "{{count}} 小时前",
    "hours_ago_plural": "{{count}} 小时前",
    "days_ago": "{{count}} 天前",
    "days_ago_plural": "{{count}} 天前"
  }
}
```

### React 组件使用

```tsx
// components/Header.tsx
import { useTranslation } from 'react-i18next';

export function Header() {
  const { t, i18n } = useTranslation();

  const changeLanguage = (lng: string) => {
    i18n.changeLanguage(lng);
  };

  return (
    <header className="flex items-center justify-between p-4">
      <h1>{t('app.name')}</h1>

      <nav className="flex gap-4">
        <a href="/">{t('nav.home')}</a>
        <a href="/products">{t('nav.products')}</a>
        <a href="/cart">{t('nav.cart')}</a>
        <a href="/profile">{t('nav.profile')}</a>
      </nav>

      <div className="flex gap-2">
        <button
          onClick={() => changeLanguage('en')}
          className={i18n.language === 'en' ? 'active' : ''}
        >
          EN
        </button>
        <button
          onClick={() => changeLanguage('zh')}
          className={i18n.language === 'zh' ? 'active' : ''}
        >
          中文
        </button>
      </div>
    </header>
  );
}

// 带命名空间的使用
export function UserProfile() {
  const { t } = useTranslation('user');

  return (
    <div>
      <h2>{t('profile.title')}</h2>
      <p>{t('profile.description')}</p>
    </div>
  );
}
```

### 复数和插值

```tsx
// components/Cart.tsx
import { useTranslation } from 'react-i18next';

export function Cart({ items }) {
  const { t } = useTranslation();

  return (
    <div>
      {/* 插值 */}
      <h1>{t('app.welcome', { name: 'John' })}</h1>

      {/* 复数 */}
      <p>
        {t('cart.items', { count: items.length })}
        {/* en: "1 item" / "5 items" */}
        {/* zh: "1 件商品" / "5 件商品" */}
      </p>

      {/* 嵌套键 */}
      <span>{t('order.status.pending')}</span>

      {/* 默认值 */}
      <span>{t('unknown.key', 'Default text')}</span>

      {/* HTML 内容 */}
      <p
        dangerouslySetInnerHTML={{
          __html: t('terms.html', { link: '/terms' }),
        }}
      />
    </div>
  );
}
```

### 日期时间格式化

```typescript
// utils/formatters.ts
import { format, formatDistance, formatRelative } from 'date-fns';
import { enUS, zhCN, ja, ko } from 'date-fns/locale';
import i18n from '../i18n';

const locales: Record<string, Locale> = {
  en: enUS,
  zh: zhCN,
  ja: ja,
  ko: ko,
};

export function formatDate(date: Date, pattern = 'PPP'): string {
  return format(date, pattern, {
    locale: locales[i18n.language] || enUS,
  });
}

export function formatRelativeTime(date: Date): string {
  return formatDistance(date, new Date(), {
    addSuffix: true,
    locale: locales[i18n.language] || enUS,
  });
}

// 使用 Intl API
export function formatNumber(value: number): string {
  return new Intl.NumberFormat(i18n.language).format(value);
}

export function formatCurrency(value: number, currency = 'USD'): string {
  return new Intl.NumberFormat(i18n.language, {
    style: 'currency',
    currency,
  }).format(value);
}

export function formatPercent(value: number): string {
  return new Intl.NumberFormat(i18n.language, {
    style: 'percent',
    minimumFractionDigits: 0,
    maximumFractionDigits: 2,
  }).format(value);
}
```

### 语言切换器组件

```tsx
// components/LanguageSwitch.tsx
import { useTranslation } from 'react-i18next';

const languages = [
  { code: 'en', name: 'English', flag: '🇺🇸' },
  { code: 'zh', name: '中文', flag: '🇨🇳' },
  { code: 'ja', name: '日本語', flag: '🇯🇵' },
  { code: 'ko', name: '한국어', flag: '🇰🇷' },
];

export function LanguageSwitch() {
  const { i18n } = useTranslation();

  return (
    <select
      value={i18n.language}
      onChange={(e) => i18n.changeLanguage(e.target.value)}
      className="px-3 py-2 border rounded-lg"
    >
      {languages.map((lang) => (
        <option key={lang.code} value={lang.code}>
          {lang.flag} {lang.name}
        </option>
      ))}
    </select>
  );
}
```

### 自定义 Hook

```typescript
// hooks/useLocale.ts
import { useTranslation } from 'react-i18next';
import { useMemo } from 'react';

export function useLocale() {
  const { i18n } = useTranslation();

  return useMemo(
    () => ({
      language: i18n.language,
      isRTL: ['ar', 'he', 'fa'].includes(i18n.language),
      dir: ['ar', 'he', 'fa'].includes(i18n.language) ? 'rtl' : 'ltr',
      changeLanguage: i18n.changeLanguage,
    }),
    [i18n]
  );
}

// hooks/useFormattedDate.ts
import { useMemo } from 'react';
import { useTranslation } from 'react-i18next';
import { formatDate, formatRelativeTime } from '../utils/formatters';

export function useFormattedDate(date: Date | string) {
  const { i18n } = useTranslation();
  const dateObj = useMemo(() => new Date(date), [date]);

  return useMemo(
    () => ({
      full: formatDate(dateObj, 'PPPPpp'),
      short: formatDate(dateObj, 'P'),
      relative: formatRelativeTime(dateObj),
    }),
    [dateObj, i18n.language]
  );
}
```

### Next.js 国际化

```typescript
// next.config.js
module.exports = {
  i18n: {
    locales: ['en', 'zh', 'ja'],
    defaultLocale: 'en',
    localeDetection: true,
  },
};

// pages/index.tsx
import { GetStaticProps } from 'next';
import { serverSideTranslations } from 'next-i18next/serverSideTranslations';
import { useTranslation } from 'next-i18next';

export default function Home() {
  const { t } = useTranslation('common');

  return (
    <div>
      <h1>{t('app.name')}</h1>
    </div>
  );
}

export const getStaticProps: GetStaticProps = async ({ locale }) => {
  return {
    props: {
      ...(await serverSideTranslations(locale ?? 'en', ['common'])),
    },
  };
};
```

### RTL 支持

```css
/* styles/rtl.css */
[dir='rtl'] {
  direction: rtl;
  text-align: right;
}

[dir='rtl'] .flex {
  flex-direction: row-reverse;
}

[dir='rtl'] .ml-4 {
  margin-left: 0;
  margin-right: 1rem;
}

[dir='rtl'] .mr-4 {
  margin-right: 0;
  margin-left: 1rem;
}
```

```tsx
// App.tsx
import { useLocale } from './hooks/useLocale';

export function App() {
  const { dir } = useLocale();

  return (
    <div dir={dir} className="min-h-screen">
      {/* 应用内容 */}
    </div>
  );
}
```

### 类型安全

```typescript
// types/i18n.d.ts
import 'i18next';
import common from '../locales/en/common.json';
import user from '../locales/en/user.json';

declare module 'i18next' {
  interface CustomTypeOptions {
    defaultNS: 'common';
    resources: {
      common: typeof common;
      user: typeof user;
    };
  }
}
```

## 最佳实践清单

- [ ] 使用命名空间组织翻译
- [ ] 实现语言自动检测
- [ ] 支持复数形式
- [ ] 日期时间本地化
- [ ] 数字货币格式化
- [ ] RTL 布局支持
- [ ] 翻译文件按需加载
- [ ] 类型安全的翻译键

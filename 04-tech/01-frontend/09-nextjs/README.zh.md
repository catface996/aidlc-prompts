# Next.js 14+ 最佳实践

## 角色设定
你是一位精通 Next.js 14+ 的全栈开发专家，擅长 App Router、React Server Components、数据获取策略和性能优化。你深刻理解服务端与客户端的边界，能够构建高性能、SEO 友好的现代 Web 应用，并充分利用 Next.js 的增量生成、边缘计算和流式渲染等高级特性。

---

## 核心原则 (NON-NEGOTIABLE)
| 原则 | 要求 | 违反后果 |
|------|------|----------|
| **Server First** | MUST 默认使用 Server Components，ONLY 在需要交互时添加 'use client' | 客户端 Bundle 膨胀，首屏性能下降，失去 RSC 优势 |
| **服务端数据获取** | MUST 在服务端（load 函数或 Server Components）获取数据，NEVER 在客户端组件中直接 fetch | SEO 受损，瀑布流请求，用户体验差 |
| **并行优先** | MUST 使用 Promise.all 并行获取独立数据，NEVER 串行 await | 请求时间线性累加，TTFB 过高 |
| **缓存策略** | MUST 为每个 fetch 明确指定缓存策略（force-cache/no-store/revalidate），NEVER 依赖默认行为 | 缓存行为不可预测，数据新鲜度问题 |
| **错误边界** | MUST 为每个路由段提供 error.tsx 和 loading.tsx，NEVER 让错误传播到根 | 用户体验差，错误粒度过粗，无即时反馈 |
| **类型安全** | MUST 使用 TypeScript 并配置 generateTypedParams，NEVER 使用 any 类型 | 运行时错误，重构困难，参数类型不安全 |
| **资源优化** | MUST 使用 next/image 和 next/font，NEVER 使用原生标签直接加载 | 性能损失，未优化的图片和字体加载 |
| **Metadata 配置** | MUST 为每个页面配置 metadata 或 generateMetadata，NEVER 忽略 SEO | 搜索引擎抓取困难，社交分享无预览 |

---

## 提示词模板

### 页面开发
```
请帮我创建一个 Next.js 页面：
- 页面路径：[路由路径，如 /posts/[id]]
- 渲染模式：[SSG/SSR/ISR/客户端]
- 数据来源：[API/数据库/CMS/第三方服务]
- 是否需要布局：[是/否，布局类型]
- 是否需要并行路由：[是/否]

功能需求：
1. [需求1]
2. [需求2]

请使用 App Router、TypeScript、Server Components，并包含 metadata 配置。
```

### Server/Client 组件划分
```
请帮我设计 Server/Client 组件架构：
- 页面功能描述：[详细描述功能]
- 需要交互的部分：[列出所有交互需求，如表单、按钮、动画]
- 数据依赖：[描述数据来源和更新频率]
- 第三方库依赖：[列出需要的库及其要求]

请说明：
1. 哪些应该是 Server Components
2. 哪些必须是 Client Components（'use client'）
3. 如何在它们之间传递数据
4. 如何优化客户端 Bundle 大小
```

### 数据获取策略
```
请帮我实现 Next.js 数据获取方案：
- 数据来源：[具体的 API/数据库/服务]
- 数据类型：[用户数据/内容数据/配置数据]
- 更新频率：[实时/频繁/较少/静态]
- 缓存策略：[no-store/force-cache/revalidate]
- 重新验证方式：[基于时间/按需/标签]
- 错误处理要求：[降级方案/重试策略]

请包含：
1. 数据获取实现（并行/串行）
2. 缓存配置
3. 加载状态处理
4. 错误状态处理
5. TypeScript 类型定义
```

### API Routes
```
请帮我创建 Next.js API 路由：
- 路由路径：[API 路径]
- HTTP 方法：[GET/POST/PUT/DELETE/PATCH]
- 请求参数：[Query/Body/Headers]
- 响应格式：[JSON 结构描述]
- 认证要求：[是否需要认证/授权]

请包含：
1. 参数验证（使用 Zod 或类似库）
2. 错误处理和适当的状态码
3. TypeScript 类型定义
4. 速率限制考虑（如需要）
5. 日志记录
```

### 性能优化
```
请帮我优化 Next.js 应用性能：
- 当前问题：
  - [ ] 首屏加载慢（LCP > 2.5s）
  - [ ] TTFB 过高（> 800ms）
  - [ ] Bundle 体积大（> 200KB）
  - [ ] 图片加载慢
  - [ ] 水合时间长
  - [ ] 其他：[描述问题]

- 应用信息：
  - 页面类型：[落地页/仪表板/电商]
  - 数据获取方式：[当前实现]
  - 使用的库：[列出主要依赖]

请提供具体的优化方案，包括代码结构调整和配置优化。
```

---

## 决策指南

### 组件类型决策树
```
创建组件？
├─ 是否需要客户端交互？
│  ├─ 否（纯展示）→ Server Component（默认）
│  ├─ 是 → 检查交互类型
│  │  ├─ 使用 Hooks（useState/useEffect/useContext）→ Client Component
│  │  ├─ 使用浏览器 API（window/document）→ Client Component
│  │  ├─ 使用事件监听器（onClick/onChange）→ Client Component
│  │  └─ 使用第三方 React 库（需要 hooks）→ Client Component
│  │
│  └─ 组件层级考虑
│     ├─ 尽可能将 'use client' 下移到叶子节点
│     ├─ 将交互逻辑提取为独立的小型 Client Component
│     └─ Server Component 可以导入 Client Component
│
└─ 数据依赖？
   ├─ 需要服务端资源（数据库/文件系统）→ Server Component
   ├─ 需要敏感信息（API 密钥/令牌）→ Server Component
   └─ 纯客户端状态 → Client Component
```

### 数据获取决策树
```
数据获取策略？
├─ 数据更新频率？
│  ├─ 实时/每次请求都不同 → fetch with cache: 'no-store'
│  ├─ 频繁更新（分钟级）→ ISR with revalidate: 60
│  ├─ 较少更新（小时级）→ ISR with revalidate: 3600
│  └─ 几乎不变 → SSG with cache: 'force-cache'
│
├─ 数据来源？
│  ├─ 外部 API → 在 Server Component 或 Route Handler 中 fetch
│  ├─ 数据库 → 在 Server Component 或 Server Action 中查询
│  ├─ CMS → 使用 SDK 在服务端获取
│  └─ 文件系统 → 在 Server Component 中读取
│
├─ 并行 vs 串行？
│  ├─ 独立数据源 → Promise.all 并行获取
│  ├─ 有依赖关系 → 串行 await
│  └─ 部分依赖 → 混合策略（先并行再串行）
│
└─ 错误处理？
   ├─ 关键数据 → 抛出错误，触发 error.tsx
   ├─ 可选数据 → try-catch 并提供降级内容
   └─ 需要重试 → 实现重试逻辑
```

### 渲染模式决策树
```
选择渲染模式？
├─ 页面特性？
│  ├─ 内容几乎不变（营销页/文档）→ SSG (generateStaticParams)
│  ├─ 内容定期更新（博客/新闻）→ ISR (revalidate)
│  ├─ 内容个性化（仪表板/用户页）→ SSR (no cache)
│  └─ 纯客户端应用（工具/游戏）→ SPA (dynamic = 'force-dynamic')
│
├─ 数据私密性？
│  ├─ 公开内容 → SSG/ISR
│  ├─ 需要认证 → SSR + Middleware
│  └─ 用户特定 → SSR
│
└─ SEO 需求？
   ├─ 需要 SEO → SSG/ISR/SSR（优先级递减）
   ├─ 不需要 SEO → 客户端渲染可接受
   └─ 动态内容需要 SEO → SSR 或 ISR
```

### 缓存策略决策树
```
配置缓存策略？
├─ fetch 请求缓存
│  ├─ 每次都要最新数据 → { cache: 'no-store' }
│  ├─ 缓存特定时间 → { next: { revalidate: 秒数 } }
│  ├─ 永久缓存 → { cache: 'force-cache' }
│  └─ 按需重新验证 → 使用 revalidateTag 或 revalidatePath
│
├─ 路由段缓存
│  ├─ 静态页面 → 默认缓存（SSG）
│  ├─ 动态页面 → export const dynamic = 'force-dynamic'
│  └─ 混合页面 → 部分预渲染（PPR）
│
└─ 数据缓存
   ├─ unstable_cache 包装数据查询
   ├─ 配置缓存标签用于重新验证
   └─ 使用 revalidateTag 按需更新
```

---

## 正反对比示例

### Server/Client Components 使用
| 场景 | ❌ 错误做法 | ✅ 正确做法 | 说明 |
|------|-----------|-----------|------|
| 纯展示组件 | 在文件顶部添加 'use client' | 默认为 Server Component，不添加 'use client' | Server Component 可以直接访问服务端资源 |
| 交互组件 | 整个页面标记为 'use client' | 只将需要交互的叶子组件标记为 'use client' | 最小化客户端 Bundle，保留 RSC 优势 |
| 数据获取 | Client Component 中使用 useEffect + fetch | Server Component 中直接 await fetch | 减少客户端请求，更好的 SEO |
| 混合使用 | Client Component 导入 Server Component | Server Component 导入 Client Component | 遵循正确的组件边界 |
| 第三方库 | 在 Server Component 中使用浏览器库 | 创建包装的 Client Component | 避免运行时错误 |

### 数据获取策略
| 场景 | ❌ 错误做法 | ✅ 正确做法 | 说明 |
|------|-----------|-----------|------|
| 独立数据 | 串行 await 多个独立请求 | 使用 Promise.all 并行获取 | 减少总等待时间，提升 TTFB |
| 缓存配置 | 不指定缓存选项，依赖默认 | 明确指定 cache 或 revalidate | 缓存行为可预测可控 |
| 数据重复 | 多个组件重复 fetch 相同 URL | 依赖 Next.js 自动去重 | 但建议提升到父组件 |
| 错误处理 | 不处理 fetch 失败 | try-catch 或抛出错误触发 error.tsx | 提供降级体验 |
| 敏感数据 | 在客户端 fetch 使用 API 密钥 | 在 Server Component 或 Route Handler 中处理 | 防止密钥泄露 |

### 路由和布局
| 场景 | ❌ 错误做法 | ✅ 正确做法 | 说明 |
|------|-----------|-----------|------|
| 共享布局 | 在每个页面重复布局代码 | 使用 layout.tsx 定义共享布局 | 复用代码，保持布局状态 |
| 加载状态 | 不提供加载反馈 | 创建 loading.tsx 提供即时反馈 | 改善用户体验 |
| 错误处理 | 让错误传播到根 error.tsx | 为每个路由段创建 error.tsx | 细粒度错误处理 |
| 动态路由 | 硬编码所有路由 | 使用 [param] 和 generateStaticParams | 灵活处理动态内容 |
| 路由组织 | 平铺所有路由 | 使用 (group) 组织相关路由 | 更好的代码组织 |

### 性能优化
| 场景 | ❌ 错误做法 | ✅ 正确做法 | 说明 |
|------|-----------|-----------|------|
| 图片加载 | 使用 img 标签 | 使用 next/image 组件 | 自动优化、懒加载、响应式 |
| 字体加载 | CDN 链接或本地 font-face | 使用 next/font 优化字体加载 | 消除布局偏移，性能更好 |
| 第三方脚本 | 直接使用 script 标签 | 使用 next/script 组件 | 控制加载时机和顺序 |
| 代码分割 | 导入所有组件 | 使用 dynamic 动态导入 | 减少初始 Bundle 大小 |
| 客户端包体积 | 在 Server Component 导入大型库 | 只在需要时导入，使用 barrel imports | 优化包体积 |

### Metadata 和 SEO
| 场景 | ❌ 错误做法 | ✅ 正确做法 | 说明 |
|------|-----------|-----------|------|
| 静态 metadata | 在 HTML 中硬编码 meta 标签 | 导出 metadata 对象 | 类型安全，更易维护 |
| 动态 metadata | 在客户端设置 document.title | 实现 generateMetadata 函数 | SSR 友好，SEO 更好 |
| OG 图片 | 不配置社交分享图 | 配置 openGraph.images | 社交分享更美观 |
| 结构化数据 | 不添加 JSON-LD | 使用 script type="application/ld+json" | 增强 SEO |

---

## 验证清单 (Validation Checklist)

### App Router 架构
- [ ] 是否最大化使用 Server Components？
- [ ] Client Components 是否只在必要时使用？
- [ ] 'use client' 是否下移到叶子节点？
- [ ] 是否正确理解 Server/Client 边界？
- [ ] 是否避免了 Client Component 导入 Server Component？

### 数据获取
- [ ] 数据是否在服务端获取（Server Components 或 Route Handlers）？
- [ ] 是否为每个 fetch 明确指定了缓存策略？
- [ ] 独立数据是否使用 Promise.all 并行获取？
- [ ] 是否处理了加载状态（loading.tsx 或 Suspense）？
- [ ] 是否处理了错误状态（error.tsx 或 try-catch）？
- [ ] 是否正确使用了 revalidate 或 revalidateTag？

### 路由和文件结构
- [ ] 是否使用了 layout.tsx 复用共享布局？
- [ ] 每个路由段是否有 loading.tsx？
- [ ] 每个路由段是否有 error.tsx？
- [ ] 动态路由是否实现了 generateStaticParams（如果适用）？
- [ ] 是否使用路由组 (group) 组织相关路由？
- [ ] 是否正确配置了 not-found.tsx？

### TypeScript 和类型安全
- [ ] 是否为所有组件提供了类型定义？
- [ ] 是否使用了自动生成的类型（如 PageProps）？
- [ ] 是否避免使用 any 类型？
- [ ] API 响应是否有类型定义？
- [ ] 是否使用 Zod 或类似库验证外部数据？

### 性能优化
- [ ] 图片是否使用 next/image？
- [ ] 字体是否使用 next/font？
- [ ] 第三方脚本是否使用 next/script？
- [ ] 是否使用 dynamic 动态导入大型组件？
- [ ] 是否优化了客户端 Bundle 大小？
- [ ] 是否配置了适当的缓存策略？
- [ ] 是否使用了 Streaming 和 Suspense？

### SEO 和 Metadata
- [ ] 每个页面是否配置了 metadata 或 generateMetadata？
- [ ] 是否配置了 openGraph 和 twitter 卡片？
- [ ] 是否提供了合适的 robots 和 sitemap？
- [ ] 动态页面是否生成了正确的 canonical URL？
- [ ] 是否添加了结构化数据（JSON-LD）？

### 用户体验
- [ ] 是否提供了即时的加载反馈？
- [ ] 错误是否有友好的提示和恢复方式？
- [ ] 表单提交是否有 pending 状态？
- [ ] 是否处理了网络失败和超时？
- [ ] 是否考虑了可访问性（ARIA 属性）？

---

## 护栏约束 (Guardrails)

**允许 (✅)**：
- 默认使用 Server Components
- 在 Server Components 中直接访问数据库和文件系统
- 在 Server Components 中使用 async/await 获取数据
- 使用 Promise.all 并行获取独立数据
- 为 fetch 明确指定缓存选项（cache/revalidate）
- 使用 loading.tsx 和 error.tsx 提供即时反馈
- 使用 generateMetadata 配置动态 metadata
- 使用 next/image、next/font、next/script 优化资源
- 使用 Server Actions 处理表单提交和数据变更
- 使用 revalidatePath 或 revalidateTag 按需重新验证

**禁止 (❌)**：
- 在 Client Components 中直接访问数据库或文件系统
- 在 Client Components 中暴露 API 密钥或敏感信息
- 串行 await 独立的数据请求
- 不指定缓存策略，完全依赖默认行为
- 让错误传播到根而不处理
- 使用 any 类型绕过类型检查
- 在 Server Components 中使用浏览器 API（window、document）
- Client Component 导入 Server Component
- 使用原生 img、script、link 标签加载优化资源
- 在页面组件中执行数据变更操作（应使用 Server Actions）

**需澄清 (⚠️)**：
- 这个组件应该是 Server Component 还是 Client Component？
- 数据应该使用什么缓存策略（force-cache/no-store/revalidate）？
- 这个页面应该使用什么渲染模式（SSG/SSR/ISR）？
- 多个数据请求是否可以并行？
- 错误应该在哪个层级处理（组件级 vs 路由段级）？
- 这个功能应该使用 Route Handler 还是 Server Action？
- 何时使用 Partial Prerendering（PPR）？
- 何时使用 Middleware vs Server Component？

---

## 常见问题诊断

| 症状 | 可能原因 | 诊断方法 | 解决方案 |
|------|---------|---------|---------|
| 水合错误 | Server/Client 渲染不一致 | 检查浏览器控制台 Hydration Error | 确保服务端和客户端渲染相同内容，避免使用随机值或时间戳 |
| 无限重渲染 | useEffect 依赖配置错误 | 检查 useEffect 依赖数组 | 修复依赖或使用 useRef 存储不变的值 |
| 数据不更新 | 缓存配置过于激进 | 检查 fetch 的 cache 选项 | 使用 no-store 或设置 revalidate |
| TTFB 过高 | 数据串行获取 | 分析 Network waterfall | 使用 Promise.all 并行获取 |
| Bundle 过大 | 导入了大型库或未分割代码 | 使用 Bundle Analyzer 分析 | 使用 dynamic 动态导入，按需加载 |
| Client Component 错误 | Server Component 使用了客户端 API | 检查是否添加了 'use client' | 将使用客户端 API 的组件标记为 Client Component |
| Metadata 不生效 | 在 Client Component 中配置 | 检查组件是否有 'use client' | 将 metadata 移至 Server Component 或使用 generateMetadata |
| 图片加载慢 | 使用原生 img 标签 | 检查是否使用 next/image | 迁移到 next/image 组件 |
| 路由跳转失败 | 使用了 a 标签或 window.location | 检查导航实现 | 使用 next/link 或 useRouter |
| 表单提交失败 | Server Action 配置错误 | 检查 action 和 formData | 确保 Server Action 正确标记 'use server' |
| 404 错误 | 动态路由未生成 | 检查 generateStaticParams | 实现 generateStaticParams 或使用动态渲染 |
| 缓存不刷新 | 未配置重新验证 | 检查缓存配置 | 添加 revalidate 或使用 revalidatePath |
| 类型错误 | 使用了过时的类型 | 检查 TypeScript 配置 | 更新到最新类型定义，使用自动生成的类型 |
| 环境变量未定义 | 未正确配置环境变量 | 检查 .env 文件和变量前缀 | 使用 NEXT_PUBLIC_ 前缀（客户端）或在服务端访问 |

---

## 输出格式要求

### 页面开发输出格式
```
1. 文件结构
   - app/路径/page.tsx（页面组件）
   - app/路径/layout.tsx（布局组件，如需要）
   - app/路径/loading.tsx（加载状态）
   - app/路径/error.tsx（错误处理）

2. 组件架构
   - Server Components 列表及职责
   - Client Components 列表及职责（标记 'use client'）
   - 组件之间的数据流

3. 数据获取实现
   - 数据源和获取方式
   - 缓存策略配置
   - 并行/串行策略
   - 错误处理逻辑

4. TypeScript 类型
   - PageProps 类型定义
   - 数据接口定义
   - API 响应类型

5. Metadata 配置
   - 静态 metadata 或 generateMetadata
   - OpenGraph 配置
   - SEO 优化说明
```

### 数据获取方案输出格式
```
1. 数据获取策略
   - 获取位置（Server Component/Route Handler）
   - 缓存配置（cache/revalidate）
   - 并行策略（Promise.all 或串行）

2. 实现步骤
   - 定义数据获取函数
   - 配置缓存选项
   - 实现错误处理
   - 添加 TypeScript 类型

3. 缓存和重新验证
   - 缓存键设计
   - 重新验证触发条件
   - 使用 revalidatePath 或 revalidateTag

4. 降级和错误处理
   - 错误边界配置
   - 降级内容设计
   - 重试策略

5. 性能优化
   - 减少请求瀑布流
   - 使用 Streaming 和 Suspense
   - 优化数据查询
```

### API Routes 输出格式
```
1. 路由文件结构
   - app/api/路径/route.ts
   - HTTP 方法实现（GET/POST/PUT/DELETE）

2. 请求处理
   - 参数获取（searchParams/body/headers）
   - 参数验证（使用 Zod）
   - 业务逻辑处理

3. 响应处理
   - 成功响应格式
   - 错误响应格式
   - 状态码使用规范

4. 类型定义
   - 请求参数类型
   - 响应数据类型
   - 错误类型

5. 安全和性能
   - 认证/授权检查
   - 速率限制
   - 错误日志记录
```

### 性能优化输出格式
```
1. 问题诊断
   - 性能指标分析（LCP/FID/CLS/TTFB）
   - 瓶颈识别
   - 优先级排序

2. 优化方案
   - 数据获取优化（并行、缓存）
   - 组件优化（Server/Client 划分）
   - 资源优化（图片、字体、脚本）
   - Bundle 优化（动态导入、Tree Shaking）

3. 实施步骤
   - 具体代码修改
   - 配置调整
   - 验证方法

4. 预期效果
   - 性能指标改善预期
   - 用户体验提升
   - 权衡说明

5. 监控和验证
   - 性能监控设置
   - A/B 测试方案
   - 回滚计划
```

# Vite 构建工具最佳实践

## 角色设定

你是一位精通 Vite 的前端工程化专家，擅长现代化构建配置和开发体验优化。

## 提示词模板

### 项目配置

```
请帮我配置 Vite 项目：
- 框架：[React/Vue/Svelte/原生]
- 语言：[JavaScript/TypeScript]
- 样式方案：[CSS/Sass/Less/Tailwind]
- 部署目标：[静态托管/SSR/SSG]

需要的功能：
- [ ] 路径别名
- [ ] 环境变量
- [ ] 代理配置
- [ ] 打包分析
```

### 插件开发

```
请帮我开发一个 Vite 插件：
- 插件名称：[名称]
- 功能描述：[描述]
- 作用时机：[配置/构建/服务器]
```

## 核心配置示例

### 基础配置

```typescript
// vite.config.ts
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');

  return {
    plugins: [react()],

    resolve: {
      alias: {
        '@': path.resolve(__dirname, 'src'),
        '@components': path.resolve(__dirname, 'src/components'),
        '@utils': path.resolve(__dirname, 'src/utils'),
      },
    },

    css: {
      modules: {
        localsConvention: 'camelCase',
      },
      preprocessorOptions: {
        scss: {
          additionalData: `@import "@/styles/variables.scss";`,
        },
      },
    },

    server: {
      port: 3000,
      open: true,
      proxy: {
        '/api': {
          target: env.VITE_API_BASE,
          changeOrigin: true,
          rewrite: (path) => path.replace(/^\/api/, ''),
        },
      },
    },

    build: {
      target: 'es2015',
      outDir: 'dist',
      sourcemap: true,
      rollupOptions: {
        output: {
          manualChunks: {
            vendor: ['react', 'react-dom'],
            router: ['react-router-dom'],
          },
        },
      },
    },

    define: {
      __APP_VERSION__: JSON.stringify(process.env.npm_package_version),
    },
  };
});
```

### 生产优化配置

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import { visualizer } from 'rollup-plugin-visualizer';
import viteCompression from 'vite-plugin-compression';
import { createHtmlPlugin } from 'vite-plugin-html';

export default defineConfig({
  plugins: [
    // Gzip 压缩
    viteCompression({
      algorithm: 'gzip',
      threshold: 10240,
    }),

    // HTML 压缩
    createHtmlPlugin({
      minify: true,
    }),

    // 打包分析
    visualizer({
      open: true,
      filename: 'stats.html',
    }),
  ],

  build: {
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },

    rollupOptions: {
      output: {
        chunkFileNames: 'js/[name]-[hash].js',
        entryFileNames: 'js/[name]-[hash].js',
        assetFileNames: '[ext]/[name]-[hash].[ext]',

        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('react')) return 'react-vendor';
            if (id.includes('lodash')) return 'lodash';
            return 'vendor';
          }
        },
      },
    },

    // 分块大小警告
    chunkSizeWarningLimit: 500,
  },
});
```

### 常用插件配置

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import legacy from '@vitejs/plugin-legacy';
import { VitePWA } from 'vite-plugin-pwa';
import svgr from 'vite-plugin-svgr';

export default defineConfig({
  plugins: [
    react(),

    // 兼容旧浏览器
    legacy({
      targets: ['defaults', 'not IE 11'],
    }),

    // PWA 支持
    VitePWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'My App',
        short_name: 'App',
        theme_color: '#ffffff',
      },
    }),

    // SVG 作为组件导入
    svgr(),
  ],
});
```

### 环境变量

```bash
# .env
VITE_APP_TITLE=My App

# .env.development
VITE_API_BASE=http://localhost:8080

# .env.production
VITE_API_BASE=https://api.example.com
```

```typescript
// 使用环境变量
const apiBase = import.meta.env.VITE_API_BASE;
const isDev = import.meta.env.DEV;
const isProd = import.meta.env.PROD;

// 类型声明 env.d.ts
/// <reference types="vite/client" />
interface ImportMetaEnv {
  readonly VITE_API_BASE: string;
  readonly VITE_APP_TITLE: string;
}
```

## 最佳实践清单

- [ ] 使用 TypeScript 配置文件
- [ ] 配置路径别名简化导入
- [ ] 合理配置 manualChunks 分包
- [ ] 使用 visualizer 分析包体积
- [ ] 生产环境启用压缩
- [ ] 配置 legacy 插件兼容旧浏览器
- [ ] 使用环境变量区分环境配置
- [ ] 利用预构建加速依赖加载

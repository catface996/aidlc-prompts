# Webpack 构建工具最佳实践

## 角色设定

你是一位精通 Webpack 5 的前端工程化专家，擅长配置优化、性能调优和构建流程设计。

## 提示词模板

### 项目配置

```
请帮我配置 Webpack 项目：
- 项目类型：[SPA/MPA/库/组件库]
- 框架：[React/Vue/原生]
- 语言：[JavaScript/TypeScript]
- 样式方案：[CSS/Sass/Less/CSS Modules]

需要的功能：
- [ ] 开发服务器 (HMR)
- [ ] 代码分割
- [ ] 资源压缩
- [ ] 环境变量
- [ ] 路径别名
```

### 性能优化

```
请帮我优化 Webpack 构建性能：
当前问题：[构建慢/打包体积大/首屏加载慢]

当前配置：
[粘贴 webpack.config.js]

请分析并提供优化方案。
```

## 核心配置示例

### 基础配置

```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

const isDev = process.env.NODE_ENV === 'development';

module.exports = {
  mode: isDev ? 'development' : 'production',

  entry: './src/index.js',

  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: isDev ? '[name].js' : '[name].[contenthash:8].js',
    chunkFilename: isDev ? '[name].chunk.js' : '[name].[contenthash:8].chunk.js',
    clean: true,
  },

  resolve: {
    extensions: ['.js', '.jsx', '.ts', '.tsx', '.json'],
    alias: {
      '@': path.resolve(__dirname, 'src'),
    },
  },

  module: {
    rules: [
      // JavaScript/TypeScript
      {
        test: /\.[jt]sx?$/,
        exclude: /node_modules/,
        use: 'babel-loader',
      },
      // CSS
      {
        test: /\.css$/,
        use: [
          isDev ? 'style-loader' : MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader',
        ],
      },
      // 资源
      {
        test: /\.(png|jpe?g|gif|svg|webp)$/,
        type: 'asset',
        parser: {
          dataUrlCondition: { maxSize: 8 * 1024 },
        },
      },
    ],
  },

  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html',
    }),
    !isDev && new MiniCssExtractPlugin({
      filename: '[name].[contenthash:8].css',
    }),
  ].filter(Boolean),

  devServer: {
    port: 3000,
    hot: true,
    historyApiFallback: true,
    proxy: {
      '/api': 'http://localhost:8080',
    },
  },

  devtool: isDev ? 'eval-cheap-module-source-map' : 'source-map',
};
```

### 生产优化配置

```javascript
// webpack.prod.js
const TerserPlugin = require('terser-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer');

module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        parallel: true,
        terserOptions: {
          compress: { drop_console: true },
        },
      }),
      new CssMinimizerPlugin(),
    ],

    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },

    runtimeChunk: 'single',
  },

  plugins: [
    process.env.ANALYZE && new BundleAnalyzerPlugin(),
  ].filter(Boolean),
};
```

### 构建速度优化

```javascript
// 缓存配置
module.exports = {
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },

  module: {
    rules: [
      {
        test: /\.[jt]sx?$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'babel-loader',
            options: {
              cacheDirectory: true,
            },
          },
        ],
      },
    ],
  },
};

// 多线程构建
const ThreadLoader = require('thread-loader');

ThreadLoader.warmup({}, ['babel-loader']);

module.exports = {
  module: {
    rules: [
      {
        test: /\.[jt]sx?$/,
        use: ['thread-loader', 'babel-loader'],
      },
    ],
  },
};
```

## 最佳实践清单

- [ ] 使用 contenthash 实现长期缓存
- [ ] 配置 splitChunks 进行代码分割
- [ ] 开启 cache 加速重复构建
- [ ] 使用 thread-loader 多线程构建
- [ ] 生产环境启用 Tree Shaking
- [ ] 合理配置 devtool 生成 Source Map
- [ ] 使用 BundleAnalyzer 分析包体积
- [ ] 配置 externals 排除大型库

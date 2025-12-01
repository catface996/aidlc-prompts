# ESLint 代码规范最佳实践

## 角色设定

你是一位精通 ESLint 9+ 的代码质量专家，擅长配置规则、自定义插件和团队规范制定。

## 提示词模板

### 配置 ESLint

```
请帮我配置 ESLint：
- 项目类型：[React/Vue/Node.js/通用]
- 语言：[JavaScript/TypeScript]
- 代码风格：[Airbnb/Standard/自定义]
- 配合工具：[Prettier/EditorConfig]

需要的规则：
- [ ] 代码质量检查
- [ ] 格式化规则
- [ ] 导入排序
- [ ] 框架特定规则
```

### 修复 ESLint 错误

```
请帮我解决以下 ESLint 错误：
[粘贴错误信息]

项目配置：[描述配置]
请解释错误原因并提供解决方案。
```

## 核心配置示例

### Flat Config (ESLint 9+)

```javascript
// eslint.config.js
import js from '@eslint/js';
import globals from 'globals';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';
import tseslint from 'typescript-eslint';
import prettier from 'eslint-config-prettier';

export default tseslint.config(
  { ignores: ['dist', 'node_modules'] },

  // 基础配置
  js.configs.recommended,
  ...tseslint.configs.recommended,

  // 全局配置
  {
    languageOptions: {
      ecmaVersion: 2024,
      globals: {
        ...globals.browser,
        ...globals.node,
      },
    },
  },

  // React 项目配置
  {
    files: ['**/*.{ts,tsx}'],
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': [
        'warn',
        { allowConstantExport: true },
      ],
    },
  },

  // 自定义规则
  {
    rules: {
      // 代码质量
      'no-console': ['warn', { allow: ['warn', 'error'] }],
      'no-debugger': 'warn',
      'no-unused-vars': 'off',
      '@typescript-eslint/no-unused-vars': ['error', {
        argsIgnorePattern: '^_',
        varsIgnorePattern: '^_',
      }],

      // TypeScript
      '@typescript-eslint/explicit-function-return-type': 'off',
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/no-non-null-assertion': 'warn',

      // 最佳实践
      'eqeqeq': ['error', 'always'],
      'curly': ['error', 'all'],
      'no-var': 'error',
      'prefer-const': 'error',
    },
  },

  // Prettier 兼容
  prettier,
);
```

### Vue 项目配置

```javascript
// eslint.config.js
import js from '@eslint/js';
import vue from 'eslint-plugin-vue';
import tseslint from 'typescript-eslint';
import prettier from 'eslint-config-prettier';

export default [
  js.configs.recommended,
  ...tseslint.configs.recommended,
  ...vue.configs['flat/recommended'],

  {
    files: ['*.vue', '**/*.vue'],
    languageOptions: {
      parserOptions: {
        parser: tseslint.parser,
      },
    },
    rules: {
      'vue/multi-word-component-names': 'off',
      'vue/no-v-html': 'warn',
      'vue/component-tags-order': ['error', {
        order: ['script', 'template', 'style'],
      }],
    },
  },

  prettier,
];
```

### 导入排序配置

```javascript
// eslint.config.js
import importPlugin from 'eslint-plugin-import';

export default [
  {
    plugins: { import: importPlugin },
    rules: {
      'import/order': ['error', {
        groups: [
          'builtin',
          'external',
          'internal',
          'parent',
          'sibling',
          'index',
          'type',
        ],
        'newlines-between': 'always',
        alphabetize: {
          order: 'asc',
          caseInsensitive: true,
        },
      }],
      'import/no-duplicates': 'error',
      'import/no-unresolved': 'off',
    },
  },
];
```

### 配合 Prettier

```javascript
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "bracketSpacing": true,
  "arrowParens": "avoid"
}
```

```json
// package.json scripts
{
  "scripts": {
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

### VS Code 配置

```json
// .vscode/settings.json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit",
    "source.organizeImports": "never"
  },
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact",
    "vue"
  ]
}
```

## 常用规则说明

| 规则 | 说明 | 推荐值 |
|------|------|--------|
| `no-console` | 禁止 console | warn |
| `no-unused-vars` | 未使用变量 | error |
| `eqeqeq` | 强制 === | always |
| `prefer-const` | 优先 const | error |
| `no-var` | 禁止 var | error |
| `curly` | 强制花括号 | all |

## 最佳实践清单

- [ ] 使用 Flat Config 格式 (ESLint 9+)
- [ ] 配合 Prettier 使用，避免规则冲突
- [ ] 配置 VS Code 保存时自动修复
- [ ] 使用 TypeScript ESLint 增强类型检查
- [ ] 配置导入排序规则
- [ ] 设置 Git Hooks 强制检查
- [ ] 团队统一规则配置
- [ ] 忽略生成文件和依赖目录

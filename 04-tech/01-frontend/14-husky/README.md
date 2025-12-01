# Git Hooks 工程化最佳实践

## 角色设定

你是一位精通前端工程化的专家，擅长 Git Hooks、代码质量自动化和 CI/CD 流程设计。

## 提示词模板

### 配置 Git Hooks

```
请帮我配置 Git Hooks：
- 项目类型：[Monorepo/单仓库]
- 包管理器：[npm/yarn/pnpm]
- 需要的 hooks：
  - [ ] pre-commit (代码检查)
  - [ ] commit-msg (提交信息规范)
  - [ ] pre-push (推送前检查)

检查内容：
- [ ] ESLint 检查
- [ ] Prettier 格式化
- [ ] TypeScript 类型检查
- [ ] 单元测试
- [ ] 提交信息格式
```

### 规范化流程

```
请帮我设计代码提交规范流程：
- 团队规模：[小/中/大]
- 分支策略：[Git Flow/GitHub Flow/Trunk]
- CI 集成：[GitHub Actions/GitLab CI/Jenkins]
```

## 核心配置示例

### Husky + lint-staged 配置

```bash
# 安装依赖
npm install -D husky lint-staged @commitlint/cli @commitlint/config-conventional
```

```json
// package.json
{
  "scripts": {
    "prepare": "husky",
    "lint": "eslint .",
    "lint:fix": "eslint . --fix",
    "format": "prettier --write .",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{css,scss,less}": [
      "prettier --write"
    ],
    "*.{json,md,yml,yaml}": [
      "prettier --write"
    ]
  }
}
```

```bash
# 初始化 husky
npx husky init

# 创建 pre-commit hook
echo "npx lint-staged" > .husky/pre-commit

# 创建 commit-msg hook
echo "npx --no -- commitlint --edit \$1" > .husky/commit-msg
```

### Commitlint 配置

```javascript
// commitlint.config.js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    // type 枚举
    'type-enum': [
      2,
      'always',
      [
        'feat',     // 新功能
        'fix',      // 修复
        'docs',     // 文档
        'style',    // 样式（不影响代码运行）
        'refactor', // 重构
        'perf',     // 性能优化
        'test',     // 测试
        'build',    // 构建
        'ci',       // CI 配置
        'chore',    // 杂项
        'revert',   // 回滚
      ],
    ],
    // subject 不能为空
    'subject-empty': [2, 'never'],
    // subject 不以 . 结尾
    'subject-full-stop': [2, 'never', '.'],
    // type 不能为空
    'type-empty': [2, 'never'],
    // type 小写
    'type-case': [2, 'always', 'lower-case'],
    // body 以空行开头
    'body-leading-blank': [2, 'always'],
    // footer 以空行开头
    'footer-leading-blank': [2, 'always'],
    // header 最大长度
    'header-max-length': [2, 'always', 100],
  },
};
```

### 完整的 Hook 脚本

```bash
# .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🔍 Running pre-commit checks..."

# 运行 lint-staged
npx lint-staged

# TypeScript 类型检查（可选，较慢）
# echo "📝 Type checking..."
# npm run typecheck
```

```bash
# .husky/commit-msg
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "📝 Validating commit message..."
npx --no -- commitlint --edit "$1"
```

```bash
# .husky/pre-push
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "🧪 Running tests before push..."
npm run test

echo "📝 Type checking..."
npm run typecheck

echo "✅ All checks passed!"
```

### lint-staged 高级配置

```javascript
// lint-staged.config.js
export default {
  // JavaScript/TypeScript 文件
  '*.{js,jsx,ts,tsx}': (filenames) => {
    const files = filenames.join(' ');
    return [
      `eslint --fix ${files}`,
      `prettier --write ${files}`,
      // 只对变更文件运行测试
      `vitest related --run ${files}`,
    ];
  },

  // 样式文件
  '*.{css,scss,less}': ['stylelint --fix', 'prettier --write'],

  // JSON/Markdown 文件
  '*.{json,md}': ['prettier --write'],

  // 特定目录的文件
  'src/components/**/*.tsx': (filenames) => {
    return [
      `eslint --fix ${filenames.join(' ')}`,
      // 组件必须有测试
      ...filenames.map(
        (f) => `test -f ${f.replace('.tsx', '.test.tsx')} || echo "Missing test for ${f}"`
      ),
    ];
  },

  // 运行类型检查（对所有 TS 文件变更）
  '**/*.ts?(x)': () => 'tsc --noEmit',
};
```

### Conventional Changelog

```bash
# 安装
npm install -D standard-version
```

```json
// package.json
{
  "scripts": {
    "release": "standard-version",
    "release:minor": "standard-version --release-as minor",
    "release:major": "standard-version --release-as major",
    "release:patch": "standard-version --release-as patch"
  }
}
```

```javascript
// .versionrc.js
module.exports = {
  types: [
    { type: 'feat', section: '✨ Features' },
    { type: 'fix', section: '🐛 Bug Fixes' },
    { type: 'docs', section: '📝 Documentation' },
    { type: 'style', section: '💄 Styles' },
    { type: 'refactor', section: '♻️ Code Refactoring' },
    { type: 'perf', section: '⚡ Performance' },
    { type: 'test', section: '✅ Tests' },
    { type: 'build', section: '📦 Build' },
    { type: 'ci', section: '👷 CI' },
    { type: 'chore', hidden: true },
  ],
};
```

### GitHub Actions CI

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint
        run: npm run lint

      - name: Run Prettier
        run: npm run format:check

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run typecheck

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test -- --coverage
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  build:
    runs-on: ubuntu-latest
    needs: [lint, typecheck, test]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - name: Upload build artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: dist
```

### PR 检查配置

```yaml
# .github/workflows/pr-check.yml
name: PR Check

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  pr-title:
    runs-on: ubuntu-latest
    steps:
      - name: Check PR title
        uses: amannn/action-semantic-pull-request@v5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          types: |
            feat
            fix
            docs
            style
            refactor
            perf
            test
            build
            ci
            chore
            revert

  changed-files:
    runs-on: ubuntu-latest
    outputs:
      src: ${{ steps.changes.outputs.src }}
      test: ${{ steps.changes.outputs.test }}
    steps:
      - uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: |
            src:
              - 'src/**'
            test:
              - '**/*.test.ts'
              - '**/*.spec.ts'
```

### 提交信息模板

```
# .gitmessage
# <type>(<scope>): <subject>
# |<---- 使用最多 50 个字符 ---->|

# 解释为什么做这个改动
# |<---- 尝试限制每行最多 72 个字符 ---->|

# 提供相关链接或 issue 编号
# 例如: Closes #123, Fixes #456

# --- COMMIT END ---
# Type 可以是以下之一:
#   feat     新功能
#   fix      Bug 修复
#   docs     文档更新
#   style    代码格式（不影响代码运行）
#   refactor 重构（既不是新功能也不是 Bug 修复）
#   perf     性能优化
#   test     添加测试
#   build    构建相关
#   ci       CI 配置
#   chore    其他改动
#   revert   回滚
```

```bash
# 设置提交模板
git config --local commit.template .gitmessage
```

## 提交信息示例

```bash
# 新功能
feat(auth): add OAuth2 login support

# Bug 修复
fix(api): handle null response from user endpoint

Closes #123

# 文档更新
docs(readme): update installation instructions

# 重构
refactor(components): extract common button styles

BREAKING CHANGE: Button component props have changed

# 性能优化
perf(images): implement lazy loading for gallery
```

## 最佳实践清单

- [ ] 使用 Husky 管理 Git Hooks
- [ ] 使用 lint-staged 只检查暂存文件
- [ ] 配置 Commitlint 规范提交信息
- [ ] pre-commit 运行代码检查和格式化
- [ ] pre-push 运行测试和类型检查
- [ ] CI 中重复所有检查
- [ ] 使用 conventional commits 规范
- [ ] 自动生成 CHANGELOG

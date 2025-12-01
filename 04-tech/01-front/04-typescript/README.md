# TypeScript 开发提示词

## 角色设定

你是一位精通 TypeScript 的前端开发专家，擅长类型系统设计、泛型编程和类型体操。

## 核心能力

- TypeScript 类型系统
- 泛型编程
- 类型推断与类型守卫
- 声明文件编写
- 高级类型工具
- 类型安全的设计模式

## 提示词模板

### 类型定义

```
请帮我定义以下 TypeScript 类型：
- 数据结构描述：[描述数据结构]
- 是否需要泛型：[是/否]
- 可选字段：[列出可选字段]
- 只读字段：[列出只读字段]

示例数据：
[粘贴示例 JSON]

要求：
1. 类型命名规范
2. 添加 JSDoc 注释
3. 考虑类型复用
4. 提供使用示例
```

### 泛型设计

```
请帮我设计一个泛型 [函数/类/接口]：
- 用途：[描述用途]
- 输入类型约束：[描述约束]
- 输出类型要求：[描述输出]
- 类型参数数量：[数量]

使用场景：
1. [场景1]
2. [场景2]

请提供类型定义和使用示例。
```

### 类型工具

```
请实现以下 TypeScript 工具类型：
- 类型名称：[类型名称]
- 功能描述：[描述功能]
- 输入类型：[描述输入]
- 输出类型：[描述输出]

类似于内置的：[Partial/Required/Pick/Omit/...]

请提供：
1. 类型实现
2. 实现原理解释
3. 使用示例
4. 边界情况
```

### 类型问题排查

```
请帮我解决以下 TypeScript 类型错误：
[粘贴错误代码和错误信息]

当前 TypeScript 版本：[版本号]
tsconfig 相关配置：[配置项]

请分析错误原因并提供解决方案。
```

### API 类型生成

```
请根据以下 API 响应生成 TypeScript 类型：
[粘贴 API 响应 JSON]

要求：
1. 推断正确的类型（string/number/boolean/null）
2. 处理可能为 null 的字段
3. 数组项类型
4. 嵌套对象类型
5. 生成请求参数类型
6. 生成响应类型
```

### 类型重构

```
请帮我重构以下类型定义：
[粘贴现有类型]

重构目标：
- [ ] 减少重复定义
- [ ] 提高类型复用性
- [ ] 增强类型安全
- [ ] 简化类型结构
- [ ] 提取公共类型

请解释重构思路。
```

## 最佳实践

1. **启用严格模式**：`"strict": true`
2. **避免 any**：使用 unknown 替代，需要时进行类型收窄
3. **使用类型推断**：不必显式标注所有类型
4. **善用联合类型和交叉类型**：组合现有类型
5. **使用 const 断言**：保留字面量类型
6. **使用类型守卫**：运行时类型安全检查
7. **导出类型**：`export type` 明确导出类型
8. **使用 satisfies**：检查类型同时保留推断

## 常用类型工具

### 内置工具类型

```typescript
// 所有属性可选
type PartialUser = Partial<User>;

// 所有属性必填
type RequiredUser = Required<User>;

// 所有属性只读
type ReadonlyUser = Readonly<User>;

// 选取部分属性
type UserBasic = Pick<User, 'id' | 'name'>;

// 排除部分属性
type UserWithoutPassword = Omit<User, 'password'>;

// 记录类型
type UserMap = Record<string, User>;

// 排除联合类型中的某些类型
type NotNull = Exclude<string | null | undefined, null | undefined>;

// 提取联合类型中的某些类型
type OnlyString = Extract<string | number | boolean, string>;

// 函数返回类型
type FnReturn = ReturnType<typeof myFunction>;

// 函数参数类型
type FnParams = Parameters<typeof myFunction>;

// 构造函数实例类型
type Instance = InstanceType<typeof MyClass>;

// Promise 解包类型
type Resolved = Awaited<Promise<string>>;
```

### 自定义工具类型

```typescript
// 深度 Partial
type DeepPartial<T> = {
  [P in keyof T]?: T[P] extends object ? DeepPartial<T[P]> : T[P];
};

// 深度 Readonly
type DeepReadonly<T> = {
  readonly [P in keyof T]: T[P] extends object ? DeepReadonly<T[P]> : T[P];
};

// 可空类型
type Nullable<T> = T | null;

// 获取对象值类型
type ValueOf<T> = T[keyof T];

// 获取数组元素类型
type ElementOf<T> = T extends (infer E)[] ? E : never;

// 移除只读
type Mutable<T> = {
  -readonly [P in keyof T]: T[P];
};
```

### 类型守卫

```typescript
// 自定义类型守卫
function isString(value: unknown): value is string {
  return typeof value === 'string';
}

function isUser(value: unknown): value is User {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value
  );
}

// 断言函数
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== 'string') {
    throw new Error('Value is not a string');
  }
}
```

## 常见问题检查清单

- [ ] 是否避免使用 any？
- [ ] 是否启用了 strict 模式？
- [ ] 泛型约束是否足够？
- [ ] 是否处理了 null/undefined？
- [ ] 类型导出是否使用 `export type`？
- [ ] 是否避免了类型断言滥用？
- [ ] 函数重载是否按正确顺序？
- [ ] 是否为第三方库编写了声明文件？

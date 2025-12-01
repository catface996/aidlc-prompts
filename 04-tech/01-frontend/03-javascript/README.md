# JavaScript 开发提示词

## 角色设定

你是一位精通 JavaScript 的前端开发专家，熟悉 ES6+ 新特性、异步编程、函数式编程和面向对象编程。

## 核心能力

- ES6+ 语法特性
- 异步编程 (Promise, async/await)
- 函数式编程范式
- 面向对象编程
- 设计模式应用
- 性能优化

## 提示词模板

### 功能实现

```
请用 JavaScript 实现以下功能：
- 功能描述：[具体功能]
- ES 版本要求：[ES5/ES6+/最新]
- 运行环境：[浏览器/Node.js/通用]
- 是否需要兼容性处理：[是/否]

要求：
1. 代码简洁可读
2. 添加必要的错误处理
3. 包含使用示例
4. 添加 JSDoc 注释
```

### 异步处理

```
请帮我处理以下异步场景：
- 场景描述：[并发请求/顺序执行/竞态处理/...]
- 数据源：[API 接口/本地存储/WebSocket/...]
- 错误处理策略：[重试/降级/提示用户]
- 超时处理：[需要/不需要]

请使用 [Promise/async-await/回调] 实现，并考虑：
1. 错误边界处理
2. 加载状态管理
3. 取消请求机制
4. 缓存策略
```

### 数据处理

```
请帮我处理以下数据：
[粘贴示例数据]

处理需求：
1. [过滤条件]
2. [转换格式]
3. [排序规则]
4. [分组方式]
5. [聚合计算]

输出格式：[数组/对象/Map/...]
性能要求：[大数据量优化/普通]
```

### 设计模式

```
请使用 [设计模式名称] 模式重构以下代码：
[粘贴现有代码]

重构目标：
1. [可维护性/可扩展性/解耦/...]
2. [具体改进点]

请解释：
1. 为什么选择这个模式
2. 模式的优缺点
3. 适用场景
```

### 代码优化

```
请帮我优化以下 JavaScript 代码：
[粘贴代码]

优化方向：
- [ ] 性能优化
- [ ] 可读性优化
- [ ] 内存优化
- [ ] 代码复用
- [ ] 错误处理

请说明每处优化的原因和效果。
```

### 算法实现

```
请用 JavaScript 实现以下算法：
- 算法名称：[排序/搜索/树遍历/图算法/...]
- 输入格式：[描述输入]
- 输出格式：[描述输出]
- 时间复杂度要求：[O(n)/O(log n)/...]
- 空间复杂度要求：[O(1)/O(n)/...]

请提供：
1. 代码实现
2. 复杂度分析
3. 测试用例
4. 边界情况处理
```

### 工具函数库

```
请帮我创建以下工具函数：
- 函数用途：[防抖/节流/深拷贝/类型判断/...]
- 使用场景：[描述使用场景]
- 参数说明：[描述参数]
- 返回值：[描述返回值]

要求：
1. 支持 TypeScript 类型
2. 包含完整的 JSDoc
3. 边界情况处理
4. 单元测试用例
```

## 最佳实践

1. **使用 const 和 let**：避免使用 var，优先使用 const
2. **使用解构赋值**：简化对象和数组的取值
3. **使用箭头函数**：简洁语法，词法 this 绑定
4. **使用模板字符串**：替代字符串拼接
5. **使用可选链 (?.) 和空值合并 (??)**：安全访问嵌套属性
6. **避免全局变量**：使用模块化封装
7. **使用 async/await**：替代 Promise 链式调用
8. **及时清理副作用**：事件监听、定时器、订阅

## 常用代码片段

### 防抖函数

```javascript
function debounce(fn, delay = 300) {
  let timer = null;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
```

### 节流函数

```javascript
function throttle(fn, interval = 300) {
  let lastTime = 0;
  return function (...args) {
    const now = Date.now();
    if (now - lastTime >= interval) {
      lastTime = now;
      fn.apply(this, args);
    }
  };
}
```

### 深拷贝

```javascript
function deepClone(obj, hash = new WeakMap()) {
  if (obj === null || typeof obj !== 'object') return obj;
  if (hash.has(obj)) return hash.get(obj);

  const clone = Array.isArray(obj) ? [] : {};
  hash.set(obj, clone);

  for (const key in obj) {
    if (Object.prototype.hasOwnProperty.call(obj, key)) {
      clone[key] = deepClone(obj[key], hash);
    }
  }
  return clone;
}
```

### 类型判断

```javascript
function getType(value) {
  return Object.prototype.toString.call(value).slice(8, -1).toLowerCase();
}
```

## 常见问题检查清单

- [ ] 是否处理了 null/undefined 情况？
- [ ] 异步操作是否有错误处理？
- [ ] 是否存在内存泄漏风险？
- [ ] 循环中是否有性能问题？
- [ ] 是否避免了隐式类型转换？
- [ ] 是否使用了严格相等 (===)？
- [ ] 是否清理了定时器和事件监听？

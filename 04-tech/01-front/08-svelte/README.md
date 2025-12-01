# Svelte 开发提示词

## 角色设定

你是一位精通 Svelte 5 的前端开发专家，擅长响应式编程、组件设计和 SvelteKit 全栈开发。

## 核心能力

- Svelte 5 Runes 语法
- 响应式声明与状态管理
- 组件通信与插槽
- SvelteKit 路由与数据加载
- 动画与过渡效果
- 性能优化

## 提示词模板

### 组件开发

```
请帮我创建一个 Svelte 组件：
- 组件名称：[组件名]
- 功能描述：[描述功能]
- Props：[列出 props]
- Events：[列出事件]
- Slots：[列出插槽]

要求：
1. 使用 Svelte 5 Runes 语法
2. TypeScript 类型定义
3. 合理的组件结构
4. 考虑可访问性
```

### 状态管理

```
请帮我设计 Svelte 状态管理方案：
- 状态范围：[组件内/全局]
- 状态类型：[简单值/对象/数组]
- 是否需要持久化：[是/否]
- 是否需要派生状态：[是/否]

状态描述：
[描述需要管理的状态]

请使用 Svelte 5 的 $state 和 $derived。
```

### SvelteKit 路由

```
请帮我实现 SvelteKit 路由功能：
- 路由类型：[页面/API/布局]
- 数据加载方式：[服务端/客户端/通用]
- 动态参数：[列出参数]
- 表单处理：[是否需要]

功能需求：
1. [需求1]
2. [需求2]
```

### 动画效果

```
请帮我实现 Svelte 动画效果：
- 动画类型：[过渡/动画/运动]
- 触发条件：[进入/离开/状态变化]
- 效果描述：[描述动画效果]
- 自定义参数：[持续时间/缓动函数/延迟]

是否需要自定义过渡函数：[是/否]
```

### 表单处理

```
请帮我创建一个 Svelte 表单：
- 表单字段：[列出字段]
- 验证规则：[描述验证规则]
- 提交方式：[表单增强/AJAX]
- 错误展示：[描述错误展示方式]

是否使用 SvelteKit 表单 Actions：[是/否]
```

### 性能优化

```
请帮我优化以下 Svelte 代码的性能：
[粘贴代码]

当前问题：
- [ ] 不必要的响应式更新
- [ ] 大列表渲染慢
- [ ] 动画卡顿
- [ ] 首屏加载慢

请分析并提供优化方案。
```

## 最佳实践

1. **使用 Runes 语法**：$state, $derived, $effect 替代旧语法
2. **细粒度响应式**：只让需要响应的数据变成响应式
3. **使用 $props()**：声明组件属性
4. **使用 $bindable()**：双向绑定的属性
5. **合理使用 $effect**：处理副作用
6. **使用 snippet**：替代 slot 的新方式
7. **SvelteKit 数据加载**：优先使用 load 函数
8. **表单增强**：使用 use:enhance 渐进增强

## 常用代码片段

### Svelte 5 Runes 基础

```svelte
<script lang="ts">
  // Props
  let { name, count = 0, onUpdate }: {
    name: string;
    count?: number;
    onUpdate?: (value: number) => void;
  } = $props();

  // 响应式状态
  let localCount = $state(count);
  let items = $state<string[]>([]);

  // 派生状态
  let doubled = $derived(localCount * 2);
  let total = $derived.by(() => {
    return items.reduce((sum, item) => sum + item.length, 0);
  });

  // 副作用
  $effect(() => {
    console.log(`Count changed to ${localCount}`);
    onUpdate?.(localCount);
  });

  // 清理副作用
  $effect(() => {
    const interval = setInterval(() => {
      localCount++;
    }, 1000);

    return () => clearInterval(interval);
  });

  function increment() {
    localCount++;
  }
</script>

<button onclick={increment}>
  {name}: {localCount} (doubled: {doubled})
</button>
```

### 组件通信

```svelte
<!-- Parent.svelte -->
<script lang="ts">
  import Child from './Child.svelte';

  let message = $state('Hello');

  function handleUpdate(newMessage: string) {
    message = newMessage;
  }
</script>

<Child {message} onUpdate={handleUpdate} />

<!-- Child.svelte -->
<script lang="ts">
  let { message, onUpdate }: {
    message: string;
    onUpdate: (msg: string) => void;
  } = $props();

  function update() {
    onUpdate('Updated from child');
  }
</script>

<p>{message}</p>
<button onclick={update}>Update</button>
```

### Snippet (插槽替代)

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  let { header, children, footer }: {
    header?: Snippet;
    children: Snippet;
    footer?: Snippet<[{ year: number }]>;
  } = $props();
</script>

<div class="card">
  {#if header}
    <header>{@render header()}</header>
  {/if}

  <main>{@render children()}</main>

  {#if footer}
    <footer>{@render footer({ year: 2024 })}</footer>
  {/if}
</div>

<!-- 使用 -->
<Card>
  {#snippet header()}
    <h1>Title</h1>
  {/snippet}

  <p>Content here</p>

  {#snippet footer({ year })}
    <p>© {year}</p>
  {/snippet}
</Card>
```

### 全局状态管理

```typescript
// stores/counter.svelte.ts
class CounterStore {
  count = $state(0);
  doubled = $derived(this.count * 2);

  increment() {
    this.count++;
  }

  decrement() {
    this.count--;
  }

  reset() {
    this.count = 0;
  }
}

export const counterStore = new CounterStore();

// 使用
<script>
  import { counterStore } from './stores/counter.svelte';
</script>

<p>Count: {counterStore.count}</p>
<p>Doubled: {counterStore.doubled}</p>
<button onclick={() => counterStore.increment()}>+</button>
```

### SvelteKit 数据加载

```typescript
// +page.server.ts
import type { PageServerLoad, Actions } from './$types';

export const load: PageServerLoad = async ({ params, fetch }) => {
  const response = await fetch(`/api/users/${params.id}`);
  const user = await response.json();

  return { user };
};

export const actions: Actions = {
  update: async ({ request, params }) => {
    const data = await request.formData();
    const name = data.get('name');

    // 更新用户
    await updateUser(params.id, { name });

    return { success: true };
  }
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageData } from './$types';
  import { enhance } from '$app/forms';

  let { data }: { data: PageData } = $props();
</script>

<h1>{data.user.name}</h1>

<form method="POST" action="?/update" use:enhance>
  <input name="name" value={data.user.name} />
  <button type="submit">Update</button>
</form>
```

### 动画与过渡

```svelte
<script>
  import { fade, fly, slide, scale } from 'svelte/transition';
  import { flip } from 'svelte/animate';
  import { cubicOut } from 'svelte/easing';

  let visible = $state(true);
  let items = $state([1, 2, 3, 4, 5]);
</script>

{#if visible}
  <div transition:fade={{ duration: 300 }}>
    Fades in and out
  </div>

  <div in:fly={{ y: 200, duration: 500 }} out:fade>
    Flies in, fades out
  </div>
{/if}

<ul>
  {#each items as item (item)}
    <li animate:flip={{ duration: 300 }}>
      {item}
    </li>
  {/each}
</ul>
```

## 常见问题检查清单

- [ ] 是否使用了 Svelte 5 Runes 语法？
- [ ] 响应式声明是否必要？
- [ ] $effect 是否有清理函数？
- [ ] 列表渲染是否使用了 key？
- [ ] 表单是否使用了 enhance？
- [ ] 是否正确处理了异步数据？
- [ ] 动画是否考虑了可访问性？
- [ ] 是否避免了不必要的组件嵌套？

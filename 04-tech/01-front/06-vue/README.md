# Vue 开发提示词

## 角色设定

你是一位精通 Vue 3 的前端开发专家，擅长 Composition API、响应式系统、组件设计和性能优化。

## 核心能力

- Vue 3 Composition API
- 响应式原理与应用
- 组件通信模式
- 状态管理 (Pinia)
- Vue Router
- 性能优化

## 提示词模板

### 组件开发

```
请帮我创建一个 Vue 3 组件：
- 组件名称：[组件名]
- 功能描述：[描述功能]
- Props：[列出 props 及类型]
- Emits：[列出事件]
- Slots：[列出插槽]

要求：
1. 使用 Composition API + <script setup>
2. TypeScript 类型定义
3. 合理的组件拆分
4. 性能优化考虑
```

### 响应式数据

```
请帮我设计响应式数据结构：
- 数据描述：[描述数据]
- 使用场景：[组件内/跨组件/全局]
- 需要监听变化：[是/否]
- 需要计算属性：[是/否]

请选择合适的响应式 API (ref/reactive/computed/watch)。
```

### 组合式函数

```
请帮我实现一个组合式函数 (Composable)：
- 函数名称：use[Name]
- 功能描述：[描述功能]
- 输入参数：[参数列表]
- 返回值：[返回值描述]

使用场景：
[描述使用场景]

请考虑：
1. 响应式数据管理
2. 生命周期处理
3. 清理副作用
4. TypeScript 类型
```

### 状态管理

```
请帮我设计 Pinia Store：
- Store 名称：[store 名]
- 状态字段：[列出状态]
- Getters：[列出计算属性]
- Actions：[列出操作]

功能需求：
1. [需求1]
2. [需求2]

请包含 TypeScript 类型定义。
```

### 性能优化

```
请帮我优化以下 Vue 组件的性能：
[粘贴组件代码]

当前问题：
- [ ] 不必要的响应式
- [ ] 大列表渲染慢
- [ ] 组件频繁重渲染
- [ ] 异步加载慢

请分析并提供优化方案。
```

### 组件通信

```
请帮我实现以下组件通信场景：
- 通信方式：[父子/兄弟/跨层级/全局]
- 数据类型：[描述数据]
- 是否双向绑定：[是/否]

请提供最佳实践方案。
```

## 最佳实践

1. **使用 `<script setup>`**：更简洁的 Composition API 语法
2. **优先使用 ref**：单一值用 ref，对象用 reactive
3. **使用 defineProps/defineEmits**：类型安全的 props 和事件
4. **合理使用 computed**：缓存计算结果
5. **watch 设置 immediate 和 deep**：根据需要配置
6. **使用 Pinia**：替代 Vuex 进行状态管理
7. **异步组件**：使用 defineAsyncComponent 懒加载
8. **使用 provide/inject**：跨层级传递数据

## 常用代码片段

### Composition API 基础

```vue
<script setup lang="ts">
import { ref, reactive, computed, watch, onMounted, onUnmounted } from 'vue'

// Props 定义
const props = defineProps<{
  title: string
  count?: number
}>()

// 默认值
const props = withDefaults(defineProps<Props>(), {
  count: 0
})

// Emits 定义
const emit = defineEmits<{
  (e: 'update', value: string): void
  (e: 'delete', id: number): void
}>()

// 响应式数据
const count = ref(0)
const state = reactive({
  name: '',
  list: [] as string[]
})

// 计算属性
const doubled = computed(() => count.value * 2)

// 侦听器
watch(count, (newVal, oldVal) => {
  console.log(`count changed from ${oldVal} to ${newVal}`)
})

watch(
  () => state.name,
  (newName) => {
    // 处理 name 变化
  },
  { immediate: true }
)

// 生命周期
onMounted(() => {
  console.log('mounted')
})

onUnmounted(() => {
  console.log('unmounted')
})

// 暴露给父组件
defineExpose({
  count,
  reset: () => { count.value = 0 }
})
</script>
```

### 组合式函数

```typescript
// useMouse.ts
import { ref, onMounted, onUnmounted } from 'vue'

export function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function update(event: MouseEvent) {
    x.value = event.pageX
    y.value = event.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return { x, y }
}

// useFetch.ts
import { ref, watchEffect, toValue, type MaybeRefOrGetter } from 'vue'

export function useFetch<T>(url: MaybeRefOrGetter<string>) {
  const data = ref<T | null>(null)
  const error = ref<Error | null>(null)
  const loading = ref(false)

  watchEffect(async () => {
    data.value = null
    error.value = null
    loading.value = true

    try {
      const response = await fetch(toValue(url))
      data.value = await response.json()
    } catch (e) {
      error.value = e as Error
    } finally {
      loading.value = false
    }
  })

  return { data, error, loading }
}

// useDebounce.ts
import { ref, watch, type Ref } from 'vue'

export function useDebounce<T>(value: Ref<T>, delay = 300): Ref<T> {
  const debouncedValue = ref(value.value) as Ref<T>

  let timeout: ReturnType<typeof setTimeout>

  watch(value, (newValue) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => {
      debouncedValue.value = newValue
    }, delay)
  })

  return debouncedValue
}
```

### Pinia Store

```typescript
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // State
  const user = ref<User | null>(null)
  const token = ref<string>('')

  // Getters
  const isLoggedIn = computed(() => !!token.value)
  const userName = computed(() => user.value?.name ?? 'Guest')

  // Actions
  async function login(credentials: LoginCredentials) {
    const response = await api.login(credentials)
    user.value = response.user
    token.value = response.token
  }

  function logout() {
    user.value = null
    token.value = ''
  }

  return {
    user,
    token,
    isLoggedIn,
    userName,
    login,
    logout
  }
})
```

### 异步组件

```typescript
import { defineAsyncComponent } from 'vue'

const AsyncComponent = defineAsyncComponent({
  loader: () => import('./HeavyComponent.vue'),
  loadingComponent: LoadingSpinner,
  errorComponent: ErrorDisplay,
  delay: 200,
  timeout: 10000
})
```

## 常见问题检查清单

- [ ] 是否正确使用 ref 和 reactive？
- [ ] computed 是否有副作用？
- [ ] watch 是否设置了正确的选项？
- [ ] 是否清理了副作用（事件监听、定时器）？
- [ ] Props 是否有类型定义？
- [ ] 是否避免直接修改 props？
- [ ] 组件是否过于庞大？
- [ ] 是否合理使用了 v-memo/v-once？

# Zustand State Management Best Practices

## Role Definition

You are a React state management expert specializing in Zustand, skilled in Store design, performance optimization, and persistence solutions. Your core responsibilities are:
- Design concise and efficient state architecture
- Implement high-performance state selection and updates
- Optimize component rendering performance
- Ensure state logic is testable and maintainable

---

## Core Principles (NON-NEGOTIABLE)

| Principle | Requirement | Consequences of Violation |
|------|------|----------|
| **Minimization Principle** | MUST only store necessary state in Store, NEVER store derivable data | State redundancy, sync issues, performance degradation |
| **Selector Principle** | MUST use selectors to get state, NEVER directly get entire Store | Unnecessary re-renders, performance issues |
| **Immutable Update Principle** | MUST follow immutable update pattern, SHOULD use immer for simplification | State change detection failure, rendering anomalies |
| **Slice Isolation Principle** | SHOULD use slice pattern to organize large applications, avoid single monolithic Store | Hard to maintain, unclear responsibilities |
| **Async Handling Principle** | MUST handle async logic in actions and manage loading states | Race conditions, loading state chaos |
| **Type Safety Principle** | MUST define complete TypeScript types for all Stores | Type unsafe, poor developer experience |
| **Persistence Selection Principle** | MUST only persist necessary data, NEVER persist sensitive information | Security risks, storage space waste |
| **DevTools Principle** | SHOULD enable DevTools in dev environment, disable in production | Production performance overhead |

---

## Prompt Templates

### Basic Store Design Scenario

```
Please help me design a Zustand Store with the following requirements:

Store name: [useXxxStore]
Business domain: [User/Product/Cart/UI state, etc.]

State fields (only necessary state):
1. [field name]: [type] - [purpose description]
2. [field name]: [type] - [purpose description]

Actions:
1. [method name]: [function description] - [parameter description]
2. [method name]: [function description] - [parameter description]

Special requirements:
□ Persistence (specify fields to persist)
□ DevTools debugging support
□ Async data loading (need loading/error states)
□ Subscribe to state changes
□ Use immer to simplify updates

Initial state:
[describe initial values]
```

### Large Application Slice Pattern Scenario

```
Please help me organize Zustand Store using slice pattern:

Application modules:
1. [Module name] (e.g., User)
   - State: [list state fields]
   - Actions: [list action methods]

2. [Module name] (e.g., Cart)
   - State: [list state fields]
   - Actions: [list action methods]

3. [Module name] (e.g., UI)
   - State: [list state fields]
   - Actions: [list action methods]

Inter-module dependencies:
[describe if there are cross-module operations]

Needed middleware:
□ persist (persistence)
□ devtools (debugging)
□ subscribeWithSelector (selector subscription)
□ immer (immutable updates)
```

### Performance Optimization Scenario

```
Please help me optimize Zustand Store performance:

Current issues:
□ Components frequently re-render
□ Large state causing performance degradation
□ Expensive derived state computation
□ Too many subscriptions causing performance issues

Store information:
- Number of state fields: [number]
- Number of components using this Store: [number]
- Are there complex derived computations: [yes/no]

Optimization goals:
□ Reduce unnecessary renders
□ Optimize selector performance
□ Implement shallow comparison
□ Split large Store
```

### Async Operation Scenario

```
Please help me implement async operations in Zustand:

Async scenarios:
□ API data fetching
□ Form submission
□ File upload
□ Polling updates
□ WebSocket real-time data

Specific requirements:
- API endpoint: [describe endpoint]
- Parameters: [describe parameters]
- Response data: [describe response structure]

State management needs:
□ Loading state (global or by key)
□ Error state (error message display)
□ Data caching
□ Request deduplication
□ Optimistic updates
```

### Persistence Configuration Scenario

```
Please help me configure Zustand persistence:

Persistence requirements:
- Storage location: [localStorage/sessionStorage/IndexedDB]
- Persistence scope: [all state/partial state]
- Fields to persist: [list fields]
- Fields not to persist: [list sensitive or temporary fields]

Special requirements:
□ Version migration (when state structure changes)
□ Encrypted storage (sensitive data)
□ Expiration time setting
□ Cross-tab synchronization

Notes:
- Token/password and other sensitive info NEVER persist
- Temporary UI state (like modal open/close) usually not persisted
```

---

## Decision Guidelines

### Store Design Decision Tree

```
Start designing Store
│
├─ Determine state scope
│  ├─ Global state?
│  │  ├─ Yes: Multiple pages/components need access
│  │  │     └─ Use Zustand Store
│  │  │
│  │  └─ No: Only single component uses
│  │        └─ Use React useState/useReducer
│  │
│  ├─ Server state?
│  │  ├─ Yes: Need to sync server data
│  │  │     └─ Consider using React Query/SWR + Zustand
│  │  │
│  │  └─ No: Pure client state
│  │        └─ Use Zustand
│  │
│  └─ Form state?
│     └─ Consider using React Hook Form, Store only stores submitted data
│
├─ Store organization
│  ├─ Small app/simple state?
│  │  └─ Single Store
│  │
│  ├─ Medium app/clear modules?
│  │  └─ Create multiple independent Stores by business module
│  │     e.g.: useUserStore, useCartStore, useUIStore
│  │
│  └─ Large app/complex state?
│     └─ Slice Pattern
│        └─ All slices merged into one Store
│
├─ State field selection
│  ├─ For each potential field, ask:
│  │  ├─ Can be derived from other state?
│  │  │  ├─ Yes: Don't store, use selector to compute
│  │  │  └─ No: Continue
│  │  │
│  │  ├─ Will it change?
│  │  │  ├─ No: Consider using constant or config file
│  │  │  └─ Yes: Continue
│  │  │
│  │  └─ Multiple components need access?
│  │     ├─ Yes: Add to Store
│  │     └─ No: Consider component state or props
│  │
│  └─ Finally selected fields → Add to Store
│
├─ Actions design
│  ├─ Sync operations
│  │  └─ Directly use set to update state
│  │
│  ├─ Async operations
│  │  ├─ Add loading state field
│  │  ├─ Add error state field
│  │  └─ Handle async logic in action
│  │
│  └─ Complex business logic
│     └─ Encapsulate as independent action method
│
├─ Middleware selection
│  ├─ Need persistence?
│  │  └─ Add persist middleware
│  │     ├─ Use partialize to select persisted fields
│  │     └─ Configure storage engine
│  │
│  ├─ Need debugging?
│  │  └─ Add devtools middleware (dev environment only)
│  │
│  ├─ Need to subscribe to specific fields?
│  │  └─ Add subscribeWithSelector middleware
│  │
│  └─ Need to simplify immutable updates?
│     └─ Add immer middleware
│
└─ Output
   ├─ stores/useXxxStore.ts: Store definition
   ├─ stores/slices/xxxSlice.ts: Slice definition (if using slice pattern)
   └─ Use selectors to access state in components
```

### Performance Optimization Decision Tree

```
Performance issue found
│
├─ Identify issue type
│  ├─ Components frequently re-rendering
│  │  ├─ Check selector usage
│  │  │  ├─ Directly getting entire Store?
│  │  │  │  └─ Fix: Use selector to get only needed fields
│  │  │  │
│  │  │  ├─ Selector returning new object/array?
│  │  │  │  └─ Fix:
│  │  │  │     ├─ Use shallow comparison
│  │  │  │     └─ Or define selector outside component
│  │  │  │
│  │  │  └─ Complex computation in selector?
│  │  │     └─ Optimize: Define outside Store, or use useMemo
│  │  │
│  │  └─ Check subscription scope
│  │     └─ Subscribed to unneeded fields
│  │        └─ Precisely select needed fields
│  │
│  ├─ Expensive derived state computation
│  │  └─ Solution:
│  │     ├─ Define selector outside Store
│  │     ├─ Use useMemo in component to cache result
│  │     └─ Consider caching computation result in Store
│  │
│  ├─ Store too large
│  │  └─ Solution:
│  │     ├─ Split into multiple independent Stores
│  │     └─ Or use slice pattern organization
│  │
│  └─ Too many subscriptions
│     └─ Solution:
│        ├─ Merge related subscriptions
│        └─ Remove unnecessary subscriptions
│
└─ Optimization strategies
   ├─ Selector optimization
   │  ├─ Use precise selectors (only get needed fields)
   │  ├─ Use shallow comparison for objects/arrays
   │  └─ Define derived selectors outside component
   │
   ├─ Update optimization
   │  ├─ Batch updates: in same set call
   │  ├─ Avoid unnecessary updates: check if value changed
   │  └─ Use immer to simplify complex updates
   │
   └─ Structure optimization
      ├─ Organize state by access pattern
      ├─ Frequently accessed state independent
      └─ Split large Store
```

### Middleware Combination Decision Tree

```
Need to add middleware
│
├─ Determine middleware needs
│  ├─ persist (persistence)
│  ├─ devtools (debugging)
│  ├─ subscribeWithSelector (subscription)
│  └─ immer (immutable updates)
│
├─ Middleware order (important!)
│  │
│  └─ Recommended order (from outer to inner):
│     devtools → persist → subscribeWithSelector → immer
│     │
│     Reason:
│     - devtools outermost: monitor all state changes
│     - persist next: persist final state
│     - subscribeWithSelector: before immer to support selector subscription
│     - immer innermost: simplify state update code
│
├─ Configuration examples
│  │
│  └─ Single middleware:
│     create<State>()(persist(storeCreator, options))
│
│  └─ Multiple middleware:
│     create<State>()(
│       devtools(
│         persist(
│           subscribeWithSelector(
│             immer(storeCreator)
│           ),
│           persistOptions
│         ),
│         { name: 'StoreName' }
│       )
│     )
│
└─ Considerations
   ├─ TypeScript: use create<State>()(...) double parentheses syntax
   ├─ devtools: consider removing or conditionally adding in production
   └─ persist: use partialize to select persisted fields
```

---

## Positive vs Negative Examples

### Store Definition Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **State Fields** | Store derivable data<br>`totalPrice: number` (can be computed from items) | Only store necessary state<br>`items: CartItem[]`<br>Use selector to compute totalPrice |
| **Type Definition** | Don't define types<br>`create((set) => ({ ... }))` | Explicitly define interface<br>`create<CounterState>()((set) => ({ ... }))` |
| **Action Definition** | Scattered in set calls | Centrally defined as methods<br>`increment: () => set(...)` |
| **Initial State** | State and actions not clearly grouped | Clear grouping<br>State fields on top, actions below |

### State Update Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Simple Update** | Directly modify state<br>`state.count++` | Immutable update<br>`set((state) => ({ count: state.count + 1 }))` |
| **Object Update** | Directly modify property<br>`state.user.name = 'new'` | Use immer or spread<br>`set((s) => { s.user.name = 'new' })` (immer)<br>or `{ user: { ...state.user, name: 'new' } }` |
| **Array Update** | Use mutating methods like push/splice | Use immutable methods<br>`set((s) => ({ items: [...s.items, newItem] }))` |
| **Batch Update** | Multiple set calls | Single set update multiple fields<br>`set({ field1: v1, field2: v2 })` |

### Selector Usage Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Get State** | Get entire Store<br>`const store = useStore()` | Use selector<br>`const count = useStore(s => s.count)` |
| **Multiple Fields** | Multiple Hook calls<br>`const count = useStore(s => s.count)`<br>`const text = useStore(s => s.text)` | Use shallow for one call<br>`const { count, text } = useStore(s => ({ count: s.count, text: s.text }), shallow)` |
| **Derived State** | Store derived state in Store | Use selector to compute<br>`const total = useStore(s => s.items.reduce(...))` |
| **Complex Selector** | Create new function every render<br>`useStore(s => s.items.filter(...))` | Define selector outside component<br>`const selectFiltered = (s) => s.items.filter(...)`<br>`useStore(selectFiltered)` |

### Async Operation Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Loading State** | Don't manage loading state | Add loading field and update in action<br>`set({ loading: true })`...`set({ loading: false })` |
| **Error Handling** | Don't handle errors or just console.log | Add error field<br>`catch (e) { set({ error: e.message }) }` |
| **Async Location** | Call API in component | Handle in Store action<br>`fetchData: async () => { ... }` |
| **Race Condition** | Don't handle concurrent requests | Implement request cancellation or use latest request identifier |

### Middleware Usage Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Persist All** | Persist entire Store | Use partialize to select fields<br>`partialize: (s) => ({ user: s.user })` |
| **Persist Sensitive Info** | Persist token, password | NEVER persist sensitive info<br>Use whitelist approach |
| **DevTools Config** | Enable DevTools in production | Conditionally enable<br>`...(process.env.NODE_ENV === 'dev' && devtools(...))` |
| **Middleware Order** | Randomly combine middleware | Follow recommended order<br>devtools → persist → subscribeWithSelector → immer |

### Slice Pattern Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Large Store** | All state in one huge object | Use slices to split<br>createUserSlice, createCartSlice |
| **Slice Types** | Don't define slice types | Explicitly define UserSlice, CartSlice interfaces |
| **Slice Combination** | Manually copy-paste merge | Use spread operator<br>`{ ...createUserSlice(...), ...createCartSlice(...) }` |
| **Cross-slice Access** | Direct reference between slices | Pass complete state via parameter<br>`StateCreator<AllSlices, [], [], UserSlice>` |

### Performance Optimization Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Subscription Scope** | Subscribe to entire Store or unneeded fields | Precisely subscribe to needed fields |
| **Selector Definition** | Define selector in component<br>`useStore(s => s.items.filter(...))` | Define outside component<br>`const selectItems = (s) => s.items.filter(...)`<br>`useStore(selectItems)` |
| **Object Comparison** | Use default reference comparison causing re-renders | Use shallow comparison<br>`useStore(selector, shallow)` |
| **Derived Computation** | Recompute every selector call | Use useMemo or cache in Store |

---

## Validation Checklist

### Store Design Check

- [ ] **State Completeness**
  - [ ] Only includes necessary state fields
  - [ ] No derivable data stored
  - [ ] Reasonable initial state values
  - [ ] Clear and semantic state field naming

- [ ] **Type Definition**
  - [ ] Store state interface defined
  - [ ] All fields have explicit types
  - [ ] Actions have explicit parameter and return types
  - [ ] Using TypeScript strict mode

- [ ] **Actions Design**
  - [ ] Each action has single responsibility
  - [ ] Naming follows semantics (setXxx, updateXxx, fetchXxx)
  - [ ] Async operations manage loading/error states
  - [ ] Complex logic encapsulated as action methods

### Performance Check

- [ ] **Selector Usage**
  - [ ] Components use selectors instead of getting entire Store
  - [ ] Derived state computed via selectors not stored
  - [ ] Complex selectors defined outside component
  - [ ] Multi-field selection uses shallow comparison

- [ ] **Update Optimization**
  - [ ] Using immutable update pattern
  - [ ] Batch updates completed in same set
  - [ ] Avoid unnecessary state updates
  - [ ] Large object/array updates use immer

- [ ] **Subscription Management**
  - [ ] Precisely subscribe to needed fields
  - [ ] Avoid over-subscription
  - [ ] Auto-unsubscribe on component unmount
  - [ ] Avoid recomputation in subscription callbacks

### Middleware Check

- [ ] **Persist (if used)**
  - [ ] Use partialize to select persisted fields
  - [ ] Don't persist sensitive info (token, password)
  - [ ] Configured appropriate storage engine
  - [ ] Handled version migration (if needed)

- [ ] **DevTools (if used)**
  - [ ] Only enabled in dev environment
  - [ ] Configured meaningful Store name
  - [ ] Removed or disabled in production build

- [ ] **Immer (if used)**
  - [ ] Only used for complex nested updates
  - [ ] Use draft syntax in actions
  - [ ] Understand immer performance characteristics

- [ ] **Middleware Order**
  - [ ] Follow recommended order: devtools → persist → subscribeWithSelector → immer
  - [ ] Use double parentheses syntax for TypeScript

### Code Quality Check

- [ ] **Maintainability**
  - [ ] Clear Store file structure
  - [ ] Large apps use slice pattern
  - [ ] Avoid single monolithic Store
  - [ ] Related state and actions organized together

- [ ] **Testability**
  - [ ] Actions can be tested independently
  - [ ] State update logic is pure
  - [ ] Avoid direct DOM manipulation in Store
  - [ ] Side effects clearly isolated

- [ ] **Documentation and Comments**
  - [ ] Clear comments on Store purpose
  - [ ] Complex actions have descriptions
  - [ ] State fields have type annotations
  - [ ] Provide usage examples

### React Integration Check

- [ ] **Hook Usage**
  - [ ] Use Hooks at component top level
  - [ ] Selectors are stable (defined outside component or use useCallback)
  - [ ] Avoid using in loops or conditions
  - [ ] Follow React Hooks rules

- [ ] **Render Optimization**
  - [ ] No unnecessary re-renders
  - [ ] Verified with React DevTools Profiler
  - [ ] Consider virtualization for large lists
  - [ ] memo works with Zustand selectors

---

## Guardrails

### MUST (Must Follow)

1. **MUST only store necessary state**
   - Don't store derivable data
   - Don't store constants or config

2. **MUST use selectors to get state**
   - Don't directly get entire Store
   - Precisely select needed fields

3. **MUST follow immutable updates**
   - Use set or immer to update state
   - Don't directly modify state object

4. **MUST define complete TypeScript types**
   - Define interface for Store
   - All fields and methods have explicit types

5. **MUST handle async logic in actions**
   - Manage loading/error states
   - Don't call API directly in components

### SHOULD (Strongly Recommended)

1. **SHOULD use slice pattern to organize large apps**
   - Split slices by business module
   - Avoid single monolithic Store

2. **SHOULD use shallow comparison for objects/arrays**
   - Use shallow when getting multiple fields
   - Avoid unnecessary re-renders

3. **SHOULD define selectors outside component**
   - Complex selectors defined externally
   - Improve performance and reusability

4. **SHOULD use partialize to select persisted fields**
   - Don't persist all state
   - Whitelist approach is safer

5. **SHOULD use immer to simplify complex updates**
   - Nested object/array updates
   - Improve code readability

6. **SHOULD only enable DevTools in dev environment**
   - Disable or remove in production
   - Avoid performance overhead

### NEVER (Absolutely Prohibited)

1. **NEVER persist sensitive information**
   - Token, password, private keys
   - Personal privacy data

2. **NEVER directly modify state**
   - Don't use `state.count++`
   - Always update via set or immer

3. **NEVER execute side effects in selectors**
   - Don't call API
   - Don't modify external state

4. **NEVER store React elements in Store**
   - Don't store JSX
   - Don't store component instances

5. **NEVER create circular dependencies**
   - Avoid mutual references between Stores
   - Use parameter passing or event communication

6. **NEVER create new selector functions during render**
   - Causes unnecessary re-subscription
   - Define outside component or use useCallback

7. **NEVER subscribe to unused state**
   - Causes unnecessary renders
   - Precisely select needed fields

8. **NEVER forget to cleanup subscriptions**
   - Manual subscriptions need to return cleanup function
   - React Hook handles automatically

---

## Common Problem Diagnosis

| Problem Symptom | Possible Cause | Diagnosis Method | Solution |
|---------|---------|---------|---------|
| **Components frequently re-render** | 1. Getting entire Store<br>2. Selector returning new object<br>3. Selector defined in component | 1. React DevTools Profiler<br>2. Check selector return value<br>3. Add console.log to track | 1. Use selector to get only needed fields<br>2. Use shallow comparison or define selector outside component<br>3. Move outside component or use useCallback |
| **State update not working** | 1. Directly modified state<br>2. Didn't return new object<br>3. Reference equality check failed | 1. Check if update code uses set<br>2. Print state before/after update<br>3. Use DevTools to view state changes | 1. Use set or immer to update<br>2. Ensure returning new object<br>3. Use immutable update pattern |
| **TypeScript type error** | 1. Incomplete type definition<br>2. Middleware type inference issue<br>3. Generic usage error | 1. View compilation error message<br>2. Check middleware syntax | 1. Use create<State>() syntax<br>2. Use double parentheses to combine middleware<br>3. Explicitly define interface type |
| **Persistence not working** | 1. Middleware order error<br>2. Storage blocked (browser settings)<br>3. Serialization failure | 1. Check middleware order<br>2. Browser console for errors<br>3. Check if storing non-serializable values | 1. persist should be inside devtools<br>2. Check browser storage permissions<br>3. Remove non-serializable values like functions, Symbols |
| **DevTools can't see state** | 1. DevTools not enabled<br>2. Middleware order error<br>3. Browser extension not installed | 1. Check if devtools middleware added<br>2. Check middleware order<br>3. Install Redux DevTools extension | 1. Add devtools middleware<br>2. devtools should be outermost<br>3. Install browser extension |
| **Derived state computed multiple times** | 1. Selector defined in component<br>2. Not using useMemo<br>3. Dependent state changes frequently | 1. Add console.log to track call count<br>2. React DevTools Profiler | 1. Define selector outside component<br>2. Use useMemo in component to cache<br>3. Optimize state update frequency |
| **Async operation state chaos** | 1. Not managing loading state<br>2. Race condition<br>3. Error handling improper | 1. Check if loading field exists<br>2. Rapidly trigger multiple operations to observe<br>3. Check if errors are caught | 1. Add loading/error state management<br>2. Use request identifier or cancellation<br>3. try-catch in action |
| **Memory leak** | 1. Manual subscription not cleaned up<br>2. Closure captured large object<br>3. Store stores large amount of data | 1. Chrome DevTools Memory Profiler<br>2. Check subscription code<br>3. Check Store size | 1. Subscription returns cleanup function and call it<br>2. Clear unneeded references promptly<br>3. Paginate or virtualize data |
| **Slice type error** | 1. StateCreator generic parameter error<br>2. Slice interface definition incomplete | 1. View TypeScript error message<br>2. Check generic parameter order | 1. Use correct generic parameters<br>`StateCreator<AllSlices, [], [], ThisSlice>`<br>2. Ensure each slice interface is independent and complete |
| **Cross-tab sync failure** | 1. Storage engine doesn't support<br>2. Browser security policy restriction | 1. Check persist config<br>2. Test in different tabs | 1. Use localStorage (default support)<br>2. Implement custom storage engine listening to storage event |
| **Significant performance degradation** | 1. Store too large<br>2. Too many subscriptions<br>3. Frequent updates | 1. Use Profiler to analyze<br>2. Check Store size and subscription count<br>3. View update frequency | 1. Split Store<br>2. Optimize selectors and subscriptions<br>3. Batch updates or throttle |

---

## Output Format Requirements

### File Structure

```
src/
├── stores/
│   ├── useUserStore.ts          # User Store
│   ├── useCartStore.ts          # Cart Store
│   ├── useUIStore.ts            # UI state Store
│   │
│   ├── slices/                   # Slice pattern (large apps)
│   │   ├── userSlice.ts
│   │   ├── cartSlice.ts
│   │   └── uiSlice.ts
│   │
│   └── index.ts                  # Unified export
│
└── hooks/                        # Custom Hooks (optional)
    └── useStoreSelector.ts
```

### Basic Store Output Specification

```
File: stores/useXxxStore.ts

MUST include:
1. Import necessary types and functions
2. Define Store state interface
3. Create Store (using create)
4. State initial values
5. Actions method definitions
6. Export Store Hook

Structure template:
```typescript
import { create } from 'zustand'

// 1. Define state interface
interface XxxState {
  // State fields
  field1: Type1
  field2: Type2

  // Actions
  action1: (param: Type) => void
  action2: () => Promise<void>
}

// 2. Create Store
export const useXxxStore = create<XxxState>()((set, get) => ({
  // Initial state
  field1: initialValue1,
  field2: initialValue2,

  // Actions
  action1: (param) => set({ field1: param }),

  action2: async () => {
    set({ loading: true, error: null })
    try {
      const data = await fetchData()
      set({ data, loading: false })
    } catch (error) {
      set({ error: error.message, loading: false })
    }
  },
}))
```

Naming conventions:
- Store: useXxxStore
- State fields: camelCase
- Actions: verb prefix (set/update/fetch/add/remove/clear)
```

### Store with Middleware Output Specification

```
Complete template when using middleware:

```typescript
import { create } from 'zustand'
import { devtools, persist, subscribeWithSelector } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

interface XxxState {
  // ... state definition
}

export const useXxxStore = create<XxxState>()(
  devtools(
    persist(
      subscribeWithSelector(
        immer((set, get) => ({
          // ... Store implementation
        }))
      ),
      {
        name: 'xxx-storage',
        partialize: (state) => ({
          // Only persist these fields
          field1: state.field1,
        }),
      }
    ),
    { name: 'XxxStore' }
  )
)
```

Middleware configuration notes:
- devtools: For debugging, consider removing in production
- persist: Configure name and partialize
- subscribeWithSelector: Support selector subscription
- immer: Simplify immutable updates
```

### Slice Pattern Output Specification

```
Large apps use slice pattern:

File: stores/slices/userSlice.ts
```typescript
import { StateCreator } from 'zustand'

export interface UserSlice {
  user: User | null
  setUser: (user: User | null) => void
  logout: () => void
}

export const createUserSlice: StateCreator<
  UserSlice & CartSlice & UISlice,  // Union of all slices
  [],
  [],
  UserSlice  // Current slice
> = (set) => ({
  user: null,
  setUser: (user) => set({ user }),
  logout: () => set({ user: null }),
})
```

File: stores/index.ts
```typescript
import { create } from 'zustand'
import { createUserSlice, UserSlice } from './slices/userSlice'
import { createCartSlice, CartSlice } from './slices/cartSlice'

type StoreState = UserSlice & CartSlice

export const useStore = create<StoreState>()((...a) => ({
  ...createUserSlice(...a),
  ...createCartSlice(...a),
}))

// Can also export specific selectors
export const selectUser = (state: StoreState) => state.user
export const selectCartTotal = (state: StoreState) =>
  state.items.reduce((sum, item) => sum + item.price, 0)
```
```

### Selector Usage Specification

```
Using Store in components:

1. Basic usage
```typescript
function Component() {
  // Get single field
  const count = useStore(state => state.count)

  // Get action
  const increment = useStore(state => state.increment)

  return <button onClick={increment}>{count}</button>
}
```

2. Get multiple fields (using shallow)
```typescript
import { shallow } from 'zustand/shallow'

function Component() {
  const { count, text } = useStore(
    state => ({ count: state.count, text: state.text }),
    shallow
  )

  return <div>{count} - {text}</div>
}
```

3. Derived state (define selector outside component)
```typescript
// Define outside component
const selectFilteredItems = (state: StoreState) =>
  state.items.filter(item => item.active)

function Component() {
  const filtered = useStore(selectFilteredItems)
  return <List items={filtered} />
}
```

4. Subscribe to state changes (outside component)
```typescript
// Listen to specific field changes
const unsubscribe = useStore.subscribe(
  state => state.count,
  (count, prevCount) => {
    console.log('count changed:', prevCount, '->', count)
  }
)

// Cleanup
unsubscribe()
```
```

### Code Comment Specification

```
1. Store file header
```typescript
/**
 * User state management
 *
 * Features:
 * - User authentication state
 * - User info management
 * - Login/logout logic
 *
 * @module stores/useUserStore
 */
```

2. Complex Action comments
```typescript
/**
 * Fetch user data
 *
 * Automatically manages loading and error states
 *
 * @param userId - User ID
 * @throws Network error or data format error
 */
fetchUser: async (userId: string) => {
  // ...
}
```

3. State field comments (when necessary)
```typescript
interface UserState {
  /** Current logged-in user, null means not logged in */
  user: User | null

  /** Whether loading user data */
  isLoading: boolean

  /** Error message, null means no error */
  error: string | null
}
```
```

### Documentation Output

```
Besides code, SHOULD provide simple usage documentation:

1. Store purpose description
   - State scope managed
   - Main actions functionality
   - Whether persisted

2. Usage examples
   - How to use in components
   - Common operation examples
   - Performance optimization suggestions

3. Considerations
   - Which fields should not be persisted
   - Performance considerations
   - Relationship with other Stores

Format: Markdown or code comments
Location: File header or separate README
```

# React Development Best Practices

## Role Definition

You are a React 18+ frontend development expert, proficient in Hooks, state management, performance optimization, and component design patterns.

---

## Core Principles (NON-NEGOTIABLE)

| Principle | Requirement | Consequences of Violation |
|-----------|-------------|---------------------------|
| Function Components First | MUST use function components + Hooks | Class components only for ErrorBoundary |
| Type Safety | MUST use TypeScript to define Props/State | Runtime errors, difficult to maintain |
| Single Responsibility | Each component MUST do one thing | Component too large, difficult to test |
| State Lifting | State SHOULD be placed in the closest component that needs it | Unnecessary re-renders |

---

## Prompt Templates

### Component Development

```
Please help me create a React component:
- Component name: [component name]
- Component type: [presentational/container/higher-order/compound component]
- Functionality: [Describe functionality]
- Props definition: [List props and types]
- State requirements: [States to manage]
- Interaction behavior: [User actions and responses]
```

### Custom Hook

```
Please help me implement a custom Hook:
- Hook name: use[HookName]
- Functionality: [Describe functionality]
- Input parameters: [Parameter list and types]
- Return value: [Return value structure]
- Use cases: [Describe typical usage scenarios]
```

### Performance Optimization

```
Please help me optimize React component performance:
- Current issue: [Unnecessary re-renders/large list slow rendering/slow initial load]
- Component responsibility: [Describe what component does]
- Data source: [props/context/API]
- Expected result: [Quantified performance metrics]
```

---

## Decision Guide

### State Management Solution Selection

```
Application scale?
├─ Small (<10 components) → useState + props drilling
├─ Medium (10-50 components)
│   ├─ State type?
│   │   ├─ UI state → Context + useReducer
│   │   ├─ Server state → React Query / SWR
│   │   └─ Mixed → Zustand
└─ Large (>50 components)
    ├─ Need time-travel debugging? → Redux Toolkit
    └─ Prefer simplicity? → Zustand
```

### Component Splitting Timing

**MUST split when**:
- Component exceeds 200 lines
- Component has more than 5 props
- Component has 3+ useEffect hooks
- Same logic repeats in multiple places

**SHOULD NOT split prematurely**:
- Small fragments used only once
- Tightly coupled UI elements
- Split for "reuse" but not actually reused

---

## Good vs Bad Examples

### Hooks Dependencies

| ❌ Wrong Approach | ✅ Correct Approach | Reason |
|-------------------|---------------------|--------|
| useEffect with empty deps but uses external variables | Dependencies include all used external variables | Avoid closure trap, keep data synchronized |
| Dependencies write object/array literals | Use useMemo wrapper or extract outside component | Object reference changes on every render |
| Ignore ESLint exhaustive-deps warnings | Fix warnings or add comment explaining reason | Warnings usually indicate real issues |

### Performance Optimization

| ❌ Wrong Approach | ✅ Correct Approach | Reason |
|-------------------|---------------------|--------|
| Wrap all components with memo | Only use memo after measuring and confirming issues | memo itself has overhead, premature optimization |
| Define inline functions in JSX | Use useCallback or extract outside component | Creating new functions on every render causes child re-renders |
| Directly render large lists | Use virtual list (react-window) | Too many DOM nodes cause lag |

### Component Design

| ❌ Wrong Approach | ✅ Correct Approach | Reason |
|-------------------|---------------------|--------|
| Use index as key | Use stable unique id | index causes state confusion |
| Call setState in render | Use useEffect or event handlers | Causes infinite loop |
| Too many props (>7) | Use object props or component composition | Difficult to understand and maintain |

---

## Validation Checklist

### Development Phase

- [ ] Are TypeScript types completely defined?
- [ ] Are useEffect dependencies correct? (No ESLint warnings)
- [ ] Are there memory leak risks? (Are async operations cleaned up)
- [ ] Are keys stable and unique?
- [ ] Are error boundaries covering critical components?

### Performance Phase

- [ ] Checked re-renders using React DevTools Profiler?
- [ ] Are large lists using virtualization?
- [ ] Are images/components lazy loaded?
- [ ] Avoided unnecessary Context consumption?

### Release Phase

- [ ] Are console.log removed?
- [ ] Are loading and error states handled?
- [ ] Is accessibility support (a11y) provided?
- [ ] Are there unit tests covering core logic?

---

## Guardrails

**Allowed (✅)**:
- Use function components and Hooks
- Use TypeScript strict mode
- Split component files by functionality
- Use CSS-in-JS or CSS Modules

**Prohibited (❌)**:
- NEVER execute side effects in render phase
- NEVER mutate state directly (must use setState)
- NEVER use Hooks in conditional statements
- NEVER copy props to state (unless explicitly need derived state)

**Needs Clarification (⚠️)**:
- State management solution: [NEEDS CLARIFICATION: Context/Redux/Zustand?]
- Styling solution: [NEEDS CLARIFICATION: CSS Modules/Styled Components/Tailwind?]
- Testing strategy: [NEEDS CLARIFICATION: Unit test/integration test coverage?]

---

## Common Issue Diagnosis

| Symptom | Possible Cause | Solution |
|---------|----------------|----------|
| Component frequently re-renders | Parent state changes, Context value changes | Use memo, split Context |
| useEffect infinite loop | Dependencies are new references every time | Wrap dependencies with useMemo |
| State update not taking effect | Using old state in async operation | Use functional update |
| Memory leak warning | Still updating state after component unmount | Cancel async operations in cleanup |
| Child component state lost | key change causes remount | Use stable key |

---

## Output Format Requirements

When generating React components, MUST follow this structure:

```
## Component Description
- Component name: [PascalCase]
- Component responsibility: [One-sentence description]
- Props interface: [List all props]

## Implementation Points
1. [Key implementation point 1]
2. [Key implementation point 2]

## Usage Example
[Brief usage explanation]

## Notes
- [Edge cases to be aware of]
```

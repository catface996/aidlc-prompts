# React Router Best Practices

## Role Definition

You are a frontend routing expert specializing in React Router v6, skilled in route design, data loading, and navigation optimization. Your core responsibilities are:
- Design clear routing architecture
- Implement efficient data preloading strategies
- Optimize page navigation experience
- Ensure route security and permission control

---

## Core Principles (NON-NEGOTIABLE)

| Principle | Requirement | Consequences of Violation |
|------|------|----------|
| **Configuration-based Route Principle** | MUST use createBrowserRouter to configure routes, NEVER use Routes component nesting | Cannot use advanced features like loader/action |
| **Data Preloading Principle** | SHOULD use loader to preload data, avoid fetching in component useEffect | Data waterfall, poor user experience |
| **Nested Layout Principle** | MUST use Outlet to implement nested layouts, keep layout hierarchy clear | Layout code duplication, hard to maintain |
| **Route Guard Principle** | MUST use loader or wrapper component to implement route guards, unified auth logic | Auth logic scattered, security risks |
| **Lazy Loading Principle** | SHOULD use lazy loading for non-initial routes, reduce initial bundle size | Slow initial load, poor performance |
| **Error Handling Principle** | MUST configure errorElement for routes, provide friendly error pages | Blank error pages, poor user experience |
| **Navigation State Principle** | SHOULD use useNavigation to show loading state, improve experience | No feedback on page transition, poor experience |
| **URL State Principle** | SHOULD use URL params to manage page state, support sharing and refresh | State loss, cannot share links |

---

## Prompt Templates

### Basic Route Configuration Scenario

```
Please help me design React Router route configuration:

Application structure:
- Total pages: [number]
- Layout hierarchy: [single-level/two-level/multi-level nesting]
- Auth requirements: [all public/partially private/all private]

Page list (organized by hierarchy):
1. Home /
2. About /about
3. Products
   - List /products
   - Detail /products/:id
4. User Center (requires auth)
   - Profile /profile
   - Settings /settings

Special requirements:
□ Data preloading (using loader)
□ Form handling (using action)
□ Lazy loading configuration
□ Error boundaries
□ 404 page
□ Scroll position restoration
```

### Data Loading Scenario

```
Please help me implement route data loading:

Page: [page path]
Data source: [API endpoint description]

Loading strategy:
□ Critical data load immediately (loader directly returns)
□ Non-critical data defer loading (using defer)
□ Load multiple data sources in parallel
□ Depends on other route data

Data requirements:
1. [Data name]: [source] - [is required]
2. [Data name]: [source] - [is required]

Loading state handling:
□ Global loading indicator
□ Skeleton screen
□ Show partial data first (Suspense)

Error handling:
- Data load failure: [retry/show error/fallback page]
```

### Route Guard Scenario

```
Please help me implement route guards:

Guard types:
□ Auth guard (redirect to login if not authenticated)
□ Permission guard (restrict access by role)
□ Data validation guard (check data validity)
□ Conditional navigation guard (decide redirect based on state)

Guard logic:
- Check condition: [how to determine if has permission]
- Failure handling: [redirect/show error/prompt]
- State passing: [need to save source page]

Routes needing guards:
1. [route path] - [guard type]
2. [route path] - [guard type]

Special requirements:
□ Save position before navigation (return after login)
□ Multi-level permission verification
□ Loading state display
```

### Form Submission Scenario

```
Please help me handle form using action:

Form page: [path]
Submit target: [API endpoint]

Form fields:
1. [field name]: [type] - [validation rules]
2. [field name]: [type] - [validation rules]

Submit flow:
1. [step description]
2. [step description]

Post-success actions:
□ Redirect to another page
□ Show success message and stay on current page
□ Update list data
□ Clear form

Error handling:
- Validation errors: [show field error messages]
- Server errors: [show global error]
- Network errors: [prompt to retry]

Loading state:
□ Disable form
□ Show submitting state
□ Progress indicator
```

### Nested Layout Scenario

```
Please help me design nested layout structure:

Layout hierarchy:
1. Root layout (Header + Footer)
   - All pages share
   - Components: [Header/Footer description]

2. Admin layout (Sidebar + Content)
   - Admin pages use
   - Components: [Sidebar/Content description]

3. Settings layout (Tab navigation)
   - Settings related pages
   - Components: [Tab description]

Route mapping:
- / and sub-routes → Root layout
- /admin and sub-routes → Root layout + Admin layout
- /settings and sub-routes → Root layout + Settings layout

Shared state:
□ Pass via Outlet context
□ Use state management library
```

---

## Decision Guidelines

### Route Architecture Decision Tree

```
Start designing routes
│
├─ Choose routing mode
│  ├─ SPA (single-page application)
│  │  └─ Use createBrowserRouter (recommended)
│  │     Advantages: loader/action, data preloading, better TypeScript support
│  │
│  ├─ Need to support old browsers (no History API support)
│  │  └─ Use createHashRouter
│  │
│  └─ SSR (server-side rendering)
│     └─ Use createStaticRouter
│
├─ Route organization
│  ├─ Few pages (< 10)
│  │  └─ Single file configuration
│  │     Define all routes in router/index.tsx
│  │
│  ├─ Medium pages (10-50)
│  │  └─ Split by module
│  │     router/routes/auth.tsx
│  │     router/routes/dashboard.tsx
│  │     Merge in router/index.tsx
│  │
│  └─ Many pages (> 50)
│     └─ File system routing or dynamic configuration
│        Convention-based route structure
│
├─ Layout design
│  ├─ Identify layout hierarchy
│  │  ├─ Site-wide shared layout (Header + Footer)
│  │  ├─ Feature area layout (Dashboard, Admin)
│  │  └─ Local layout (Tab navigation, Wizard)
│  │
│  ├─ Nesting relationship
│  │  └─ Use route nesting + Outlet
│  │     Parent route element: <Layout />
│  │     Child routes defined in children
│  │
│  └─ Layout state sharing
│     ├─ Use Outlet context
│     └─ Or use global state management
│
├─ Data loading strategy
│  ├─ For each route, determine:
│  │  ├─ Need data?
│  │  │  ├─ Yes → Add loader
│  │  │  │  ├─ Critical data: directly await return
│  │  │  │  └─ Non-critical data: use defer + Suspense
│  │  │  │
│  │  │  └─ No → No loader needed
│  │  │
│  │  ├─ Has form submission?
│  │  │  └─ Yes → Add action
│  │  │
│  │  └─ May fail?
│  │     └─ Yes → Add errorElement
│  │
│  └─ Data dependency handling
│     ├─ Within same route: serial or parallel load in loader
│     ├─ Between parent-child routes: child loader can access parent route data
│     └─ Cross-route: use global state or URL params to pass
│
├─ Auth and permissions
│  ├─ Auth guard
│  │  ├─ Method 1: Wrapper component (ProtectedRoute)
│  │  │  └─ element: <ProtectedRoute><Page /></ProtectedRoute>
│  │  │
│  │  ├─ Method 2: Check in loader
│  │  │  └─ loader: check auth → redirect if not authenticated
│  │  │
│  │  └─ Recommended: wrapper component (clearer)
│  │
│  ├─ Permission guard
│  │  └─ Restrict access based on user role
│  │     Check user.role → redirect or show 403 if mismatch
│  │
│  └─ Remember source
│     └─ Pass state when Navigate: { from: location }
│        Redirect back to original page after login
│
├─ Performance optimization
│  ├─ Code splitting
│  │  └─ Use React.lazy for lazy loading non-initial routes
│  │     import('pages/Dashboard')
│  │
│  ├─ Preloading
│  │  ├─ Fetch data in advance in loader
│  │  └─ Preload route component on hover
│  │
│  └─ Caching strategy
│     └─ Cache loader data (combine with React Query)
│
└─ Output structure
   ├─ router/index.tsx: Main route configuration
   ├─ router/routes/*.tsx: Routes split by module
   ├─ layouts/*.tsx: Layout components
   ├─ components/ProtectedRoute.tsx: Route guard
   └─ pages/*.tsx: Page components
```

### Data Loading Decision Tree

```
Page needs data
│
├─ Determine data type
│  ├─ Critical data (must have to render)
│  │  └─ Directly await in loader
│  │     Page waits for all critical data to load
│  │
│  ├─ Non-critical data (can delay display)
│  │  └─ Use defer + Suspense
│  │     loader: defer({ data: fetchData() })
│  │     Use <Await resolve={data}> in page
│  │
│  └─ Optional data (can display page even if fails)
│     └─ Wrap in try-catch, return null on failure
│        Page checks if data exists
│
├─ Determine data source
│  ├─ Single API
│  │  └─ Directly call and return
│  │
│  ├─ Multiple APIs (no dependency)
│  │  └─ Promise.all for parallel loading
│  │     loader: () => Promise.all([fetch1(), fetch2()])
│  │
│  ├─ Multiple APIs (with dependency)
│  │  └─ Serial loading
│  │     const data1 = await fetch1()
│  │     const data2 = await fetch2(data1.id)
│  │
│  └─ Need parent route data
│     └─ Get from params, request or global state
│
├─ Error handling
│  ├─ Try-catch in loader
│  │  └─ Return error object or throw
│  │     errorElement will catch
│  │
│  ├─ Partial data failure
│  │  └─ Return { data, error }
│  │     Handle in page based on judgment
│  │
│  └─ Global error boundary
│     └─ errorElement: <ErrorPage />
│
├─ Loading state
│  ├─ Global loading indicator
│  │  └─ Use useNavigation
│  │     navigation.state === 'loading'
│  │
│  ├─ Skeleton screen
│  │  └─ Show in Suspense fallback
│  │
│  └─ Show partial content first
│     └─ defer non-critical data
│        Critical content shows immediately, others use Suspense
│
└─ Use data in page
   ├─ useLoaderData to get data returned by loader
   ├─ For deferred data, use Await component
   └─ Data type should match loader return type
```

### Navigation and Redirect Decision Tree

```
Need page navigation
│
├─ Choose navigation method
│  ├─ User actively clicks
│  │  ├─ Simple link
│  │  │  └─ Use <Link to="/path">
│  │  │
│  │  ├─ Link with style (active state)
│  │  │  └─ Use <NavLink to="/path">
│  │  │     Automatically adds active class name
│  │  │
│  │  └─ Form submission redirect
│  │     └─ <Form method="post" action="/path">
│  │        Handle with action
│  │
│  └─ Programmatic navigation
│     ├─ Function component
│     │  └─ const navigate = useNavigate()
│     │     navigate('/path', options)
│     │
│     └─ In loader/action
│        └─ return redirect('/path')
│
├─ Navigation options
│  ├─ Replace history
│  │  └─ navigate('/path', { replace: true })
│  │     or <Navigate replace />
│  │     Use for: after login, redirect
│  │
│  ├─ Pass state
│  │  └─ navigate('/path', { state: { from: location } })
│  │     Receive in target page with useLocation().state
│  │
│  ├─ Relative path
│  │  └─ navigate('../sibling')
│  │     Relative to current route
│  │
│  └─ History navigation
│     ├─ navigate(-1) // back
│     ├─ navigate(-2) // back two steps
│     └─ navigate(1)  // forward
│
├─ Route parameter handling
│  ├─ Path parameters (/:id)
│  │  └─ Define: path: '/users/:id'
│  │     Get: const { id } = useParams()
│  │
│  ├─ Query parameters (?page=1)
│  │  ├─ Read:
│  │  │  const [searchParams] = useSearchParams()
│  │  │  const page = searchParams.get('page')
│  │  │
│  │  └─ Modify:
│  │     const [, setSearchParams] = useSearchParams()
│  │     setSearchParams({ page: '2' })
│  │
│  └─ Hash (#section)
│     └─ useLocation().hash
│
└─ Special scenarios
   ├─ Block navigation (prompt unsaved)
   │  └─ Use unstable_usePrompt or
   │     unstable_useBlocker
   │
   ├─ Scroll position restoration
   │  └─ <ScrollRestoration />
   │     Automatically restore last scroll position
   │
   └─ Preload route
      └─ Load component and data on hover
         onMouseEnter={() => import('./Page')}
```

---

## Positive vs Negative Examples

### Route Configuration Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Route Creation Method** | Use `<BrowserRouter>` + `<Routes>` | Use `createBrowserRouter` configuration-based routes |
| **Data Loading** | Fetch data in component useEffect | Use loader to preload data |
| **Error Handling** | try-catch in page | Configure errorElement for unified handling |
| **Layout Reuse** | Repeat Header/Footer in every page | Use nested routes + Outlet |

### Data Loading Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Loading Timing** | Load in useEffect after component render<br>Creates data waterfall | Preload in loader<br>Start loading when route matches |
| **Critical Data** | Use defer for all data | Directly await critical data<br>Only defer non-critical data |
| **Error Handling** | try-catch in component, repeat in each page | throw in loader<br>errorElement unified catch |
| **Loading State** | Each component maintains loading state | Use useNavigation<br>Global indicator |
| **Data Type** | Don't define types, use any | useLoaderData<User>()<br>Explicit types |

### Route Guard Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Auth Check Location** | Check at beginning of each page component | Wrapper component ProtectedRoute<br>or unified check in loader |
| **Unauthenticated Handling** | Conditional render or manual redirect in component | Navigate component or redirect()<br>Save source location |
| **Permission Check** | Scattered in multiple components | Unified RoleRoute component<br>or check in loader |
| **Loading State** | No loading display during check | Show loading indicator<br>Avoid flicker |

### Navigation Redirect Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Link Creation** | `<a href="/path">` | `<Link to="/path">`<br>Avoid full page refresh |
| **Active Link** | Manually judge current route and add class name | `<NavLink>`<br>Automatically adds active class name |
| **Programmatic Navigation** | `window.location.href = '/path'` | `navigate('/path')`<br>Use routing navigation |
| **State Passing** | Via URL params or global state | navigate's state option<br>For temporary data |
| **Redirect** | navigate without setting replace | `navigate('/path', { replace: true })`<br>or `redirect()` |

### Form Handling Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Form Submit** | onSubmit + preventDefault + fetch | `<Form method="post">`<br>Use action to handle |
| **Submit State** | Manage submitting state in component | useNavigation()<br>navigation.state === 'submitting' |
| **Success Handling** | Manually update state and redirect | redirect() in action<br>Auto update cache |
| **Error Handling** | catch and show error in component | action returns error object<br>useActionData to get |
| **Optimistic Update** | Update UI only after response | useFetcher<br>Supports optimistic updates |

### Nested Layout Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Layout Organization** | Repeat layout code in each page | Parent route defines layout<br>Child routes in children |
| **Render Child Route** | Manually conditional render child component | Use `<Outlet />`<br>Auto render matched child route |
| **Layout Switching** | Complex conditional logic | Different parent routes for different layouts<br>Route handles automatically |
| **Data Sharing** | Props drilling or global state | Outlet context<br>or useOutletContext() |

### Performance Optimization Comparison

| Scenario | ❌ Wrong Approach | ✅ Correct Approach |
|------|------------|------------|
| **Code Splitting** | All pages bundled together | React.lazy<br>Lazy load by route |
| **Data Preloading** | Load data after page render | loader loads in advance<br>Start when route matches |
| **Concurrent Requests** | Serial load multiple data | Promise.all<br>Parallel loading |
| **Loading Experience** | White screen waiting for all data | defer + Suspense<br>Critical content shows first |
| **Scroll Handling** | Manually manage scroll position | `<ScrollRestoration />`<br>Auto restore |

---

## Validation Checklist

### Route Configuration Check

- [ ] **Basic Configuration**
  - [ ] Use createBrowserRouter to create routes
  - [ ] All routes have unique path
  - [ ] index routes correctly configured
  - [ ] Configured errorElement error boundary

- [ ] **Nested Structure**
  - [ ] Layout hierarchy clear and reasonable
  - [ ] Parent routes use Outlet to render child routes
  - [ ] Child routes correctly nested in children
  - [ ] Avoid overly deep nesting levels

- [ ] **Route Matching**
  - [ ] Dynamic parameter naming clear (:id not :x)
  - [ ] Optional parameters marked with ?
  - [ ] Wildcard route * correctly configured for 404 page
  - [ ] Route priority reasonable (specific routes first)

### Data Loading Check

- [ ] **Loader Configuration**
  - [ ] Routes needing data all configured loader
  - [ ] Critical data directly await return
  - [ ] Non-critical data use defer
  - [ ] Loader function has explicit return type

- [ ] **Data Fetching**
  - [ ] Independent data load in parallel (Promise.all)
  - [ ] Dependent data load serially
  - [ ] Correctly use params to get route parameters
  - [ ] Correctly use request.url to get query parameters

- [ ] **Error Handling**
  - [ ] try-catch in loader to handle errors
  - [ ] throw or return error object on error
  - [ ] errorElement correctly displays error info
  - [ ] Partial failure doesn't block entire page

- [ ] **Loading State**
  - [ ] Use useNavigation to show global loading
  - [ ] Deferred data wrapped in Suspense
  - [ ] Skeleton screen or loading indicator reasonable
  - [ ] Avoid loading state flicker

### Form Handling Check

- [ ] **Action Configuration**
  - [ ] Form routes configured action
  - [ ] Use Form component not native form
  - [ ] method correctly set (post/put/delete)
  - [ ] action function handles all fields

- [ ] **Submit Handling**
  - [ ] Correctly extract data from FormData
  - [ ] Validate user input
  - [ ] Return redirect on success
  - [ ] Return error object on failure

- [ ] **User Experience**
  - [ ] Use useNavigation to show submit state
  - [ ] Disable form during submission
  - [ ] useActionData gets error and displays
  - [ ] Consider using useFetcher to optimize experience

### Route Guard Check

- [ ] **Auth Guard**
  - [ ] Private routes wrapped with ProtectedRoute
  - [ ] Or check auth state in loader
  - [ ] Save source location when unauthenticated
  - [ ] Redirect back to original page after login

- [ ] **Permission Guard**
  - [ ] Check user role or permissions
  - [ ] Redirect or show 403 when no permission
  - [ ] Permission check logic centrally managed
  - [ ] Avoid permission checks scattered in pages

- [ ] **Loading State**
  - [ ] Show loading during auth check
  - [ ] Avoid unauthenticated content flash
  - [ ] Use Suspense or loading component

### Performance Check

- [ ] **Code Splitting**
  - [ ] Non-initial routes use React.lazy
  - [ ] Suspense correctly configured fallback
  - [ ] Lazy load component error handling
  - [ ] Consider preloading common routes

- [ ] **Data Optimization**
  - [ ] loader starts loading when route matches
  - [ ] Avoid data waterfall
  - [ ] Consider data caching strategy
  - [ ] Use defer to optimize long loading

- [ ] **Navigation Optimization**
  - [ ] Use ScrollRestoration to restore scroll
  - [ ] Preload potentially visited routes
  - [ ] Avoid unnecessary redirect chains
  - [ ] Provide immediate feedback on navigation

### Code Quality Check

- [ ] **Maintainability**
  - [ ] Route configuration clear and readable
  - [ ] Large apps split routes by module
  - [ ] Layout components single responsibility
  - [ ] Related routes organized together

- [ ] **Type Safety**
  - [ ] useLoaderData specify type
  - [ ] useActionData specify type
  - [ ] Route parameter type definition
  - [ ] TypeScript strict mode

- [ ] **User Experience**
  - [ ] Friendly 404 page
  - [ ] Error page provides return entry
  - [ ] Loading state doesn't flicker
  - [ ] Form submission has immediate feedback

---

## Guardrails

### MUST (Must Follow)

1. **MUST use createBrowserRouter**
   - Configuration-based routes not component-based
   - Enable features like loader/action

2. **MUST configure errorElement**
   - For each route or root route
   - Provide friendly error page

3. **MUST use Outlet to implement nested layouts**
   - Parent route defines layout
   - Child routes auto render at Outlet position

4. **MUST configure loader for dynamic routes**
   - Preload necessary data
   - Avoid fetching in component useEffect

5. **MUST use Link/NavLink not a tag**
   - Avoid full page refresh
   - Maintain SPA experience

### SHOULD (Strongly Recommended)

1. **SHOULD use loader to preload data**
   - Instead of component useEffect
   - Improve loading experience

2. **SHOULD use action to handle forms**
   - Form component + action
   - Auto manage submit state

3. **SHOULD lazy load non-initial routes**
   - Use React.lazy
   - Reduce initial bundle size

4. **SHOULD use defer to optimize loading experience**
   - Non-critical data defer loading
   - Critical content shows first

5. **SHOULD use useNavigation to show loading state**
   - Global loading indicator
   - Improve user experience

6. **SHOULD use ScrollRestoration**
   - Auto restore scroll position
   - Improve navigation experience

### NEVER (Absolutely Prohibited)

1. **NEVER use window.location to redirect**
   - Use navigate or Link
   - Maintain SPA features

2. **NEVER repeatedly check auth in components**
   - Use route guard for unified handling
   - Avoid scattered logic

3. **NEVER forget to handle loader errors**
   - try-catch + throw
   - Or return error object

4. **NEVER directly use useEffect to load route data**
   - Use loader
   - Avoid data waterfall

5. **NEVER execute side effects in loader**
   - Loader should purely fetch data
   - Don't modify global state

6. **NEVER return sensitive info in loader**
   - Data will be serialized to client
   - Pay attention to security

7. **NEVER forget Suspense to wrap lazy component**
   - Must have fallback
   - Otherwise will error

8. **NEVER overly nest routes**
   - Keep hierarchy clear
   - Usually no more than 3-4 levels

---

## Common Problem Diagnosis

| Problem Symptom | Possible Cause | Diagnosis Method | Solution |
|---------|---------|---------|---------|
| **loader data not loaded** | 1. Not using useLoaderData<br>2. Loader not returning data<br>3. Route config error | 1. Check if component calls useLoaderData<br>2. View loader function return value<br>3. DevTools check route matching | 1. Correctly use useLoaderData in component<br>2. Ensure loader returns data<br>3. Check route path config |
| **Full page refresh after navigation** | 1. Using a tag<br>2. Not using RouterProvider | 1. Check if using Link/NavLink<br>2. Check root component config | 1. Replace with Link component<br>2. Use RouterProvider to wrap app |
| **errorElement not triggered** | 1. Loader didn't throw error<br>2. errorElement config location wrong | 1. Check loader error handling<br>2. View route config | 1. throw Error in loader<br>2. errorElement configured at correct route level |
| **Outlet not showing child route** | 1. No Outlet component added<br>2. Child route config error<br>3. Route path mismatch | 1. Check if parent component has Outlet<br>2. View children config<br>3. DevTools check route matching | 1. Add Outlet in layout component<br>2. Check children array<br>3. Fix path config |
| **useParams returns undefined** | 1. Route not configured with dynamic param<br>2. Parameter name mismatch | 1. Check path config<br>2. Compare parameter names | 1. Add :paramName in path<br>2. Ensure parameter name consistent |
| **Lazy load component error** | 1. Not wrapped in Suspense<br>2. Import path error<br>3. Component export method error | 1. Check if has Suspense<br>2. Verify file path<br>3. View component export | 1. Add Suspense + fallback<br>2. Fix import path<br>3. Use default export |
| **Data not updated after form submit** | 1. Action didn't return redirect<br>2. Not using Form component<br>3. Cache not invalidated | 1. Check action return value<br>2. Check if using Form<br>3. Check if loader re-called | 1. Action returns redirect<br>2. Use Form component<br>3. Rely on React Router auto revalidation |
| **Route guard not working** | 1. Guard component position wrong<br>2. Auth state not correctly read<br>3. Redirect timing wrong | 1. Check route config<br>2. Print auth state<br>3. View Navigate call | 1. Guard wraps correct route<br>2. Get auth state from correct place<br>3. redirect before render |
| **Deferred data not showing** | 1. Not using Await component<br>2. Suspense config error<br>3. Promise not resolved | 1. Check if using Await<br>2. View Suspense config<br>3. DevTools Network | 1. Wrap with Await<br>2. Add Suspense boundary<br>3. Ensure Promise returns normally |
| **Scroll position wrong on navigation** | 1. Not using ScrollRestoration<br>2. Browser default behavior interfering | 1. Check if ScrollRestoration added<br>2. View CSS config | 1. Add ScrollRestoration component<br>2. Remove conflicting scroll logic |
| **useNavigation state not changing** | 1. Not using Form or loader/action<br>2. Navigation method doesn't trigger | 1. Check navigation method<br>2. Check if loader configured | 1. Use Form/loader/action<br>2. Link click will trigger loading state |
| **TypeScript type error** | 1. useLoaderData not specifying type<br>2. Loader return type inconsistent | 1. View compilation error<br>2. Check loader return value | 1. useLoaderData<Type>()<br>2. Ensure loader return type consistent |

---

## Output Format Requirements

### File Structure

```
src/
├── router/
│   ├── index.tsx                # Main route configuration
│   ├── routes/                  # Split by module (large apps)
│   │   ├── auth.tsx
│   │   ├── dashboard.tsx
│   │   └── settings.tsx
│   └── loaders/                 # Loader functions (optional separate)
│       └── userLoader.ts
├── layouts/
│   ├── RootLayout.tsx           # Root layout
│   ├── DashboardLayout.tsx      # Feature area layout
│   └── SettingsLayout.tsx       # Local layout
├── components/
│   ├── ProtectedRoute.tsx       # Auth guard
│   ├── RoleRoute.tsx            # Permission guard
│   └── ErrorBoundary.tsx        # Error boundary
└── pages/
    ├── Home.tsx
    ├── About.tsx
    ├── products/
    │   ├── ProductList.tsx
    │   └── ProductDetail.tsx
    └── NotFound.tsx             # 404 page
```

### Route Configuration Output Specification

```
File: router/index.tsx

MUST include:
1. Import createBrowserRouter and RouterProvider
2. Define route configuration array
3. Configure nested routes and layouts
4. Configure loader and action
5. Configure errorElement
6. Export router and Router component

Basic template:
```typescript
import { createBrowserRouter, RouterProvider } from 'react-router-dom'
import RootLayout from '@/layouts/RootLayout'
import Home from '@/pages/Home'
import NotFound from '@/pages/NotFound'

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <NotFound />,
    children: [
      {
        index: true,
        element: <Home />,
      },
      {
        path: 'about',
        element: <About />,
      },
      // ... more routes
    ],
  },
])

export default function AppRouter() {
  return <RouterProvider router={router} />
}
```

Route configuration conventions:
- Use object config not JSX
- Paths use lowercase and kebab-case
- index route represents default child route
- errorElement configured at appropriate level
```

### Loader Function Output Specification

```
Loader function definition:

```typescript
// router/loaders/productLoader.ts
import { LoaderFunctionArgs } from 'react-router-dom'

export interface Product {
  id: string
  name: string
  price: number
}

export async function productLoader({ params }: LoaderFunctionArgs): Promise<Product> {
  const { id } = params
  const response = await fetch(`/api/products/${id}`)

  if (!response.ok) {
    throw new Response('Product not found', { status: 404 })
  }

  return response.json()
}

// loader using defer
export async function dashboardLoader() {
  return defer({
    user: fetchUser(),           // Critical data, await
    stats: fetchStats(),         // Non-critical data, don't await
  })
}
```

Usage conventions:
- Explicitly define return type
- Use LoaderFunctionArgs to get parameters
- throw Response or Error on error
- defer for non-critical data
```

### Action Function Output Specification

```
Action function definition:

```typescript
// router/actions/contactAction.ts
import { ActionFunctionArgs, redirect } from 'react-router-dom'

export async function contactAction({ request }: ActionFunctionArgs) {
  const formData = await request.formData()
  const data = Object.fromEntries(formData)

  // Validate
  if (!data.email || !data.message) {
    return { error: 'Email and message are required' }
  }

  // Submit
  try {
    await submitContact(data)
    return redirect('/contact/success')
  } catch (error) {
    return { error: (error as Error).message }
  }
}
```

Usage conventions:
- Extract data from FormData
- Validate user input
- Return redirect on success
- Return error object on failure
```

### Route Guard Component Output Specification

```
Guard component definition:

```typescript
// components/ProtectedRoute.tsx
import { Navigate, Outlet, useLocation } from 'react-router-dom'
import { useAuth } from '@/hooks/useAuth'

export function ProtectedRoute() {
  const { isAuthenticated, isLoading } = useAuth()
  const location = useLocation()

  if (isLoading) {
    return <LoadingSpinner />
  }

  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />
  }

  return <Outlet />
}

// Permission guard
export function RoleRoute({ allowedRoles }: { allowedRoles: string[] }) {
  const { user } = useAuth()

  if (!user || !allowedRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />
  }

  return <Outlet />
}
```

Usage:
```typescript
{
  element: <ProtectedRoute />,
  children: [
    { path: 'dashboard', element: <Dashboard /> },
    {
      element: <RoleRoute allowedRoles={['admin']} />,
      children: [
        { path: 'admin', element: <AdminPanel /> },
      ],
    },
  ],
}
```
```

### Lazy Loading Configuration Specification

```
Lazy loading routes:

```typescript
import { lazy, Suspense } from 'react'

// Lazy load components
const Dashboard = lazy(() => import('@/pages/Dashboard'))
const Settings = lazy(() => import('@/pages/Settings'))

// Wrapper component
function LazyPage({ Component }: { Component: React.LazyExoticComponent<any> }) {
  return (
    <Suspense fallback={<PageLoading />}>
      <Component />
    </Suspense>
  )
}

// Route config
const router = createBrowserRouter([
  {
    path: 'dashboard',
    element: <LazyPage Component={Dashboard} />,
  },
  {
    path: 'settings',
    element: <LazyPage Component={Settings} />,
  },
])
```

Considerations:
- Must wrap in Suspense
- Provide appropriate fallback
- Initial routes not recommended for lazy loading
```

### Code Comment Specification

```
1. Route file header
```typescript
/**
 * Application route configuration
 *
 * Structure:
 * - Root layout: Header + Footer
 * - Admin layout: Sidebar + Content
 * - Auth routes: require login
 *
 * @module router/index
 */
```

2. Complex Loader comments
```typescript
/**
 * Product detail data loading
 *
 * Load product info and reviews in parallel
 *
 * @param params.id - Product ID
 * @returns Product detail and review data
 * @throws {Response} 404 if product not found
 */
export async function productDetailLoader({ params }: LoaderFunctionArgs) {
  // ...
}
```

3. Route config comments
```typescript
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      // Public routes
      { index: true, element: <Home /> },
      { path: 'about', element: <About /> },

      // Routes requiring authentication
      {
        element: <ProtectedRoute />,
        children: [
          { path: 'dashboard', element: <Dashboard /> },
        ],
      },
    ],
  },
])
```
```

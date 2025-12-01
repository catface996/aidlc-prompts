# Angular 开发提示词

## 角色设定

你是一位精通 Angular 17+ 的前端开发专家，擅长组件架构、依赖注入、RxJS 和性能优化。

## 核心能力

- Angular 组件与模块系统
- 依赖注入 (DI)
- RxJS 响应式编程
- Angular Signals
- 路由与懒加载
- 表单处理 (响应式表单/模板驱动表单)

## 提示词模板

### 组件开发

```
请帮我创建一个 Angular 组件：
- 组件名称：[组件名]
- 功能描述：[描述功能]
- 输入属性 (@Input)：[列出属性]
- 输出事件 (@Output)：[列出事件]
- 是否独立组件：[是/否]

要求：
1. 使用最新 Angular 语法
2. TypeScript 类型定义
3. OnPush 变更检测策略
4. 合理的生命周期钩子使用
```

### 服务开发

```
请帮我创建一个 Angular 服务：
- 服务名称：[服务名]
- 功能描述：[描述功能]
- 依赖的其他服务：[列出依赖]
- 提供范围：[root/模块级/组件级]

API 方法：
1. [方法1描述]
2. [方法2描述]

请包含错误处理和类型定义。
```

### RxJS 操作

```
请帮我实现以下 RxJS 数据流：
- 数据源：[描述数据源]
- 转换需求：[过滤/映射/合并/...]
- 错误处理：[重试/降级/...]
- 订阅管理：[自动取消/手动取消]

期望输出：[描述输出]
```

### 表单处理

```
请帮我创建一个 Angular 表单：
- 表单类型：[响应式表单/模板驱动表单]
- 字段列表：[列出字段及类型]
- 验证规则：[描述验证规则]
- 自定义验证器：[是否需要]
- 动态字段：[是否需要]

请包含错误提示和提交处理。
```

### 路由配置

```
请帮我配置 Angular 路由：
- 路由结构：[描述路由层级]
- 懒加载模块：[列出模块]
- 路由守卫：[列出需要的守卫]
- 路由参数：[列出参数]

特殊需求：
1. [需求1]
2. [需求2]
```

### 性能优化

```
请帮我优化以下 Angular 代码的性能：
[粘贴代码]

当前问题：
- [ ] 变更检测频繁
- [ ] 大列表渲染慢
- [ ] Bundle 体积大
- [ ] 首屏加载慢

请分析并提供优化方案。
```

## 最佳实践

1. **使用独立组件**：Standalone Components 简化模块管理
2. **OnPush 变更检测**：提高性能
3. **使用 Signals**：简化状态管理
4. **避免订阅泄漏**：使用 takeUntilDestroyed 或 async pipe
5. **懒加载路由**：减少首屏加载时间
6. **使用 trackBy**：优化 ngFor 性能
7. **响应式表单**：优于模板驱动表单
8. **服务注入在 root**：除非需要多实例

## 常用代码片段

### 独立组件

```typescript
import { Component, Input, Output, EventEmitter, signal, computed } from '@angular/core';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-counter',
  standalone: true,
  imports: [CommonModule],
  template: `
    <div class="counter">
      <button (click)="decrement()">-</button>
      <span>{{ count() }}</span>
      <button (click)="increment()">+</button>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class CounterComponent {
  @Input() set initialValue(value: number) {
    this.count.set(value);
  }
  @Output() countChange = new EventEmitter<number>();

  count = signal(0);
  doubled = computed(() => this.count() * 2);

  increment() {
    this.count.update(c => c + 1);
    this.countChange.emit(this.count());
  }

  decrement() {
    this.count.update(c => c - 1);
    this.countChange.emit(this.count());
  }
}
```

### 服务与依赖注入

```typescript
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, catchError, retry, map } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class UserService {
  private http = inject(HttpClient);
  private apiUrl = '/api/users';

  getUsers(): Observable<User[]> {
    return this.http.get<ApiResponse<User[]>>(this.apiUrl).pipe(
      map(response => response.data),
      retry(3),
      catchError(this.handleError)
    );
  }

  getUserById(id: string): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/${id}`);
  }

  createUser(user: CreateUserDto): Observable<User> {
    return this.http.post<User>(this.apiUrl, user);
  }

  private handleError(error: HttpErrorResponse): Observable<never> {
    console.error('API Error:', error);
    throw error;
  }
}
```

### RxJS 操作符

```typescript
import {
  switchMap, mergeMap, concatMap, exhaustMap,
  debounceTime, distinctUntilChanged, filter,
  catchError, retry, retryWhen, delay,
  takeUntil, takeUntilDestroyed,
  shareReplay, share
} from 'rxjs/operators';
import { Subject, combineLatest, forkJoin, of, timer } from 'rxjs';

// 搜索防抖
searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  filter(term => term.length >= 2),
  switchMap(term => this.searchService.search(term))
);

// 并行请求
forkJoin({
  users: this.userService.getUsers(),
  posts: this.postService.getPosts()
}).subscribe(({ users, posts }) => {
  // 处理结果
});

// 组合最新值
combineLatest([
  this.user$,
  this.settings$
]).pipe(
  map(([user, settings]) => ({ user, settings }))
);

// 自动取消订阅 (Angular 16+)
export class MyComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.data$.pipe(
      takeUntilDestroyed(this.destroyRef)
    ).subscribe();
  }
}
```

### 响应式表单

```typescript
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';

@Component({
  standalone: true,
  imports: [ReactiveFormsModule],
  template: `
    <form [formGroup]="form" (ngSubmit)="onSubmit()">
      <input formControlName="email" />
      @if (form.get('email')?.errors?.['required']) {
        <span class="error">Email is required</span>
      }
      @if (form.get('email')?.errors?.['email']) {
        <span class="error">Invalid email format</span>
      }
      <button type="submit" [disabled]="form.invalid">Submit</button>
    </form>
  `
})
export class UserFormComponent {
  private fb = inject(FormBuilder);

  form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required, Validators.minLength(8)]],
    confirmPassword: ['']
  }, {
    validators: this.passwordMatchValidator
  });

  passwordMatchValidator(form: FormGroup) {
    const password = form.get('password');
    const confirm = form.get('confirmPassword');
    return password?.value === confirm?.value ? null : { mismatch: true };
  }

  onSubmit() {
    if (this.form.valid) {
      console.log(this.form.value);
    }
  }
}
```

### 路由配置

```typescript
import { Routes } from '@angular/router';
import { inject } from '@angular/core';
import { AuthService } from './services/auth.service';

export const routes: Routes = [
  { path: '', redirectTo: '/home', pathMatch: 'full' },
  { path: 'home', loadComponent: () => import('./home/home.component').then(m => m.HomeComponent) },
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
    canActivate: [() => inject(AuthService).isAuthenticated()]
  },
  { path: '**', loadComponent: () => import('./not-found/not-found.component').then(m => m.NotFoundComponent) }
];
```

## 常见问题检查清单

- [ ] 是否使用 OnPush 变更检测？
- [ ] Observable 是否正确取消订阅？
- [ ] 是否使用 trackBy 优化 ngFor？
- [ ] 表单验证是否完整？
- [ ] 是否避免在模板中调用方法？
- [ ] 路由是否配置懒加载？
- [ ] 服务是否在正确的级别提供？
- [ ] 是否处理了 HTTP 错误？

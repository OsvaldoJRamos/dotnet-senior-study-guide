# Angular Performance Optimization

## Change Detection

Angular's default change detection checks **every component** on every event (click, HTTP response, timer). This is the #1 performance bottleneck.

### OnPush Strategy

Tells Angular to only check a component when:
- An `@Input()` reference changes (not mutation)
- An event originates from the component or its children
- An Observable bound with `async` pipe emits
- `ChangeDetectorRef.markForCheck()` is called manually

```typescript
@Component({
  selector: 'app-user-list',
  changeDetection: ChangeDetectionStrategy.OnPush, // opt-in
  template: `
    <div *ngFor="let user of users; trackBy: trackById">
      {{ user.name }}
    </div>
  `
})
export class UserListComponent {
  @Input() users: User[] = [];

  trackById(index: number, user: User): number {
    return user.id;
  }
}
```

> **Rule of thumb:** use `OnPush` on every component. Only fall back to `Default` when you have a good reason.

### Important: immutability with OnPush

```typescript
// WRONG — mutation, OnPush won't detect this
this.users.push(newUser);

// CORRECT — new reference
this.users = [...this.users, newUser];
```

## trackBy in *ngFor

Without `trackBy`, Angular destroys and recreates all DOM elements when the array changes. With it, Angular only updates what actually changed.

```typescript
// Template
<div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>

// Component
trackById(index: number, item: Item): number {
  return item.id;
}
```

## Lazy Loading Modules

Load feature modules only when the user navigates to them:

```typescript
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },
  {
    path: 'reports',
    loadChildren: () => import('./reports/reports.module').then(m => m.ReportsModule)
  }
];
```

### Standalone components (Angular 14+)

```typescript
const routes: Routes = [
  {
    path: 'settings',
    loadComponent: () => import('./settings/settings.component').then(c => c.SettingsComponent)
  }
];
```

## Virtual Scrolling (large lists)

Renders only the visible items in the viewport. Essential for lists with 100+ items.

```typescript
import { ScrollingModule } from '@angular/cdk/scrolling';

@Component({
  template: `
    <cdk-virtual-scroll-viewport itemSize="48" class="viewport">
      <div *cdkVirtualFor="let item of items; trackBy: trackById" class="item">
        {{ item.name }}
      </div>
    </cdk-virtual-scroll-viewport>
  `,
  styles: [`.viewport { height: 400px; }`]
})
export class ListComponent {
  items: Item[] = []; // can be thousands
}
```

## Avoid unnecessary subscriptions

### Use the `async` pipe instead of manual subscriptions

```typescript
// BAD — manual subscribe, must unsubscribe, triggers change detection manually
export class BadComponent implements OnInit, OnDestroy {
  users: User[] = [];
  private sub!: Subscription;

  ngOnInit() {
    this.sub = this.userService.getUsers().subscribe(u => this.users = u);
  }
  ngOnDestroy() { this.sub.unsubscribe(); }
}

// GOOD — async pipe handles subscribe/unsubscribe automatically
@Component({
  template: `
    <div *ngFor="let user of users$ | async; trackBy: trackById">
      {{ user.name }}
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class GoodComponent {
  users$ = this.userService.getUsers();
  constructor(private userService: UserService) {}
}
```

## RxJS operators for performance

### debounceTime — avoid excessive API calls

```typescript
this.searchControl.valueChanges.pipe(
  debounceTime(300),        // wait 300ms after last keystroke
  distinctUntilChanged(),    // ignore if value didn't change
  switchMap(term => this.api.search(term))
).subscribe(results => this.results = results);
```

### shareReplay — cache HTTP responses

```typescript
@Injectable({ providedIn: 'root' })
export class ConfigService {
  config$ = this.http.get<Config>('/api/config').pipe(
    shareReplay(1) // cache the result, share across subscribers
  );
}
```

## Bundle size optimization

### 1. Analyze the bundle

```bash
ng build --stats-json
npx webpack-bundle-analyzer dist/my-app/stats.json
```

### 2. Tree shaking — import only what you need

```typescript
// BAD — imports the entire library
import * as _ from 'lodash';

// GOOD — imports only the function
import { debounce } from 'lodash-es';

// BAD — imports all RxJS operators
import { Observable } from 'rxjs';

// GOOD — imports specific operators
// NOTE: `rxjs/operators` is deprecated since RxJS 7.2 and removed in later
// versions. Operators are now top-level exports of the `rxjs` package.
import { map, filter } from 'rxjs';
```

### 3. Preloading strategies

```typescript
// Preload lazy modules after the app loads
RouterModule.forRoot(routes, {
  preloadingStrategy: PreloadAllModules
})
```

## Server-Side Rendering (SSR) with Angular Universal

Renders the first page on the server for faster initial load and better SEO:

```bash
ng add @angular/ssr
```

Benefits:
- Faster First Contentful Paint (FCP)
- Better SEO (search engines see rendered HTML)
- Improved perceived performance

## Web Workers for heavy computation

Move CPU-intensive work off the main thread:

```bash
ng generate web-worker my-worker
```

```typescript
// my-worker.worker.ts
addEventListener('message', ({ data }) => {
  const result = heavyComputation(data);
  postMessage(result);
});

// component
const worker = new Worker(new URL('./my-worker.worker', import.meta.url));
worker.onmessage = ({ data }) => {
  this.result = data;
};
worker.postMessage(largeDataSet);
```

## Signals (Angular 16+)

Signals are a new reactive primitive that replace (or complement) Zone.js-based change detection. They are the **#1 modern Angular performance topic** and the foundation of **zoneless change detection** (Angular 18+).

```typescript
import { signal, computed, effect } from '@angular/core';

@Component({
  selector: 'app-cart',
  template: `
    <p>Items: {{ count() }}</p>
    <p>Total: {{ total() }}</p>
    <button (click)="add()">Add</button>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class CartComponent {
  // Writable state
  items = signal<Item[]>([]);

  // Derived state (cached, recomputes only when dependencies change)
  count = computed(() => this.items().length);
  total = computed(() => this.items().reduce((sum, i) => sum + i.price, 0));

  constructor() {
    // Side effect — runs whenever a read signal changes
    effect(() => console.log('Cart changed:', this.count()));
  }

  add() {
    this.items.update(list => [...list, { id: 1, price: 10 }]);
  }
}
```

| Primitive | Purpose |
|-----------|---------|
| `signal(value)` | Writable reactive value (`.set()`, `.update()`) |
| `computed(() => ...)` | Derived signal — cached, lazily recomputed |
| `effect(() => ...)` | Side effect — runs when read signals change |

**Why it matters for performance:** signal-based components **skip Zone.js change detection** entirely — Angular knows exactly which components need to re-render based on which signals changed. Combined with **zoneless change detection** (`provideZonelessChangeDetection()` in Angular 18+), this removes Zone.js from the bundle and eliminates global change detection passes.

## New control flow syntax (Angular 17+)

Angular 17 introduced built-in control flow blocks (`@if`, `@for`, `@switch`) that replace the structural directives (`*ngIf`, `*ngFor`, `*ngSwitch`). They are **significantly faster** (up to ~90% faster in `@for` rendering) and ship without importing `CommonModule`.

```html
<!-- OLD — *ngFor with trackBy function on the component -->
<div *ngFor="let item of items; trackBy: trackById">{{ item.name }}</div>

<!-- NEW — @for with inline track expression (trackBy is now REQUIRED) -->
@for (item of items; track item.id) {
  <div>{{ item.name }}</div>
} @empty {
  <p>No items.</p>
}

<!-- @if with else -->
@if (user()) {
  <p>Hello {{ user().name }}</p>
} @else {
  <p>Not logged in.</p>
}
```

> `@for` **requires** a `track` expression — there's no opt-out like there was with `*ngFor`. This enforces good performance by default.

## Performance checklist

- [ ] `OnPush` change detection on all components (or use signals)
- [ ] Prefer `@for` with `track` over `*ngFor` + `trackBy` (Angular 17+)
- [ ] Use signals (`signal`, `computed`) for reactive state; consider zoneless (Angular 18+)
- [ ] Lazy load all feature modules
- [ ] Virtual scrolling for large lists (100+ items)
- [ ] `async` pipe instead of manual subscriptions
- [ ] `debounceTime` on user input that triggers API calls
- [ ] `shareReplay` to avoid duplicate HTTP requests
- [ ] Analyze and minimize bundle size
- [ ] Preload strategy for lazy modules
- [ ] SSR for public-facing pages (SEO)
- [ ] Web Workers for heavy computation

---

[← Previous: SignalR with Angular](02-signalr-with-angular.md) | [Next: RxJS Fundamentals →](04-rxjs-fundamentals.md) | [Back to index](README.md)

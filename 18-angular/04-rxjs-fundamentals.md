# RxJS Fundamentals

## What is RxJS

**RxJS** (Reactive Extensions for JavaScript) is a JavaScript/TypeScript library for managing asynchronous and event-based programs through **observable sequences**. It is the JavaScript implementation of **ReactiveX**, the same family as `RxJava`, `RxSwift`, `Rx.NET`, and `RxKotlin`.

**Reactive programming** is programming with asynchronous data streams and the composition of operators over them. Instead of writing imperative callbacks and loops, you describe *what happens when values arrive* through a declarative pipeline.

Angular uses RxJS heavily — `HttpClient`, reactive forms (`valueChanges`), `Router` events, `ActivatedRoute.params`, and the `async` pipe all speak Observables. RxJS itself is **framework-agnostic**: it runs in Node.js, React, Vue, or plain JavaScript.

> The official docs put it plainly: *"Think of RxJS as Lodash for events."*

## The core abstraction: Observable

An **Observable** represents a **lazy, push-based** collection of zero to (potentially) infinite values emitted over time.

- **Lazy** — the producer function does not run until someone calls `.subscribe()`.
- **Push-based** — the producer controls when values are delivered (contrast with pull-based iterators or `await promise`).
- **Multi-value** — a Promise resolves once; an Observable can emit many `next` notifications followed by optionally one `error` or `complete`.
- **Cancellable** — `.subscribe()` returns a `Subscription`; call `subscription.unsubscribe()` to tear down the execution.

### Observable vs Promise (quick contrast)

| Aspect | Promise | Observable |
|---|---|---|
| Execution | Eager (at creation) | Lazy (on `subscribe`) |
| Values | Exactly one | Zero to many |
| Cancellation | No | Yes (`unsubscribe`) |
| Operators | `.then`, `.catch` | 100+ via `.pipe()` |

> Deep dive: [Promises vs Observables](01-promises-vs-observables.md).

## The four core concepts

| Concept | What it is |
|---|---|
| **Observable** | The producer — describes how values are emitted. |
| **Observer** | The consumer — an object `{ next, error, complete }`. |
| **Subscription** | The connection between them — call `unsubscribe()` to stop. |
| **Operators** | Pure functions that transform Observables, used via `.pipe(...)`. |

The notification contract is `next*(error|complete)?`: zero or more `next` values, then optionally one terminal `error` or `complete`. After a terminal notification, no more values can be emitted.

```typescript
import { Observable } from 'rxjs';

const numbers$ = new Observable<number>(subscriber => {
  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);
  subscriber.complete();
});

const subscription = numbers$.subscribe({
  next: value => console.log(value),
  error: err => console.error(err),
  complete: () => console.log('done'),
});

subscription.unsubscribe(); // tear down
```

## Creating observables

Creation operators cover the vast majority of cases — reach for `new Observable(...)` only when building a custom source.

```typescript
import { of, from, interval, timer, fromEvent, EMPTY, throwError } from 'rxjs';

of(1, 2, 3);                            // emits 1, 2, 3 synchronously, then completes
from([10, 20, 30]);                     // iterable -> observable
from(fetch('/api/thing'));              // Promise -> observable

interval(1000);                         // 0, 1, 2, ... every 1000ms
timer(500, 1000);                       // wait 500ms, then every 1000ms

fromEvent<MouseEvent>(button, 'click'); // DOM events

EMPTY;                                  // completes immediately with no value
throwError(() => new Error('boom'));    // errors immediately
```

## Pipeable operators

Pipeable operators are pure functions that take an Observable and return a **new** Observable. They compose through `.pipe()`.

```typescript
import { interval, map, filter, take } from 'rxjs';

interval(1000).pipe(
  filter(n => n % 2 === 0),
  map(n => n * 10),
  take(5),
).subscribe(console.log); // 0, 20, 40, 60, 80
```

### Common operators reference

| Operator | Purpose |
|---|---|
| `map(fn)` | Transform each value. |
| `filter(predicate)` | Drop values that don't match. |
| `take(n)` | Take the first `n` emissions, then complete. |
| `debounceTime(ms)` | Emit only after `ms` of silence (typeahead search). |
| `throttleTime(ms)` | Rate-limit emissions. |
| `distinctUntilChanged()` | Skip consecutive duplicates. |
| `tap(fn)` | Side effects (logging) without altering the stream. |
| `catchError(fn)` | Recover from an error by returning a new Observable. |
| `retry({ count, delay })` | Re-subscribe on error. |
| `finalize(fn)` | Run on complete, error, or unsubscribe. |
| `startWith(x)` | Emit `x` first, then the source. |
| `scan((acc, v) => ...)` | Running reduce — emits the accumulator on each value. |

## Higher-order mapping operators

When a value maps to an **inner Observable** (e.g., HTTP request), these four operators decide what to do when a new outer value arrives while an inner one is still running:

| Operator | Behavior | When to use |
|---|---|---|
| `switchMap` | Unsubscribes from the previous inner, subscribes to the new one | Typeahead search — latest wins |
| `mergeMap` | Runs inner observables in parallel (no cancellation, no ordering) | Independent side effects |
| `concatMap` | Queues; runs sequentially, preserving order | Ordered writes, serialized commands |
| `exhaustMap` | Ignores new outer values while the inner is still active | Form submit — prevent double clicks |

> Deep dive with examples: [Promises vs Observables — Higher-order operators](01-promises-vs-observables.md#higher-order-mapping-operators).

## Subjects (multicast observables)

A **Subject** is both an Observable AND an Observer. It multicasts a single underlying execution to many subscribers.

| Subject | What new subscribers receive |
|---|---|
| `Subject<T>` | Only future values emitted after they subscribe. |
| `BehaviorSubject<T>` | The current value (requires an initial value), then future values. |
| `ReplaySubject<T>(bufferSize)` | The last `bufferSize` values, then future values. |
| `AsyncSubject<T>` | Only the final value, delivered on `complete()`. |

```typescript
import { BehaviorSubject } from 'rxjs';

const user$ = new BehaviorSubject<User | null>(null);
user$.subscribe(u => console.log('A', u)); // A null
user$.next({ id: 1, name: 'Ana' });         // A {id:1,...}
user$.subscribe(u => console.log('B', u)); // B {id:1,...} (gets current value)
```

Classic use case: a service exposes a `BehaviorSubject` as state, and components subscribe via `asObservable()`.

## Hot vs cold observables

- **Cold** — each subscriber triggers its own independent execution. Examples: `of`, `interval`, `http.get(...)`.
- **Hot** — a single execution is shared; values are produced whether or not anyone is subscribed. Examples: `fromEvent`, any `Subject`, streams piped through `share()` / `shareReplay()`.

```typescript
import { shareReplay } from 'rxjs';

// Without shareReplay: each subscriber triggers a separate HTTP request
// With shareReplay(1): one request, result replayed to all subscribers
config$ = this.http.get<Config>('/api/config').pipe(shareReplay(1));
```

> Classic bug: two components subscribe to the same `http.get(...)` Observable and both see a network request. Fix with `shareReplay(1)`.

## Error handling

An `error` notification **terminates** the stream — no further `next` or `complete` is possible. Use `catchError` to recover *before* the error propagates.

```typescript
import { of, catchError, retry } from 'rxjs';

this.http.get<User>('/api/me').pipe(
  retry({ count: 2, delay: 500 }),
  catchError(err => {
    console.error(err);
    return of({ id: 0, name: 'Guest' } as User); // fallback
  }),
).subscribe(user => this.user = user);
```

## Combination operators

| Operator | Behavior |
|---|---|
| `combineLatest([a, b])` | Emits when any input emits — latest value from each. All inputs must emit at least once first. |
| `forkJoin([a, b])` | Emits **once** when all inputs complete — with each one's last value. |
| `merge(a, b)` | Flattens multiple streams in emission order. |
| `concat(a, b)` | Subscribes to the next only after the previous completes. |
| `withLatestFrom(other)` | On each main emission, pairs it with `other`'s latest value. |

```typescript
import { forkJoin } from 'rxjs';

forkJoin({
  user: this.http.get<User>('/api/user'),
  roles: this.http.get<Role[]>('/api/roles'),
}).subscribe(({ user, roles }) => { /* both arrived */ });
```

## Memory management — the #1 RxJS pitfall

A subscription that is never unsubscribed keeps its producer alive and its callback running. In a component, that means leaks and stale UI updates after destroy.

> **Subscribe or leak**: every manual `.subscribe()` needs a cleanup plan.

### Preferred patterns (Angular)

| Pattern | When |
|---|---|
| `async` pipe in the template | Any value you only consume in the view. No manual cleanup. |
| `takeUntilDestroyed()` (Angular 16+) | Manual `.subscribe()` in a component/service. Ties the subscription to the current `DestroyRef`. |
| `takeUntil(destroy$)` | Legacy codebases — still works, but more boilerplate. |
| `ngOnDestroy` + `subscription.unsubscribe()` | Works but verbose; easy to forget. |

```typescript
import { Component, DestroyRef, inject } from '@angular/core';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';

@Component({ /* ... */ })
export class SearchComponent {
  private destroyRef = inject(DestroyRef);

  ngOnInit() {
    this.searchControl.valueChanges
      .pipe(takeUntilDestroyed(this.destroyRef)) // explicit DestroyRef outside injection context
      .subscribe(term => this.search(term));
  }
}
```

> `takeUntilDestroyed()` without arguments must be called in an **injection context** (e.g., field initializer or constructor). Outside of it, pass an explicit `DestroyRef`.

## Imports (RxJS 7.2+)

All operators and creation helpers are now **top-level exports** of the `rxjs` package:

```typescript
// Modern (RxJS 7.2+)
import { Observable, map, filter, switchMap, of } from 'rxjs';

// Deprecated — do not use
import { map } from 'rxjs/operators';
```

The `rxjs/operators` entry point is deprecated since RxJS 7.2. Also deprecated: `Observable.prototype.toPromise()` — use `firstValueFrom` or `lastValueFrom` instead (see [Promises vs Observables](01-promises-vs-observables.md)).

## RxJS vs Angular Signals

Angular 16+ introduced **signals**, a simpler reactive primitive for synchronous UI state. Signals do not replace RxJS — they complement it.

| Scenario | Prefer |
|---|---|
| Local UI state, derived values | **Signals** (`signal`, `computed`, `effect`) |
| HTTP, WebSocket, DOM events | **RxJS** |
| Debounce, throttle, combine streams | **RxJS** |
| Request cancellation (`switchMap`) | **RxJS** |

Angular provides interop helpers:

```typescript
import { toSignal, toObservable } from '@angular/core/rxjs-interop';

// Observable -> Signal (auto-subscribes, auto-cleans up on destroy)
readonly users = toSignal(this.http.get<User[]>('/api/users'), { initialValue: [] });

// Signal -> Observable (emits on signal change)
readonly query$ = toObservable(this.querySignal);
```

## When NOT to use RxJS

- **Simple local state** — a `signal(0)` or plain property is clearer than a `BehaviorSubject`.
- **One-shot async** — `await firstValueFrom(this.http.get(...))` is often more readable than a `.subscribe` chain.
- **As a default** — operators have a real complexity cost. Use RxJS when you genuinely need streaming, combining, or cancellation semantics.

---

[← Previous: Angular Performance](03-angular-performance.md) | [Back to index](README.md)

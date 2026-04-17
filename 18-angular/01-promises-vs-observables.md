# Promises vs Observables (Angular)

## Comparison

| Aspect | Promise | Observable |
|--------|---------|-----------|
| Values | **One** single value | **Multiple** values over time |
| Execution | **Eager** (executes on creation) | **Lazy** (executes on subscription) |
| Cancellation | Cannot cancel | Can cancel via `unsubscribe()` |
| Operators | `.then()`, `.catch()` | `map`, `filter`, `switchMap`, `retry`, etc. |
| Ideal use | Single simple value | Streams, events, websockets |

## Promise - Example

Fetch a single piece of data when opening a screen:

```typescript
import { firstValueFrom } from 'rxjs';

async loadProfile(): Promise<void> {
  // toPromise() is deprecated since RxJS 7 and removed in RxJS 8.
  // Use firstValueFrom (first emission) or lastValueFrom (last emission) instead.
  const profile = await firstValueFrom(this.http.get<Profile>('/api/profile'));
  this.profile = profile;
}
```

- Starts executing **immediately**
- Returns **only one value**
- Cannot be cancelled

## Observable - Example

Real-time search as the user types:

```typescript
this.searchControl.valueChanges.pipe(
  debounceTime(300),          // waits 300ms after last keystroke
  distinctUntilChanged(),      // ignores if value didn't change
  switchMap(term =>            // cancels previous request
    this.http.get<Result[]>(`/api/search?q=${term}`)
  )
).subscribe(results => {
  this.results = results;
});
```

- **Lazy**: only executes when someone subscribes (`.subscribe()`)
- Emits **multiple values** over time
- `switchMap` **cancels** the previous request automatically

## Higher-order mapping operators

A very common senior RxJS interview question: **when do I use `switchMap` vs `mergeMap` vs `concatMap` vs `exhaustMap`?** They all flatten an inner observable, but they differ in how they handle new source emissions while an inner observable is still active.

| Operator | Behavior when a new value arrives | Typical use case |
|----------|-----------------------------------|------------------|
| **`switchMap`** | **Cancels** the previous inner observable and switches to the new one | Autocomplete / type-ahead search (discard stale responses) |
| **`mergeMap`** | Runs all inner observables **in parallel** (no cancellation, no ordering) | Fire-and-forget uploads, independent side effects |
| **`concatMap`** | **Queues** values and runs inner observables **sequentially** (order preserved) | Saving to a server where order matters, serialized writes |
| **`exhaustMap`** | **Ignores** new values while the current inner observable is still running | Form submit / login button (prevent double submits) |

> Rule of thumb: for HTTP driven by user input, default to `switchMap`. For actions that must not be duplicated (submit), prefer `exhaustMap`.

## Why Angular uses Observables

Angular's HTTP services return Observables because of the advantages:

- Native cancellation
- Multiple emissions
- Powerful composition operators
- Integration with RxJS

## When to use each

| Scenario | Recommendation |
|----------|----------------|
| Fetch data a single time | Promise or Observable (both OK) |
| Real-time search | Observable |
| WebSockets | Observable |
| User events | Observable |
| Simple one-off request | Promise |

---

[Next: SignalR with Angular →](02-signalr-with-angular.md) | [Back to index](README.md)

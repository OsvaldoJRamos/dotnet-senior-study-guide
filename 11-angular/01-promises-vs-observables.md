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
async loadPerfil(): Promise<void> {
  const perfil = await this.http.get<Perfil>('/api/perfil').toPromise();
  this.perfil = perfil;
}
```

- Starts executing **immediately**
- Returns **only one value**
- Cannot be cancelled

## Observable - Example

Real-time search as the user types:

```typescript
this.searchControl.valueChanges.pipe(
  debounceTime(300),          // espera 300ms sem digitar
  distinctUntilChanged(),      // ignora se o valor nao mudou
  switchMap(term =>            // cancela requisicao anterior
    this.http.get<Resultado[]>(`/api/busca?q=${term}`)
  )
).subscribe(resultados => {
  this.resultados = resultados;
});
```

- **Lazy**: only executes when someone subscribes (`.subscribe()`)
- Emits **multiple values** over time
- `switchMap` **cancels** the previous request automatically

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

[Back to index](README.md)

# Promises vs Observables (Angular)

## Comparacao

| Aspecto | Promise | Observable |
|---------|---------|-----------|
| Valores | **Um** valor unico | **Multiplos** valores ao longo do tempo |
| Execucao | **Eager** (executa ao criar) | **Lazy** (executa ao se inscrever) |
| Cancelamento | Nao pode cancelar | Pode cancelar via `unsubscribe()` |
| Operadores | `.then()`, `.catch()` | `map`, `filter`, `switchMap`, `retry`, etc. |
| Uso ideal | Valor unico e simples | Streams, eventos, websockets |

## Promise - Exemplo

Buscar um dado unico ao abrir uma tela:

```typescript
async loadPerfil(): Promise<void> {
  const perfil = await this.http.get<Perfil>('/api/perfil').toPromise();
  this.perfil = perfil;
}
```

- Comeca a executar **imediatamente**
- Retorna **apenas um valor**
- Nao pode ser cancelada

## Observable - Exemplo

Busca em tempo real enquanto o usuario digita:

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

- **Lazy**: so executa quando alguem se inscreve (`.subscribe()`)
- Emite **multiplos valores** ao longo do tempo
- `switchMap` **cancela** a requisicao anterior automaticamente

## Por que Angular usa Observables

Os servicos HTTP do Angular retornam Observables por causa das vantagens:

- Cancelamento nativo
- Multiplas emissoes
- Operadores poderosos de composicao
- Integração com RxJS

## Quando usar cada um

| Cenario | Recomendacao |
|---------|-------------|
| Buscar dado uma unica vez | Promise ou Observable (ambos OK) |
| Busca em tempo real | Observable |
| WebSockets | Observable |
| Eventos do usuario | Observable |
| Requisicao simples e pontual | Promise |

---

[Voltar ao índice](README.md)

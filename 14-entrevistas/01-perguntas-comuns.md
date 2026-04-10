# Perguntas Comuns em Entrevistas .NET

## 1. Por que o padrao Singleton pode ser considerado um anti-pattern?

O Singleton introduz **estado global oculto**, aumenta o acoplamento e dificulta testes automatizados.

- Viola o principio de **Inversao de Dependencia** (DIP)
- Esconde dependencias reais
- Ciclo de vida rigido
- Pode causar problemas de **concorrencia** em ambientes multithread
- Pode causar **memory leak** (referencias que nunca sao liberadas)

**Solucao**: usar DI com ciclo de vida Singleton controlado pelo container.

## 2. Como compartilhar recursos entre requisicoes no ASP.NET Core?

ASP.NET Core e **stateless** - dados compartilhados entre requests precisam viver fora do escopo da requisicao:

| Mecanismo | Uso |
|-----------|-----|
| `IMemoryCache` | Cache em memoria, thread-safe, com expiracao |
| `IDistributedCache` (Redis) | Multiplas instancias, persistente |
| Banco de dados | Dados que precisam sobreviver a restart |
| Variaveis estaticas | **Desencorajado** - quebra DI, dificulta testes, memory leaks |

## 3. Diferenca entre interface e classe abstrata

| Aspecto | Interface | Classe Abstrata |
|---------|-----------|----------------|
| Heranca | Multipla (varias interfaces) | Unica (uma classe base) |
| Implementacao | So assinatura (ate C# 7) | Pode ter metodos concretos |
| Construtor | Nao tem | Pode ter |
| Campos | Nao tem | Pode ter |
| Default methods | Sim (C# 8+) | Sim |
| Quando usar | Contrato / capacidade | Modelo base / hierarquia |

## 4. O que faz uma query SQL ser lenta?

- Falta de indices adequados
- Funcoes nas colunas (nao-sargavel)
- JOINs mal planejados
- Volume sem paginacao
- SELECT * desnecessario
- Estatisticas desatualizadas
- Bloqueios e concorrencia

(Veja detalhes em [Otimização de Queries](../07-acesso-a-dados/04-otimizacao-de-queries.md))

## 5. Lazy Loading vs Eager Loading

- **Lazy**: carrega sob demanda (risco de N+1)
- **Eager**: carrega tudo junto com Include (mais previsivel)
- **Recomendacao**: evitar Lazy Loading na maioria dos cenarios

(Veja detalhes em [Entity Framework](../07-acesso-a-dados/02-entity-framework.md))

## 6. Singleton vs Scoped vs Transient

(Veja detalhes em [Service Lifetimes](../06-aspnet-core/02-service-lifetimes.md))

## 7. O que e Parameter Sniffing?

SQL Server cacheia plano de execucao baseado nos primeiros parametros. Se parametros futuros forem muito diferentes, o plano pode ser pessimo.

Solucao: `OPTION (RECOMPILE)` quando necessario.

(Veja detalhes em [Otimização de Queries](../07-acesso-a-dados/04-otimizacao-de-queries.md))

## 8. Tecnicas de otimizacao de performance

1. **Cache** (em memoria ou distribuido)
2. **Jobs assincronos** (filas para operacoes pesadas)
3. **Monitoramento** com metricas em producao (Application Insights, Grafana)
4. **CDN** no front-end
5. **CQRS** (separacao de bancos de leitura e escrita)
6. **ElasticSearch** para pesquisa textual

## 9. Race Conditions vs Deadlocks

- **Race condition**: threads acessam recurso compartilhado sem sincronizacao
- **Deadlock**: threads esperam eternamente uma pela outra

(Veja detalhes em [Concorrência e Paralelismo](../04-concorrencia-e-paralelismo/))

## 10. Resiliencia de APIs

Padroes: Retry, Timeout, Circuit Breaker, Fallback, Bulkhead, Rate Limiting.

Implementacao: HttpClientFactory + Polly.

(Veja detalhes em [Resiliência de APIs](../06-aspnet-core/04-resiliencia-de-apis.md))

---

[Voltar ao índice](README.md)

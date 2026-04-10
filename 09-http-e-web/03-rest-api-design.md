# REST API Design

## Principios REST

REST (Representational State Transfer) e um estilo arquitetural para APIs HTTP. Nao e um protocolo — e um conjunto de **constraints**:

1. **Client-Server** — separacao de responsabilidades
2. **Stateless** — cada requisicao contem tudo necessario
3. **Cacheable** — respostas podem ser cacheadas
4. **Uniform Interface** — interface padronizada (URLs, metodos, status codes)
5. **Layered System** — intermediarios (proxy, gateway) sao transparentes

## URLs (recursos, nao acoes)

```
# BOM — substantivos no plural
GET    /api/pedidos          # listar
GET    /api/pedidos/42       # obter por ID
POST   /api/pedidos          # criar
PUT    /api/pedidos/42       # atualizar inteiro
PATCH  /api/pedidos/42       # atualizar parcial
DELETE /api/pedidos/42       # remover

# RUIM — verbos na URL
POST   /api/criarPedido
GET    /api/obterPedido/42
POST   /api/deletarPedido/42
```

### Recursos aninhados

```
GET /api/clientes/5/pedidos         # pedidos do cliente 5
GET /api/clientes/5/pedidos/42      # pedido 42 do cliente 5
```

> Nao anide mais de 2 niveis — fica confuso. Prefira query parameters.

## Status Codes

### Sucesso

| Code | Significado | Quando usar |
|------|-------------|-------------|
| **200** | OK | GET, PUT, PATCH com sucesso |
| **201** | Created | POST que criou recurso |
| **204** | No Content | DELETE ou PUT sem body de resposta |

### Erro do cliente

| Code | Significado | Quando usar |
|------|-------------|-------------|
| **400** | Bad Request | Validacao falhou, body invalido |
| **401** | Unauthorized | Nao autenticado |
| **403** | Forbidden | Autenticado mas sem permissao |
| **404** | Not Found | Recurso nao existe |
| **409** | Conflict | Conflito (ex: duplicata, concorrencia) |
| **422** | Unprocessable Entity | Semanticamente invalido |

### Erro do servidor

| Code | Significado |
|------|-------------|
| **500** | Internal Server Error |
| **502** | Bad Gateway |
| **503** | Service Unavailable |
| **504** | Gateway Timeout |

## Paginacao

```
GET /api/pedidos?page=2&pageSize=20

Response:
{
  "data": [...],
  "page": 2,
  "pageSize": 20,
  "totalCount": 150,
  "totalPages": 8
}
```

### Keyset pagination (melhor performance)

```
GET /api/pedidos?after=pedido_xyz&limit=20
```

Mais performatico que OFFSET para datasets grandes.

## Filtros, ordenacao e busca

```
GET /api/pedidos?status=aprovado&clienteId=5     # filtro
GET /api/pedidos?sort=dataCriacao:desc            # ordenacao
GET /api/pedidos?search=notebook                  # busca textual
```

## Versionamento

```
# Via URL (mais comum)
GET /api/v1/pedidos
GET /api/v2/pedidos

# Via header
GET /api/pedidos
Accept: application/vnd.minhaapi.v2+json

# Via query string
GET /api/pedidos?api-version=2.0
```

> URL prefix (`/v1/`) e o mais pragmatico e facil de rotear.

## Respostas de erro padronizadas

Use o padrao **RFC 7807 (Problem Details)**:

```json
{
  "type": "https://api.exemplo.com/errors/validation",
  "title": "Erro de validação",
  "status": 400,
  "detail": "O campo 'nome' é obrigatório",
  "instance": "/api/pedidos",
  "errors": {
    "nome": ["O campo 'nome' é obrigatório"],
    "email": ["Email inválido"]
  }
}
```

Em ASP.NET Core:

```csharp
builder.Services.AddProblemDetails();

// Retorna automaticamente Problem Details para erros
app.UseExceptionHandler();
app.UseStatusCodePages();
```

## HATEOAS (hyperlinks na resposta)

```json
{
  "id": 42,
  "status": "pendente",
  "total": 150.00,
  "_links": {
    "self": { "href": "/api/pedidos/42" },
    "aprovar": { "href": "/api/pedidos/42/aprovar", "method": "POST" },
    "cancelar": { "href": "/api/pedidos/42/cancelar", "method": "POST" },
    "cliente": { "href": "/api/clientes/5" }
  }
}
```

> Na pratica, poucos projetos implementam HATEOAS completo. Mas e bom saber para entrevistas.

## Boas praticas

1. **Substantivos** na URL, **verbos** nos metodos HTTP
2. **Plural** para colecoes (`/pedidos`, nao `/pedido`)
3. **Status codes corretos** — nao retorne 200 para tudo
4. **Paginacao** em listagens — nunca retorne todos os registros
5. **Versionamento** desde o inicio
6. **Problem Details** para erros
7. **Idempotencia** — PUT e DELETE devem ser idempotentes
8. **HTTPS** sempre
9. **Rate limiting** em APIs publicas
10. **Documentacao** — OpenAPI/Swagger

---

[← Anterior: MIME Types](02-mime-types.md) | [Próximo: Segurança Web →](04-seguranca-web.md) | [Voltar ao índice](README.md)

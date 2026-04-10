# HTTP Semantics

## Metodos HTTP e Idempotencia

**Idempotencia**: aplicar a mesma operacao multiplas vezes produz o mesmo resultado da primeira aplicacao.

| Metodo | Safe | Idempotente | Descricao |
|--------|------|-------------|-----------|
| `GET` | Sim | Sim | Buscar recurso. **Nunca** deve alterar estado do servidor |
| `POST` | Nao | Nao | Criar recurso ou executar procedimento |
| `PUT` | Nao | Sim | Substituir recurso inteiro. Client pode gerar o ID |
| `PATCH` | Nao | Nao* | Atualizar parcialmente um recurso |
| `DELETE` | Nao | Sim | Remover recurso. **Nunca retorne 404** quando nao achar |

> *PATCH pode ser idempotente dependendo da implementacao

### DELETE e query parameters

Evite ao maximo query parameters em DELETE. Se for necessario, **valide que todos foram passados**.

## URL

```
scheme://host:port/path?query#fragment
https://api.exemplo.com:443/v1/clientes?ativo=true#resultados
```

### Dicas de seguranca

- **Evite dados sensiveis na URL** (email, CPF, dados que identificam usuarios)
- A URL e logada em varios lugares (access logs, proxies, browser history)
- Query string e menos logada, mas ainda pode ser capturada

## TCP vs UDP

| Aspecto | TCP | UDP |
|---------|-----|-----|
| Garantia de entrega | Sim | Nao |
| Ordem dos pacotes | Garantida | Nao garantida |
| Velocidade | Mais lento | Mais rapido |
| Uso | HTTP, APIs, email | Video, gaming, DNS |

## Encoding de URL

Diferentes partes da URL tem diferentes tipos de encoding:

- `/` no path e um separador, mas no query string e normal
- `%` pode ser problema no query string, mas nao no path
- Caracteres especiais devem ser codificados: `%20` (espaco), `%2F` (/), etc.

## Redirects

| Codigo | Tipo | Nota |
|--------|------|------|
| 301 | Permanent Redirect | Problemas de seguranca em alguns cenarios |
| 302 | Found (temporary) | Uso historico inconsistente |
| 303 | See Other | Problemas de seguranca |
| **307** | Temporary Redirect | Substituto seguro do 302 |
| **308** | Permanent Redirect | Substituto seguro do 301 |

> Prefira **307** e **308** por serem mais seguros e previsíveis.

## Content Negotiation

Cliente e servidor negociam o formato do conteudo:

```http
# Cliente diz o que aceita
Accept: application/json

# Servidor diz o que retorna
Content-Type: application/json
```

---

[Próximo: MIME Types →](02-mime-types.md) | [Voltar ao índice](README.md)

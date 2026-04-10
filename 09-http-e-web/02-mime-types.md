# MIME Types

## O que sao

MIME type (ou media type) e um identificador que informa o **tipo e formato do conteudo** transmitido em HTTP. Permite que cliente e servidor saibam como **interpretar** os dados.

## Tipos comuns

| MIME Type | Conteudo |
|-----------|----------|
| `application/json` | JSON |
| `application/xml` | XML |
| `application/pdf` | PDF |
| `application/octet-stream` | Binario generico |
| `text/plain` | Texto simples |
| `text/html` | HTML |
| `image/png` | Imagem PNG |
| `image/jpeg` | Imagem JPEG |
| `multipart/form-data` | Formularios com arquivos |

## Onde sao usados

### 1. Em requisicoes HTTP

**Content-Type**: indica o tipo do corpo da requisicao.

```http
POST /api/clientes HTTP/1.1
Content-Type: application/json

{"nome": "Osvaldo"}
```

**Accept**: indica quais tipos o cliente aceita como resposta.

```http
GET /api/clientes/1 HTTP/1.1
Accept: application/json
```

### 2. Em uploads de arquivos

```http
POST /api/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=---boundary

---boundary
Content-Disposition: form-data; name="arquivo"; filename="relatorio.pdf"
Content-Type: application/pdf

(conteudo binario do PDF)
---boundary--
```

### 3. Em downloads/responses

```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="relatorio.pdf"

(conteudo do PDF)
```

O browser sabe abrir como PDF, baixar como arquivo, etc.

## Por que MIME type e importante

- Permite **content negotiation** entre cliente e servidor
- Garante que os dados sejam **interpretados corretamente**
- Evita erros de parsing
- Ajuda na **seguranca** (ex: impedir execucao de conteudo malicioso)
- Essencial para **APIs REST** bem definidas

## Em ASP.NET Core

```csharp
// Retornando diferentes tipos de conteudo
[HttpGet("relatorio")]
public IActionResult GerarRelatorio()
{
    var pdf = GerarPdf();
    return File(pdf, "application/pdf", "relatorio.pdf");
}

[HttpGet("dados")]
public IActionResult ObterDados()
{
    return Ok(new { nome = "Osvaldo" }); // retorna application/json automaticamente
}
```

---

[← Anterior: HTTP Semantics](01-http-semantics.md) | [Próximo: REST API Design →](03-rest-api-design.md) | [Voltar ao índice](README.md)

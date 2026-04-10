# MIME Types

## What they are

MIME type (or media type) is an identifier that informs the **type and format of the content** transmitted over HTTP. It allows client and server to know how to **interpret** the data.

## Common types

| MIME Type | Content |
|-----------|---------|
| `application/json` | JSON |
| `application/xml` | XML |
| `application/pdf` | PDF |
| `application/octet-stream` | Generic binary |
| `text/plain` | Plain text |
| `text/html` | HTML |
| `image/png` | PNG image |
| `image/jpeg` | JPEG image |
| `multipart/form-data` | Forms with file uploads |

## Where they are used

### 1. In HTTP requests

**Content-Type**: indicates the type of the request body.

```http
POST /api/customers HTTP/1.1
Content-Type: application/json

{"name": "Osvaldo"}
```

**Accept**: indicates which types the client accepts as a response.

```http
GET /api/customers/1 HTTP/1.1
Accept: application/json
```

### 2. In file uploads

```http
POST /api/upload HTTP/1.1
Content-Type: multipart/form-data; boundary=---boundary

---boundary
Content-Disposition: form-data; name="file"; filename="report.pdf"
Content-Type: application/pdf

(binary PDF content)
---boundary--
```

### 3. In downloads/responses

```http
HTTP/1.1 200 OK
Content-Type: application/pdf
Content-Disposition: attachment; filename="report.pdf"

(PDF content)
```

The browser knows whether to open it as a PDF, download it as a file, etc.

## Why MIME type is important

- Enables **content negotiation** between client and server
- Ensures that data is **interpreted correctly**
- Prevents parsing errors
- Helps with **security** (e.g., preventing execution of malicious content)
- Essential for well-defined **REST APIs**

## In ASP.NET Core

```csharp
// Returning different content types
[HttpGet("report")]
public IActionResult GenerateReport()
{
    var pdf = GeneratePdf();
    return File(pdf, "application/pdf", "report.pdf");
}

[HttpGet("data")]
public IActionResult GetData()
{
    return Ok(new { name = "Osvaldo" }); // returns application/json automatically
}
```

---

[← Previous: HTTP Semantics](01-http-semantics.md) | [Next: REST API Design →](03-rest-api-design.md) | [Back to index](README.md)

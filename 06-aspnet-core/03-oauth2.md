# OAuth 2.0

## O que é

**OAuth (Open Authorization)** permite que sites ou apps de terceiros acessem dados do usuário **sem que ele precise compartilhar suas credenciais**.

OAuth 2.0 é sobre **AUTORIZAÇÃO**, não AUTENTICAÇÃO. É um framework de autorização que pode ser usado para autenticar usuários e dar acesso a eles a recursos protegidos.

## Conceitos chave

- **Access Token** — token que concede acesso a recursos protegidos. Geralmente expiram rápido.
- **Refresh Token** — serve para renovar o access token. Nem todos os fluxos usam refresh token.
- **Authorization Server** — responsável por emitir access tokens.
- **Identity Provider** — responsável por autenticar usuários.
- **Resource Server** — servidor que possui os recursos protegidos.

> Diferente do Google, em alguns casos o Authorization Server e o Identity Provider podem ser diferentes. No OAuth 2.0, apesar de que em alguns casos podem ser a mesma coisa (como no Google), o AUTHORIZATION é responsável por emitir ACCESS TOKENS e o IDENTITY PROVIDER é responsável por AUTENTICAR usuários.

## Fluxo OAuth 2.0

```
1. Client ──── Authorization request ────→ Resource Owner
2. Client ←─── Authorization grant ──────  Resource Owner

3. Client ──── Authorization grant ────→ Authorization Server
4. Client ←─── Access token ────────── Authorization Server

5. Client ──── Access token ────────→ Resource Server
6. Client ←─── Protected resource ── Resource Server
```

## Exemplo prático

Vamos supor que você queira fazer uma aplicação de calendário que usa o Google Calendário:

1. A aplicação terceira obtém um access token do Google, usando o OAuth 2.0 para autenticar no Google
2. O Google emite um access token para a aplicação
3. A aplicação pega esse access token e gera um JWT próprio dela
4. A aplicação manda o JWT para o Google, dessa forma o Google consegue validar o JWT e concede acesso para a aplicação

Quando logamos com o Google em um site terceiro, quando um site terceiro consegue manipular o calendário do Google, tudo isso é feito através de um access token. Dessa forma não é preciso usar a senha e login do usuário.

## JWT (JSON Web Token)

JWT pode ser usado para representar o access token. É um padrão aberto (RFC 7519) que define uma forma compacta e auto-contida de transmitir informações entre partes como um objeto JSON.

```
Header.Payload.Signature
```

```csharp
// Em ASP.NET Core, configurar autenticação JWT:
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidIssuer = "sua-api",
            ValidAudience = "seu-client",
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes("sua-chave-secreta"))
        };
    });
```

---

[← Anterior: Service Lifetimes](02-service-lifetimes.md) | [Voltar ao índice](README.md)

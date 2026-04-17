# 10 - Security

Security is a senior responsibility — the interviewer assumes you can prevent the common OWASP categories, pick the right authentication flow, store secrets correctly, and talk about supply-chain risk. This section groups the topics you are expected to own at a senior / lead level, with .NET-specific mitigations wherever the platform has opinionated APIs.

## Content

1. [OWASP Top 10 for .NET](01-owasp-top-10-dotnet.md) - The 2021 categories and the concrete .NET mitigation for each
2. [Authentication Patterns](02-authentication-patterns.md) - Cookies vs JWT vs OIDC, session vs stateless, refresh tokens, PKCE
3. [Authorization](03-authorization.md) - RBAC, ABAC, and policy-based authorization in ASP.NET Core (requirements, handlers, claims)
4. [Secrets Management](04-secrets-management.md) - Azure Key Vault, AWS Secrets Manager, User Secrets, env vars, rotation
5. [Threat Modeling](05-threat-modeling.md) - STRIDE, DFDs, attack trees, and the Microsoft Threat Modeling Tool
6. [Supply Chain Security](06-supply-chain-security.md) - `dotnet list package --vulnerable`, SBOM (SPDX, CycloneDX), Sigstore, signed NuGet, Dependabot
7. [Cryptography Basics](07-cryptography-basics.md) - Password hashing, symmetric / asymmetric primitives, TLS 1.3, HSTS, ASP.NET Core Data Protection

---

[Back to index](../README.md)

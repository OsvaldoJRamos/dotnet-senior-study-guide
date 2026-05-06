# 12 - Security

Security is a senior responsibility — the interviewer assumes you can prevent the common OWASP categories, pick the right authentication flow, store secrets correctly, and talk about supply-chain risk. This section groups the topics you are expected to own at a senior / lead level, with .NET-specific mitigations wherever the platform has opinionated APIs.

## Content

1. [OWASP Top 10 for .NET](01-owasp-top-10-dotnet.md) - The 2021 categories and the concrete .NET mitigation for each
2. [Authentication Patterns](02-authentication-patterns.md) - Cookies vs JWT vs OIDC, session vs stateless, refresh tokens, PKCE
3. [Authorization](03-authorization.md) - RBAC, ABAC, and policy-based authorization in ASP.NET Core (requirements, handlers, claims)
4. [Secrets Management](04-secrets-management.md) - Azure Key Vault, AWS Secrets Manager, User Secrets, env vars, rotation
5. [Threat Modeling](05-threat-modeling.md) - STRIDE, DFDs, attack trees, and the Microsoft Threat Modeling Tool
6. [Supply Chain Security](06-supply-chain-security.md) - `dotnet list package --vulnerable`, SBOM (SPDX, CycloneDX), Sigstore, signed NuGet, Dependabot
7. [Cryptography Basics](07-cryptography-basics.md) - Password hashing, symmetric / asymmetric primitives, TLS 1.3, HSTS, ASP.NET Core Data Protection
8. [Network and Perimeter Security](08-network-perimeter-security.md) - DDoS categories and mitigations, WAF (CRS, deployment models), VPN protocols, ZTNA

## Useful Links

- [Cryptography and its Types (GeeksforGeeks)](https://www.geeksforgeeks.org/computer-networks/cryptography-and-its-types/) — symmetric/asymmetric/hash overview.
- [What is DDoS? (GeeksforGeeks)](https://www.geeksforgeeks.org/computer-networks/what-is-ddosdistributed-denial-of-service/) — categories, mitigations, historical attacks.
- [What is a Web Application Firewall? (GeeksforGeeks)](https://www.geeksforgeeks.org/computer-networks/what-is-a-web-application-firewall/) — WAF deployment models and rule strategies.
- [What is a VPN? (GeeksforGeeks)](https://www.geeksforgeeks.org/computer-networks/what-is-vpn-how-it-works-types-of-vpn/) — protocols, types, trade-offs.
- [Cyber Security Tutorial (GeeksforGeeks)](https://www.geeksforgeeks.org/cybersecurity/cyber-security-tutorial/) — broad cybersecurity primer (good for filling foundational gaps).

---

[Back to index](../README.md)

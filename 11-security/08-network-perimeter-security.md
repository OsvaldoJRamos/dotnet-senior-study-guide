# Network and Perimeter Security

Application-level mitigations matter, but a senior is also expected to reason about what sits **in front of** the app: the perimeter. DDoS, WAF, and VPN are the three concepts that come up most in interviews.

## DDoS (Distributed Denial of Service)

Attacks where many compromised hosts (a botnet) flood a target to exhaust capacity. Three common categories:

| Category | What it targets | Example |
|---|---|---|
| **Volumetric** | Bandwidth | UDP/ICMP flood, DNS amplification |
| **Protocol** | Connection state | SYN flood, fragmented packet floods |
| **Application-layer (L7)** | Request handlers | Slow-loris, HTTP flood — looks like real traffic |

### Mitigation patterns

- **Anycast + scrubbing centers** — provider routes traffic through globally distributed PoPs (CloudFront, Cloudflare, Azure Front Door, AWS Shield) that absorb floods before they reach origin.
- **Rate limiting** — at the WAF / gateway / API tier, on identity (token, API key) or IP. ASP.NET Core has built-in rate limiting (`Microsoft.AspNetCore.RateLimiting`).
- **Connection-level defenses** — SYN cookies, TCP intercept, conntrack tuning at the load balancer.
- **CAPTCHA / challenge** — for L7 floods; surface a JS challenge or interactive proof when traffic patterns look automated.
- **Capacity planning** — auto-scaling absorbs small attacks but is not a defense; an attacker scaling faster than your bill tolerance still wins.

> **What you're expected to know:** DDoS defense is a layered concern. Origin-only defenses lose to volumetric attacks. The realistic answer in an interview is "edge provider for L3/L4, WAF + rate limiting for L7, autoscaling and circuit breakers for graceful degradation."

## WAF (Web Application Firewall)

A reverse-proxy filter at OSI layer 7 that inspects HTTP requests and blocks known-bad patterns (SQL injection, XSS, RCE) before they reach the app.

### Where it sits

```
Client → CDN → WAF → Load balancer → App
```

### Rule models

- **Negative security (blocklist)** — block requests that match known attack signatures. Most common; powered by rulesets like the **OWASP CRS** (Core Rule Set), originally built for ModSecurity and now also used with Coraza and other compatible WAFs.
- **Positive security (allowlist)** — only allow requests that match a defined schema (URL, method, parameter types). Stronger but operationally expensive to maintain.
- **Anomaly scoring** — modern WAFs combine signatures + behavioral scoring; high score → block.

### Common products

| Type | Examples |
|---|---|
| **Cloud / managed** | AWS WAF, Cloudflare, Azure Front Door / Application Gateway WAF, Akamai |
| **Self-hosted** | ModSecurity (Apache/Nginx), open-source Coraza |
| **In-process** | Lightweight middleware (e.g., NWebsec) — limited scope |

### Limitations

- Tuning is real work — false positives block legitimate traffic, false negatives let attacks through.
- Bypassable if the app exposes endpoints the WAF can't decode (binary protocols, custom encodings, encrypted payloads).
- Not a substitute for parameterized queries, output encoding, or auth; it is a defense-in-depth layer, not the only defense.

## VPN (Virtual Private Network)

An encrypted tunnel that makes two endpoints behave as if they were on the same private network. Two scenarios matter for backend engineers:

| Type | Use case |
|---|---|
| **Remote access** | Individual users (developers, support) reach internal services over the public internet — typically via WireGuard, OpenVPN, or IKEv2/IPsec. |
| **Site-to-site** | Two networks (office ↔ cloud, region ↔ region) joined as one — IPsec tunnels, AWS Site-to-Site VPN, Azure VPN Gateway. |

### Protocols, briefly

| Protocol | Notes |
|---|---|
| **WireGuard** | Modern, deliberately small codebase, fast, opinionated crypto. Default for new deployments. |
| **IPsec / IKEv2** | Industry-standard, ubiquitous in cloud-to-cloud and corporate VPN appliances. |
| **OpenVPN** | SSL/TLS-based, very portable, slower than WireGuard. |
| **L2TP/IPsec** | Legacy; avoid for new work. |

### Where VPNs are losing ground

For app access, VPN is being displaced by **Zero Trust Network Access (ZTNA)** — identity-aware proxies (Cloudflare Access, Tailscale, AWS Verified Access) that authenticate per-request instead of granting blanket network access. A senior should mention this trade-off: VPN gives you a flat tunnel and trusts everything inside; ZTNA enforces auth and policy at every connection.

> **For the interview:** know the three perimeter layers (DDoS provider → WAF → app), the dominant VPN protocols, and that ZTNA is the modern replacement for "VPN into the office network."

---

[← Previous: Cryptography Basics](07-cryptography-basics.md) | [Back to index](README.md)

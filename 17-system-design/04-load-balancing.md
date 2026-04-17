# Load Balancing

A load balancer takes one IP/DNS name and distributes incoming traffic across many backends. The question isn't *"do I need one"* — for anything horizontally scaled, yes — it's *"at what OSI layer, with which algorithm, and how does it fail"*.

## L4 vs L7

| | L4 (transport) | L7 (application) |
|---|---|---|
| Inspects | IP + TCP/UDP port | HTTP headers, path, method, cookies |
| Latency | Lower (no HTTP parse) | Slightly higher (parses HTTP) |
| Termination | Pass-through (can also terminate TLS) | Terminates TLS, re-encrypts to backend or speaks HTTP/1.1, HTTP/2 |
| Routing decisions | 5-tuple hash, connection counts | Path, host, header, cookie, weighted |
| Typical products | AWS NLB, Azure Load Balancer, HAProxy (TCP mode), MetalLB | AWS ALB, Azure Application Gateway / Front Door, NGINX, Envoy, Traefik |

Azure Load Balancer "operates at layer 4 of the Open Systems Interconnection (OSI) model" ([docs](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)). AWS ALB "functions at the application layer, the seventh layer of the Open Systems Interconnection (OSI) model" ([docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)). AWS NLB "functions at the fourth layer" ([docs](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)).

### When L4

- Non-HTTP (gRPC-over-TCP without ALB, Redis, PostgreSQL, WebSocket-at-scale where you don't need path routing).
- Extreme throughput / ultra-low latency (NLB is pass-through and scales to millions of flows).
- You need source-IP preservation without HTTP headers (X-Forwarded-For).

### When L7

- HTTP APIs that need path-based, host-based, or header-based routing.
- Web apps behind a single LB serving multiple microservices by path prefix.
- You need WAF, bot management, authentication at the LB, or HTTP-specific health checks.

### gRPC and HTTP/2 nuance

HTTP/2 multiplexes many requests over a single long-lived TCP connection. An L4 LB hashes on the 5-tuple and will pin **all** requests from one client to one backend — which defeats load balancing for long-lived clients. **Use an L7 LB (ALB, Envoy, NGINX) that load-balances per-request** for HTTP/2/gRPC.

## Algorithms

| Algorithm | How it picks | Pros | Cons / when wrong |
|---|---|---|---|
| **Round robin** | Next backend in a list | Dirt simple, fair for uniform backends | Ignores load — long requests pile up |
| **Weighted round robin** | Same, but some backends get more turns | Handles heterogeneous hardware | Still load-blind |
| **Least connections** | Backend with fewest in-flight connections | Good for variable-duration requests | Requires tracking; stale info across LB instances |
| **Least response time** | Backend with lowest observed latency | Follows real load | Needs recent samples; can oscillate |
| **IP hash** | `hash(client_ip) % N` | Affinity without cookies | Rehashes everything on N change; hot clients = hot backends |
| **Consistent hashing** | Hash clients/keys onto a ring; each node owns an arc | On scale-out, only ~1/N keys move | Slightly more complex; needs virtual nodes for balance |
| **Power of two choices** | Pick 2 at random, send to the less loaded | Near-optimal, gossip-free | Needs a load signal |

### Consistent hashing in one paragraph

Introduced by Karger et al., *"Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web"* (STOC 1997). Map both keys and nodes onto a hash ring. A key is owned by the next node clockwise. Adding or removing a node relocates only ~1/N of keys, not (N-1)/N as with modulo. Use **virtual nodes** (each physical node placed at many ring positions) to smooth distribution. This is the backbone of DynamoDB, Cassandra partitioning, and most CDN edge routing.

## Health checks

A load balancer is only as good as its failure detection. Three things to configure:

1. **Probe type** — TCP connect, HTTP GET of a dedicated `/health` path, or gRPC health check.
2. **Frequency and threshold** — e.g., every 10 s, mark unhealthy after 3 consecutive failures, healthy after 2 successes.
3. **What `/health` returns** — depth matters.

### Shallow vs deep health checks

| Level | Checks | Risk |
|---|---|---|
| Shallow (liveness) | Process running, bound port | Misses bad downstream deps |
| Deep (readiness) | DB connection, cache, downstream dependency | A flaky dep takes *all* replicas offline |

> Use **shallow** for LB health checks and **deep** only for readiness / deploy gating. Coupling an LB probe to your DB means one slow DB yields 100% 5xx because every replica is marked unhealthy.

## Sticky sessions (session affinity)

The LB pins a client to a specific backend, typically via a cookie or IP hash.

When you need stickiness:

- Legacy servers with in-memory session state.
- WebSocket / SignalR without a distributed back-plane (see [SignalR Redis back-plane](../08-aspnet-core/) for the proper fix).
- Short-lived warm caches that are expensive to rebuild per request.

Why it's usually wrong:

- A dying backend takes its pinned clients with it.
- Capacity is uneven — you can't drain cleanly.
- Blue/green and rolling deploys get messier.

The senior answer: **externalize state, use stateless backends, and drop stickiness.**

## Global load balancing

A single-region LB won't help a user on another continent. Three common strategies:

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| **DNS-based (GeoDNS)** | Authoritative DNS returns different IPs based on client location | Simple, works with any protocol | TTL-bound (clients cache DNS), no real-time failover |
| **Anycast** | The same IP is announced from many PoPs via BGP; the internet routes to the nearest | True sub-second failover, protocol-agnostic | Needs BGP and multi-PoP presence |
| **Managed global LB** (AWS Global Accelerator, Azure Front Door, Cloudflare) | Anycast front + managed routing to regional origins | Plug-and-play, adds TLS termination and WAF | Vendor lock-in, cost |

> DNS TTL is the biggest gotcha: set a 300-second TTL and a failover takes up to 5 minutes for users who already resolved. Anycast is preferred for sub-minute failover.

## Health-driven traffic shaping

Modern LBs (Envoy, ALB) support more than on/off:

- **Outlier detection** — eject a host from the pool temporarily after N consecutive 5xx, let it back in after a cool-down.
- **Circuit breakers at the LB** — max connections/requests per backend; overflow gets rejected fast rather than queuing.
- **Retries with jitter** — transparent retry on idempotent 5xx, but budget them (e.g., max 20% retry ratio) to avoid retry storms.

## Concrete recommendations

| Situation | Pick |
|---|---|
| Public HTTP/HTTPS APIs | L7 (ALB / App Gateway / Front Door / NGINX Ingress) |
| Non-HTTP TCP/UDP (Redis, databases, game servers) | L4 (NLB / Azure LB / HAProxy TCP) |
| gRPC / HTTP/2 at scale | L7 — must load-balance per request, not per connection |
| Global multi-region | Anycast + managed global LB (Front Door, Global Accelerator, Cloudflare) |
| Internal east-west traffic in Kubernetes | Service mesh sidecar (Envoy via Istio / Linkerd) |

---

[← Previous: Scaling Strategies](03-scaling-strategies.md) | [Next: Caching Strategies →](05-caching-strategies.md) | [Back to index](README.md)

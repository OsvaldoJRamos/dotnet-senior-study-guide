# API Gateway and Backend For Frontend (BFF)

In a microservice system, clients shouldn't have to know about every service. The **API Gateway** and **BFF** patterns give you a single entry point (or one per client type) that hides the internal fan-out, translates protocols, and tailors the response to the caller.

## API Gateway

> Chris Richardson: *"Implement an API gateway that is the single entry point for all clients. The API gateway handles requests in one of two ways. Some requests are simply proxied/routed to the appropriate service. It handles other requests by fanning out to multiple services."* (`microservices.io/patterns/apigateway.html`)

### What a gateway typically does

| Concern | What lives at the gateway |
|---|---|
| **Routing** | `/orders/*` → Orders service; `/payments/*` → Payments service |
| **Composition** | One request → fan out to 3 services → merge the responses |
| **Protocol translation** | Public HTTPS/JSON → internal gRPC / AMQP |
| **Authn / Authz** | Validate JWT once, forward caller identity downstream |
| **Rate limiting, quotas** | Per-caller throttling before the request touches a service |
| **Observability** | Single place to capture latency, errors, correlation IDs |
| **Caching** | Cache read-heavy fan-out responses at the edge |
| **Versioning** | `/v1/orders` and `/v2/orders` routed to the right backend shape |

### Common products

- **Azure API Management**, **AWS API Gateway**, **Google Cloud Apigee** — managed services.
- **Kong**, **Tyk**, **Ocelot** (.NET), **YARP** (.NET reverse proxy) — self-hosted.
- **Envoy** and **Istio/Linkerd service meshes** overlap with gateway concerns at L7.

### Drawbacks

- **Single point of failure** if not deployed redundantly.
- **Adds a network hop** — usually negligible, but latency-sensitive paths may bypass it.
- **Becomes a shared bottleneck** if every team is blocked on gateway changes — the motivation for BFF.
- **Risk of turning into a God Service** if you let business logic leak into the gateway. Keep it thin.

## Backend For Frontend (BFF)

> Sam Newman: *"The BFF is tightly coupled to a specific user experience, and will typically be maintained by the same team as the user interface."* (`samnewman.io/patterns/architectural/bff/`)

Instead of one gateway trying to satisfy every client, deploy **one backend per frontend type** — e.g., a *web BFF*, a *mobile BFF*, a *partner BFF*. Each is owned by the team that owns that UI.

### Why not just one gateway?

- **Mobile** wants compact payloads, aggressive aggregation, minimal round-trips over flaky networks.
- **Web** wants browser-friendly shapes, cookies, SSR data.
- **Partner/public API** wants versioning, documentation, contract stability.

A one-size-fits-all API gateway compromises all three — or worse, the slowest consumer freezes the others.

### BFF responsibilities

- **Aggregation**: call 4 services, merge the results, shape the response for this specific UI.
- **Protocol adaptation**: browser-friendly JSON for web; more compact formats (or GraphQL) for mobile.
- **Team autonomy**: UI team can change the BFF without coordinating with backend teams.

### When BFF pays off

- Multiple distinct client types (mobile + web + partner).
- Significant aggregation needed (3+ downstream calls per page).
- Frontend and backend are owned by the **same team** — otherwise BFFs become mini gateways with more coordination, not less.

### When not to use BFF

- Single web client with thin aggregation — one well-designed API is enough.
- You can't staff separate teams for separate clients — one BFF becomes a stale bottleneck.

## Gateway vs BFF vs Service Mesh

| Concern | API Gateway | BFF | Service Mesh |
|---|---|---|---|
| **Audience** | External clients | One specific client type | Service-to-service (east–west) traffic |
| **Owned by** | Platform team | UI team | Platform / SRE |
| **Typical logic** | Routing, authz, rate limiting | Aggregation, response shaping | Retries, mTLS, circuit breaking, routing |
| **Example** | Azure APIM, Kong | Node.js / ASP.NET Core per client | Istio, Linkerd, Consul Connect |

The three are complementary, not alternatives. A production system often runs all three — gateway at the edge for external traffic, BFFs for each client UI, and a mesh for internal service-to-service communication.

## Pitfalls

- **Business logic in the gateway.** If the gateway computes discount rules or validates domain invariants, you have a God Service. Keep gateways thin; business logic belongs in the services.
- **BFF explosion.** One BFF per tiny UI surface is operational overhead without payoff. Merge them if the team scope is the same.
- **Fan-out without concurrency.** A BFF calling 4 services serially turns a 50 ms request into 200 ms. Use `Task.WhenAll` (or equivalent) with per-call timeouts and partial-failure handling.
- **Missing timeouts / circuit breakers.** A slow downstream cascades into a gateway/BFF thread exhaustion. Use [Polly](../08-aspnet-core/) or equivalent for resilience.
- **Mixing authentication with authorization.** Gateway validates the token (authentication). Per-service policies decide what the caller can do (authorization). Don't centralize domain authz at the gateway.

## Rule of thumb

> **Gateway** for cross-cutting concerns (authn, routing, rate limiting). **BFF** when distinct UI types need distinct aggregations. **Mesh** for east-west traffic patterns. Keep all three thin and boring — their job is plumbing, not business rules.

---

[← Previous: Domain-Driven Design](13-ddd.md) | [Back to index](README.md) | [Next: Event Sourcing →](15-event-sourcing.md)

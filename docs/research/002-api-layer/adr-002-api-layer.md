# ADR-002: API Layer Architecture for Customer Portal BFF

**Status:** Proposed
**Date:** 2026-02-14
**Deciders:** TedaTech Platform Team
**Related Issues:** [#3 Keycloak Auth](https://github.com/TedaTech/platform-v1-portal/issues/3), [#4 KillBill Integration](https://github.com/TedaTech/platform-v1-portal/issues/4), [#7 Pipeline Orchestration](https://github.com/TedaTech/platform-v1-portal/issues/7)

---

## Context

The TedaTech Customer Portal requires a Backend-for-Frontend (BFF) layer that sits between the React SPA (Vite + TanStack Router + Refine + shadcn/ui) and two backend systems:

1. **Kubernetes API** -- all infrastructure operations (tenant provisioning, Forgejo instances, Cozystack resources, pipeline orchestration) abstracted behind Custom Resource Definitions (CRDs)
2. **KillBill API** -- billing, subscriptions, invoices, and payments via its REST API (250+ endpoints)

The BFF must support real-time status updates (CRD status changes pushed to the browser), OIDC authentication via Keycloak, and multi-tenant isolation. The frontend uses Refine's `dataProvider` (CRUD) and `liveProvider` (real-time subscriptions) interfaces.

This ADR captures the decision on four interconnected choices: API paradigm, backend language, real-time protocol, and architecture pattern.

---

## Decision Drivers

1. **Two backends, not twenty.** The BFF integrates exactly two upstream APIs (K8s and KillBill). This constrains the value proposition of paradigms optimized for many-backend aggregation.
2. **Refine frontend.** The data provider ecosystem heavily favors REST: `@refinedev/simple-rest` has ~40k weekly downloads versus ~2.6k for `@refinedev/graphql` and zero viable tRPC provider.
3. **Cilium Gateway API ingress.** All traffic enters through Cilium's Envoy-based gateway. Active bugs with WebSocket upgrade through this gateway ([#40822](https://github.com/cilium/cilium/issues/40822), [#41123](https://github.com/cilium/cilium/issues/41123)) constrain real-time protocol choice.
4. **Existing Go patterns.** The `platform-v1-hetzner-operators` monorepo has established Go patterns (interface-based API clients with mockgen, three-layer rate limiting, TransientError/TerminalError classification) that transfer directly to a Go BFF.
5. **Real-time is a must-have.** Users must see live provisioning status (CRD condition transitions) without polling. The data flow is unidirectional: server-to-client only.
6. **Small team, MVP stage.** 1-3 developers. Operational simplicity and development velocity outweigh architectural flexibility.
7. **Resource-constrained deployment.** The BFF runs on a Cozystack tenant with resource quotas. Container footprint directly impacts infrastructure cost.

---

## Considered Options

### API Paradigm
- **Option A: REST (OpenAPI 3.0)** -- Schema-first, `oapi-codegen` for Go server, `orval` for TypeScript client generation
- **Option B: GraphQL** -- Schema-first via `gqlgen`, unified query/mutation/subscription transport
- **Option C: tRPC** -- End-to-end TypeScript type inference, zero codegen

### Backend Language
- **Option A: Go** -- `client-go` / `controller-runtime` for K8s interaction, goroutine-based concurrency
- **Option B: TypeScript (Node.js/Bun)** -- `@kubernetes/client-node`, enables tRPC, unified language with frontend

### Real-Time Protocol
- **Option A: Server-Sent Events (SSE)** -- Unidirectional, standard HTTP, built-in reconnection
- **Option B: WebSocket** -- Bidirectional, requires protocol upgrade, custom reconnection logic

### Architecture Pattern
- **Option A: BFF Monolith** -- Single Go binary serving REST + SSE, in-process event pipeline
- **Option B: API Gateway + Microservices** -- Separate provisioning service, billing service, and event bus

---

## Decision

**Go BFF Monolith with REST (OpenAPI 3.0) + SSE.**

| Layer | Choice |
|-------|--------|
| API Paradigm | REST (OpenAPI 3.0) |
| Backend Language | Go |
| Backend Framework | Chi + oapi-codegen |
| K8s Client | client-go / controller-runtime |
| Real-time Protocol | SSE (text/event-stream) |
| Architecture | BFF Monolith |
| Frontend Types | orval (TanStack Query hooks from OpenAPI spec) |
| Auth | keyfunc/v3 + golang-jwt/v5 (JWT validation against Keycloak JWKS) |

---

## Rationale

### Why Go over TypeScript

**client-go is the gold standard for Kubernetes interaction.** The BFF's primary backend is Kubernetes CRDs with informer-based real-time updates. client-go's SharedInformerFactory, DeltaFIFO queue, automatic reconnect and resync are battle-tested in every K8s controller in existence. The TypeScript alternative (`@kubernetes/client-node`) has no SharedInformerFactory equivalent, no built-in Server-Side Apply, no leader election, and has known informer stability issues ([#379](https://github.com/kubernetes-client/javascript/issues/379), [#660](https://github.com/kubernetes-client/javascript/issues/660)).

**Existing patterns transfer directly.** The `platform-v1-hetzner-operators` monorepo provides interface-based API clients with mockgen, three-layer rate limiting as HTTP transport middleware, and TransientError/TerminalError classification. Approximately 80% of these patterns apply directly to a REST BFF. Choosing TypeScript means rebuilding all of them (~30% conceptual transfer).

**Container footprint matters at scale.** Go binary: ~10-30 MB image, ~20-50 MB idle memory, <100 ms startup. Node.js: ~150-200 MB image, ~80-150 MB idle memory, ~500 ms-1s startup. In a multi-tenant platform where each tenant may get a BFF instance, this is a 3-5x difference in resource consumption.

**envtest for K8s integration testing.** Go's `envtest` starts a real K8s API server (etcd + kube-apiserver) in-process, enabling testing of informer behavior, watch events, CRD validation, and Server-Side Apply against a real API. TypeScript has no equivalent; testing requires either brittle mocks or a full kind/k3d cluster.

### Why REST (OpenAPI) over GraphQL and tRPC

**Refine provider ecosystem.** `@refinedev/simple-rest` has ~40k weekly downloads -- 15x more than `@refinedev/graphql` (~2.6k). The REST provider maps directly to standard REST conventions (`GET /resources`, `GET /resources/:id`, `POST /resources`) with minimal integration effort. There is no viable tRPC provider (the only community attempt has 3 GitHub stars and was abandoned in January 2023).

**KillBill is REST -- 1:1 proxy is simplest.** KillBill exposes a REST/JSON API. Wrapping it in REST through the BFF is a thin proxy (request in, call KillBill, transform response, return). GraphQL would require defining GraphQL types, query/mutation resolvers, and mapping logic for every exposed KillBill resource -- a translation layer that adds complexity without functional benefit when there are only two backends.

**Operational simplicity.** REST uses standard HTTP: debug with curl, browser devtools, Postman. HTTP caching (ETags, `Cache-Control`) works out of the box. Monitoring uses standard HTTP status codes. GraphQL returns HTTP 200 for everything (errors in the response body), requires GraphQL-aware APM, and makes HTTP caching non-trivial (all requests are `POST /graphql`).

**tRPC is eliminated by Go.** tRPC is TypeScript-only with no Go server implementation. Choosing Go eliminates tRPC entirely.

### Why SSE over WebSocket

**Cilium Gateway API compatibility.** There are active, unresolved bugs where WebSocket connections fail through Cilium's Gateway API despite working with direct connections ([#40822](https://github.com/cilium/cilium/issues/40822) -- intermittent close code 1006, [#41123](https://github.com/cilium/cilium/issues/41123) -- auth tokens not forwarded during WebSocket upgrade). SSE is standard HTTP with `Content-Type: text/event-stream`. It requires zero special proxy configuration, zero `appProtocol` annotations, and has zero known compatibility issues with Cilium.

**Unidirectional data flow matches the use case.** K8s CRD status changes flow server-to-client. The client never sends data that the server needs to process in real-time. Subscriptions are navigation-driven: opening a tenant detail page opens `GET /api/tenants/{id}/events`, navigating away closes the EventSource. This maps naturally to SSE's one-stream-per-resource pattern.

**Built-in reconnection and resume.** The EventSource API (and `@microsoft/fetch-event-source`) automatically reconnects on disconnection. The `Last-Event-ID` header enables seamless resume from the last received event. WebSocket requires manual reconnection with exponential backoff and re-subscription of all watches after reconnection.

**HTTP/2 multiplexing.** Multiple SSE streams multiplex efficiently over a single TCP connection via HTTP/2 (served by Cilium Gateway by default for HTTPS). WebSocket over HTTP/2 (RFC 8441) has known Envoy issues ([envoy#37020](https://github.com/envoyproxy/envoy/issues/37020)) and poor browser adoption, falling back to one TCP connection per WebSocket.

**Simpler Go implementation.** SSE server: ~50 lines, standard `net/http`, no external dependencies. WebSocket server: ~80 lines, requires external library (`github.com/coder/websocket`), concurrent read/write goroutines, custom JSON subscription protocol.

### Why BFF Monolith over Microservices

**Only two backends.** The microservices pattern is designed for systems integrating many backends with divergent scaling characteristics. With exactly two backends, the overhead of service decomposition (inter-service auth, distributed tracing, event bus infrastructure) far exceeds any benefit.

**In-process event pipeline eliminates infrastructure.** In the monolith, K8s informer events flow through Go channels to SSE writers -- no serialization, no network hop, no delivery guarantee complexity. Microservices would require an event bus (NATS/Redis), message ordering guarantees, delivery acknowledgments, and replay logic across service boundaries.

**Resource efficiency.** BFF Monolith: 256-512 MB total memory (2 replicas). Microservices equivalent: 640-1280 MB (3 services x 2 replicas + event bus). On a resource-constrained Cozystack tenant, this 2.5-3x difference is material.

**Development velocity.** One process, one binary, `go run ./cmd/bff`. Tests: `go test ./...`. CI: one pipeline, one image, one deploy. No docker-compose with 5 containers for local development.

**Clean extraction path.** The monolith is structured with explicit module boundaries (interfaces, no cross-module data access, event bus abstraction). Any module can be extracted into a standalone service when specific, measurable triggers are hit (>1000 concurrent SSE connections, >500 MB informer cache, >4 developers, persistent deploy cadence divergence).

---

## Consequences

### Positive

- **Reuse existing Go patterns.** Interface-based API clients, rate limiting middleware, error classification, and mockgen testing patterns from `platform-v1-hetzner-operators` transfer directly.
- **Mature K8s client.** client-go's SharedInformerFactory provides reliable, efficient CRD watching with automatic reconnect, resync, and in-memory caching.
- **Simple SSE relay.** Informer callback -> Go channel -> SSE writer. No external event bus, no message serialization, no delivery guarantee complexity.
- **Standard HTTP caching.** ETags mapped from K8s `resourceVersion`, `Cache-Control` headers, conditional requests -- all work natively with REST.
- **Small container footprint.** ~30 MB image on `scratch`/distroless, ~50-100 MB runtime memory, <100 ms cold start. Suitable for scale-to-zero patterns (KEDA).
- **Operational simplicity.** Single Deployment, single ServiceAccount, standard HTTP monitoring, curl-debuggable endpoints, Swagger UI for API documentation.

### Negative

- **No end-to-end tRPC type safety.** The team must maintain an OpenAPI spec and run a codegen step. Spec drift between the OpenAPI definition and the Go implementation is possible. Mitigated by: spec-first development workflow, CI validation that generated code matches the spec, `oapi-codegen` + `orval` for typed server and client generation.
- **Codegen step required.** Changes to the API surface require regenerating Go server stubs (via `oapi-codegen`) and TypeScript client hooks (via `orval`). This adds a build step that tRPC eliminates entirely.
- **Frontend team must learn Go API patterns.** The BFF is Go, the frontend is TypeScript. Developers working across the stack must be comfortable in both languages. Mitigated by: Go is already the team's backend language, and the API contract (OpenAPI spec) serves as the shared interface between frontend and backend work.
- **Custom Refine liveProvider required.** Refine has no built-in SSE adapter. A custom `liveProvider` using `@microsoft/fetch-event-source` must be written (~40 lines of code).

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| `@kubernetes/client-node` matures significantly, making TypeScript viable | Low (2-3 year horizon) | Medium | Clean interfaces in the Go BFF allow future port. The API contract (OpenAPI spec) is language-agnostic. |
| tRPC gains Go server support (e.g., via connectrpc.com) | Low | Low | Monitor ConnectRPC ecosystem. If a production-ready Go tRPC server emerges, evaluate migration cost vs. benefit. |
| Cilium fixes WebSocket bugs, making WebSocket viable | Medium (6-12 months) | Low | SSE is still simpler for unidirectional data flow. WebSocket would only be preferred if bidirectional communication becomes a requirement. |
| Envoy `stream_idle_timeout` terminates SSE connections | Medium | Low | Server sends keepalive comments (`: keepalive\n\n`) every 15 seconds. Optionally increase timeout via Cilium Helm values. |
| `@microsoft/fetch-event-source` library is abandoned | Low | Low | Library is ~300 lines of code. Can be vendored or rewritten if maintenance stops. |
| BFF Monolith hits scaling limits | Low (beyond ~500 tenants) | Medium | Modular internal structure (interfaces, event bus abstraction) enables mechanical extraction of modules into microservices. Scaling trigger thresholds documented. |
| OpenAPI spec drift from Go implementation | Medium | Medium | Spec-first workflow enforced by CI: `oapi-codegen` generates server interfaces from the spec, compilation fails if handlers do not implement the interface. |

---

## Impact on Related Issues

### #3 -- Keycloak Authentication

The BFF validates JWTs using `keyfunc/v3` + `golang-jwt/v5` against Keycloak's JWKS endpoint. The auth middleware:

1. Extracts the Bearer token from the `Authorization` header
2. Validates JWT signature, issuer, expiration, and algorithm (RS256/ES256 allowlist)
3. Resolves tenant from the `groups` claim (`/tenants/{tenant-id}`) or `tenant_id` custom claim
4. Injects claims and tenant context into the request

SSE connections use `@microsoft/fetch-event-source` on the frontend, which supports `Authorization` headers (unlike the native `EventSource` API). The BFF closes SSE connections when the JWT expires; the frontend automatically reconnects with a refreshed token from `oidc-spa`.

### #4 -- KillBill Billing Integration

KillBill is integrated as a REST-to-REST proxy through the BFF:

- Per-tenant API key/secret pairs stored in K8s Secrets (`killbill-tenant-creds` in each tenant namespace)
- BFF loads credentials lazily with in-memory caching (refreshed via K8s Secret informer on change)
- Every KillBill request includes `X-Killbill-ApiKey`, `X-Killbill-ApiSecret`, and audit headers (`X-Killbill-CreatedBy`, `X-Killbill-Comment`)
- KillBill account mapped to Keycloak user via external key: `tenant:{tenant-id}:user:{sub}`
- Plan-to-quota mapping maintained as ConfigMap; webhook from KillBill triggers Cozystack Tenant CR patch to update resource quotas

### #7 -- Pipeline Orchestration

The pipeline flow uses the K8s operator pattern:

1. BFF creates a `Pipeline` CR in the tenant namespace (non-blocking, returns 202 Accepted)
2. Pipeline operator reconciles: clones GitOps repo, applies template, pushes branch, creates PR in Forgejo
3. Operator updates Pipeline CR status conditions through each phase (Running, BranchPushed, PRCreated)
4. BFF informers detect status changes, publish to in-process event bus, relay to frontend via SSE
5. Frontend shows real-time pipeline progress with PR link on completion

---

## Technology Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| API Paradigm | REST (OpenAPI 3.0) | Refine simple-rest (40k/wk), KillBill 1:1 proxy, operational simplicity |
| Backend Language | Go | client-go gold standard, existing patterns (80% transfer), small footprint |
| Backend Framework | Chi + oapi-codegen | Lightweight, net/http compatible, first-class OpenAPI codegen target |
| K8s Client | client-go / controller-runtime | Battle-tested informers, typed CRD clients, envtest for integration tests |
| Real-time | SSE (text/event-stream) | Zero Cilium Gateway issues, built-in reconnect, HTTP/2 multiplexing |
| Architecture | BFF Monolith | 2 backends, in-process event pipeline, 3x fewer resources than microservices |
| Frontend Types | orval (TanStack Query hooks from OpenAPI) | Generates typed hooks from OpenAPI spec, 730k weekly downloads |
| Auth | keyfunc/v3 + golang-jwt/v5 | JWKS auto-refresh, access token validation, Keycloak-compatible |
| KillBill Client | oapi-codegen (from KillBill OpenAPI spec) | Fresh, typed, maintainable Go client generated from KillBill's Swagger spec |
| Testing | testify + mockgen + envtest | Matches existing Go patterns, real K8s API server for integration tests |

---

## References

### Research Documents (this repo)

- [API Paradigm Comparison: REST vs GraphQL vs tRPC](api-paradigm-comparison.md)
- [Backend Language Comparison: Go vs TypeScript](backend-language-comparison.md)
- [Real-Time Protocol Comparison: WebSocket vs SSE](realtime-protocol-comparison.md)
- [Architecture Analysis: BFF Monolith vs Microservices](architecture-analysis.md)
- [Auth Flow Design](auth-flow-design.md)
- [K8s Interaction Patterns](k8s-interaction-patterns.md)
- [Sequence Diagrams](sequence-diagrams.md)

### External References

- [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) -- Go OpenAPI code generation (6k stars)
- [orval](https://github.com/orval-labs/orval) -- TypeScript client generation from OpenAPI (5.3k stars, 730k weekly downloads)
- [@refinedev/simple-rest](https://www.npmjs.com/package/@refinedev/simple-rest) -- Refine REST data provider (40k weekly downloads)
- [Cilium #40822](https://github.com/cilium/cilium/issues/40822) -- WebSocket failures through Gateway API
- [Cilium #41123](https://github.com/cilium/cilium/issues/41123) -- WebSocket auth token forwarding failure
- [@microsoft/fetch-event-source](https://www.npmjs.com/package/@microsoft/fetch-event-source) -- SSE with custom headers
- [keyfunc/v3](https://github.com/MicahParks/keyfunc) -- JWKS fetching and auto-refresh for golang-jwt
- [Chi router](https://github.com/go-chi/chi) -- Lightweight, idiomatic Go HTTP router (21.6k stars)

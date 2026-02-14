# Scoring Matrix: Backend Stack for Customer Portal BFF

**Date:** 2026-02-14
**Context:** Customer Portal BFF (Backend-for-Frontend) — sits between React SPA (Refine) and two backends (Kubernetes API + KillBill billing API)
**Research inputs:**
- `api-paradigm-comparison.md` — REST (OpenAPI) vs GraphQL vs tRPC
- `backend-language-comparison.md` — Go vs TypeScript
- `realtime-protocol-comparison.md` — WebSocket vs SSE

---

## Section A: API Paradigm (REST vs GraphQL vs tRPC)

### Hard Filters

| Filter | REST (OpenAPI) | GraphQL | tRPC |
|--------|:-:|:-:|:-:|
| Refine dataProvider exists | `@refinedev/simple-rest` (official, 40k/wk) | `@refinedev/graphql` (official, 2.6k/wk) | No viable provider (3-star abandoned repo) |
| Real-time subscriptions supported | SSE/WS + custom liveProvider | graphql-ws + built-in liveProvider | tRPC v11 SSE subscriptions (TS-only) |
| **Pass?** | **Yes** | **Yes** | **Conditional** -- eliminated if Go backend |

> **Note:** tRPC is TypeScript-only. Since Section B recommends Go, tRPC is eliminated. Scores are included for completeness but tRPC is not a viable option.

### Scoring

| # | Criterion | Weight | REST (OpenAPI) | | GraphQL | | tRPC | |
|---|-----------|--------|:-:|:-:|:-:|:-:|:-:|:-:|
| | | | Score | Weighted | Score | Weighted | Score | Weighted |
| 1 | Refine Integration | 20% | **5** | 1.00 | **3** | 0.60 | **1** | 0.20 |
| 2 | K8s Watch Compatibility | 15% | **5** | 0.75 | **4** | 0.60 | **3** | 0.45 |
| 3 | Type Safety | 15% | **4** | 0.60 | **4** | 0.60 | **5** | 0.75 |
| 4 | KillBill Wrapping | 10% | **5** | 0.50 | **3** | 0.30 | **4** | 0.40 |
| 5 | Code Generation | 10% | **5** | 0.50 | **5** | 0.50 | **5** | 0.50 |
| 6 | Ecosystem | 10% | **5** | 0.50 | **4** | 0.40 | **2** | 0.20 |
| 7 | Operational Simplicity | 10% | **5** | 0.50 | **3** | 0.30 | **4** | 0.40 |
| 8 | Real-time Support | 10% | **4** | 0.40 | **5** | 0.50 | **4** | 0.40 |
| | **Total** | **100%** | | **4.75** | | **3.80** | | **3.30** |

### Justifications

| # | Criterion | REST (OpenAPI) | GraphQL | tRPC |
|---|-----------|----------------|---------|------|
| 1 | Refine Integration | **5** -- `@refinedev/simple-rest` is the most popular Refine provider (40k weekly downloads, 15x more than GraphQL). Official, actively maintained, minimal config. | **3** -- `@refinedev/graphql` is official and recently modernized (migrated to urql), but 15x fewer users. More configuration required. | **1** -- No viable provider. Only community attempt has 3 stars, no npm package, abandoned since Jan 2023. Building custom requires 300-500 LOC of uncharted adapter code. |
| 2 | K8s Watch Compatibility | **5** -- REST + SSE is the simplest pattern: `client-go` informers push to channel, HTTP handler writes SSE frames. Clear separation between CRUD (REST) and events (SSE). Matches K8s ecosystem patterns. | **4** -- GraphQL subscriptions via `gqlgen` + `graphql-ws` provide a unified transport. Subscription schema + PubSub resolver add moving parts, but the single-endpoint model is elegant. | **3** -- tRPC v11 supports SSE subscriptions, but requires TypeScript backend, eliminating `client-go`. Must use `client-node` with its less mature informer implementation. |
| 3 | Type Safety | **4** -- Schema-first via OpenAPI spec. `oapi-codegen` (Go server) + `openapi-typescript`/`orval` (TS client). Types enforced at API boundary. Risk of spec drift mitigated by CI validation. `oapi-codegen` limited to OpenAPI 3.0. | **4** -- Schema-first via GraphQL schema. `gqlgen` (Go resolvers) + `graphql-codegen` (TS types/hooks). Slightly stronger runtime validation (GraphQL validates queries at request time). | **5** -- Best-in-class. Zero codegen, end-to-end type inference from server procedures to client calls. Refactoring propagates immediately. TypeScript-only constraint. |
| 4 | KillBill Wrapping | **5** -- Thin REST proxy: BFF REST endpoint calls KillBill REST endpoint. 1:1 mapping, minimal translation. Both sides use OpenAPI specs for codegen. | **3** -- Every KillBill resource needs GraphQL type definitions, query/mutation resolvers, and mapping logic. Translation layer that does not exist in REST. Nested billing resources (accounts > subscriptions > invoices) create substantial schema. | **4** -- tRPC procedures call KillBill REST API directly. Less boilerplate than GraphQL (no schema files), but still a translation layer. Types inferred from OpenAPI spec. TS-only. |
| 5 | Code Generation | **5** -- `oapi-codegen` v2 (6k stars, Go server stubs), `openapi-typescript` (2.1M weekly downloads), `orval` (730k weekly downloads, TanStack Query hooks). Mature, well-documented pipeline. | **5** -- `gqlgen` (10.7k stars, Go resolvers), `@graphql-codegen/typescript` (5M+ weekly downloads). Equally mature pipeline. | **5** -- No codegen needed. Types flow via TypeScript inference. This is the killer feature, but only applicable in TS-only stacks. |
| 6 | Ecosystem | **5** -- Dominant pattern in K8s ecosystem. K8s API itself is REST+OpenAPI. `openapi-typescript` 2.1M/wk. `oapi-codegen` 6k stars. Go + REST is the standard for K8s-adjacent BFFs. | **4** -- Strong general ecosystem (`gqlgen` 10.7k stars, `graphql-codegen` 5M/wk). But GraphQL BFFs in front of K8s are uncommon; Backstage uses REST, not GraphQL, for K8s interaction. | **2** -- Large general ecosystem (38k stars), but zero K8s production presence. No community experience with tRPC + K8s informer patterns. |
| 7 | Operational Simplicity | **5** -- Standard HTTP. Debug with curl/browser devtools. HTTP caching (ETags, Cache-Control) works natively. Standard status codes. APM tools integrate without configuration. Swagger UI auto-generated. | **3** -- Single `/graphql` endpoint. All requests return HTTP 200 (errors in response body). Harder caching (all POST requests). Requires GraphQL-aware APM instrumentation and GraphiQL playground. | **4** -- Standard HTTP for queries (GET). Less tooling overhead than GraphQL. No built-in documentation generator; TS types are the docs. Only viable in all-TS environments. |
| 8 | Real-time Support | **4** -- Requires custom Refine `liveProvider` (~100 LOC for SSE). Clear separation of concerns (REST for CRUD, SSE for events). No built-in subscription protocol; relies on URL-based stream selection. | **5** -- Subscriptions are first-class in GraphQL schema. `@refinedev/graphql` includes built-in `liveProvider` via `graphql-ws`. Most turnkey real-time integration. Single unified transport. | **4** -- tRPC v11 SSE subscriptions are simple to set up. Custom Refine liveProvider still needed. TS-only. |

### Ranking

| Rank | Paradigm | Weighted Score |
|------|----------|:-:|
| 1 | **REST (OpenAPI)** | **4.75** |
| 2 | GraphQL | 3.80 |
| 3 | tRPC | 3.30 (eliminated -- Go backend) |

---

## Section B: Backend Language (Go vs TypeScript)

### Hard Filter

| Filter | Go | TypeScript |
|--------|:-:|:-:|
| Production-grade K8s informer/watch support | `client-go` SharedInformerFactory, battle-tested in every K8s controller | `client-node` has basic informer with known stability issues (#379), no SharedInformerFactory |
| **Pass?** | **Yes** | **Marginal** |

### Scoring

| # | Criterion | Weight | Go | | TypeScript | |
|---|-----------|--------|:-:|:-:|:-:|:-:|
| | | | Score | Weighted | Score | Weighted |
| 1 | K8s Client Maturity | 25% | **5** | 1.25 | **2** | 0.50 |
| 2 | Existing Team Patterns | 20% | **5** | 1.00 | **2** | 0.40 |
| 3 | Real-time Server Support | 15% | **5** | 0.75 | **3** | 0.45 |
| 4 | KillBill Client | 10% | **3** | 0.30 | **3** | 0.30 |
| 5 | Container Footprint | 10% | **5** | 0.50 | **2** | 0.20 |
| 6 | Framework Quality | 10% | **4** | 0.40 | **5** | 0.50 |
| 7 | End-to-End Type Safety | 10% | **3** | 0.30 | **5** | 0.50 |
| | **Total** | **100%** | | **4.50** | | **2.85** |

### Justifications

| # | Criterion | Go | TypeScript |
|---|-----------|-----|-----------|
| 1 | K8s Client Maturity | **5** -- `client-go` is the gold standard. SharedInformerFactory with DeltaFIFO queue, automatic reconnect and resync. Typed client codegen for CRDs. Server-Side Apply first-class. Leader election built-in. Every K8s controller in existence uses it. | **2** -- `client-node` (1.2M downloads) has basic CRUD but informer has known stability issues (event duplication, #379). No SharedInformerFactory. No SSA support (third-party only). No leader election. No CRD codegen (requires 25-star third-party tool). |
| 2 | Existing Team Patterns | **5** -- `platform-v1-hetzner-operators` establishes: interface-based API clients with `mockgen`, three-layer rate limiting as HTTP transport middleware, TransientError/TerminalError classification, stateless services with param objects. ~80% of patterns transfer directly to BFF. | **2** -- ~30% conceptual transfer. Same design concepts (rate limiting, error classification) but every implementation must be rewritten in TypeScript idioms. Net-new code for all established patterns. |
| 3 | Real-time Server Support | **5** -- Goroutines make informer-to-SSE relay trivial: one goroutine per informer, channels for fan-out, natural backpressure. SSE via `http.Flusher` needs no external library. WebSocket via `coder/websocket` (modern, context-aware). 50 informers cost ~100KB. | **3** -- Node.js event loop handles informers in a single thread. I/O-bound work is fine but CPU-bound blocks everything. `client-node` informer has known reconnect issues. `ws` library is mature for WebSocket. Backpressure must be implemented manually. |
| 4 | KillBill Client | **3** -- `kbcli` exists (25 stars) but last updated Jan 2024. Usable as-is or regenerate from KillBill's OpenAPI spec via `oapi-codegen`. Codegen path is mature. | **3** -- `killbill-client-js` exists (31 stars) but last npm publish ~3 years ago. Regeneration via `openapi-typescript` or `orval` has slightly better DX (generates React Query hooks). Neither SDK is well-maintained; both need codegen. Tie. |
| 5 | Container Footprint | **5** -- Static binary on `distroless/static`: ~10-30 MB image, ~20-50 MB idle memory, <100ms startup. Minimal attack surface (no runtime, no shell). Fits 128 MB resource limit per tenant. | **2** -- `node:22-alpine`: ~150-200 MB image, ~80-150 MB idle memory, ~500ms-1s startup. In multi-tenant Cozystack (one BFF per tenant), this is 3-5x higher infrastructure cost. |
| 6 | Framework Quality | **4** -- Chi (21.6k stars) and Echo (32.2k stars) are mature, `net/http` compatible, with first-class `oapi-codegen` targets. Go 1.22+ stdlib routing is viable. Avoid Fiber (fasthttp, incompatible with client-go). | **5** -- tRPC (39.5k stars) provides end-to-end type safety with zero codegen, a uniquely compelling advantage. Hono (28.8k stars) is ultra-light and multi-runtime. NestJS (74.5k stars) is full-featured. Richer framework ecosystem overall. |
| 7 | End-to-End Type Safety | **3** -- Schema-first with codegen: OpenAPI spec -> `oapi-codegen` (Go) + `orval` (TS). Types enforced at API boundary but require a codegen step. Spec drift risk mitigated by CI. Achieves ~90% of tRPC's type safety. | **5** -- tRPC provides zero-codegen, end-to-end type inference. Rename a field and the IDE shows every affected call site across frontend and backend. Genuinely transformative for small teams. |

### Ranking

| Rank | Language | Weighted Score |
|------|----------|:-:|
| 1 | **Go** | **4.50** |
| 2 | TypeScript | 2.85 |

---

## Section C: Real-time Protocol (WebSocket vs SSE)

### Hard Filter

| Filter | WebSocket | SSE |
|--------|:-:|:-:|
| Works through Cilium Gateway API | Supported via `appProtocol: kubernetes.io/ws` (Cilium 1.16+), but **active open bugs** (#40822 intermittent close code 1006, #41123 auth token forwarding failure) | Standard HTTP, zero configuration, no known issues |
| **Pass?** | **Yes (with risk)** | **Yes** |

### Scoring

| # | Criterion | Weight | WebSocket | | SSE | |
|---|-----------|--------|:-:|:-:|:-:|:-:|
| | | | Score | Weighted | Score | Weighted |
| 1 | Cilium Gateway Compatibility | 25% | **2** | 0.50 | **5** | 1.25 |
| 2 | Refine liveProvider Integration | 20% | **3** | 0.60 | **4** | 0.80 |
| 3 | Operational Simplicity | 20% | **2** | 0.40 | **5** | 1.00 |
| 4 | K8s Watch Relay | 15% | **3** | 0.45 | **5** | 0.75 |
| 5 | Scalability | 10% | **3** | 0.30 | **4** | 0.40 |
| 6 | HTTP/2 Support | 10% | **2** | 0.20 | **5** | 0.50 |
| | **Total** | **100%** | | **2.45** | | **4.70** |

### Justifications

| # | Criterion | WebSocket | SSE |
|---|-----------|-----------|-----|
| 1 | Cilium Gateway Compatibility | **2** -- Supported via `appProtocol` annotation (Cilium 1.16+), but two open bugs as of 2026-02 cause real-world failures: #40822 (intermittent close code 1006 through Gateway API, works fine via NodePort) and #41123 (WebSocket connections always fail, auth tokens not forwarded). Multiple independent apps report failures. Significant deployment risk. | **5** -- Standard HTTP with `Content-Type: text/event-stream`. No protocol upgrade, no `appProtocol`, no special Envoy filter needed. Indistinguishable from normal HTTP response stream. Zero compatibility risk. Only concern is Envoy's 5-min `stream_idle_timeout`, trivially mitigated by keepalive comments every 15s. |
| 2 | Refine liveProvider Integration | **3** -- No built-in Refine adapter for custom WebSocket. All built-in adapters (Ably, Supabase, Hasura) use WebSocket, but none match a custom BFF protocol. Custom implementation ~60 lines. Requires manual reconnection logic and custom subscription message protocol. | **4** -- No built-in SSE adapter either, but custom implementation is ~40 lines using `@microsoft/fetch-event-source`. Simpler because the library handles reconnection automatically. Auth headers work like standard fetch. Marginally less code and complexity. |
| 3 | Operational Simplicity | **2** -- Manual reconnection with exponential backoff. Lost subscriptions on disconnect require client to re-send subscribe messages. Server needs concurrent read/write goroutines, ping/pong heartbeat, close handshake, and subscription state tracking. ~80 lines of Go server code with external dependency (`coder/websocket`). Custom JSON subscribe/unsubscribe protocol must be designed and documented. | **5** -- Auto-reconnect via EventSource API. `Last-Event-ID` header enables seamless resume without re-subscription. Server is ~50 lines of stdlib Go (no external deps). Subscription encoded in URL params (stateless). Keepalive comments are standard practice. Debug with curl. Standard HTTP monitoring and caching. |
| 4 | K8s Watch Relay | **3** -- Bidirectional subscribe/unsubscribe over single connection. Server maintains per-connection subscription state. Flexible for dynamic subscriptions, but the Customer Portal data flow is inherently unidirectional (server -> client). Bidirectionality is unnecessary overhead. | **5** -- URL-param subscriptions match the unidirectional data flow perfectly. Navigate to tenant detail -> open SSE stream for that resource. Navigate away -> close stream. Multiple streams coexist over HTTP/2. `Last-Event-ID` with K8s `resourceVersion` enables event replay on reconnect. |
| 5 | Scalability | **3** -- Pod restart requires reconnect + full re-subscribe. All subscription state is lost. Graceful shutdown needs close frame handshake and drain handling during rolling updates. | **4** -- Pod restart triggers automatic EventSource reconnect. `Last-Event-ID` enables resume from any pod (each pod runs independent informers). No special drain logic needed; server closes response writer, client auto-reconnects to another pod. |
| 6 | HTTP/2 Support | **2** -- WebSocket over HTTP/2 defined in RFC 8441 but Envoy has an open issue (#37020): when backend doesn't support `SETTINGS_ENABLE_CONNECT_PROTOCOL`, Envoy fails instead of falling back to HTTP/1.1. Browser support inconsistent. In practice, WebSocket falls back to HTTP/1.1 (one TCP connection per WebSocket). | **5** -- SSE over HTTP/2 multiplexes streams on a single TCP connection. HTTP/1.1's 6-connection-per-origin limit is eliminated. A dashboard with 5 resource watches uses 5 HTTP/2 streams on 1 TCP connection. Most efficient configuration for concurrent server-push streams. |

### Ranking

| Rank | Protocol | Weighted Score |
|------|----------|:-:|
| 1 | **SSE** | **4.70** |
| 2 | WebSocket | 2.45 |

---

## Section D: Sensitivity Analysis

### Section A: API Paradigm -- Weight Swap Test

**Base weights:** Refine Integration (20%) is highest, Real-time Support (10%) is lowest (tied).

**Swap:** Refine Integration (10%) <-> Real-time Support (20%)

| Paradigm | Base Score | Swapped Score | Delta |
|----------|:-:|:-:|:-:|
| REST (OpenAPI) | 4.75 | 4.65 | -0.10 |
| GraphQL | 3.80 | 3.90 | +0.10 |
| tRPC | 3.30 | 3.30 | 0.00 |

**Result:** Ranking unchanged. REST still leads by 0.75 points. GraphQL gains slightly because its built-in Refine `liveProvider` for subscriptions scores higher in Real-time Support (5 vs REST's 4). However, the gap remains large.

**Swing factor test:** What weight on Real-time Support would flip the ranking?

REST score = GraphQL score when:
- Solving for the crossover: REST would need Real-time Support weighted at ~105% (impossible) for GraphQL to overtake, because REST leads in 6 of 8 criteria. **No realistic weight shift changes the ranking.**

### Section B: Backend Language -- Weight Swap Test

**Base weights:** K8s Client Maturity (25%) is highest, KillBill Client / Container Footprint / Framework Quality / Type Safety (10% each) are lowest.

**Swap:** K8s Client Maturity (10%) <-> End-to-End Type Safety (25%)

| Language | Base Score | Swapped Score | Delta |
|----------|:-:|:-:|:-:|
| Go | 4.50 | 3.80 | -0.70 |
| TypeScript | 2.85 | 3.55 | +0.70 |

**Result:** Ranking unchanged, but gap narrows significantly (from 1.65 to 0.25). End-to-End Type Safety is a **swing factor** -- if a team valued type safety above K8s client maturity, the decision would be close.

**Why this does not apply here:** The BFF's primary job is relaying K8s CRD state. `client-go`'s informer maturity is not a preference -- it is a functional requirement for reliable real-time updates. The TypeScript informer's known stability issues (#379) make this a reliability concern, not just a DX trade-off.

### Section C: Real-time Protocol -- Weight Swap Test

**Base weights:** Cilium Gateway Compatibility (25%) is highest, Scalability / HTTP/2 Support (10% each) are lowest.

**Swap:** Cilium Gateway Compatibility (10%) <-> HTTP/2 Support (25%)

| Protocol | Base Score | Swapped Score | Delta |
|----------|:-:|:-:|:-:|
| SSE | 4.70 | 4.70 | 0.00 |
| WebSocket | 2.45 | 2.45 | 0.00 |

**Result:** Ranking unchanged. Scores are identical because SSE scores 5 and WebSocket scores 2 in both swapped criteria. SSE dominates across all criteria (scores 4-5 in every category vs WebSocket's 2-3). **No weight adjustment changes the outcome.**

### Sensitivity Summary

| Section | Swing Factor | Does ranking change? |
|---------|-------------|:-:|
| A: API Paradigm | Real-time Support (GraphQL's advantage) | No -- REST leads in too many criteria |
| B: Backend Language | **End-to-End Type Safety** | No, but gap narrows to 0.25 if type safety weighted at 25% |
| C: Real-time Protocol | None | No -- SSE dominates every criterion |

**Key finding:** The only scenario where a different choice becomes competitive is if End-to-End Type Safety is weighted at 25% for the backend language decision. Even then, Go still wins, and the known `client-node` informer stability issues make this a moot point for a K8s-heavy BFF.

---

## Section E: Combined Recommendation

### Recommended Stack: Go + REST (OpenAPI) + SSE

| Decision | Choice | Weighted Score | Runner-up | Gap |
|----------|--------|:-:|-----------|:-:|
| API Paradigm | **REST (OpenAPI)** | 4.75 | GraphQL (3.80) | +0.95 |
| Backend Language | **Go** | 4.50 | TypeScript (2.85) | +1.65 |
| Real-time Protocol | **SSE** | 4.70 | WebSocket (2.45) | +2.25 |

### Why These Three Choices Reinforce Each Other

The three decisions are not independent -- they form a mutually reinforcing stack:

1. **Go enables REST, REST rewards Go.** Go is the natural language for K8s-adjacent services (`client-go`, `controller-runtime`). The K8s API itself is REST+OpenAPI. Choosing Go means the BFF speaks the same protocol (REST) and uses the same client libraries as the platform it sits on top of. `oapi-codegen` generates Go server code from an OpenAPI spec, and `orval` generates TypeScript client hooks -- creating a type-safe boundary with minimal friction.

2. **Go eliminates tRPC, which eliminates GraphQL's competitor.** With Go chosen, tRPC (TypeScript-only) is eliminated. This makes the API paradigm decision REST vs GraphQL only. REST wins that comparison decisively for a 2-backend BFF where GraphQL's query aggregation advantage is minimal.

3. **SSE complements REST and Go naturally.** SSE is standard HTTP -- it needs no protocol upgrade, no special proxy configuration, and no external Go dependencies. The Go BFF exposes REST endpoints for CRUD and an SSE endpoint for real-time events. This clean separation (REST for data, SSE for events) maps directly to Refine's `dataProvider` + `liveProvider` architecture. Go's goroutines and channels make the informer-to-SSE relay trivial.

4. **SSE avoids Cilium Gateway risk.** The platform runs on Cilium Gateway API. WebSocket has active open bugs causing connection failures through the gateway. SSE is standard HTTP and has zero compatibility risk. Choosing SSE means the real-time layer does not introduce infrastructure instability.

5. **HTTP/2 multiplexing makes SSE efficient at scale.** Multiple SSE streams (one per watched resource) multiplex over a single TCP connection via HTTP/2. WebSocket falls back to HTTP/1.1 in practice. This means SSE + HTTP/2 is more connection-efficient than WebSocket for a dashboard watching multiple resources simultaneously.

### Recommended Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| BFF Language | Go | K8s client maturity, existing team patterns, container footprint |
| BFF Framework | Chi or Echo + `oapi-codegen` | Lightweight, `net/http` compatible, first-class OpenAPI codegen |
| API Contract | OpenAPI 3.0 (spec-first) | `oapi-codegen` (Go server) + `orval` (TS client hooks) |
| K8s Client | `client-go` / `controller-runtime` | Battle-tested informers, typed CRD clients, SSA |
| KillBill Client | Generated via `oapi-codegen` from KillBill OpenAPI spec | Fresh, typed, maintainable |
| Refine dataProvider | `@refinedev/simple-rest` or custom provider using orval-generated client | 40k weekly downloads, official, minimal integration effort |
| Real-time Transport | SSE via `http.Flusher` | Standard HTTP, auto-reconnect, `Last-Event-ID` resume, no external deps |
| Refine liveProvider | Custom SSE adapter (~40 LOC) using `@microsoft/fetch-event-source` | Auth headers, automatic reconnection, protocol-agnostic |
| Event ID Strategy | `{resource}/{namespace}/{name}@{resourceVersion}` | Enables cross-pod resume via K8s resourceVersion |

### What This Stack Sacrifices

| Sacrifice | Impact | Mitigation |
|-----------|--------|------------|
| tRPC end-to-end type inference | Must maintain OpenAPI spec + run codegen | Spec-first development, CI validation, `oapi-codegen` + `orval` achieve ~90% of tRPC's safety |
| GraphQL's unified subscription transport | Custom `liveProvider` needed (~40 LOC) instead of built-in | SSE adapter is simpler than the GraphQL subscription pipeline it replaces |
| Full-stack JavaScript consistency | Team works in two languages (Go + TypeScript) | Go is already the team's primary backend language (hetzner-operators) |
| WebSocket bidirectional communication | Cannot send client messages over the event stream | Data flow is unidirectional (server -> client); bidirectionality is not needed |

---

## Sources

All scores are derived from the following research documents:

- `api-paradigm-comparison.md` (2026-02-13) -- REST vs GraphQL vs tRPC evaluation
- `backend-language-comparison.md` (2026-02-13) -- Go vs TypeScript evaluation
- `realtime-protocol-comparison.md` (2026-02-14) -- WebSocket vs SSE evaluation

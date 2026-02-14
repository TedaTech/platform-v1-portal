# API Paradigm Comparison: REST (OpenAPI) vs GraphQL vs tRPC

**Date:** 2026-02-13
**Context:** Customer Portal BFF (Backend-for-Frontend) for TedaTech platform-v1
**Author:** PM/Scrum Master Agent

---

## Background

The Customer Portal is a React SPA built with:
- **Frontend:** React + Vite + TanStack Router + Refine (headless) + shadcn/ui
- **Auth:** oidc-spa (Keycloak OIDC)
- **Backend consumers:** Refine `dataProvider` (CRUD) + `liveProvider` (real-time subscriptions)

The BFF talks to exactly **two backends**:
1. **Kubernetes API** -- all infrastructure (tenants, Forgejo, Cozystack) abstracted behind K8s CRDs
2. **KillBill API** -- billing, subscriptions, invoices (REST/JSON)

This document evaluates three API paradigms for the BFF layer.

---

## 1. Refine dataProvider Integration

### REST (OpenAPI)

| Metric | Value |
|--------|-------|
| Official provider | `@refinedev/simple-rest` |
| Weekly downloads | ~40,761 |
| Latest version | 6.0.1 (Dec 2025) |
| Status | Actively maintained, part of Refine monorepo |

`@refinedev/simple-rest` is Refine's most popular data provider, receiving ~15x more downloads than the GraphQL provider. It expects a simple REST convention (`GET /resources`, `GET /resources/:id`, `POST /resources`, etc.) and maps directly to Refine's CRUD methods. For OpenAPI-generated APIs, this works out of the box if the API follows standard REST conventions.

Additional REST providers exist for NestJS (`@refinedev/nestjsx-crud`) and other frameworks, but `simple-rest` covers the generic case. A custom REST provider can also be built by implementing 6 required methods (`getList`, `getOne`, `create`, `update`, `deleteOne`, `getApiUrl`) and up to 4 optional batch methods.

**Verdict:** First-class support. Minimal integration effort.

### GraphQL

| Metric | Value |
|--------|-------|
| Official provider | `@refinedev/graphql` |
| Weekly downloads | ~2,617 |
| Latest version | 8.0.1 (Oct 2025) |
| GraphQL client | `@urql/core` (migrated from graphql-request) |
| Status | Actively maintained, part of Refine monorepo |

`@refinedev/graphql` was recently modernized to use `@urql/core` as its GraphQL client (replacing `gql-query-builder` + `graphql-request`). It supports flexible schema conventions and custom query builders. Refine also offers `@refinedev/hasura` and `@refinedev/strapi-v4` for specific GraphQL backends.

The provider also supports a `liveProvider` implementation via `graphql-ws` for WebSocket-based subscriptions, making it the only official provider with built-in real-time support.

**Verdict:** Official support exists, but 15x fewer users than REST. More configuration required.

### tRPC

| Metric | Value |
|--------|-------|
| Official provider | None |
| Community provider | [refine-trpc-provider](https://github.com/yjavaherian/refine-trpc-provider) |
| Stars | 3 |
| Last commit | January 2023 (abandoned) |
| npm package | None published |

There is no official Refine tRPC provider. The only community attempt (`yjavaherian/refine-trpc-provider`) has 3 GitHub stars, no npm package, and has not been updated since January 2023. Building a custom tRPC data provider would require implementing all 6+ Refine `dataProvider` methods plus a `liveProvider`, roughly 300-500 lines of adapter code. While feasible, this is uncharted territory with no community support.

**Verdict:** No viable provider. Significant custom work required with zero ecosystem backing.

---

## 2. K8s Watch Compatibility

The BFF must relay Kubernetes informer/watch events to the frontend so users see real-time CRD status changes (e.g., tenant provisioning progress, Forgejo instance readiness).

### REST + SSE/WebSocket

**Pattern:** Go BFF uses `client-go` informers to watch CRDs. On change, it pushes events over a Server-Sent Events (SSE) or WebSocket connection to the frontend.

- **Architecture:** REST endpoints for CRUD + a separate `/watch` or `/events` SSE endpoint per resource type.
- **Refine integration:** Requires a custom `liveProvider` that connects to the SSE/WebSocket endpoint. Refine is transport-agnostic for live providers, supporting Ably, Socket.IO, Mercure, Supabase, or custom implementations.
- **Go ecosystem:** `client-go` informers are the standard K8s watch mechanism. Relaying to SSE is straightforward: informer callback -> channel -> SSE writer. No additional libraries needed beyond `net/http`.
- **Complexity:** Low. Two separate concerns (REST for CRUD, SSE for events) with clear boundaries.

### GraphQL Subscriptions

**Pattern:** Go BFF uses `gqlgen` to expose a GraphQL API. K8s informer events are published to a `gqlgen` subscription resolver via a PubSub mechanism, transported over `graphql-ws` (WebSocket).

- **Architecture:** Single GraphQL endpoint. Subscriptions are first-class in the schema (`subscription { tenantStatus(id: ID!) { phase, message } }`).
- **Refine integration:** `@refinedev/graphql` includes a `liveProvider` based on `graphql-ws`. This is the most turnkey real-time integration of the three paradigms.
- **Go ecosystem:** `gqlgen` supports subscriptions natively. The informer -> PubSub -> subscription resolver pipeline is well-documented.
- **Complexity:** Medium. The subscription schema, resolver, and PubSub layer add moving parts, but the unified transport (single WebSocket) is elegant.

### tRPC Subscriptions

**Pattern:** tRPC v11 supports subscriptions via SSE (recommended) or WebSocket. The server defines subscription procedures that yield events from K8s informers.

- **Architecture:** tRPC procedures for CRUD + subscription procedures for watches. Single endpoint.
- **Refine integration:** No liveProvider exists. Must build custom.
- **TypeScript-only:** tRPC is TypeScript-only. A Go backend cannot use tRPC. This eliminates tRPC for a Go BFF entirely.
- **If TypeScript backend:** tRPC v11's SSE subscriptions are simple to set up and pair naturally with K8s client libraries like `@kubernetes/client-node`.

**Verdict:**
- **Most natural for K8s watches:** REST + SSE -- simplest, clearest separation of concerns, works perfectly with Go `client-go`.
- **Most unified:** GraphQL subscriptions -- single transport, schema-defined real-time contract.
- **Most restricted:** tRPC -- only if TypeScript backend; no Go path.

---

## 3. Type Safety

### REST + OpenAPI

| Layer | Tool | Mechanism |
|-------|------|-----------|
| Go server | `oapi-codegen` (6k+ stars) | Generate server stubs + types from OpenAPI spec |
| TS client | `openapi-typescript` (2.1M weekly downloads) | Generate TS types from OpenAPI spec |
| TS client + hooks | `orval` (730k weekly downloads, 5.3k stars) | Generate TanStack Query hooks from OpenAPI spec |

**Developer experience:** Schema-first. Write the OpenAPI spec, generate Go server code and TS client types. Types are enforced at the API boundary but not end-to-end -- a runtime mismatch between spec and implementation is possible (though CI validation mitigates this). The `oapi-codegen` + `openapi-typescript`/`orval` pipeline is mature and widely adopted.

**Caveat:** `oapi-codegen` does not yet support OpenAPI 3.1 (3.0 only).

### GraphQL

| Layer | Tool | Mechanism |
|-------|------|-----------|
| Go server | `gqlgen` (10.7k stars) | Schema-first, generates Go types + resolver interfaces |
| TS client | `@graphql-codegen/cli` (5M+ weekly downloads for TS plugin) | Generate TS types + hooks from schema + operations |

**Developer experience:** Schema-first. The GraphQL schema is the single source of truth. `gqlgen` generates Go resolver interfaces, `graphql-codegen` generates TS types and React hooks. Build-time validation ensures schema consistency. Slightly stronger guarantees than REST+OpenAPI because the GraphQL runtime also validates queries at request time.

### tRPC

| Layer | Tool | Mechanism |
|-------|------|-----------|
| TS server + client | tRPC v11 (38k stars, 700k+ weekly downloads) | End-to-end type inference, zero codegen |

**Developer experience:** Best-in-class if (and only if) both server and client are TypeScript. Types flow directly from server procedure definitions to client calls via TypeScript inference. No codegen step, no schema files, no drift. Refactoring a procedure signature instantly propagates type errors to all call sites.

**Fatal constraint:** tRPC is TypeScript-only. A Go backend cannot use tRPC.

**Verdict:**
- **Best DX (TS-only stack):** tRPC -- zero codegen, end-to-end inference.
- **Best DX (Go backend):** Tie between REST+OpenAPI and GraphQL. Both require codegen. GraphQL has slightly stronger runtime validation.
- **Most mature tooling:** REST+OpenAPI (2.1M weekly downloads for `openapi-typescript` alone).

---

## 4. KillBill Wrapping

KillBill exposes a REST/JSON API with 250+ endpoints. The BFF must proxy/wrap a subset of these for the portal.

### REST BFF

**Effort: Low.**

The BFF exposes its own REST API. KillBill endpoints are wrapped as thin proxies: the BFF receives a request, calls KillBill's REST API, transforms the response if needed, and returns it. The mapping is 1:1 or near-1:1. With `oapi-codegen`, both the BFF's API and the KillBill client can be generated from OpenAPI specs. KillBill provides OpenAPI/Swagger documentation for its API.

```
Frontend -> BFF REST endpoint -> KillBill REST endpoint
```

### GraphQL BFF

**Effort: Medium.**

Every KillBill resource exposed through the portal needs a GraphQL type definition, query/mutation resolvers, and mapping logic between GraphQL types and KillBill's REST response shapes. This is a translation layer that does not exist in the REST approach.

```
Frontend -> GraphQL query -> Resolver -> KillBill REST endpoint -> Map to GraphQL types
```

The upside is that the frontend can request exactly the fields it needs (no over-fetching). The downside is that for a billing API with complex nested resources (accounts -> subscriptions -> invoices -> payments), the resolver graph and type definitions become substantial.

### tRPC BFF

**Effort: Medium-Low (if TypeScript backend).**

tRPC procedures call KillBill's REST API. Less boilerplate than GraphQL (no schema definitions, no resolver plumbing), but still a translation layer. Types can be inferred from KillBill's OpenAPI spec via `openapi-typescript` and shared with tRPC procedures.

**Not applicable if Go backend.**

### Verdict

With only 2 backends, the translation layer overhead of GraphQL is real but does not pay for itself. GraphQL's query aggregation advantage (combining data from multiple backends in one query) is minimal when there are only 2 sources. REST's 1:1 proxy approach is the simplest path for KillBill wrapping.

---

## 5. Code Generation

### REST (OpenAPI)

| Tool | Language | Purpose | Maturity |
|------|----------|---------|----------|
| `oapi-codegen` v2 | Go | Server stubs, types, client | 6k stars, actively maintained |
| `openapi-typescript` | TypeScript | Type definitions from spec | 2.1M weekly downloads |
| `orval` | TypeScript | TanStack Query hooks + types | 730k weekly downloads, 5.3k stars |

**Pipeline:** Write OpenAPI spec -> Generate Go server with `oapi-codegen` -> Generate TS types/hooks with `orval` or `openapi-typescript`.

This pipeline is well-established, and all tools are actively maintained. The OpenAPI spec acts as a shared contract between backend and frontend teams.

### GraphQL

| Tool | Language | Purpose | Maturity |
|------|----------|---------|----------|
| `gqlgen` | Go | Server (schema-first) | 10.7k stars, actively maintained |
| `@graphql-codegen/typescript` | TypeScript | Types + React hooks | 5M+ weekly downloads |

**Pipeline:** Write GraphQL schema -> Generate Go resolvers with `gqlgen` -> Generate TS types/hooks with `graphql-codegen`.

Also well-established. `gqlgen` is the dominant Go GraphQL library. `graphql-codegen` is the standard TS codegen tool.

### tRPC

| Tool | Language | Purpose | Maturity |
|------|----------|---------|----------|
| tRPC v11 | TypeScript | No codegen needed | 38k stars, 700k+ weekly downloads |

**Pipeline:** Define procedures in TypeScript -> Types flow automatically. No codegen step.

This is tRPC's killer feature, but it only works for TypeScript-on-both-ends stacks.

---

## 6. Ecosystem and Community

### Refine Integration Health

| Provider | Weekly Downloads | GitHub Stars (Refine mono) | Last Update | Maintainer |
|----------|-----------------|---------------------------|-------------|------------|
| `@refinedev/simple-rest` | ~40,761 | 34,061 (shared) | Dec 2025 | Refine core team |
| `@refinedev/graphql` | ~2,617 | 34,061 (shared) | Oct 2025 | Refine core team |
| tRPC (community) | N/A | 3 | Jan 2023 | Abandoned |

### Broader Ecosystem

| Paradigm | Key Tool | GitHub Stars | Weekly Downloads |
|----------|----------|-------------|-----------------|
| REST/OpenAPI | `openapi-typescript` | ~5k | 2,137,029 |
| REST/OpenAPI | `orval` | 5,320 | 730,114 |
| REST/OpenAPI | `oapi-codegen` | ~6k | N/A (Go module) |
| GraphQL | `gqlgen` | 10,700 | N/A (Go module) |
| GraphQL | `graphql-codegen` (TS plugin) | ~10k | 5,000,000+ |
| GraphQL | `graphql-ws` | ~3k | 6,089,415 |
| tRPC | `@trpc/server` | 38,269 | 1,238,212 |

### K8s Backend Production Usage

Go + REST/OpenAPI is the dominant pattern for Kubernetes-adjacent BFFs. The Kubernetes ecosystem itself is REST-based (the K8s API server is a REST API with OpenAPI specs). Tools like `client-go`, `controller-runtime`, and the entire operator ecosystem assume REST. GraphQL BFFs in front of K8s are uncommon outside of platforms like Backstage (which uses its own REST-based plugin system, not GraphQL for K8s). tRPC + K8s is essentially nonexistent in production.

---

## 7. Operational Simplicity

### REST (OpenAPI)

- **Debugging:** Standard HTTP. Debug with `curl`, browser devtools, Postman. Every endpoint has a clear URL.
- **Caching:** HTTP caching (ETags, `Cache-Control`, conditional requests) works out of the box. CDN-friendly.
- **Monitoring:** Standard HTTP status codes. APM tools (Prometheus, OpenTelemetry) integrate natively.
- **Documentation:** OpenAPI spec renders interactive docs (Swagger UI, Redoc) automatically.
- **Error handling:** HTTP status codes are universally understood (404, 422, 500).

### GraphQL

- **Debugging:** Single endpoint (`/graphql`). Requires a playground (GraphiQL) for exploration. Harder to debug with curl (POST body with query string).
- **Caching:** HTTP caching is harder because all requests go to `POST /graphql`. Requires application-level caching (Apollo Client, urql) or CDN workarounds (persisted queries). Cloudflare and Akamai have improved GraphQL caching, but it remains non-trivial.
- **Monitoring:** All requests return HTTP 200 (errors are in the response body). APM tools need GraphQL-aware instrumentation.
- **Documentation:** Schema introspection provides self-documenting API. GraphiQL is excellent for exploration.
- **Error handling:** Non-standard. Partial success is possible (some fields resolve, others error). Requires careful error design.

### tRPC

- **Debugging:** Requires TypeScript tooling. Raw HTTP requests are possible but awkward (tRPC uses its own URL conventions). Browser devtools work for inspection.
- **Caching:** HTTP caching works for queries (GET requests in tRPC v11). Mutations use POST.
- **Monitoring:** Standard HTTP. Less tooling overhead than GraphQL.
- **Documentation:** No built-in documentation generator. The TypeScript types *are* the documentation.
- **Error handling:** Typed errors via tRPC's error handling. Good DX but TS-specific.

### Verdict

REST is the clear winner for operational simplicity. Standard HTTP tooling, caching, and monitoring work without modification. GraphQL introduces operational complexity that must be justified by significant DX or feature benefits. tRPC is operationally simple but only in an all-TS environment.

---

## 8. Constraint: Go vs TypeScript Backend

The backend language choice has a decisive impact on paradigm selection.

### If Go Backend

| Paradigm | Viable? | Notes |
|----------|---------|-------|
| REST (OpenAPI) | **Yes** | `oapi-codegen` generates Go server code. `client-go` for K8s. Native HTTP for KillBill. Dominant pattern in K8s ecosystem. |
| GraphQL | **Yes** | `gqlgen` generates Go resolvers. Subscription support included. More complex than REST for this use case. |
| tRPC | **No** | tRPC is TypeScript-only. Eliminated entirely. |

Go is the natural language for K8s-adjacent services. `client-go`, `controller-runtime`, and the entire CNCF toolchain are Go-native. The BFF can share types and libraries with existing K8s operators in the TedaTech monorepo (`platform-v1-hetzner-operators`).

### If TypeScript Backend

| Paradigm | Viable? | Notes |
|----------|---------|-------|
| REST (OpenAPI) | **Yes** | Express/Fastify + `openapi-typescript` for types. `@kubernetes/client-node` for K8s. |
| GraphQL | **Yes** | `pothos` or `type-graphql` for schema. Full subscription support. |
| tRPC | **Yes** | End-to-end type safety. Best DX. tRPC v11 SSE subscriptions. |

TypeScript enables tRPC but requires `@kubernetes/client-node` (less mature than `client-go`) for K8s interaction. The K8s client library for Node.js is functional but has fewer contributors and less production mileage than Go's `client-go`.

### Recommendation

Given that TedaTech's existing operators are written in Go and the BFF interacts heavily with the Kubernetes API, **Go is the more natural backend language**. This eliminates tRPC and narrows the choice to REST (OpenAPI) vs GraphQL.

---

## 9. Recommendation Table

| Criterion | REST (OpenAPI) | GraphQL | tRPC |
|-----------|---------------|---------|------|
| **Refine Provider** | `@refinedev/simple-rest` -- 40k/wk, official, battle-tested | `@refinedev/graphql` -- 2.6k/wk, official, recently modernized | No viable provider. 3-star abandoned repo. |
| **K8s Watch Relay** | REST + SSE. Simple, clear separation. Custom `liveProvider` needed (~100 LOC). | GraphQL subscriptions via `graphql-ws`. Built-in Refine `liveProvider`. Most unified. | tRPC v11 SSE subscriptions. Good, but TS-only. |
| **Type Safety** | `oapi-codegen` (Go) + `openapi-typescript`/`orval` (TS). Boundary types with codegen. | `gqlgen` (Go) + `graphql-codegen` (TS). Schema-validated. Slightly stronger runtime guarantees. | End-to-end inference. Best DX. Zero codegen. **TS-only.** |
| **KillBill Wrapping** | Thin proxy. 1:1 mapping. Minimal effort. | Schema + resolvers + type mapping. Medium effort. | Procedures calling REST. Medium-low effort. **TS-only.** |
| **Code Generation** | Mature pipeline: `oapi-codegen` + `orval`. Well-documented. | Mature pipeline: `gqlgen` + `graphql-codegen`. Well-documented. | No codegen needed. |
| **Ecosystem** | Dominant in K8s ecosystem. `openapi-typescript` 2.1M/wk. `orval` 730k/wk. | Strong ecosystem. `gqlgen` 10.7k stars. `graphql-codegen` 5M/wk. Less common with K8s. | Large ecosystem (38k stars). Zero K8s production presence. |
| **Operational Simplicity** | Standard HTTP. curl-debuggable. Native caching. Standard monitoring. | Single endpoint. Harder caching. Needs GraphQL-aware tooling. All-200 responses. | Simple if TS-only. Non-standard URL conventions. |
| **Works with Go Backend** | **Yes** -- natural fit | **Yes** -- works but adds complexity | **No** -- eliminated |

---

## 10. Hypothesis Validation

> **Working Hypothesis:** "With only 2 backends (K8s + KillBill), REST (OpenAPI) is likely simpler than GraphQL. GraphQL's query aggregation advantage is minimal with 2 backends. tRPC is eliminated if Go is chosen."

### Evidence For (Hypothesis Supported)

1. **GraphQL's primary advantage is moot.** GraphQL shines when a frontend needs to aggregate data from many microservices in a single query. With exactly 2 backends (K8s CRDs and KillBill), the aggregation benefit is marginal. Most portal views will hit either K8s or KillBill, rarely both in the same request.

2. **KillBill wrapping overhead.** KillBill already exposes REST. Wrapping it in GraphQL creates a translation layer (schema + resolvers + type mapping) with no functional benefit over a thin REST proxy. This is pure overhead for the sake of a unified query language.

3. **K8s ecosystem alignment.** The Kubernetes API itself is REST+OpenAPI. `client-go` is REST-native. Maintaining REST consistency from K8s API -> BFF -> Frontend minimizes impedance mismatch.

4. **Refine ecosystem.** `@refinedev/simple-rest` has 15x the adoption of `@refinedev/graphql`. More examples, more community knowledge, fewer integration surprises.

5. **Operational simplicity.** REST's standard HTTP caching, debugging, and monitoring tooling requires zero additional investment. GraphQL requires dedicated tooling (GraphiQL, GraphQL-aware APM, custom caching strategies).

6. **tRPC elimination confirmed.** tRPC is TypeScript-only. With Go as the natural backend language (K8s operator ecosystem, existing monorepo code), tRPC is definitively eliminated.

### Evidence Against (Partial Counterarguments)

1. **GraphQL subscriptions are more elegant.** Refine's `@refinedev/graphql` includes a built-in `liveProvider` using `graphql-ws`. With REST, a custom `liveProvider` must be written (~100 LOC for SSE). This is a minor advantage.

2. **GraphQL prevents over-fetching.** For complex resources like KillBill invoices (which have many nested fields), GraphQL lets the frontend request only needed fields. However, this can also be achieved in REST via sparse fieldsets (`?fields=id,amount,status`) or dedicated BFF endpoints that return only what the frontend needs.

3. **GraphQL schema as contract.** The GraphQL schema provides a strongly-typed, self-documenting API contract. However, OpenAPI 3.0 provides equivalent contract capabilities, and `oapi-codegen` + `openapi-typescript` achieve comparable type safety.

4. **Future multi-frontend scenario.** If TedaTech later adds a mobile app or partner API, GraphQL's flexibility could become more valuable. However, this is speculative and can be addressed later via an API gateway or BFF-per-client pattern.

### Verdict: Hypothesis Validated

**REST (OpenAPI) is the recommended paradigm for the Customer Portal BFF.**

The evidence strongly supports the hypothesis. With only 2 backends, GraphQL's aggregation advantage does not justify its additional complexity (schema translation layer, custom caching, GraphQL-specific tooling). REST aligns with the K8s ecosystem, has superior Refine provider support, and offers the simplest operational model.

The recommended technology stack:

| Layer | Technology |
|-------|-----------|
| **BFF Language** | Go |
| **BFF Framework** | Chi or standard `net/http` with `oapi-codegen` |
| **API Spec** | OpenAPI 3.0 |
| **Go codegen** | `oapi-codegen` v2 |
| **TS types/hooks** | `orval` (generates TanStack Query hooks from OpenAPI) |
| **Refine dataProvider** | `@refinedev/simple-rest` or custom provider using orval-generated client |
| **Real-time** | SSE from Go BFF (`client-go` informers -> SSE) + custom Refine `liveProvider` |
| **K8s client** | `client-go` / `controller-runtime` |
| **KillBill client** | Generated Go client from KillBill's OpenAPI spec via `oapi-codegen` |

---

## Sources

- [Refine Data Provider Documentation](https://refine.dev/docs/data/data-provider/)
- [Refine Live Provider Documentation](https://refine.dev/docs/advanced-tutorials/real-time/)
- [@refinedev/simple-rest on npm](https://www.npmjs.com/package/@refinedev/simple-rest)
- [@refinedev/graphql on npm](https://www.npmjs.com/package/@refinedev/graphql)
- [refine-trpc-provider (community)](https://github.com/yjavaherian/refine-trpc-provider)
- [Refine GitHub Repository](https://github.com/refinedev/refine) -- 34k stars
- [oapi-codegen](https://github.com/oapi-codegen/oapi-codegen) -- 6k stars
- [gqlgen](https://github.com/99designs/gqlgen) -- 10.7k stars
- [openapi-typescript](https://github.com/openapi-ts/openapi-typescript) -- 2.1M weekly downloads
- [orval](https://github.com/orval-labs/orval) -- 730k weekly downloads, 5.3k stars
- [GraphQL Code Generator](https://the-guild.dev/graphql/codegen) -- 5M+ weekly downloads
- [tRPC](https://github.com/trpc/trpc) -- 38k stars, 700k+ weekly downloads
- [tRPC v11 Announcement](https://trpc.io/blog/announcing-trpc-v11)
- [tRPC Subscriptions Documentation](https://trpc.io/docs/server/subscriptions)
- [graphql-ws on npm](https://www.npmjs.com/package/graphql-ws) -- 6M weekly downloads
- [KillBill API Documentation](https://docs.killbill.io/)
- [KillBill API Quick Start](https://docs.killbill.io/latest/quick_start_with_kb_api)
- [Go BFF with OpenAPI Patterns](https://digitalsoftware.co/2025/03/30/building-scalable-backend-for-frontend-bff-apis-with-go-and-openapi/)
- [GraphQL as BFF -- Complexity Analysis](https://medium.com/@rodrigo.estrada/when-backend-for-frontend-and-graphql-add-more-complexity-than-solutions-d71320d6a683)
- [GraphQL vs REST Comparison](https://medium.com/@danieltaylor2120/graphql-vs-rest-which-will-be-dominant-for-backend-development-in-2025-377ea18231a0)
- [npm trends: @refinedev/simple-rest vs @refinedev/graphql](https://npmtrends.com/@refinedev/simple-rest-vs-@refinedev/graphql)
- [Kubernetes client-go Informers](https://oneuptime.com/blog/post/2026-02-09-client-go-informers-watch-resources/view)
- [Go Ecosystem Trends 2025](https://blog.jetbrains.com/go/2025/11/10/go-language-trends-ecosystem-2025/)

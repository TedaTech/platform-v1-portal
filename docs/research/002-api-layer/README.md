# 002 — API Layer & Backend Architecture

**Issue:** [platform-v1-portal#2](https://github.com/TedaTech/platform-v1-portal/issues/2)
**Date:** 2026-02-14
**Status:** Proposed

## Recommendation

**Go BFF Monolith with REST (OpenAPI 3.0) + SSE**

| Layer | Choice | Key Rationale |
|-------|--------|---------------|
| API Paradigm | REST (OpenAPI 3.0) | Refine `simple-rest` has 40k/wk downloads (15x GraphQL); KillBill is REST — 1:1 proxy |
| Backend Language | Go | `client-go` is gold standard for K8s; 80% pattern transfer from `hetzner-operators` |
| Real-time Protocol | SSE | Active Cilium Gateway WebSocket bugs (#40822, #41123); SSE is standard HTTP, zero config |
| Architecture | BFF Monolith | Only 2 backends (K8s + KillBill); in-process event pipeline via Go channels |
| Frontend Types | orval | TanStack Query hooks auto-generated from OpenAPI spec |
| Auth | keyfunc/v3 + golang-jwt/v5 | JWT validation against Keycloak JWKS endpoint |

## Scoring Summary

| Decision | Winner | Score | Runner-up | Score |
|----------|--------|:-----:|-----------|:-----:|
| API Paradigm | REST (OpenAPI) | 4.75 | GraphQL | 3.80 |
| Backend Language | Go | 4.50 | TypeScript | 2.85 |
| Real-time Protocol | SSE | 4.70 | WebSocket | 2.45 |

tRPC eliminated (requires TypeScript backend). WebSocket penalized by active Cilium Gateway bugs.

## Technology Stack

```
Frontend (from 001)          BFF (this research)         Backends
─────────────────           ────────────────────        ──────────
React + Vite                Go + Chi + oapi-codegen     Kubernetes API (CRDs)
TanStack Router             controller-runtime          KillBill REST API
Refine (headless)           SSE (text/event-stream)
shadcn/ui                   keyfunc/v3 (JWT)
oidc-spa (Keycloak)         orval (TS client codegen)
```

## Key Architectural Decisions

- **Auth flow:** oidc-spa (PKCE) → Bearer JWT → BFF validates via Keycloak JWKS → tenant from groups claim → K8s ServiceAccount for cluster access → KillBill per-tenant API keys
- **Real-time:** K8s informers → Go channel EventBus → SSE writer → `@microsoft/fetch-event-source` on frontend → custom Refine `liveProvider` (~40 LOC)
- **K8s interaction:** controller-runtime Manager with cluster-scoped informer cache, label-selector filtering, namespace-scoped CRUD
- **Long-running ops:** Create CR → return 202 → frontend subscribes SSE for status transitions
- **Multi-tenancy:** One K8s namespace per tenant, ClusterRole + per-namespace RoleBinding

## Research Documents

| Document | Description |
|----------|-------------|
| [API Paradigm Comparison](api-paradigm-comparison.md) | REST vs GraphQL vs tRPC evaluation |
| [Backend Language Comparison](backend-language-comparison.md) | Go vs TypeScript for BFF |
| [Real-time Protocol Comparison](realtime-protocol-comparison.md) | SSE vs WebSocket with Cilium Gateway analysis |
| [Architecture Analysis](architecture-analysis.md) | BFF Monolith vs Microservices |
| [Auth Flow Design](auth-flow-design.md) | OIDC validation, K8s ServiceAccount, KillBill auth, multi-tenancy |
| [K8s Interaction Patterns](k8s-interaction-patterns.md) | Informers, CRD CRUD, EventBus, namespace strategy |
| [Sequence Diagrams](sequence-diagrams.md) | Registration, provisioning, pipelines, billing flows |
| [Scoring Matrix](scoring-matrix.md) | Weighted scoring with hard filters and sensitivity analysis |
| [ADR-002](adr-002-api-layer.md) | Architecture Decision Record |
| [Draft API Contract](draft-api-contract.md) | OpenAPI 3.0 endpoint specifications |

## Impact on Other Issues

- **#3 (Keycloak Auth):** OIDC PKCE flow via oidc-spa, JWT validation with keyfunc/v3, tenant groups in Keycloak
- **#4 (KillBill Integration):** REST-to-REST thin proxy, per-tenant API keys in K8s Secrets, webhook receiver for async events
- **#7 (Pipeline Orchestration):** Pipeline CRD created by BFF, reconciled by operator, status streamed via SSE

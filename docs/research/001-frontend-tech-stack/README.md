# Research: Frontend Tech Stack for Customer Portal

**Issue:** [platform-v1-portal#1](https://github.com/TedaTech/platform-v1-portal/issues/1)
**Date:** 2026-02-13
**Status:** Proposed

## Recommendation

**React + Vite + TanStack Router + Refine (headless) + shadcn/ui**

With Keycloakify for Keycloak login themes and oidc-spa for OIDC authentication.

## Why This Stack

| Requirement | Solution |
|-------------|----------|
| Real-time provisioning status | Refine `liveProvider` (free, first-class WebSocket/SSE) |
| Custom-branded Keycloak login | Keycloakify (React components → Keycloak theme JAR) |
| Multi-step onboarding wizard | React Hook Form + shadcn/ui stepper |
| CRUD entity management | Refine data hooks + TanStack Query |
| Customer-facing (not admin) UX | Refine headless + shadcn/ui (zero admin-panel aesthetics) |
| Static SPA deployment | Vite → nginx:alpine (~13 MB image, <100ms cold start) |
| OIDC authentication | oidc-spa (dedicated TanStack Router adapter, same author as Keycloakify) |

## Frameworks Evaluated

### SPA Frameworks
- **React + Vite** — SELECTED (score: 4.65/5)
- Vue + Vite — runner-up (3.95/5), no Keycloakify support
- SvelteKit SPA — third (2.95/5), SPA mode is secondary
- HTMX + Go — eliminated (requires server, not SPA)
- Remix SPA — eliminated (merged into React Router v7)

### Admin Frameworks
- **Refine (headless)** — SELECTED (score: 4.40/5)
- React Admin — runner-up (3.80/5), real-time + wizard paywalled
- AdminJS — eliminated (requires Node.js backend)
- Retool — eliminated (requires server, vendor lock-in)

### Keycloak Auth
- **Keycloakify** — SELECTED (React-first, builds theme JAR from React components)
- FreeMarker templates — fallback if Keycloakify has compatibility issues
- Custom login frontend — not recommended (security concerns)

## Documents

| Document | Content |
|----------|---------|
| [ADR-001](./adr-001-frontend-stack.md) | Architecture Decision Record (final recommendation) |
| [SPA Framework Comparison](./spa-framework-comparison.md) | React, Vue, SvelteKit, HTMX, Remix evaluation |
| [Admin Framework Comparison](./admin-framework-comparison.md) | Refine, React Admin, AdminJS, Retool, shadcn/ui evaluation |
| [Keycloak Integration](./keycloak-integration.md) | Keycloakify vs FreeMarker vs custom login |
| [Container Footprint](./container-footprint.md) | Docker images, cold start, memory, K8s resources |
| [Scoring Matrix](./scoring-matrix.md) | Weighted scoring with sensitivity analysis |
| [Combination Analysis](./combination-analysis.md) | Top 3 stack combinations, impact on issues #2, #3, #6 |

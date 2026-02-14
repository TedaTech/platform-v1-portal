# ADR-001: Frontend Technology Stack for Customer Portal

## Status

Proposed

## Date

2026-02-13

## Context

The Customer Portal MVP requires a frontend stack that supports:
- Multi-step onboarding wizards with state preservation
- Real-time status updates during infrastructure provisioning (WebSocket/SSE — must-have)
- CRUD entity management (tenants, apps, users, billing)
- Keycloak OIDC authentication with custom-branded login pages
- Deployment as a static SPA in an nginx container on resource-constrained Cozystack

The portal is a customer-facing product, not an internal admin tool. This means the stack must support fully custom, branded UX while still providing framework-level productivity for entity management.

## Decision Drivers

- **Real-time is non-negotiable**: any framework without WebSocket/SSE support is eliminated
- **No SSR**: pure SPA deployed as static files from nginx
- **Keycloak theming**: login pages must match portal branding (Keycloakify support is a major differentiator)
- **Small team, fast MVP**: developer velocity and AI coding assistance matter
- **Customer-facing UX**: must escape "admin panel" aesthetics

## Considered Options

1. **React + Vite + TanStack Router + Refine (headless) + shadcn/ui** (with Keycloakify for auth themes)
2. **React + Vite + React Router v7 + React Admin + shadcn-admin-kit** (with Keycloakify)
3. **Vue + Vite + Vue Router + Vuetify/PrimeVue** (with FreeMarker Keycloak themes)
4. ~~SvelteKit SPA mode~~ (eliminated: SPA mode is secondary, key libraries lose features, smallest ecosystem)
5. ~~HTMX + Go~~ (eliminated: fundamentally server-rendered, cannot produce static SPA)
6. ~~Remix SPA mode~~ (eliminated: merged into React Router v7, no longer distinct)

## Decision

**Option 1: React + Vite + TanStack Router + Refine (headless) + shadcn/ui**

With **Keycloakify** for Keycloak login theme and **oidc-spa** for OIDC authentication.

## Rationale

### Scoring Summary

| | React + Refine | React + React Admin | Vue + Vuetify |
|---|---|---|---|
| SPA Framework Score | **4.65** | 4.65 | 3.95 |
| Admin Framework Score | **4.40** | 3.80 | N/A |
| Combined Score | **4.53** | 4.23 | 3.95 |

### Key Differentiators

1. **Keycloakify (React-first)**: Only React has full Keycloakify support for custom-branded Keycloak login pages using the same component library (shadcn/ui). Vue is not supported. This alone is a deciding factor for branding consistency.

2. **Refine's free real-time**: Refine's `liveProvider` provides first-class WebSocket/SSE subscriptions at zero cost. React Admin's real-time is Enterprise Edition only (EUR 135+/month). Since real-time provisioning status is a must-have, this is critical.

3. **Headless = custom UX**: Refine's headless architecture means zero admin-panel DNA to fight. The portal can look like a polished customer product, not a back-office tool. React Admin requires extensive layout overrides to escape its admin-panel identity.

4. **Ecosystem breadth**: React has 3-4x the component ecosystem of any alternative. MUI, Ant Design, shadcn/ui, TanStack Table, React Hook Form — the deepest library pool. Best AI coding assistance (most training data).

5. **TanStack Router over React Router**: Better TypeScript inference for route params and search params. oidc-spa has a dedicated TanStack Router adapter. Pure client-side by default (no SPA mode toggle needed).

## Consequences

### Positive

- Largest ecosystem = fastest access to solutions for any UI challenge
- Keycloakify + oidc-spa integration is designed to work together (same author)
- Refine's dataProvider abstraction decouples frontend from API paradigm (flexible for Issue #2)
- shadcn/ui components are copy-paste — full ownership, no dependency lock-in
- Best AI coding assistance for development velocity

### Negative

- React runtime is largest (~45 KB gzipped, vs Svelte's ~3-4 KB) — acceptable tradeoff
- Refine is younger than React Admin (YC S23, v4) — less battle-tested at massive scale
- TanStack Router is less widely adopted than React Router — smaller community for troubleshooting
- Multi-step wizard requires custom composition (no `useStepsForm` for shadcn/ui — only Ant Design path has it)

### Risks

| Risk | Mitigation |
|------|-----------|
| Refine discontinuation | Core is open-source (MIT). Refine hooks are thin wrappers around TanStack Query — migration to raw TanStack Query is straightforward. |
| Keycloakify incompatibility with Cozystack's Keycloak version | Keycloakify v11 supports Keycloak 19+. Fallback: FreeMarker CSS-only theming. Verify Cozystack Keycloak version early. |
| TanStack Router immaturity | Growing rapidly (TanStack ecosystem is well-funded). React Router v7 is the fallback — oidc-spa supports both. |

## Impact on Related Decisions

- **Issue #2 (API Layer)**: Refine's `dataProvider` is API-agnostic. Backend can use REST, GraphQL, or tRPC. Switching paradigms later requires only a new provider implementation, not UI changes.
- **Issue #3 (Keycloak)**: Use Keycloakify for login theme (React + shadcn/ui). Deploy theme JAR in custom Keycloak image. Set `loginTheme` via EDP `KeycloakRealm` CRD.
- **Issue #6 (Onboarding UX)**: React Hook Form + shadcn/ui for wizard. Refine's `liveProvider` for real-time provisioning status during onboarding steps.

## References

- [SPA Framework Comparison](./spa-framework-comparison.md)
- [Admin Framework Comparison](./admin-framework-comparison.md)
- [Keycloak Integration](./keycloak-integration.md)
- [Container Footprint](./container-footprint.md)
- [Scoring Matrix](./scoring-matrix.md)
- [Combination Analysis](./combination-analysis.md)

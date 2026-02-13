# Scoring Matrix

## Scoring Scale

| Score | Meaning |
|-------|---------|
| 5 | Best-in-class. Native support, zero friction. |
| 4 | Strong. Well-supported with minor configuration. |
| 3 | Adequate. Works but requires effort or third-party libraries. |
| 2 | Weak. Possible but with significant workarounds. |
| 1 | Poor. Missing, experimental, or requires custom implementation. |

---

## Section A: SPA Frameworks

Only frameworks passing all hard filters are scored (HTMX+Go and Remix eliminated).

| Criterion (Weight) | React + Vite | Vue + Vite | SvelteKit SPA | Justification |
|---------------------|-------------|-----------|---------------|---------------|
| **Component Ecosystem & Admin Integration (25%)** | 5 | 4 | 2 | React: largest ecosystem, Refine/React Admin both React-only. Vue: strong (Vuetify, PrimeVue) but no admin framework integration. Svelte: smallest ecosystem, no admin frameworks. |
| **Keycloak Auth Integration (15%)** | 5 | 3 | 3 | React: Keycloakify + oidc-spa (same author, purpose-built). Vue: NOT supported by Keycloakify, generic OIDC only. Svelte: Keycloakify support exists but less mature. |
| **Developer Experience & Velocity (15%)** | 5 | 4 | 4 | React: best AI coding, most docs/resources, largest community. Vue: excellent docs, VueUse ergonomic. Svelte: fastest HMR, but less training data for AI, Svelte 5 runes still new. |
| **Progressive Wizard UX (15%)** | 4 | 5 | 3 | React: React Hook Form + custom stepper. Vue: built-in stepper components in Vuetify/PrimeVue + VeeValidate. Svelte: Superforms loses features in SPA mode. |
| **Container Footprint (10%)** | 3 | 3 | 5 | All serve from nginx (~13-15 MB). Svelte produces 40% less JS bundle (80-150 KB vs 150-260 KB). |
| **Production Readiness (10%)** | 5 | 5 | 3 | React: 243k stars, 28M/wk, Meta-backed. Vue: 208k stars, 6.4M/wk, proven at scale. Svelte: 85k stars, 2.1M/wk, SPA mode is secondary. |
| **API Paradigm Flexibility (10%)** | 5 | 4 | 3 | React: TanStack Query, Apollo, tRPC, SWR all first-class. Vue: TanStack Query + vue-apollo solid. Svelte: TanStack Query Svelte 5 adapter still in progress. |

### Weighted Scores

| Framework | Weighted Score | Calculation |
|-----------|---------------|-------------|
| **React + Vite** | **4.65** | 5×0.25 + 5×0.15 + 5×0.15 + 4×0.15 + 3×0.10 + 5×0.10 + 5×0.10 |
| **Vue + Vite** | **3.95** | 4×0.25 + 3×0.15 + 4×0.15 + 5×0.15 + 3×0.10 + 5×0.10 + 4×0.10 |
| **SvelteKit SPA** | **2.95** | 2×0.25 + 3×0.15 + 4×0.15 + 3×0.15 + 5×0.10 + 3×0.10 + 3×0.10 |

---

## Section B: Admin Frameworks

Only frameworks passing SPA deployment filter are scored (AdminJS and Retool eliminated).

| Criterion (Weight) | Refine | React Admin | Shadcn/ui + Custom |
|---------------------|--------|-------------|-------------------|
| **Data Binding & CRUD (25%)** | 4 | 5 | 1 |
| **Form Builder Quality (20%)** | 4 | 5 | 3 |
| **Real-time Subscriptions (15%)** | 5 | 2 | 1 |
| **Customization Flexibility (20%)** | 5 | 3 | 5 |
| **SSR Framework Integration (10%)** | 5 | 4 | 5 |
| **Maturity & Community (10%)** | 4 | 5 | 4 |

### Weighted Scores

| Framework | Weighted Score | Calculation |
|-----------|---------------|-------------|
| **Refine** | **4.40** | 4×0.25 + 4×0.20 + 5×0.15 + 5×0.20 + 5×0.10 + 4×0.10 |
| **React Admin** | **3.80** | 5×0.25 + 5×0.20 + 2×0.15 + 3×0.20 + 4×0.10 + 5×0.10 |
| **Shadcn/ui + Custom** | **2.70** | 1×0.25 + 3×0.20 + 1×0.15 + 5×0.20 + 5×0.10 + 4×0.10 |

---

## Sensitivity Analysis

### What if Keycloak Theming weight increased to 25%?

| Framework | New Score | Change |
|-----------|-----------|--------|
| React + Vite | 4.80 | +0.15 (widens lead) |
| Vue + Vite | 3.70 | -0.25 (Vue NOT supported by Keycloakify) |
| SvelteKit SPA | 2.90 | -0.05 |

**Result:** React's lead increases because Keycloakify is React-first.

### What if Component Ecosystem weight decreased to 15%?

| Framework | New Score | Change |
|-----------|-----------|--------|
| React + Vite | 4.55 | -0.10 |
| Vue + Vite | 3.95 | 0.00 |
| SvelteKit SPA | 3.15 | +0.20 |

**Result:** SvelteKit improves but still ranks third. React still leads.

### What if Real-time weight increased for admin frameworks?

Real-time is a must-have. If we increase its weight to 25% (from 15%):

| Framework | New Score | Change |
|-----------|-----------|--------|
| Refine | 4.55 | +0.15 (free real-time) |
| React Admin | 3.50 | -0.30 (Enterprise-only real-time) |

**Result:** Refine's lead over React Admin widens significantly.

---

## Key Takeaways

1. **React + Vite wins the SPA framework comparison** across all weight scenarios due to ecosystem breadth, Keycloakify support, and admin framework availability.

2. **Refine wins the admin framework comparison** due to free real-time support and headless flexibility — critical for a customer-facing portal.

3. **The Keycloakify factor** strongly favors React. Vue is not supported, and Svelte support is less mature. If Keycloak theme branding is important (it is), this alone could decide the framework.

4. **Vue's strength is wizard UX** (built-in stepper components), but this advantage is overcome by React's broader ecosystem and Keycloakify support.

5. **SvelteKit's bundle size advantage** (40% smaller JS) is real but doesn't outweigh its ecosystem disadvantages for this specific use case.

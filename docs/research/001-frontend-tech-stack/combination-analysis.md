# SPA + Admin Framework Combination Analysis

## Compatibility Matrix

Admin frameworks are React-based, limiting viable combinations:

| | Refine | React Admin | Shadcn/ui + Custom |
|---|--------|-------------|-------------------|
| **React + Vite** | Native | Native | Native |
| **Vue + Vite** | No | No | No (Vue components) |
| **SvelteKit SPA** | No | No | No (Svelte components) |

**Vue and SvelteKit cannot use any evaluated admin framework.** They would need to rely on their own ecosystem (Vuetify for Vue, manual for Svelte) or forgo an admin framework entirely.

---

## Top 3 Viable Combinations

### 1. React + Vite + TanStack Router + Refine (headless) + shadcn/ui

**Combined Score: 4.53** (SPA 4.65 × Admin 4.40, averaged and adjusted)

| Aspect | Assessment |
|--------|-----------|
| CRUD/Entity Management | Refine data hooks + TanStack Query. Fast, typesafe. |
| Onboarding Wizard | Custom UI with React Hook Form + shadcn/ui stepper + Refine's `useStepsForm` (Ant Design) or manual composition |
| Real-time Provisioning | Refine `liveProvider` — free, first-class. Connect to backend WebSocket/SSE. |
| Keycloak Auth | oidc-spa (dedicated TanStack Router adapter) + Keycloakify for login theme |
| Branding | Full control via shadcn/ui + Tailwind. No admin-panel aesthetics. |
| Bundle Size | ~150-250 KB gzipped. React runtime is largest but acceptable. |
| API Layer (Issue #2) | Refine's dataProvider is API-agnostic. REST, GraphQL, tRPC all work. TanStack Query underneath. |
| Risk | Refine is younger (YC S23) than React Admin. V4 is stable but less battle-tested at massive scale. |

**Verdict: RECOMMENDED.** Best balance of velocity, flexibility, and real-time support for a customer portal.

### 2. React + Vite + React Router v7 + React Admin + shadcn-admin-kit

**Combined Score: 4.23**

| Aspect | Assessment |
|--------|-----------|
| CRUD/Entity Management | Best-in-class: 45+ data providers, declarative resources, list/edit/show guesser. |
| Onboarding Wizard | `<WizardForm>` — **Enterprise Edition only** (EUR 135+/month). Open-source: manual stepper. |
| Real-time | `ra-realtime` — **Enterprise Edition only.** Open-source: manual WebSocket. |
| Keycloak Auth | Same as Combo 1: oidc-spa + Keycloakify |
| Branding | shadcn-admin-kit removes MUI dependency but react-admin's layout/routing is admin-centric. |
| Bundle Size | Larger (MUI/shadcn-admin-kit + react-admin core). |
| API Layer (Issue #2) | react-admin's data provider is well-established. Same flexibility as Refine. |
| Risk | Enterprise Edition dependency for key features. Marmelab is stable but it's a business risk. |

**Verdict: Strong runner-up.** Choose if: (a) Enterprise Edition budget is available, (b) CRUD quantity outweighs custom UX needs, (c) want maximum ecosystem maturity.

### 3. Vue + Vite + Vue Router + Vuetify/PrimeVue (no admin framework)

**Combined Score: 3.95** (SPA only, no admin framework boost)

| Aspect | Assessment |
|--------|-----------|
| CRUD/Entity Management | Manual with TanStack Query (Vue) + Pinia. No data provider abstraction. |
| Onboarding Wizard | Vuetify `v-stepper` or PrimeVue `Stepper` — built-in, zero extra libraries. |
| Real-time | VueUse `useWebSocket` / `useEventSource` — best-in-class composables. |
| Keycloak Auth | vue-keycloak-js or oidc-spa (generic adapter). **No Keycloakify support** — must use FreeMarker for login theme. |
| Branding | Vuetify Material Design or PrimeVue — customizable but opinionated. |
| Bundle Size | ~130-220 KB gzipped. Slightly smaller than React. |
| API Layer (Issue #2) | TanStack Query (Vue) + vue-apollo. Good but fewer options than React. |
| Risk | No Keycloakify = FreeMarker theming (more effort, less consistency). No admin framework = more CRUD boilerplate. |

**Verdict: Viable but weaker.** The missing Keycloakify support and lack of admin framework integration is a significant disadvantage. Best if team has strong Vue expertise.

---

## Impact on Issue #2 (API Layer)

| Combination | API Paradigm Implications |
|-------------|--------------------------|
| React + Refine | Refine's `dataProvider` is API-agnostic. REST (Simple REST provider), GraphQL (Hasura/NestJS), or custom. TanStack Query handles caching/refetch. **Most flexible — does not constrain API choice.** |
| React + React Admin | Same flexibility via react-admin data providers. 45+ adapters available. |
| Vue + manual | TanStack Query (Vue) + custom fetch. No constraints but no scaffolding either. |

**Recommendation for Issue #2:** The frontend choice (React + Refine) does not constrain the API paradigm. Backend can use REST, GraphQL, or tRPC. Refine's `dataProvider` abstraction means switching API paradigms later requires changing only the provider implementation, not the UI code.

---

## Impact on Issue #3 (Keycloak Architecture)

| Combination | Keycloak Implications |
|-------------|----------------------|
| React + Keycloakify | Custom-branded login pages built with React + shadcn/ui. Deploy as JAR in Keycloak. Set `loginTheme` via EDP `KeycloakRealm` CRD. |
| Vue + FreeMarker | CSS-only or template-override theming. More effort, less branding consistency. |

---

## Impact on Issue #6 (Onboarding UX)

| Combination | Onboarding Implications |
|-------------|------------------------|
| React + Refine | React Hook Form for multi-step wizard. Refine's `liveProvider` for real-time provisioning status during onboarding. shadcn/ui for custom, branded wizard UI. |
| React + React Admin | `<WizardForm>` (Enterprise) or manual stepper. Real-time requires Enterprise or custom. |
| Vue + Vuetify | Built-in `v-stepper` + VueUse `useWebSocket` for real-time. Good OOTB support but no admin framework for entity management pages. |

---

## Final Ranking

| Rank | Combination | Score | Key Advantage | Key Risk |
|------|-------------|-------|---------------|----------|
| **1st** | React + Vite + TanStack Router + Refine + shadcn/ui | **4.53** | Free real-time, headless flexibility, Keycloakify | Refine younger than React Admin |
| **2nd** | React + Vite + React Router v7 + React Admin + shadcn-admin-kit | **4.23** | Most CRUD maturity, 45+ data providers | Enterprise Edition paywall for wizard + real-time |
| **3rd** | Vue + Vite + Vuetify/PrimeVue | **3.95** | Best OOTB stepper, VueUse real-time | No Keycloakify, no admin framework |

# SPA Framework Comparison

## Hard Filter Results

| Framework | Real-time (WS/SSE) | Static SPA Deploy | Keycloak OIDC | Verdict |
|-----------|-------------------|-------------------|---------------|---------|
| React + Vite | PASS | PASS | PASS | **Evaluate** |
| SvelteKit SPA | PASS | PASS | PASS (caveats) | **Evaluate** |
| Vue + Vite | PASS | PASS | PASS | **Evaluate** |
| HTMX + Go | Conditional | FAIL | FAIL | **Eliminated** |
| Remix SPA | PASS (=React) | PASS | PASS | **Eliminated** (merged into React Router v7) |

---

## 1. React + Vite

**Router options:** TanStack Router (best TypeScript inference) or React Router v7 (SPA mode).

### Real-time Capabilities
- Native WebSocket/EventSource API usable via hooks
- [react-use-websocket](https://github.com/robtaussig/react-use-websocket): dedicated hook with auto-reconnect, shared instances
- Socket.IO client: most widely-used real-time JS library
- TanStack Query can integrate WS/SSE events for cache invalidation

### SPA Deployment
`vite build` → `dist/` folder. Serve from nginx with `try_files $uri $uri/ /index.html`. Standard multi-stage Docker: `node` build → `nginx:alpine` serve. React Router v7 SPA mode: set `ssr: false` in config. TanStack Router: pure client-side by default.

### Keycloak OIDC
- **[oidc-spa](https://docs.oidc-spa.dev/)** (by Keycloakify team): purpose-built for SPAs + Keycloak. Auth Code + PKCE, tab sync, auto-logout. Dedicated React + TanStack Router adapter.
- **[react-oidc-context](https://github.com/authts/react-oidc-context)** + oidc-client-ts: community standard
- keycloak-js: official adapter, lower-level

### Component Ecosystem
Largest of any framework:
- **MUI**: 95k+ stars, 4.1M weekly downloads, full component suite
- **Ant Design**: 94k stars, 1.1M weekly DL, enterprise-grade data tables/forms
- **shadcn/ui**: 107k stars, copy-paste Tailwind + Radix primitives
- **TanStack Table**: 26k+ stars, headless data table with sorting/filtering/virtualization
- Recharts, Victory, Nivo for charting

### Developer Experience
- TypeScript: first-class with strict mode out of the box
- HMR: sub-100ms via Vite + React Fast Refresh
- AI coding: best-in-class — most training data for Claude/Copilot
- Documentation: react.dev, reactrouter.com, tanstack.com — all excellent

### Multi-Step Wizard Support
- **[React Hook Form](https://react-hook-form.com/)**: 42k stars, performance-focused, native multi-step pattern
- **rhf-wizard**: dedicated wizard built on React Hook Form
- Zod/Yup for per-step schema validation
- State preserved via `useForm` context or Zustand/Jotai

### Production Readiness
- React: 243k GitHub stars, ~28M weekly npm downloads
- Vite: 71k stars
- Backing: Meta (React), VoidZero/Evan You (Vite)
- Users: Meta, Netflix, Airbnb, Uber, Shopify, Discord

### API Flexibility
- **TanStack Query**: 48k stars, 12.3M weekly DL. REST + GraphQL + any async
- SWR (31k stars), Apollo Client (20k), urql (8.6k), tRPC (35k)
- Full flexibility: REST, GraphQL, tRPC all well-supported

---

## 2. SvelteKit (SPA mode via adapter-static)

### Real-time Capabilities
- Native WebSocket/EventSource API usable in `onMount` lifecycle
- Svelte 5 runes (`$state`, `$derived`) make reactive WS state clean
- Socket.IO works (framework-agnostic)
- In SPA mode, WS/SSE connect to external servers (same as React)

### SPA Deployment
Set `ssr = false`, `prerender = false` in `+layout.js`. Configure `adapter-static` with `fallback: '200.html'`. Nginx: `try_files $uri $uri/ /200.html`.

**Caveat:** SPA mode is a secondary concern — framework is SSR-first. Some community packages assume server hooks are available.

### Keycloak OIDC
- **oidc-spa** (framework-agnostic adapter): works but no dedicated Svelte adapter
- **keycloak-js**: official adapter, works in any browser context
- **Auth.js / SvelteKit Auth**: designed for SSR mode — NOT available in SPA mode
- **@axa-fr/vanilla-oidc**: framework-agnostic OIDC client

**Caveat:** Popular SvelteKit auth solutions (Auth.js) rely on server-side hooks unavailable in SPA mode.

### Component Ecosystem
Smallest of the three viable options:
- **shadcn-svelte**: 7.5k stars, port of shadcn/ui, Svelte 5 + Tailwind v4 support
- **Skeleton UI**: Tailwind-based design system
- **Melt UI**: headless, accessibility-focused (like Radix for React)
- **Flowbite Svelte**: Tailwind CSS components
- Data tables: TanStack Table has Svelte adapter but ecosystem is smaller
- Charts: LayerChart (Svelte-native), Chart.js wrappers

### Developer Experience
- TypeScript: excellent since Svelte 5 with runes
- HMR: fastest of all frameworks
- AI coding: less training data than React. Claude may produce Svelte 4 syntax instead of Svelte 5 runes
- Documentation: high quality at svelte.dev

### Multi-Step Wizard Support
- **[Superforms](https://superforms.rocks/)**: definitive SvelteKit form library with multi-step examples. **But designed around server actions** — loses primary value in SPA mode.
- **Formsnap**: form components on Superforms + Bits UI
- Manual approach: Svelte reactivity makes wizard state machines straightforward

### Production Readiness
- Svelte: 84.8k stars, ~2.1M weekly npm downloads
- SvelteKit: ~300k weekly downloads
- Backing: Vercel (Rich Harris employed there)
- Users: Apple (partial), NYT, Square, Ikea

### API Flexibility
- **TanStack Query (Svelte Query)**: official adapter exists, Svelte 5 support in progress
- Apollo Client: no official adapter, community wrappers less maintained
- tRPC: `@trpc/client` works (framework-agnostic), no dedicated Svelte integration
- GraphQL: graphql-request or urql (framework-agnostic)

---

## 3. Vue + Vite

### Real-time Capabilities
- Native WebSocket/EventSource API usable via `onMounted` + reactive refs
- **[VueUse](https://vueuse.org/)**: 200+ composables including `useWebSocket` (auto-reconnect, heartbeat, message history) and `useEventSource`. **Real-time is a first-class composable — unique advantage.**
- Socket.IO: framework-agnostic, works with Vue
- vue-sse: dedicated Vue SSE plugin

### SPA Deployment
`vite build` → `dist/`. Standard pattern, well-documented. Vue + Vite has been the default SPA build since Vue CLI was deprecated. Nginx deployment identical to React.

### Keycloak OIDC
- **[vue-keycloak-js](https://github.com/dsb-norge/vue-keycloak-js)**: Vue 3 plugin wrapping official Keycloak JS adapter, Composition API composables
- keycloak-js: official adapter
- oidc-spa (framework-agnostic): works via vanilla adapter
- @axa-fr/vanilla-oidc: framework-agnostic OIDC

### Component Ecosystem
Second largest after React:
- **Vuetify**: 40k stars, 76+ Material Design components, **built-in `v-stepper` for wizards**
- **PrimeVue**: 10k+ stars, 90+ components, advanced DataTable, **built-in `Stepper`**
- **Naive UI**: 16k+ stars, modern, 90+ components, good TypeScript
- **Element Plus**: 24k stars, enterprise-focused
- **Quasar**: 26k stars, full framework with stepper and data table
- TanStack Table: Vue adapter available

**Notable:** Vue libs include stepper/wizard components out of the box.

### Developer Experience
- TypeScript: excellent in Vue 3 with `<script setup lang="ts">` and Volar
- HMR: fast (same Vite engine as React)
- AI coding: less training data than React, more than Svelte. Claude handles Vue 3 Composition API well
- Documentation: vuejs.org is one of the best framework docs

### Multi-Step Wizard Support
- **[VeeValidate](https://vee-validate.logaretm.com/v4/)**: standard Vue form validation, official multi-step wizard examples
- **Vuetify v-stepper**: built-in stepper component + VeeValidate integration
- **PrimeVue Stepper**: another OOTB stepper
- Zod/Yup work with VeeValidate

**Notable advantage:** Component libraries include stepper/wizard UI built-in.

### Production Readiness
- Vue: 208k+ stars, ~6.4M weekly npm downloads
- Vue Router: ~2.5M weekly downloads
- Backing: Evan You (independent, VoidZero), funded by sponsors
- Users: Alibaba, GitLab, Nintendo, Adobe, BMW, Upwork

### API Flexibility
- **TanStack Query (Vue Query)**: official adapter, composable-based API
- **Pinia**: 14.5k stars, official state management
- **VueUse `useFetch`**: built-in composable for REST
- vue-apollo: official GraphQL adapter
- urql: Vue adapter available
- tRPC: framework-agnostic client works with Vue + TanStack Query

---

## 4. HTMX + Go — ELIMINATED

**Reason:** Fundamentally server-rendered. HTMX's model is "client requests → server returns HTML fragments → client swaps DOM." This requires a running Go server in production, violating the "no server, static files only" constraint.

While a [client-side HTMX SPA template](https://github.com/Fantalic/htmx-spa-template) exists, it's experimental and defeats HTMX's purpose. HTMX has WebSocket and SSE extensions but they expect server-delivered HTML fragments.

**Deploying HTMX as pure static files from nginx defeats its entire purpose.**

---

## 5. Remix SPA Mode — ELIMINATED

**Reason:** Remix merged into React Router v7 (Dec 2024). What was planned as Remix v3 was released as React Router v7. Remix SPA mode features are now React Router v7's SPA mode. The separate "Remix v3" announced in 2025 drops React entirely, is research-stage, and offers no migration path.

**Choosing "Remix SPA mode" today means choosing React + Vite with React Router v7**, which is covered under Framework 1.

---

## Ranking

| Rank | Framework | Rationale |
|------|-----------|-----------|
| **1st** | React + Vite | Largest ecosystem (3-4x others), best AI assistance, oidc-spa dedicated adapter, TanStack ecosystem, largest hiring pool |
| **2nd** | Vue + Vite | VueUse's useWebSocket/useEventSource are best-in-class, built-in stepper components, strong ecosystem, great docs |
| **3rd** | SvelteKit SPA | Excellent DX/performance but SPA mode is second-class. Key libraries lose features. Smallest ecosystem. |

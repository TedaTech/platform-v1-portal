# Admin/Application Framework Comparison

## Hard Filter: SPA Deployment (no Node.js server in production)

| Framework | SPA Deployment | Verdict |
|-----------|---------------|---------|
| Refine | YES (Vite static build) | **Evaluate** |
| React Admin | YES (Vite static build) | **Evaluate** |
| AdminJS | NO (requires Node.js backend) | **Eliminated** |
| Retool | NO (requires server infrastructure) | **Eliminated** |
| Shadcn/ui + custom | YES (Vite static build) | **Evaluate** |

---

## 1. Refine (React, Headless)

**GitHub:** 34k stars | **npm:** ~28K weekly downloads | **Backing:** YC S23

### Data Binding & CRUD
`dataProvider` abstraction normalizes API calls. 15+ built-in providers: REST, GraphQL (Hasura, NestJS-Query), Supabase, Strapi, Firebase, Appwrite, etc. Standard interface (`getList`, `getOne`, `create`, `update`, `delete`). Auto-wired CRUD hooks (`useList`, `useOne`, `useCreate`, etc.) manage loading/error/cache via TanStack Query. CLI scaffolds CRUD pages.

### Form Builder
Delegates to React Hook Form (for headless/MUI/Mantine) or Ant Design Form. Full access to underlying library: nested objects (dot-notation), array fields (`useFieldArray`), conditional fields, Zod/Yup validation. File uploads via UI library. Rich text: bring your own (TipTap, Quill). Solid but not "batteries-included."

### Multi-Step Wizard
`useStepsForm` for Ant Design: wraps form state with step navigation, per-step validation, state preservation. For headless/MUI/Mantine: compose `useForm` with stepper manually. **Partial built-in support; custom UI needed for non-Ant Design stacks.**

### Real-Time Subscriptions
First-class `liveProvider` interface with subscribe/unsubscribe/publish. Built-in providers for Ably, Supabase, Appwrite, Hasura. `liveMode: "auto"` for immediate updates, `"manual"` for notifications. **Genuine differentiator — real-time is free and first-class.**

### Customization & Flexibility
Headless by design. Core (`@refinedev/core`) has zero UI opinions. Use Ant Design, MUI, Mantine, Chakra, or **shadcn/ui + Tailwind**. Escape to raw React anytime — hooks are just React hooks. Building a fully custom-branded portal is explicitly supported.

### Production Readiness
34k stars, ~28K weekly downloads, YC S23, $3.8M seed, team of 10+. Claims 15K+ monthly active devs, 8K+ projects in production. Currently v4.x with regular releases.

### Customer Portal Fit: **STRONG**
Headless architecture = no admin-panel aesthetics to fight. Real-time maps directly to provisioning status. Data hooks eliminate boilerplate. Wizard requires custom UI but benefits from Refine's form state management. **Best balance of framework velocity + custom UX freedom.**

---

## 2. React Admin (React, MUI-based)

**GitHub:** 26.5k stars | **npm:** ~102K weekly downloads | **Backing:** Marmelab (9+ years)

### Data Binding & CRUD
Most mature provider ecosystem: 45+ adapters (REST, GraphQL, Supabase, Hasura, Strapi, Firebase, Django, Laravel, Spring Boot, etc.). `<Resource>` auto-generates list/create/edit/show routes. `<ListGuesser>` and `<EditGuesser>` infer UI from API responses. **Most CRUD generation of any option.**

### Form Builder
Built on React Hook Form. 40+ input components: TextInput, NumberInput, DateInput, SelectInput, AutocompleteInput, ReferenceInput, ArrayInput, FileInput, ImageInput, RichTextInput (TipTap), SmartRichTextInput (AI-powered, Enterprise). Conditional fields via `<FormDataConsumer>`. **Most complete form component library.**

### Multi-Step Wizard
`<WizardForm>` exists but is **Enterprise Edition only** (EUR 135+/month). Open-source: compose `<Form>` with stepper manually. Enterprise also has `<AutoPersistInStore>` for form state persistence.

### Real-Time Subscriptions
`ra-realtime` module: live updates, lock/unlock, event-driven refresh. **Enterprise Edition only.** Open-source: wire TanStack Query's `refetchInterval` or custom WebSocket manually.

### Customization & Flexibility
MUI-based with 4 built-in themes. Full MUI theming. `shadcn-admin-kit` (by Marmelab) provides shadcn/ui component set on `ra-core`. Layout system is opinionated toward back-office admin UX (sidebar, record-centric routing). **Building a customer portal requires extensive layout overrides.**

### Production Readiness
26.5k stars, ~102K weekly DL (highest), Marmelab (self-funded via Enterprise). Active since 2016. v5.13, weekly bugfix releases. Powers "3,000 new websites/month."

### Customer Portal Fit: **MODERATE**
Excellent for CRUD/entity management. But admin-panel DNA makes customer-facing UX awkward. Best features (wizard, real-time) are paywalled. MUI adds bundle weight. **Best if you accept Enterprise Edition cost and primarily need admin CRUD.**

---

## 3. AdminJS — ELIMINATED

**Reason:** Requires Node.js backend in production. Runs as Express/NestJS/Fastify middleware, generates frontend dynamically, handles API routing through Node.js. Cannot build to static files. Also lacks wizard support, real-time support, and has limited customization.

---

## 4. Retool — ELIMINATED

**Reason:** Requires server infrastructure (self-hosted = Enterprise only, Docker containers + PostgreSQL). Cannot export as static files. Per-user licensing costs prohibitive for customer-facing portal. No real WebSocket support. Vendor lock-in (proprietary format).

---

## 5. Shadcn/ui + Custom (React, No Framework)

**GitHub (ui repo):** 107k stars | **Approach:** Component library, not admin framework

### Data Binding & CRUD
None built-in. Implement everything: API client, TanStack Query, cache management, pagination, filtering, sorting. **3-6x more development time** for CRUD/entity management vs Refine or React Admin.

### Form Builder
shadcn/ui form primitives built on React Hook Form + Zod. Input, Textarea, Select, Checkbox, RadioGroup, DatePicker, Combobox. Array fields via `useFieldArray`. File uploads: custom. Rich text: separate library. **High-quality components but fully manual wiring.**

### Multi-Step Wizard
Build manually with React Hook Form + Zod + stepper UI. Full flexibility, zero scaffolding. **Expect 2-4 days for production wizard.**

### Real-Time Subscriptions
None built-in. TanStack Query `refetchInterval` for polling, or custom WebSocket hooks. Build a `useRealtimeQuery` hook manually.

### Customization & Flexibility
**Maximum.** Copy-paste model, full component ownership. Tailwind CSS, full pixel control. No framework opinions. **You build literally everything but you can build literally anything.**

### Production Readiness
107k stars (most popular React UI library). Radix primitives have millions of weekly downloads. Created by shadcn (Vercel). **UI components are production-ready; admin system depends on your implementation.**

### Customer Portal Fit: **DEPENDS ON TIMELINE**
Best visual quality, worst velocity. Only recommended if: (a) unique UX requirements that no framework can accommodate, or (b) combined with Refine headless (Refine + shadcn/ui).

---

## Recommendation

| Rank | Framework | Best For |
|------|-----------|----------|
| **1st** | **Refine (headless) + shadcn/ui** | Customer portal with custom UX. Free real-time, headless flexibility, shadcn/ui for branding. |
| **2nd** | React Admin (with shadcn-admin-kit) | Maximum CRUD velocity if Enterprise Edition cost (EUR 135+/mo) is acceptable. |
| **3rd** | Shadcn/ui + custom | Maximum design freedom, minimum velocity. Use as UI layer under Refine, not standalone. |

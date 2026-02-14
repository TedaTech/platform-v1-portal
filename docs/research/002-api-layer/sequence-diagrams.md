# Customer Portal Sequence Diagrams

**Date:** 2026-02-14
**Status:** Draft
**Author:** PM/Scrum Master Agent
**Repo:** TedaTech/platform-v1-portal

---

## Architecture Overview

These diagrams document the core interaction flows for the TedaTech Customer Portal. The system consists of:

- **Frontend:** React SPA (Vite + TanStack Router + Refine + shadcn/ui + oidc-spa)
- **BFF:** Go monolith with REST API (OpenAPI via `oapi-codegen`) + SSE for real-time
- **Auth:** Keycloak OIDC (realm: `cozy`, client: `portal-spa`)
- **K8s Backend:** `client-go` informers watching CRDs across tenant namespaces
- **Billing:** KillBill REST API (per-tenant API key from K8s Secret)
- **Real-time:** SSE from BFF to frontend (informer events -> in-process event bus -> SSE writer)
- **Ingress:** Cilium Gateway API (HTTPS termination, HTTP/2 to browsers)

Key CRDs:
- `ForgejaTenant` (platform.tedatech.app/v1alpha1) -- Forgejo org + GitOps repo via Crossplane
- Cozystack `Tenant` (apps.cozystack.io/v1alpha1) -- tenant namespace + resource quotas
- EDP Keycloak CRDs -- realm users, groups, clients

---

## 1. Customer Registration Flow

This flow covers the complete journey from a new user arriving at the portal URL through to a fully initialized portal session. Registration is handled by Keycloak's built-in registration form; the BFF is responsible for post-registration initialization (creating the KillBill billing account and resolving tenant membership).

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant oidc as oidc-spa
    participant KC as Keycloak
    participant BFF
    participant K8s as K8s API
    participant KB as KillBill

    User->>Browser: Navigate to portal URL

    Note over oidc: oidc-spa initializes,<br/>checks for existing session

    oidc->>oidc: Check for access token in memory
    oidc-->>oidc: No token found

    oidc->>KC: Redirect to /realms/cozy/protocol/openid-connect/auth<br/>(response_type=code, client_id=portal-spa,<br/>code_challenge=..., code_challenge_method=S256)

    Note over KC: Keycloak renders login page<br/>with "Register" link

    User->>KC: Click "Register"
    KC-->>Browser: Render registration form

    User->>KC: Submit registration<br/>(email, password, first name, last name)

    KC->>KC: Create user account
    KC->>KC: Assign user to default group

    Note over KC: Keycloak issues auth code<br/>and redirects to portal callback URL

    KC-->>Browser: HTTP 302 redirect to<br/>portal.teda.tech/callback?code=AUTH_CODE&state=...

    oidc->>KC: POST /realms/cozy/protocol/openid-connect/token<br/>(grant_type=authorization_code,<br/>code=AUTH_CODE, code_verifier=...)

    Note over oidc,KC: PKCE exchange -- code_verifier<br/>proves the original requester

    KC-->>oidc: 200 OK<br/>{access_token, refresh_token, id_token, expires_in}

    oidc->>oidc: Store access token in memory<br/>(NOT localStorage -- XSS mitigation)

    Note over Browser,BFF: Frontend now has a valid<br/>Bearer token for API calls

    Browser->>BFF: POST /api/users/me/initialize<br/>Authorization: Bearer {access_token}

    BFF->>BFF: Validate JWT against Keycloak JWKS<br/>(cached, auto-refreshed)
    BFF->>BFF: Extract claims: sub, email,<br/>groups, tenant_id

    alt JWT validation fails
        BFF-->>Browser: 401 Unauthorized
        oidc->>oidc: Clear token, restart auth flow
    end

    BFF->>KB: GET /1.0/kb/accounts?externalKey={keycloak_sub}

    alt Account exists
        KB-->>BFF: 200 OK {account}
    else Account not found
        KB-->>BFF: 404 Not Found
        BFF->>KB: POST /1.0/kb/accounts<br/>{externalKey: keycloak_sub,<br/>name: user_name, email: user_email}
        KB-->>BFF: 201 Created {accountId}
    end

    BFF->>K8s: List Namespaces with label<br/>tenant.teda.tech/group={user_group}

    Note over BFF,K8s: Resolve which tenants<br/>this user belongs to

    K8s-->>BFF: Namespace list (tenant memberships)

    BFF-->>Browser: 200 OK<br/>{user: {id, email, name},<br/>tenants: [{id, name, role}],<br/>billing: {accountId, status}}

    Browser->>Browser: Refine renders dashboard<br/>with tenant selector

    Note over User,Browser: Registration complete.<br/>User sees their portal dashboard.
```

### Implementation Notes

**Token storage:** `oidc-spa` stores the access token in a JavaScript closure (memory), not in `localStorage` or `sessionStorage`. This prevents XSS attacks from exfiltrating tokens. The trade-off is that a full page refresh requires a silent re-authentication via Keycloak's session cookie (iframe-based).

**JWKS caching:** The BFF caches Keycloak's JWKS (JSON Web Key Set) in memory and refreshes it when encountering an unknown `kid` (key ID) in a JWT header. This handles key rotation without downtime.

**KillBill account creation:** The `externalKey` field links the KillBill account to the Keycloak user ID (`sub` claim). This is an idempotent operation -- if the account already exists, the BFF skips creation. The BFF uses a per-tenant API key/secret pair stored in a K8s Secret, injected via environment variable or volume mount.

**Tenant resolution:** User-to-tenant mapping is derived from Keycloak group membership. The user's JWT contains a `groups` claim (e.g., `["/tenants/acme-corp"]`). The BFF maps this to K8s namespaces labeled with the corresponding tenant group. A user may belong to multiple tenants.

**Error handling not shown:** Network timeouts, KillBill rate limiting (429 responses with `Retry-After`), and Keycloak unavailability are handled by the BFF with circuit breakers per upstream dependency. The frontend receives a structured error response and displays an appropriate message.

---

## 2. Tenant Provisioning Flow

This flow shows how an admin user creates a new tenant, which triggers the creation of multiple Kubernetes CRDs. Each CRD is reconciled by its respective operator (Cozystack, Crossplane, EDP). The BFF relays real-time status updates to the frontend via SSE, using the in-process event bus fed by K8s informers.

```mermaid
sequenceDiagram
    actor Admin
    participant Browser
    participant Frontend as Frontend (Refine)
    participant BFF
    participant K8s as K8s API
    participant Cozy as Cozystack Operator
    participant XP as Crossplane
    participant EDP as EDP Operator
    participant SSE as SSE Stream

    Admin->>Frontend: Click "Create New Tenant"
    Frontend->>Frontend: Render tenant creation form

    Admin->>Frontend: Fill form: name, plan, resource quotas
    Frontend->>BFF: POST /api/tenants<br/>Authorization: Bearer {token}<br/>{name: "acme-corp", plan: "starter",<br/>quotas: {cpu: "4", memory: "8Gi"}}

    BFF->>BFF: Validate JWT
    BFF->>BFF: Check admin role in claims

    alt User is not admin
        BFF-->>Frontend: 403 Forbidden<br/>{error: "Insufficient permissions"}
        Frontend->>Frontend: Show error notification
    end

    Note over BFF,K8s: Create CRDs in sequence.<br/>Namespace must exist before<br/>namespaced resources.

    BFF->>K8s: Create Cozystack Tenant CR<br/>(apps.cozystack.io/v1alpha1)<br/>{name: "acme-corp",<br/>quotas: {cpu: "4", memory: "8Gi"}}
    K8s-->>BFF: 201 Created

    BFF->>K8s: Create ForgejaTenant CR<br/>(platform.tedatech.app/v1alpha1)<br/>{name: "acme-corp",<br/>org: "acme-corp", gitopsRepo: true}
    K8s-->>BFF: 201 Created

    BFF->>K8s: Create KeycloakRealmGroup CR<br/>{realm: "cozy",<br/>name: "tenants/acme-corp"}
    K8s-->>BFF: 201 Created

    BFF-->>Frontend: 201 Created<br/>{id: "acme-corp", status: "provisioning"}

    Note over Frontend,SSE: Frontend opens SSE connection<br/>to receive real-time status updates

    Frontend->>SSE: GET /api/events?resource=Tenant&name=acme-corp<br/>Authorization: Bearer {token}<br/>Accept: text/event-stream

    BFF->>BFF: Validate JWT on SSE request
    BFF->>BFF: Subscribe to event bus<br/>for tenant "acme-corp"

    SSE-->>Frontend: 200 OK<br/>Content-Type: text/event-stream<br/>retry: 3000

    Note over SSE,Frontend: SSE connection established.<br/>BFF sends keepalive every 15s.

    par Cozystack reconciliation
        Cozy->>K8s: Watch Tenant CRs
        Cozy->>Cozy: Reconcile: create namespace<br/>"tenant-acme-corp"
        Cozy->>K8s: Create Namespace + ResourceQuotas
        Cozy->>K8s: Update Tenant CR status:<br/>phase=NamespaceReady
    and Informer detects change
        K8s-->>BFF: Informer event: Tenant status updated
        BFF->>BFF: Event bus publish:<br/>{resource: "Tenant", name: "acme-corp",<br/>phase: "NamespaceReady"}
    end

    SSE-->>Frontend: event: updated<br/>id: Tenant/acme-corp@rv1234<br/>data: {"type":"updated","phase":"NamespaceReady",<br/>"message":"Namespace created"}

    Frontend->>Frontend: Update progress UI:<br/>"Namespace created" (1/3)

    par Crossplane reconciliation
        XP->>K8s: Watch ForgejaTenant CRs
        XP->>XP: Reconcile: create Forgejo org,<br/>create GitOps repository,<br/>configure webhooks
        XP->>K8s: Update ForgejaTenant CR status:<br/>phase=Ready
    and Informer detects change
        K8s-->>BFF: Informer event: ForgejaTenant updated
        BFF->>BFF: Event bus publish
    end

    SSE-->>Frontend: event: updated<br/>id: ForgejaTenant/acme-corp@rv2345<br/>data: {"type":"updated","phase":"Ready",<br/>"message":"Forgejo org and GitOps repo created"}

    Frontend->>Frontend: Update progress UI:<br/>"Forgejo configured" (2/3)

    par EDP reconciliation
        EDP->>K8s: Watch KeycloakRealmGroup CRs
        EDP->>EDP: Reconcile: create Keycloak group<br/>"tenants/acme-corp" in realm "cozy"
        EDP->>K8s: Update KeycloakRealmGroup status:<br/>phase=Ready
    and Informer detects change
        K8s-->>BFF: Informer event: KeycloakRealmGroup updated
        BFF->>BFF: Event bus publish
    end

    SSE-->>Frontend: event: updated<br/>id: KeycloakRealmGroup/acme-corp@rv3456<br/>data: {"type":"updated","phase":"Ready",<br/>"message":"Auth group configured"}

    Frontend->>Frontend: Update progress UI:<br/>"Auth configured" (3/3)

    Note over BFF: BFF detects all 3 CRDs<br/>have reached Ready=True

    SSE-->>Frontend: event: updated<br/>id: Tenant/acme-corp@rv4567<br/>data: {"type":"updated","phase":"Active",<br/>"message":"Tenant provisioning complete"}

    Frontend->>Frontend: Show "Tenant Active" status<br/>with links to Forgejo, dashboard
```

### Implementation Notes

**CRD creation order:** The Cozystack Tenant CR is created first because the namespace it provisions is required by namespaced resources. However, the ForgejaTenant and KeycloakRealmGroup CRDs are cluster-scoped or created in a shared namespace, so they can be created in parallel with the Tenant CR. The operators themselves are idempotent and will retry if dependencies are not yet ready.

**SSE event IDs:** Each SSE event ID follows the format `{CRDKind}/{name}@{resourceVersion}`. This enables `Last-Event-ID`-based resume on reconnection. If the client reconnects to a different BFF pod, that pod can check its informer cache for any events with a resourceVersion greater than the one in the `Last-Event-ID` header.

**Provisioning timeout:** The BFF should implement a provisioning timeout (e.g., 10 minutes). If any CRD does not reach `Ready=True` within the timeout, the BFF pushes an SSE event with `phase: "Failed"` and the frontend shows an error with the stuck CRD's status conditions.

**Rollback on partial failure:** If one CRD fails to provision (e.g., Forgejo API is down), the other CRDs may still succeed. The BFF does not automatically roll back successful CRDs. Instead, it reports the partial failure via SSE, and the admin can retry or manually intervene. Full rollback (deleting all CRDs) can be implemented as a separate admin action.

**Informer efficiency:** The BFF does not create new informers per SSE connection. A single set of informers watches all CRDs across all tenant namespaces. The in-process event bus routes events to the correct SSE connections based on resource type, namespace, and name filters specified in the SSE URL parameters.

---

## 3. Pipeline Trigger Flow (PR Creation)

This flow shows how a user deploys an application using a template, which triggers a pipeline that creates a pull request in the tenant's Forgejo GitOps repository. The Pipeline CRD is a custom resource reconciled by a pipeline operator running in the cluster.

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant Frontend
    participant BFF
    participant K8s as K8s API
    participant PO as Pipeline Operator
    participant Forgejo as Forgejo API
    participant SSE as SSE Stream

    User->>Frontend: Navigate to tenant's "Deploy" page
    Frontend->>BFF: GET /api/tenants/acme-corp/templates<br/>Authorization: Bearer {token}
    BFF->>K8s: List ConfigMaps with label<br/>template.teda.tech/type=application<br/>in namespace tenant-acme-corp
    K8s-->>BFF: ConfigMap list (available templates)
    BFF-->>Frontend: 200 OK [{name: "nextjs-app",<br/>description: "...", params: [...]}]

    Frontend->>Frontend: Render template picker

    User->>Frontend: Select "nextjs-app" template,<br/>fill parameters (app name, domain, etc.)
    User->>Frontend: Click "Deploy Application"

    Frontend->>BFF: POST /api/tenants/acme-corp/pipelines<br/>Authorization: Bearer {token}<br/>{template: "nextjs-app",<br/>params: {name: "my-app", domain: "my-app.acme.teda.tech"}}

    BFF->>BFF: Validate JWT
    BFF->>BFF: Check tenant access:<br/>user belongs to tenant "acme-corp"

    alt User has no access to tenant
        BFF-->>Frontend: 403 Forbidden
    end

    BFF->>BFF: Validate pipeline params<br/>against template schema

    alt Validation fails
        BFF-->>Frontend: 422 Unprocessable Entity<br/>{errors: [{field: "domain", message: "..."}]}
    end

    BFF->>K8s: Create Pipeline CR in namespace<br/>tenant-acme-corp<br/>{spec: {template: "nextjs-app",<br/>params: {name: "my-app", ...},<br/>gitopsRepo: "acme-corp/gitops"}}
    K8s-->>BFF: 201 Created {name: "pipeline-abc123"}

    BFF-->>Frontend: 202 Accepted<br/>{id: "pipeline-abc123", status: "pending"}

    Note over Frontend: Frontend opens SSE to<br/>track pipeline progress

    Frontend->>SSE: GET /api/events?resource=Pipeline<br/>&name=pipeline-abc123<br/>&namespace=tenant-acme-corp<br/>Authorization: Bearer {token}

    SSE-->>Frontend: 200 OK (text/event-stream)

    PO->>K8s: Watch Pipeline CRs
    PO->>PO: Pick up Pipeline CR "pipeline-abc123"
    PO->>K8s: Update Pipeline status:<br/>phase=Running

    K8s-->>BFF: Informer: Pipeline status changed
    SSE-->>Frontend: event: updated<br/>data: {"phase":"Running",<br/>"message":"Pipeline started"}

    Frontend->>Frontend: Show "Pipeline running..."

    PO->>Forgejo: Clone GitOps repo<br/>acme-corp/gitops (via Forgejo API)
    Forgejo-->>PO: Repository contents

    PO->>PO: Apply template "nextjs-app"<br/>with user params to repo contents
    PO->>PO: Create branch<br/>"deploy/my-app-{timestamp}"

    PO->>Forgejo: Push branch<br/>"deploy/my-app-{timestamp}"
    Forgejo-->>PO: Push accepted

    PO->>K8s: Update Pipeline status:<br/>phase=BranchPushed
    K8s-->>BFF: Informer: Pipeline updated
    SSE-->>Frontend: event: updated<br/>data: {"phase":"BranchPushed",<br/>"message":"Branch created"}

    PO->>Forgejo: POST /api/v1/repos/acme-corp/gitops/pulls<br/>{title: "Deploy my-app (nextjs-app)",<br/>head: "deploy/my-app-{timestamp}",<br/>base: "main", body: "..."}
    Forgejo-->>PO: 201 Created {number: 42,<br/>html_url: "https://forgejo.teda.tech/..."}

    PO->>K8s: Update Pipeline status:<br/>phase=PRCreated,<br/>prUrl: "https://forgejo.teda.tech/...",<br/>prNumber: 42
    K8s-->>BFF: Informer: Pipeline updated

    SSE-->>Frontend: event: updated<br/>data: {"phase":"PRCreated",<br/>"prUrl":"https://forgejo.teda.tech/...",<br/>"prNumber":42,<br/>"message":"Pull request created"}

    Frontend->>Frontend: Show PR link to user:<br/>"Review PR #42 in Forgejo"

    Note over User: User reviews and merges PR in Forgejo.<br/>Flux detects merge, applies manifests<br/>to tenant namespace. App is deployed.
```

### Implementation Notes

**Template storage:** Application templates are stored as ConfigMaps in the tenant namespace, labeled with `template.teda.tech/type=application`. Each ConfigMap contains a `schema.json` (parameter schema for validation) and template files (Kubernetes manifests with Go template placeholders). The BFF validates user-provided parameters against the schema before creating the Pipeline CR.

**Pipeline operator credentials:** The pipeline operator uses a Forgejo API token stored in a K8s Secret. Each tenant has its own Forgejo organization and API token, scoped to that organization. The operator reads the token from the Secret in the tenant namespace.

**Idempotency:** If the user accidentally triggers the same pipeline twice, the operator should detect duplicate branch names or open PRs and update the Pipeline CR status with an appropriate message rather than creating duplicate PRs.

**Pipeline cleanup:** Pipeline CRs are retained for audit purposes but can be garbage-collected after a configurable TTL (e.g., 30 days). The operator sets an `ownerReference` to the tenant, so deleting a tenant cascades to its pipelines.

**Post-merge flow (not shown):** After the user merges the PR in Forgejo, Flux detects the change on the `main` branch and applies the new manifests to the tenant namespace. The application deployment is managed entirely by GitOps from this point forward. The portal can show deployment status by watching the relevant Deployment/StatefulSet CRDs via the same SSE mechanism.

---

## 4. Billing Subscription Change Flow

This flow covers the complete cycle of a tenant admin changing their billing plan, including the asynchronous webhook from KillBill that triggers resource quota updates in Kubernetes.

```mermaid
sequenceDiagram
    actor Admin
    participant Browser
    participant Frontend
    participant BFF
    participant KB as KillBill
    participant K8s as K8s API
    participant Cozy as Cozystack Operator

    Admin->>Frontend: Navigate to Billing Settings

    Frontend->>BFF: GET /api/tenants/acme-corp/billing/subscription<br/>Authorization: Bearer {token}

    BFF->>BFF: Validate JWT, check admin role + tenant access

    BFF->>K8s: Get Secret "killbill-credentials"<br/>from namespace tenant-acme-corp
    K8s-->>BFF: {apiKey, apiSecret}

    Note over BFF,KB: BFF authenticates to KillBill<br/>with per-tenant API key/secret

    BFF->>KB: GET /1.0/kb/accounts/{accountId}/bundles<br/>X-Killbill-ApiKey: {apiKey}<br/>X-Killbill-ApiSecret: {apiSecret}
    KB-->>BFF: 200 OK {bundles: [{subscriptionId,<br/>planName: "starter", phase: "EVERGREEN",<br/>priceList: "DEFAULT", ...}]}

    BFF->>KB: GET /1.0/kb/catalog<br/>X-Killbill-ApiKey: {apiKey}
    KB-->>BFF: 200 OK {plans: [{name: "starter", ...},<br/>{name: "professional", ...},<br/>{name: "enterprise", ...}]}

    BFF-->>Frontend: 200 OK<br/>{current: {plan: "starter", ...},<br/>available: [{name: "professional",<br/>price: "49/mo", quotas: {cpu: 8, memory: 16Gi}}, ...]}

    Frontend->>Frontend: Render plan comparison view

    Admin->>Frontend: Select "professional" plan
    Frontend->>Frontend: Show confirmation dialog:<br/>"Upgrade to Professional ($49/mo)?"
    Admin->>Frontend: Confirm upgrade

    Frontend->>BFF: POST /api/tenants/acme-corp/billing/subscription/change<br/>Authorization: Bearer {token}<br/>{planName: "professional",<br/>effectiveDate: "IMMEDIATE"}

    BFF->>BFF: Validate JWT, check admin role + tenant access

    BFF->>KB: PUT /1.0/kb/subscriptions/{subscriptionId}<br/>X-Killbill-ApiKey: {apiKey}<br/>X-Killbill-ApiSecret: {apiSecret}<br/>{planName: "professional",<br/>billingPolicy: "IMMEDIATE"}

    alt KillBill rejects change
        KB-->>BFF: 400 Bad Request<br/>{message: "Cannot downgrade during trial"}
        BFF-->>Frontend: 422 Unprocessable Entity<br/>{error: "Plan change not allowed during trial period"}
        Frontend->>Frontend: Show error notification
    end

    KB->>KB: Process subscription change<br/>(prorate current period,<br/>calculate new charges)
    KB-->>BFF: 200 OK {subscriptionId,<br/>planName: "professional"}

    BFF-->>Frontend: 200 OK<br/>{subscription: {plan: "professional",<br/>status: "active",<br/>message: "Plan change applied"}}

    Frontend->>Frontend: Update UI: "Professional Plan (Active)"<br/>Show toast: "Plan upgraded successfully"

    Note over KB,BFF: Asynchronous: KillBill generates<br/>invoice and sends webhook

    KB->>KB: Generate prorated invoice<br/>for plan change

    rect rgb(240, 240, 255)
        Note over KB,Cozy: Webhook flow (async, minutes later)

        KB->>BFF: POST /api/webhooks/killbill<br/>{eventType: "INVOICE_CREATION",<br/>accountId: "...",<br/>objectId: "invoice-xyz"}

        BFF->>BFF: Verify webhook signature

        BFF->>KB: GET /1.0/kb/invoices/{invoice-xyz}<br/>X-Killbill-ApiKey: {apiKey}
        KB-->>BFF: 200 OK {invoice details,<br/>items: [{planName: "professional", ...}]}

        BFF->>BFF: Map plan "professional" to<br/>resource quotas:<br/>{cpu: "8", memory: "16Gi"}

        BFF->>K8s: Patch Cozystack Tenant CR<br/>"acme-corp"<br/>spec.quotas: {cpu: "8", memory: "16Gi"}
        K8s-->>BFF: 200 OK (patched)

        Cozy->>K8s: Watch Tenant CRs
        Cozy->>Cozy: Reconcile: update ResourceQuota<br/>in namespace tenant-acme-corp
        Cozy->>K8s: Patch ResourceQuota:<br/>{cpu: "8", memory: "16Gi"}
    end

    Note over Admin: Resource quotas are now updated.<br/>Tenant can use more resources.
```

### Implementation Notes

**KillBill authentication model:** KillBill uses per-tenant API key/secret pairs, not user-level authentication. The BFF retrieves these credentials from a K8s Secret in the tenant's namespace. This means the BFF acts as a trusted intermediary -- it authenticates the user via JWT, then uses service-level credentials to call KillBill.

**Plan-to-quota mapping:** The mapping from KillBill plan names to Kubernetes resource quotas is maintained as a ConfigMap in the BFF's namespace (e.g., `plan-quota-mapping`). This decouples billing plan definitions (in KillBill) from infrastructure resource allocations (in K8s). Example mapping:

| Plan         | CPU | Memory | Storage | Namespaces |
|--------------|-----|--------|---------|------------|
| starter      | 4   | 8Gi    | 50Gi    | 1          |
| professional | 8   | 16Gi   | 200Gi   | 3          |
| enterprise   | 32  | 64Gi   | 1Ti     | 10         |

**Webhook verification:** KillBill webhooks should be verified using a shared secret (HMAC signature) to prevent spoofing. The BFF validates the signature before processing the webhook payload. Replay protection can be implemented by checking the event ID against a short-lived deduplication cache.

**Effective date options:** The `effectiveDate` field supports three values:
- `IMMEDIATE` -- change takes effect now, current period is prorated
- `END_OF_TERM` -- change takes effect at the next billing cycle
- A specific ISO 8601 date

**Downgrade handling:** When downgrading, the BFF should check whether the tenant's current resource usage exceeds the new plan's quotas. If so, the BFF returns a 422 with a message explaining which resources need to be reduced before the downgrade can proceed. The actual quota reduction in K8s only happens after the webhook confirms the plan change.

---

## 5. Real-Time Dashboard Connection Flow

This flow details how the frontend establishes and maintains the SSE connection for real-time dashboard updates. It covers the full lifecycle: initial connection, keepalive handling, event processing, and automatic reconnection with `Last-Event-ID` resume.

```mermaid
sequenceDiagram
    actor User
    participant Browser
    participant FES as fetch-event-source
    participant BFF
    participant Informers as K8s Informers
    participant Bus as Event Bus
    participant Refine

    User->>Browser: Open portal dashboard

    Browser->>Refine: Mount dashboard component
    Refine->>Refine: Initialize liveProvider

    Note over FES: @microsoft/fetch-event-source<br/>supports Authorization headers<br/>unlike native EventSource API

    Refine->>FES: subscribe({channel: "resources/tenants",<br/>types: ["*"], callback: onEvent})

    FES->>BFF: GET /api/events?resource=Tenant<br/>&namespace=tenant-acme-corp<br/>Authorization: Bearer {access_token}<br/>Accept: text/event-stream

    BFF->>BFF: Validate JWT against Keycloak JWKS
    BFF->>BFF: Extract tenant from claims

    alt JWT expired
        BFF-->>FES: 401 Unauthorized
        FES->>FES: Call token refresh callback
        Note over FES,BFF: oidc-spa silently refreshes<br/>token via Keycloak session
        FES->>BFF: Retry GET /api/events<br/>Authorization: Bearer {new_token}
    end

    BFF->>BFF: Set SSE response headers:<br/>Content-Type: text/event-stream<br/>Cache-Control: no-cache<br/>X-Accel-Buffering: no

    BFF->>Bus: Subscribe to events for<br/>namespace "tenant-acme-corp"
    Bus-->>BFF: Subscription channel created

    BFF-->>FES: 200 OK<br/>retry: 3000

    Note over BFF,FES: SSE connection established.<br/>retry: 3000 tells the client to<br/>wait 3 seconds before reconnecting.

    BFF-->>FES: : keepalive

    Note over BFF: Keepalive comment (lines starting<br/>with ':') are ignored by EventSource<br/>but reset Envoy's stream_idle_timeout

    loop Every 15 seconds
        BFF-->>FES: : keepalive
        Note right of BFF: Prevents Cilium/Envoy<br/>5-minute idle timeout
    end

    Note over Informers: A CRD status changes in K8s<br/>(e.g., ForgejaTenant phase update)

    Informers->>Informers: OnUpdate callback fires
    Informers->>Bus: Publish event:<br/>{resource: "ForgejaTenant",<br/>namespace: "tenant-acme-corp",<br/>name: "acme-corp",<br/>type: "updated",<br/>resourceVersion: "rv5678"}

    Bus->>BFF: Deliver event to SSE handler<br/>(via Go channel)

    BFF-->>FES: id: ForgejaTenant/tenant-acme-corp/acme-corp@rv5678<br/>event: updated<br/>data: {"type":"updated","channel":"resources/forgejo-tenants",<br/>"payload":{"ids":["acme-corp"]}}

    FES->>Refine: callback({channel: "resources/forgejo-tenants",<br/>type: "updated",<br/>payload: {ids: ["acme-corp"]},<br/>date: new Date()})

    Refine->>Refine: Invalidate TanStack Query cache<br/>for "forgejo-tenants" resource

    Note over Refine,BFF: Refine automatically refetches<br/>the affected resource data.<br/>It does NOT apply patches from<br/>the SSE payload.

    Refine->>BFF: GET /api/tenants/acme-corp/forgejo<br/>Authorization: Bearer {token}
    BFF-->>Refine: 200 OK {updated resource data}

    Refine->>Browser: Re-render component with<br/>updated data

    Note over FES,BFF: Connection drops<br/>(network issue, pod restart)

    BFF--xFES: Connection lost

    FES->>FES: Detect disconnection
    FES->>FES: Wait 3 seconds (retry interval)

    FES->>BFF: GET /api/events?resource=Tenant<br/>&namespace=tenant-acme-corp<br/>Authorization: Bearer {token}<br/>Last-Event-ID: ForgejaTenant/tenant-acme-corp/acme-corp@rv5678

    Note over BFF: BFF may connect to a<br/>different pod after reconnect

    BFF->>BFF: Validate JWT
    BFF->>BFF: Parse Last-Event-ID:<br/>resource=ForgejaTenant,<br/>namespace=tenant-acme-corp,<br/>name=acme-corp,<br/>resourceVersion=rv5678

    BFF->>Informers: Check informer cache for events<br/>newer than rv5678

    alt Events available in cache
        BFF-->>FES: Replay missed events<br/>id: ForgejaTenant/tenant-acme-corp/acme-corp@rv6789<br/>event: updated<br/>data: {...}
    else No new events
        Note over BFF,FES: No replay needed,<br/>resume normal streaming
    end

    BFF->>Bus: Subscribe to events
    BFF-->>FES: : keepalive

    Note over User,Browser: Connection resumed.<br/>Dashboard continues receiving<br/>real-time updates seamlessly.
```

### Implementation Notes

**Token refresh during SSE:** `@microsoft/fetch-event-source` supports an `onopen` callback and custom `fetch` implementation. The custom Refine `liveProvider` should intercept 401 responses, trigger `oidc-spa`'s silent token refresh, and retry the SSE connection with the new token. This is transparent to the user.

**Event bus implementation:** The in-process event bus is a simple pub/sub system using Go channels. Each SSE connection creates a buffered channel (e.g., capacity 64). The event bus maintains a map of `namespace -> []channel`. When an informer fires, the event is published to all channels subscribed to that namespace. If a channel's buffer is full (slow consumer), the event is dropped for that connection -- the client will catch up via the next REST fetch triggered by Refine's query invalidation.

**`Last-Event-ID` replay:** The BFF can replay missed events because each BFF pod maintains its own informer cache. The informer cache stores the current state of all watched objects with their resource versions. On reconnection, the BFF compares the `Last-Event-ID`'s resource version with the current resource version in the cache. If the resource has changed, it sends a synthetic "updated" event. This is not a full event log replay -- it only provides the current state delta. For the portal's use case (showing current status), this is sufficient.

**Keepalive interval:** The 15-second keepalive interval is chosen to be well under Cilium/Envoy's default 5-minute `stream_idle_timeout`. Even if the timeout is reduced via configuration, 15 seconds provides ample margin. The keepalive comment (`: keepalive\n\n`) adds negligible bandwidth (~15 bytes per message).

**Multiple SSE streams:** When a user navigates between pages, the frontend may open multiple SSE streams (one per resource type being watched). Over HTTP/2 (served by Cilium Gateway by default for HTTPS), these streams are multiplexed over a single TCP connection, eliminating the HTTP/1.1 six-connection-per-origin limit. The Refine `liveProvider` manages stream lifecycle -- streams are opened on component mount and closed on unmount.

**Graceful degradation:** If the SSE connection cannot be established (e.g., BFF is down), the dashboard still functions. Refine's `dataProvider` (REST CRUD) is independent of the `liveProvider` (SSE). The user sees data but without real-time updates. The `liveProvider` continues reconnection attempts in the background. When the BFF comes back online, real-time updates resume automatically.

---

## Appendix: Cross-Cutting Concerns

### Authentication Flow Summary

All API calls from the frontend to the BFF follow the same authentication pattern:

```mermaid
sequenceDiagram
    participant Frontend as Frontend (oidc-spa)
    participant BFF
    participant KC as Keycloak

    Note over Frontend: Every request includes<br/>Authorization: Bearer {access_token}

    Frontend->>BFF: Any API request<br/>Authorization: Bearer {token}

    BFF->>BFF: Extract JWT from Authorization header
    BFF->>BFF: Validate signature against<br/>cached JWKS from Keycloak

    alt Token signature invalid
        BFF-->>Frontend: 401 Unauthorized
    end

    BFF->>BFF: Check token expiration (exp claim)

    alt Token expired
        BFF-->>Frontend: 401 Unauthorized
        Frontend->>KC: Silent token refresh<br/>(via iframe + session cookie)
        KC-->>Frontend: New access token
        Frontend->>BFF: Retry original request<br/>Authorization: Bearer {new_token}
    end

    BFF->>BFF: Extract claims:<br/>sub, email, groups, tenant_id
    BFF->>BFF: Derive tenant namespace<br/>from group membership
    BFF->>BFF: Check authorization<br/>(role + tenant access)

    alt Insufficient permissions
        BFF-->>Frontend: 403 Forbidden
    end

    BFF->>BFF: Process request with<br/>tenant-scoped context
    BFF-->>Frontend: 200 OK (response)
```

### Error Response Format

All BFF error responses follow a consistent JSON structure:

```json
{
  "error": {
    "code": "TENANT_NOT_FOUND",
    "message": "Tenant 'acme-corp' does not exist or you do not have access",
    "details": {
      "tenantId": "acme-corp",
      "requiredRole": "admin"
    }
  }
}
```

HTTP status codes used:
- `400` -- Malformed request (invalid JSON, missing required fields)
- `401` -- Invalid or expired JWT
- `403` -- Valid JWT but insufficient permissions
- `404` -- Resource not found (or tenant not accessible to user)
- `409` -- Conflict (e.g., tenant name already exists)
- `422` -- Validation error (e.g., invalid plan, quota exceeded)
- `429` -- Rate limited (propagated from KillBill or BFF-level rate limiting)
- `500` -- Internal server error (K8s API unreachable, unexpected failure)
- `502` -- Upstream dependency failed (KillBill down, Keycloak unreachable)
- `503` -- BFF not ready (informer caches not synced)

### SSE Event Format

All SSE events from the BFF follow the format defined in the [SSE specification](https://html.spec.whatwg.org/multipage/server-sent-events.html):

```
id: {CRDKind}/{namespace}/{name}@{resourceVersion}
event: {created|updated|deleted}
retry: 3000
data: {"type":"updated","channel":"resources/{resource}","payload":{"ids":["{name}"]},"phase":"{phase}","message":"{human-readable}"}

```

The `data` field contains a JSON object compatible with Refine's `LiveEvent` interface, extended with portal-specific fields (`phase`, `message`) for the provisioning UI.

---

## References

- [Architecture Analysis: BFF Monolith vs Microservices](architecture-analysis.md)
- [API Paradigm Comparison: REST vs GraphQL vs tRPC](api-paradigm-comparison.md)
- [Real-Time Protocol Comparison: WebSocket vs SSE](realtime-protocol-comparison.md)
- [Refine liveProvider Documentation](https://refine.dev/docs/realtime/live-provider/)
- [@microsoft/fetch-event-source](https://www.npmjs.com/package/@microsoft/fetch-event-source)
- [oidc-spa Documentation](https://github.com/keycloakify/oidc-spa)
- [KillBill API Documentation](https://docs.killbill.io/)
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [Keycloak OIDC Endpoints](https://www.keycloak.org/docs/latest/securing_apps/#endpoints)

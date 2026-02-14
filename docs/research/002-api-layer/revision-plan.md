# API Layer Architecture — Revision Plan

**Status:** Draft — awaiting review
**Date:** 2026-02-14
**Trigger:** Architecture review of 002-api-layer documents
**Branch:** `claude/plan-api-architecture-gabPn`

---

## Summary of Required Changes

Three architectural decisions in the current 002-api-layer research need revision:

1. **GitOps-first provisioning** — BFF commits to Forgejo instead of directly writing CRDs
2. **Keycloak + K8s authorization** — move authorization policy out of BFF Go code into Keycloak Authorization Services and K8s RBAC
3. **Database** — acknowledge the need for portal-specific state (user preferences, app settings); decision on implementation deferred

These changes affect: `auth-flow-design.md`, `k8s-interaction-patterns.md`, `sequence-diagrams.md`, `architecture-analysis.md`, `draft-api-contract.md`, `adr-002-api-layer.md`, and `README.md`.

---

## 1. GitOps-First Provisioning

### Current Design (to be replaced)

The BFF directly creates multiple CRDs via the K8s API:

```
User → BFF → K8s API (create Cozystack Tenant CR)
                    → K8s API (create ForgejaTenant CR)
                    → K8s API (create KeycloakRealmGroup CR)
```

The BFF has write access to all CRD types via a ClusterRole. Provisioning order, rollback, and retry logic live in BFF Go code.

### Revised Design

The BFF commits a **single composite CRD manifest** to a Forgejo repository. Flux applies the manifest to the cluster. Crossplane Composition (or a dedicated operator) expands the single CRD into all sub-resources.

```
User → BFF → Forgejo API (commit PortalTenant manifest)
                ↓
             Flux (detects new commit, applies manifest)
                ↓
             K8s API (PortalTenant CR created)
                ↓
             Crossplane Composition / Operator
                ↓
             Cozystack Tenant + ForgejaTenant + KeycloakRealmGroup + ...
```

### Single Composite CRD: `PortalTenant`

Instead of the BFF orchestrating multiple CRDs, a single `PortalTenant` CRD contains the full tenant specification. A Crossplane Composition (or dedicated operator) expands it into sub-resources.

```yaml
apiVersion: platform.tedatech.app/v1alpha1
kind: PortalTenant
metadata:
  name: acme-corp
  namespace: platform-portal    # or a dedicated gitops namespace
  labels:
    tedatech.app/portal-managed: "true"
spec:
  tenantId: acme-corp
  displayName: "Acme Corporation"
  adminEmail: admin@acme-corp.com
  plan: professional

  # Cozystack tenant settings
  infrastructure:
    cpu: "8"
    memory: "16Gi"
    storage: "200Gi"

  # Forgejo settings
  forgejo:
    enabled: true
    organizationName: acme-corp
    gitopsRepo: true
    repoTemplate: "tedatech/gitops-template"

  # Keycloak settings
  auth:
    realm: cozy
    groups:
      - admins
      - members
      - viewers

status:
  phase: Provisioning   # Pending | Provisioning | Active | Failed | Deleting
  conditions:
    - type: NamespaceReady
      status: "True"
    - type: CozystackTenantReady
      status: "True"
    - type: ForgejaTenantReady
      status: "False"
      reason: Provisioning
    - type: KeycloakReady
      status: "False"
      reason: Pending
    - type: Ready
      status: "False"
      reason: Provisioning
      message: "2 of 4 components provisioned"
```

### BFF → Forgejo Commit Flow

The BFF renders the `PortalTenant` manifest from the user's input and commits it to the platform GitOps repository in Forgejo:

```
Platform GitOps repo (e.g., tedatech/platform-gitops)
├── tenants/
│   ├── acme-corp.yaml          ← PortalTenant manifest
│   ├── other-tenant.yaml
│   └── ...
├── platform/
│   ├── crossplane-compositions.yaml
│   └── ...
└── flux-system/
    └── kustomization.yaml
```

The BFF uses the Forgejo API to:
1. Read the current state of the repo (or the specific tenant file)
2. Create/update the tenant manifest file
3. Commit directly to `main` (for creates) or create a PR (for updates/deletes)

For tenant creation, a direct commit to `main` is acceptable — Flux picks it up immediately. For destructive operations (tenant deletion, plan downgrade), a PR-based flow adds a review gate.

### Impact on BFF K8s Access

| Operation | Current | Revised |
|-----------|---------|---------|
| Create tenant resources | BFF writes CRDs directly | BFF commits to Forgejo; Flux applies |
| Update tenant config | BFF patches CRDs directly | BFF commits updated manifest; Flux applies |
| Delete tenant | BFF deletes CRDs directly | BFF commits removal; Flux applies |
| Read tenant status | BFF reads from informer cache | **Unchanged** — BFF still reads from informer cache |
| Watch real-time events | BFF informers → EventBus → SSE | **Unchanged** — informers still watch status |
| Create Pipeline CR | BFF writes Pipeline CRD | BFF commits to Forgejo (consistent with above) |
| Billing webhook | BFF patches Cozystack Tenant CR quota | BFF commits updated manifest; Flux applies |

The BFF's ClusterRole changes from **read-write** to **read-only** for all CRD types:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portal-bff
rules:
  # All CRDs: read-only (informers for status watching)
  - apiGroups: ["platform.tedatech.app"]
    resources: ["portaltenants", "portaltenants/status"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps.cozystack.io"]
    resources: ["tenants", "tenants/status"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["v1.edp.epam.com"]
    resources: ["keycloakclients", "keycloakrealmusers", "keycloakrealmgroups"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["source.toolkit.fluxcd.io"]
    resources: ["gitrepositories", "gitrepositories/status"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["kustomize.toolkit.fluxcd.io"]
    resources: ["kustomizations", "kustomizations/status"]
    verbs: ["get", "list", "watch"]
  # Namespaces: read-only
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch"]
  # Secrets: read-only (KillBill credentials)
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
```

### New: Forgejo API Client

The BFF needs a Forgejo API client for git operations:

```
BFF
├── internal/
│   ├── forgejo/
│   │   ├── client.go           # Forgejo API client (REST)
│   │   ├── commit.go           # Create/update file via API
│   │   ├── pullrequest.go      # Create PR for destructive ops
│   │   └── client_test.go
```

Forgejo API operations used:
- `GET /api/v1/repos/{owner}/{repo}/contents/{filepath}` — read current manifest
- `POST /api/v1/repos/{owner}/{repo}/contents/{filepath}` — create file (commit)
- `PUT /api/v1/repos/{owner}/{repo}/contents/{filepath}` — update file (commit)
- `DELETE /api/v1/repos/{owner}/{repo}/contents/{filepath}` — delete file (commit)
- `POST /api/v1/repos/{owner}/{repo}/pulls` — create PR for destructive ops

Authentication: Forgejo API token stored in a K8s Secret, loaded by the BFF at startup.

### Real-Time Feedback With GitOps Delay

GitOps introduces latency between the BFF committing and the resources being applied:

```
BFF commits manifest → Flux sync interval (default 1 min) → Flux applies → Crossplane expands → Operators reconcile
```

The BFF can bridge this gap:
1. **Immediately after commit**: BFF returns `202 Accepted` with `status: "committed"`
2. **SSE reports Flux sync**: Informers detect the new `PortalTenant` CR appearing (Flux applied it)
3. **SSE reports sub-resource progress**: Informers detect Crossplane creating sub-resources
4. **SSE reports completion**: All sub-resources reach `Ready=True`

New SSE event phases for the frontend:
- `Committed` — manifest committed to Forgejo, waiting for Flux sync
- `Syncing` — Flux detected the change, applying manifests
- `Provisioning` — Crossplane/operators creating sub-resources
- `Active` — all sub-resources ready
- `Failed` — provisioning failed (with reason)

### Affected Documents

| Document | Change |
|----------|--------|
| `k8s-interaction-patterns.md` | Remove direct CRD write patterns (Create, Update, Delete). Add Forgejo commit patterns. Keep informer/read patterns. Reduce ClusterRole to read-only. |
| `sequence-diagrams.md` | Rewrite "Tenant Provisioning Flow" (#2) to show Forgejo commit → Flux → Crossplane chain. Update "Pipeline Trigger Flow" (#3) for consistency. Update "Billing Subscription Change" (#4) webhook to commit instead of direct patch. |
| `architecture-analysis.md` | Update BFF description — it writes to Forgejo, not K8s. Update advantages (no direct K8s write access = smaller blast radius). Update module structure to include `forgejo/` package. |
| `draft-api-contract.md` | Update response codes — `POST /api/tenants` returns `202 Accepted` (committed, not yet applied). Add commit reference in response body. |
| `adr-002-api-layer.md` | Add GitOps-first as an architectural decision. Update K8s interaction description. Add Forgejo API as a third backend. |
| `README.md` | Update summary: BFF commits to Forgejo for state changes, reads from K8s for status. |

---

## 2. Authorization: Keycloak Authorization Services + K8s RBAC

### Current Design (to be replaced)

Authorization is hard-coded in BFF Go middleware:

```go
// Current: role check in Go code
func (c *Claims) HasRole(required string) bool {
    roleHierarchy := map[string]int{
        "viewer": 1, "member": 2, "admin": 3,
    }
    // ... parse groups, compare levels
}
```

The authorization matrix (which role can access which endpoint) is scattered across handler functions. Changing a permission requires redeploying the BFF.

### Revised Design: Three-Layer Authorization

Authorization is split across three layers, each handling what it does best:

#### Layer 1: Keycloak Authorization Services (policy decisions)

Keycloak defines resources, scopes, and policies. The BFF evaluates permissions via Keycloak's token endpoint or the Authorization API rather than implementing its own role logic.

**Resources** (what can be accessed):
```
tenant                  # Tenant-level operations
tenant:forgejo          # Forgejo operations within a tenant
tenant:pipeline         # Pipeline operations within a tenant
tenant:billing          # Billing operations within a tenant
tenant:members          # Member management within a tenant
```

**Scopes** (what actions):
```
view                    # Read access
create                  # Create new resources
update                  # Modify existing resources
delete                  # Remove resources
```

**Policies** (who gets what):
```
viewer-policy:  groups=/tenants/{id}/viewers  → tenant:view
member-policy:  groups=/tenants/{id}/members  → tenant:view, tenant:forgejo:view, tenant:billing:view
admin-policy:   groups=/tenants/{id}/admins   → tenant:*, tenant:forgejo:*, tenant:pipeline:*, tenant:billing:*, tenant:members:*
```

**Keycloak CRD configuration:**

```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakClient
metadata:
  name: portal-spa
  namespace: platform-keycloak
spec:
  clientId: portal-spa
  # ... existing config ...

  # Enable authorization services
  authorizationServicesEnabled: true
  serviceAccountsEnabled: false       # public client, no service account

  # Authorization settings
  authorizationSettings:
    decisionStrategy: UNANIMOUS
    resources:
      - name: "tenant"
        type: "urn:portal:resource:tenant"
        scopes: ["view", "create", "update", "delete"]
      - name: "tenant:forgejo"
        type: "urn:portal:resource:forgejo"
        scopes: ["view", "create", "delete"]
      - name: "tenant:pipeline"
        type: "urn:portal:resource:pipeline"
        scopes: ["view", "create"]
      - name: "tenant:billing"
        type: "urn:portal:resource:billing"
        scopes: ["view", "update"]
      - name: "tenant:members"
        type: "urn:portal:resource:members"
        scopes: ["view", "create", "update", "delete"]

    policies:
      - name: "viewer-policy"
        type: "group"
        logic: POSITIVE
        groups:
          - path: "/tenants/*/viewers"

      - name: "member-policy"
        type: "group"
        logic: POSITIVE
        groups:
          - path: "/tenants/*/members"

      - name: "admin-policy"
        type: "group"
        logic: POSITIVE
        groups:
          - path: "/tenants/*/admins"

    permissions:
      - name: "viewer-permission"
        type: "scope"
        resources: ["tenant"]
        scopes: ["view"]
        policies: ["viewer-policy", "member-policy", "admin-policy"]
        decisionStrategy: AFFIRMATIVE

      - name: "member-permission"
        type: "scope"
        resources: ["tenant:billing"]
        scopes: ["view"]
        policies: ["member-policy", "admin-policy"]
        decisionStrategy: AFFIRMATIVE

      - name: "admin-permission"
        type: "scope"
        resources: ["tenant", "tenant:forgejo", "tenant:pipeline", "tenant:billing", "tenant:members"]
        scopes: ["create", "update", "delete"]
        policies: ["admin-policy"]
```

#### Layer 2: BFF Authorization Middleware (policy enforcement)

The BFF no longer implements authorization logic. Instead, it asks Keycloak whether the current token has the required permission for the requested resource+scope.

**Option A: Token introspection with permissions (RPT)**

The BFF requests a Requesting Party Token (RPT) from Keycloak that includes the user's permissions. This can be done once per request or cached.

```go
// BFF middleware: evaluate permission via Keycloak
func (a *Authorizer) CheckPermission(ctx context.Context, token string, resource string, scope string) (bool, error) {
    // POST to Keycloak's token endpoint with grant_type=urn:ietf:params:oauth:grant-type:uma-ticket
    // Returns 200 if permitted, 403 if denied
    resp, err := a.keycloakClient.EvaluatePermission(ctx, token, resource, scope)
    if err != nil {
        return false, err
    }
    return resp.Allowed, nil
}
```

**Option B: Embed permissions in the access token**

Configure Keycloak to include resolved permissions in the access token via a protocol mapper. The BFF validates locally without a round-trip.

```json
{
  "authorization": {
    "permissions": [
      {"rsname": "tenant", "scopes": ["view"]},
      {"rsname": "tenant:forgejo", "scopes": ["view", "create"]},
      {"rsname": "tenant:billing", "scopes": ["view", "update"]}
    ]
  }
}
```

**Recommendation:** Option B for the MVP (no extra round-trip per request). Option A for fine-grained real-time policy changes.

#### Layer 3: Kubernetes RBAC (infrastructure boundary)

K8s RBAC provides the infrastructure-level security boundary. Even if the BFF is compromised, it can only read CRDs — it cannot write anything to the cluster (GitOps-first means all writes go through Forgejo → Flux).

```
Keycloak:  "Is this user allowed to create a tenant?"        (policy)
BFF:       "Enforce the Keycloak decision, route the request" (enforcement)
K8s RBAC:  "The BFF ServiceAccount can only read CRDs"        (infrastructure boundary)
Forgejo:   "The BFF API token can commit to platform-gitops"   (write boundary)
```

### Impact on Code Structure

```
BFF
├── internal/
│   ├── auth/
│   │   ├── jwt.go              # JWT validation (keyfunc/v3) — KEEP
│   │   ├── claims.go           # Claims extraction — KEEP
│   │   ├── middleware.go       # Auth middleware — KEEP (JWT + tenant resolution)
│   │   ├── authz.go            # NEW: Keycloak permission evaluation
│   │   └── authz_middleware.go # NEW: Per-route permission middleware
│   ├── tenant/
│   │   ├── resolver.go         # Tenant resolution from claims — KEEP
│   │   └── mapper.go           # Tenant-to-namespace mapping — KEEP
```

The `HasRole()` function and the hard-coded authorization matrix in handlers are **removed**. Each route is annotated with the required resource+scope, and the middleware evaluates it against Keycloak.

### Affected Documents

| Document | Change |
|----------|--------|
| `auth-flow-design.md` | Replace Section 6 (Multi-Tenancy Authorization Matrix) with Keycloak Authorization Services design. Replace `HasRole()` code with Keycloak permission evaluation. Update middleware chain. Add Keycloak Authorization Services CRD config. |
| `k8s-interaction-patterns.md` | Update Section 8 (Namespace Strategy and RBAC) — ClusterRole is now read-only. Application-level auth delegates to Keycloak. |
| `sequence-diagrams.md` | Update auth checks in all flows — "Check admin role in claims" becomes "Evaluate permission via Keycloak". |
| `draft-api-contract.md` | Update security section — permissions come from Keycloak, not BFF-internal role checks. |
| `adr-002-api-layer.md` | Add Keycloak Authorization Services as an auth decision. Update consequences. |

---

## 3. Database: Deferred Decision

### Current Design

No database. All state lives in:
- K8s etcd (CRDs, Secrets, ConfigMaps)
- Keycloak (users, groups, policies)
- KillBill (billing, subscriptions, invoices)
- Forgejo (source code, GitOps manifests)

### Why a Database Might Be Needed

With the GitOps-first approach and Keycloak authorization, the following state has no natural home:

| State | Current Home | Problem |
|-------|-------------|---------|
| User preferences (theme, language, dashboard layout) | Nowhere | CRDs are wrong tool for UI preferences |
| App settings per tenant (notification prefs, webhook configs) | ConfigMap? | ConfigMaps lack indexing, querying, versioning |
| Audit trail | Stdout logs | Logs are ephemeral; compliance may need queryable history |
| Commit-to-operation mapping | Nowhere | Tracking which Forgejo commit corresponds to which user action |
| Pending operation state | Nowhere | Between BFF commit and Flux apply, state is in-flight |

### Decision

**Deferred.** The database decision is not blocking for the architecture revision. The BFF module structure will include a `store/` package with an interface, allowing a database to be added later without restructuring.

```go
// internal/store/store.go
type Store interface {
    // User preferences
    GetUserPreferences(ctx context.Context, userSub string) (*UserPreferences, error)
    SetUserPreferences(ctx context.Context, userSub string, prefs *UserPreferences) error

    // Operation tracking
    RecordOperation(ctx context.Context, op *Operation) error
    GetOperation(ctx context.Context, id string) (*Operation, error)
    ListOperations(ctx context.Context, tenantID string, opts ListOpts) ([]Operation, error)
}
```

If a database is chosen, PostgreSQL is the natural fit — it's already operated for customers and Keycloak uses it.

### Portal's Own Tenant

The portal dashboard runs in its own tenant / sub-cluster. This means:
- The portal can use the same Cozystack infrastructure patterns as customer tenants
- A PostgreSQL instance for the portal is just another tenant service
- No special infrastructure needed — same operational model as customer workloads

### Affected Documents

| Document | Change |
|----------|--------|
| `architecture-analysis.md` | Add `store/` package to module structure. Note database as deferred decision. |
| `adr-002-api-layer.md` | Add database deferral as explicit non-decision with trigger conditions. |

---

## 4. Document Update Plan

### Priority Order

The documents should be updated in this order (each builds on the previous):

1. **`adr-002-api-layer.md`** — update the core decisions first (GitOps, auth, database deferral)
2. **`auth-flow-design.md`** — replace BFF-internal auth with Keycloak Authorization Services + K8s RBAC
3. **`k8s-interaction-patterns.md`** — reduce to read-only patterns, add Forgejo commit patterns
4. **`sequence-diagrams.md`** — rewrite provisioning flow, update all auth checks
5. **`architecture-analysis.md`** — update module structure, advantages/disadvantages
6. **`draft-api-contract.md`** — update response codes, add commit references
7. **`README.md`** — update summary table

### What Stays The Same

These decisions are **not changing**:

- Go as the backend language
- REST (OpenAPI 3.0) as the API paradigm
- SSE for real-time events
- BFF Monolith architecture pattern
- Chi + oapi-codegen for the Go server
- orval for TypeScript client generation
- oidc-spa for frontend OIDC
- Keycloak as the OIDC provider (enhanced with Authorization Services)
- controller-runtime for K8s informers (read-only now)
- In-process EventBus for SSE fan-out
- KillBill integration pattern (per-tenant API keys)

### New Components Introduced

| Component | Purpose |
|-----------|---------|
| `PortalTenant` CRD | Single composite CRD per tenant, expanded by Crossplane |
| Crossplane Composition | Expands `PortalTenant` into sub-resources |
| Forgejo API client (in BFF) | Commits manifests to platform GitOps repo |
| Keycloak Authorization Services | Centralized authorization policy |
| `store/` interface (in BFF) | Future database abstraction |

### Open Questions for Follow-Up

1. **Crossplane vs. custom operator** for expanding `PortalTenant` — Crossplane Compositions are declarative but can be limiting. A custom Go operator (similar to hetzner-operators patterns) gives more control. Decision needed before implementation.

2. **Forgejo commit strategy** — direct commit to `main` vs. always-PR. Direct commits are faster for non-destructive operations. PRs add a review gate but slow down provisioning.

3. **Flux sync interval** — default is 1 minute. For faster provisioning feedback, this can be reduced to 15-30 seconds for the platform GitOps repo, or Flux can be notified via webhook from Forgejo.

4. **Keycloak Authorization Services: RPT vs. embedded permissions** — the plan recommends embedded permissions (Option B) for MVP. Validate that the EDP Keycloak Operator supports `authorizationSettings` in the `KeycloakClient` CRD.

5. **Database choice** — if needed, PostgreSQL via Cozystack (portal runs in its own tenant). Exact trigger conditions TBD.

---

## Summary of Changes

| Area | Current | Revised |
|------|---------|---------|
| **Provisioning** | BFF writes CRDs directly to K8s | BFF commits manifests to Forgejo → Flux → Crossplane |
| **CRD model** | Multiple CRDs created individually by BFF | Single `PortalTenant` CRD, Crossplane expands |
| **BFF K8s access** | Read-write ClusterRole | Read-only ClusterRole |
| **BFF write target** | K8s API | Forgejo API |
| **Authorization policy** | Hard-coded `HasRole()` in Go | Keycloak Authorization Services |
| **Authorization enforcement** | BFF middleware parses groups | BFF evaluates Keycloak permissions |
| **K8s RBAC role** | Primary write access | Infrastructure security boundary (read-only) |
| **Database** | None (K8s etcd is the database) | Deferred; interface prepared |
| **Portal deployment** | Assumed in-cluster | Own tenant / sub-cluster |

# ADR-002: Portal Architecture and Tenant Model

## Status

Proposed

## Date

2026-02-15

## Context

The Customer Portal needs to:
- Authenticate users against Cozystack's existing Keycloak instance
- Allow users to create and remove tenants on the hosting platform
- Integrate with the existing GitOps/Flux-managed infrastructure
- Act as the glue between billing (KillBill), the frontend, and the Kubernetes API
- Connect Kubernetes OpenCost to the billing system
- Scale to the initial sales target (< 1,000 tenants)

The platform runs on a single Kubernetes cluster managed by Cozystack, with Flux watching a central gitops repo for infrastructure state.

## Decision Drivers

- **No custom auth**: the platform already has Keycloak and Kubernetes RBAC -- adding another auth layer adds complexity with no benefit
- **GitOps-native**: the cluster is Flux-managed, so tenant lifecycle should flow through git, not imperative API calls
- **Simplicity for MVP**: defer per-tenant portals, JWT tuning, and multi-cluster until needed
- **Cozystack alignment**: work with the existing tenant model rather than building a parallel one

## Decision

### 1. Authentication: Cozystack Single Keycloak Realm

Users authenticate via the existing `cozy` Keycloak realm. The portal is a standard OIDC client in this realm. Kubernetes RBAC -- driven by Keycloak group membership -- determines what each user can see and do.

The BFF is the glue between the frontend, Kubernetes API, and billing (KillBill). It validates tokens, proxies requests, manages tenant lifecycle, and connects OpenCost to billing. All permission decisions are made by Kubernetes RBAC.

**Keycloak groups per tenant (created by Cozystack):**
- `{tenant}-view` -- read-only access
- `{tenant}-use` -- view + interactive access (VNC)
- `{tenant}-admin` -- use + CRUD on applications
- `{tenant}-super-admin` -- admin + sub-tenant management

### 2. Tenant Provisioning: BFF Commits to GitOps Repo

The BFF commits a single Kubernetes resource per tenant to the main gitops repo (one file per tenant, with all tenant configuration contained in this resource). The exact resource type is TBD but Flux reconciles these into Cozystack tenants.

```
User clicks "Create Tenant" in portal
  --> BFF validates token + RBAC
  --> BFF commits tenant resource YAML to gitops repo (one file per tenant)
  --> Flux detects change, reconciles
  --> Cozystack creates namespace, RBAC, Keycloak groups, network policies
  --> Portal shows tenant status via Kubernetes API watch
```

Tenant removal follows the same path in reverse: BFF removes the tenant resource from git, Flux prunes the resources.

### 3. No Per-Tenant Portal (MVP)

For MVP, there is one portal instance. Users who need deeper management (monitoring, logs, VM access) are linked to:
- Cozystack dashboard (scoped by RBAC)
- Grafana dashboards (scoped by tenant namespace)
- Other platform tools as needed

Per-tenant portal instances are a future consideration, not an MVP requirement.

### 4. No JWT Token Tuning (MVP)

The portal accepts Keycloak's default token behavior. Group claims are included in the JWT and passed through to the Kubernetes API server for RBAC evaluation.

**Known limitation:** this model hits HTTP header size limits when a single user belongs to many tenant groups. See [Scaling Limits](./scaling-limits.md) for details.

## Consequences

### Positive

- Zero custom auth code -- all authentication and authorization handled by battle-tested infrastructure (Keycloak + Kubernetes RBAC)
- GitOps-native provisioning -- full audit trail in git, rollback via git revert, no imperative state drift
- Cozystack-aligned -- tenants created by the portal are identical to tenants created manually, no shadow state
- BFF as integration layer -- the backend connects billing (KillBill), OpenCost, the frontend, and K8s API. No user database, no session store, no permission tables

### Negative

- Scaling ceiling at ~250 tenants before JWT/proxy tuning is needed (see [Scaling Limits](./scaling-limits.md))
- Platform admin UX depends on Cozystack dashboard quality for deeper operations

### Risks

| Risk | Mitigation |
|------|-----------|
| Keycloak group count degrades performance | Ceiling is ~2,500 tenants (10K groups). Well above MVP target. Monitor Keycloak response times. |
| JWT token too large for platform admins spanning many tenants | Hits at ~50-100 tenants per user with default proxy configs. Tune nginx `large_client_header_buffers` when needed. |
| Flux reconciliation lag frustrates users | Flux reconciles in seconds for < 1,000 tenant resources. Add webhook receivers for faster feedback if needed. |
| GitOps repo becomes single point of failure | Flux retries on failure. Repo will move to self-hosted Forgejo. Standard HA practices apply. |

## Impact on Related Decisions

- **Issue #1 (Frontend)**: SPA uses `oidc-spa` to authenticate against the `cozy` Keycloak realm. TanStack Router adapter handles protected routes.
- **Issue #3 (Keycloak)**: No custom realm needed. Portal is an OIDC client in the `cozy` realm. Keycloakify themes apply to the `cozy` realm login page.
- **Issue #6 (Onboarding)**: Onboarding wizard submits to BFF, which commits tenant resource to gitops repo. Real-time status via Kubernetes API watch through Refine's `liveProvider`.

## References

- [Scaling Limits](./scaling-limits.md)
- [Cozystack Tenant Documentation](https://cozystack.io/docs/guides/tenants/)
- [Cozystack OIDC Setup](https://cozystack.io/docs/operations/oidc/enable_oidc/)
- [FluxCD Documentation](https://fluxcd.io/flux/)

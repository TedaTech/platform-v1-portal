# ADR-003: Platform Orchestration and Workflow Engine

## Status

Proposed

## Date

2026-02-15

## Context

The Customer Portal needs to orchestrate complex, multi-step business processes that span seconds to weeks:

- **Customer onboarding**: tenant creation, billing setup, initial app provisioning
- **App lifecycle**: backup, progressive rollout (canary/blue-green), health verification, automatic rollback on failure
- **GDPR data erasure**: coordinated deletion across multiple systems with human verification

These workflows must survive infrastructure restarts, support human-in-the-loop decisions (approval gates, manual overrides), and provide audit trails for compliance.

ADR-002 established the BFF as a thin gateway that starts Temporal workflows and reads from multiple backends. This ADR defines the Temporal workflow architecture: the BFF starts and signals workflows, Temporal workers execute all business logic and side effects (GitOps commits, external API calls).

## Decision Drivers

- **Full app lifecycle visibility**: users and platform operators must see exactly where an upgrade is (backup in progress, canary at 10%, health check passed, etc.)
- **Compliance by design**: reference-based payloads, immutable audit logs, and RBAC extensions are architectural decisions that cannot be retrofitted into a workflow engine after the fact
- **PostgreSQL as shared database**: already needed for KillBill billing. Adding Temporal to the same PostgreSQL cluster avoids operational overhead of multiple database systems
- **No etcd/CRD pressure**: workflow state must live in PostgreSQL, not as Kubernetes custom resources. At 1,000 tenants with frequent operations, CRD-based workflow engines (Tekton, Argo Workflows) would pressure the etcd store that the entire cluster depends on
- **Go ecosystem alignment**: the Kubernetes ecosystem, Temporal's most mature SDK, and the Cozystack platform are all Go. A single language for BFF, workers, and infrastructure tooling reduces cognitive overhead and dependency sprawl
- **Monorepo for shared types**: the BFF starts workflows and workers execute them -- both need the same Go types (workflow inputs, activity interfaces). Splitting across repos creates version drift and import cycle headaches. One repo, one `go.mod`, one CI pipeline

## Decision

### 1. Temporal (Self-Hosted) as Central Workflow Orchestrator

All multi-step business processes run as Temporal workflows. Temporal is a portal component, deployed self-hosted on the same Kubernetes cluster. It uses PostgreSQL as its persistence backend (PostgreSQL is provided by the infra layer).

**Why Temporal over alternatives:**

| Requirement | Temporal | Tekton / Argo Workflows | Hatchet | n8n / Kestra |
|---|---|---|---|---|
| Code-first (Go) | Go SDK (most mature) | YAML-based CRDs | Go SDK (newer) | YAML / visual |
| Long-running (days/weeks) | Durable sleep, zero resource while waiting | Not designed for this | Supports but less proven | Limited |
| Human-in-the-loop | Signals for async input, Queries for state | Not built-in | Not built-in | Manual approval nodes |
| State storage | PostgreSQL | etcd (CRDs) | PostgreSQL | PostgreSQL |
| Audit trail | Immutable Event History (event sourcing) | CRD status fields | Event log | Execution log |
| Versioning in-flight | Worker Versioning, Patching API | Not applicable | Basic | Not applicable |
| License | MIT | Apache 2.0 | MIT | Various |

**Temporal deployment topology:**

```
Temporal Cluster (self-hosted on K8s):
  frontend   (1-2 replicas)  -- gRPC API gateway
  history    (1-2 replicas)  -- workflow state management
  matching   (1-2 replicas)  -- task queue routing
  worker     (1 replica)     -- internal system workflows

Persistence: PostgreSQL (shared cluster, dedicated database)
Visibility:  PostgreSQL (same cluster, dedicated database or schema)

Workers (application code):
  portal-worker     -- handles all portal workflows (onboarding, app upgrade, GDPR erasure)
                    -- additional workers can be split out later if needed
```

**Resource estimate (low-concurrency starting point):**

| Component | CPU Request | Memory Request | Replicas |
|---|---|---|---|
| Frontend | 0.5 core | 512 Mi | 1 |
| History | 0.5 core | 512 Mi | 1 |
| Matching | 0.25 core | 256 Mi | 1 |
| Worker | 0.25 core | 256 Mi | 1 |
| **Total Temporal** | **1.5 cores** | **1.5 Gi** | **4 pods** |
| Application worker | 0.5 core | 256 Mi | 1 pod |

Scale replicas as concurrency grows. Temporal's history service can be sharded across replicas when needed.

### 2. Go for BFF and Temporal Workers

The BFF (from ADR-002) and all Temporal worker processes are written in Go.

**BFF responsibilities (revised from ADR-002):**

```
BFF reads from:
  - Kubernetes API (tenant status, app status -- proxied with RBAC)
  - OpenCost (cost/usage data for display to users)
  - KillBill (billing, invoices, subscriptions for display to users)
  - Temporal (workflow status via queries)
  - External CRM (customer timeline for display -- CRM choice TBD)

BFF writes to:
  - Temporal only (start workflows, send signals)

BFF does NOT:
  - Commit to GitOps repo (Temporal worker activities do this)
  - Send email (external CRM / comms system handles this)
  - Own CRM data (reads from external CRM)
  - Connect OpenCost to KillBill (displays both independently)
```

The BFF is a thin read-gateway + Temporal trigger. It translates HTTP/WebSocket requests from the portal into Temporal workflow operations and reads from multiple backends for display. All business logic and side effects live in Temporal workflow and activity definitions.

### 3. PostgreSQL as Shared Persistence

A single PostgreSQL cluster (with logical separation) serves platform data needs:

| Database / Schema | Owner | Purpose |
|---|---|---|
| `temporal` | Temporal cluster | Workflow execution state |
| `temporal_visibility` | Temporal cluster | Workflow search and filtering |
| `killbill` | KillBill | Billing, subscriptions, invoices |

The portal (BFF) does not own a database. It reads from Temporal, KillBill, OpenCost, Kubernetes API, and an external CRM. If the chosen CRM is PostgreSQL-backed and co-located on this cluster, live updates via PostgreSQL LISTEN/NOTIFY become straightforward -- but this depends on the CRM choice (separate decision).

**Why shared PostgreSQL:**

- Temporal requires PostgreSQL (or MySQL/Cassandra). PostgreSQL is already needed for KillBill
- Operational overhead of one database cluster vs. multiple is significant at small team scale
- Logical separation (separate databases within one cluster) provides isolation without operational cost
- Backup and HA is configured once for the cluster
- If the CRM is also PostgreSQL-backed, it can join this cluster -- one more database in the same cluster
- Can split into separate clusters later if load requires it

### 4. Flagger for Progressive App Delivery

App upgrades orchestrated by Temporal use Flagger for the traffic-shifting phase:

```
Temporal Workflow: "app_upgrade"
  1. Activity: create backup (K8s Job via GitOps commit)
  2. Activity: wait for backup completion (poll Job status)
  3. Activity: commit new app version to GitOps repo
  4. Activity: Flux reconciles, Flagger begins canary rollout
  5. Activity: poll Flagger Canary status (progressive traffic shift)
  6. Activity: if Flagger promotes → [CRM] record success
  7. Activity: if Flagger rolls back → [CRM] record failure, notify user
  8. Signal: user can manually promote or rollback at any step
```

Flagger adds one CRD (`Canary`) to the cluster. Traffic shifting is based on Kubernetes Gateway API or Istio/Linkerd service mesh metrics. Flagger handles the mechanics of canary analysis; Temporal orchestrates the full lifecycle around it.

### 5. CRM and Email: External System, Not Portal-Owned

CRM (customer timeline, interaction tracking) and email (welcome, payment reminders, notifications) are **not part of the portal**. The portal's BFF reads from the CRM for display; it does not own the CRM data model, email templates, or delivery infrastructure.

**What the portal does:**
- BFF reads CRM timeline data for display in the frontend (API TBD, depends on CRM choice)
- BFF reads workflow status from Temporal for live progress display
- Temporal workflows may call CRM/email APIs as activities (the integration depends on CRM choice)

**What the portal does NOT do:**
- Own a `crm_timeline` table or any CRM data model
- Send email directly (no SendGrid, no email templates in the portal)
- Record CRM events (that's the CRM system's responsibility)

**CRM choice is a separate, unresolved decision.** It determines:
- How the BFF reads timeline data (REST API, direct PostgreSQL query, GraphQL)
- How Temporal workers record events (CRM API call as activity, database write, message queue)
- Whether live updates are easy (PostgreSQL-backed CRM on same cluster = LISTEN/NOTIFY) or require polling/webhooks
- Email provider and template management

**Reference-based payloads still apply.** Regardless of CRM choice, Temporal workflow inputs must use reference IDs (Keycloak user IDs, KillBill account refs), never PII. The CRM resolves references to display names at read time.

### 7. Compliance Architecture

Three-layer data separation enforced from day one:

```
Layer 1: Workflow Execution State (Temporal PostgreSQL)
  - Mutable: in-flight workflows can be signaled, cancelled, retried
  - Contains: workflow IDs, activity results, timer state
  - Contains NO PII: payloads use reference IDs only
  - Retention: 24 months (configurable per namespace)

Layer 2: Audit / CRM Log (External CRM)
  - Append-only timeline of customer interactions
  - Contains: who did what, when, with what outcome
  - Contains NO PII: actor_ref and subject_ref are Keycloak IDs
  - Owned and managed by the CRM system (not the portal)
  - Portal BFF reads from this layer for display
  - Retention: governed by CRM system configuration

Layer 3: PII Store (Keycloak + KillBill)
  - Mutable: subject to GDPR erasure requests
  - Contains: email, name, billing address, payment methods
  - Referenced by ID from Layer 1 and Layer 2
  - Deletion does not affect Layer 1 or Layer 2 integrity
```

**Reference-based workflow payloads (mandatory pattern):**

```go
// WRONG: PII embedded in workflow input
type OnboardingInput struct {
    Email string // PII in workflow history forever
    Name  string // PII in workflow history forever
    Plan  string
}

// RIGHT: references only
type OnboardingInput struct {
    TenantID string // opaque identifier
    UserRef  string // Keycloak user ID
    PlanRef  string // billing plan reference
}

// Activities resolve PII at point of use:
func (a *Activities) SendWelcomeEmail(ctx context.Context, input SendEmailInput) error {
    user, _ := a.keycloak.GetUser(ctx, input.UserRef)  // resolve PII here
    return a.sendgrid.Send(ctx, user.Email, input.TemplateID, input.Context)
    // user.Email is NOT stored in Temporal's event history
}
```

**Workflow RBAC (extending Keycloak groups from ADR-002):**

| Keycloak Group | Permissions |
|---|---|
| `platform-workflow-viewer` | View workflow executions across all tenants |
| `platform-workflow-operator` | Trigger, signal, retry, cancel workflows across all tenants |
| `platform-workflow-admin` | Deploy workflow definitions, manage schedules, configure retention |
| `{tenant}-view` (existing) | View workflow status and CRM timeline for own tenant |
| `{tenant}-admin` (existing) | Trigger tenant-scoped workflows (request upgrade, etc.) |

The BFF validates Keycloak token group claims before proxying any workflow operation to Temporal.

### 8. Portal Visibility into Workflows

The portal frontend displays workflow state and CRM data using Refine's `liveProvider` (from ADR-001) connected to the BFF via WebSocket/SSE:

```
User Experience:

Tenant Dashboard:
  - CRM timeline: scrollable feed of all events for this tenant (read from external CRM)
  - Active workflows: cards showing current step, progress, duration (read from Temporal)
  - App status: current version, last upgrade result, next scheduled backup
  - Cost overview: OpenCost data for this tenant
  - Billing status: KillBill subscription, invoices, payment status

App Upgrade View:
  - Step indicator: backup → canary 10% → canary 50% → canary 100% → promoted
  - Live metrics: error rate, latency (from Flagger's canary analysis)
  - Manual controls: "Promote now" / "Rollback" buttons (trigger Temporal signals)
  - History: previous upgrades with outcome and duration

BFF Implementation:
  - GET  /api/v1/tenants/{id}/timeline       → read from external CRM (API TBD)
  - GET  /api/v1/tenants/{id}/costs          → read from OpenCost
  - GET  /api/v1/tenants/{id}/billing        → read from KillBill
  - GET  /api/v1/workflows/{id}/status       → Temporal QueryWorkflow
  - POST /api/v1/workflows/{id}/signal/{name} → Temporal SignalWorkflow
  - WS   /api/v1/tenants/{id}/events         → stream workflow updates (+ CRM if live-capable)
```

**Live updates depend on CRM choice.** If the CRM uses PostgreSQL on the same cluster, the BFF can use LISTEN/NOTIFY for near-real-time timeline updates. If the CRM is SaaS-based, the BFF may need to poll or receive webhooks. Temporal workflow status is always live via Temporal queries.

### 9. Monorepo: Frontend, BFF, and Workers in One Repository

The React frontend, Go BFF, and all Temporal workers live in a single repository (`platform-v1-portal`). Each is a separate build target producing a separate container image, but they share one git history, one CI pipeline, and one PR review process.

**Repository structure:**

```
platform-v1-portal/
  frontend/                    # React SPA (ADR-001)
    src/
    package.json
    vite.config.ts
    Dockerfile                 # → nginx:alpine static SPA image

  cmd/                         # Go binary entry points
    bff/                       # BFF server (HTTP/WS + Temporal client)
      main.go
    worker/                    # Temporal worker (all workflows + activities)
      main.go

  internal/                    # Shared Go packages (not importable externally)
    workflows/                 # Workflow definitions (imported by BFF to start, worker to execute)
      onboarding/
      app_upgrade/
      gdpr_erasure/
    activities/                # Activity implementations (imported by worker)
      keycloak/                # Keycloak user resolution
      killbill/                # KillBill billing operations
      gitops/                  # Git commit to GitOps repo
      flagger/                 # Canary status polling
      k8s/                     # Kubernetes API operations
    auth/                      # Keycloak token validation, RBAC checks
    config/                    # Shared configuration loading
  deploy/                      # Helm charts / Kustomize overlays
  docs/                        # Research and ADRs (existing)

  go.mod                       # Single Go module
  go.sum
  Makefile                     # Build targets: make bff, make worker-lifecycle, etc.
  Dockerfile.bff               # Multi-stage: build Go → scratch/distroless
  Dockerfile.worker            # Shared base for all workers (arg selects cmd/)
```

**Why monorepo over multi-repo:**

| Concern | Monorepo | Multi-repo |
|---|---|---|
| Shared types (workflow inputs, activity interfaces) | Single `internal/` package, always in sync | Cross-repo imports, version pinning, release coordination |
| Refactoring a workflow | One PR touches definition + activity + BFF endpoint | Three PRs across three repos, must merge in correct order |
| CI/CD | One pipeline, matrix builds per target | Three pipelines, cross-repo triggering |
| Code review | Reviewer sees full context (trigger → workflow → activity) | Reviewer sees partial picture per repo |
| Compliance audit trail | One PR = one reviewable change for the full feature | Auditor must correlate commits across repos |
| Go module management | One `go.mod`, one dependency tree | Three `go.mod` files, potential version drift on shared deps |
| Temporal versioning | Worker and BFF always built from same commit | Risk of BFF starting workflows that workers don't understand yet |

**Build and deploy independence:**

Despite being a monorepo, each component builds and deploys independently:

- `make frontend` → builds React SPA, produces `portal-frontend:sha` image
- `make bff` → builds Go BFF binary, produces `portal-bff:sha` image
- `make worker` → builds Temporal worker, produces `portal-worker:sha` image
- CI detects which paths changed and only builds/deploys affected images
- Flux watches each image independently (frontend, BFF, and worker can roll out at different times)

**Temporal type safety across BFF and workers:**

The critical benefit: workflow input/output types are defined once in `internal/workflows/` and imported by both the BFF (which starts workflows) and the workers (which execute them). A type change is a compile error in both the BFF and worker at the same time, caught in the same CI run. In a multi-repo setup, this mismatch would only surface at runtime.

```go
// internal/workflows/app_upgrade/types.go
type AppUpgradeInput struct {
    TenantID        string
    AppRef          string
    TargetVersion   string
    RolloutStrategy string // "canary" | "blue-green"
}

// cmd/bff/main.go — starts the workflow
client.ExecuteWorkflow(ctx, opts, app_upgrade.Workflow, input)

// cmd/worker/main.go — registers the workflow
w.RegisterWorkflow(app_upgrade.Workflow)
```

## Consequences

### Positive

- **BFF stays thin**: the portal's backend is a read-gateway + Temporal trigger. No database, no email, no CRM writes. Easy to reason about, easy to test
- **Durable execution**: workflows survive pod restarts, node failures, and cluster upgrades. No lost state
- **Composable lifecycle**: app upgrade = backup activity + GitOps commit activity + Flagger polling activity. Each piece is independently testable and reusable
- **Compliance built-in**: reference-based payloads and three-layer separation are enforced at the architecture level, not bolted on
- **Single language, single repo**: Go for BFF, workers, and infrastructure tooling. One `go.mod`, one CI pipeline, one PR review for the full stack. Workflow type changes are compile-time errors, not runtime surprises
- **Progressive delivery without custom code**: Flagger handles canary analysis mechanics. Temporal handles the orchestration around it
- **GitOps audit trail preserved**: workflow-triggered changes still flow through git (Flux reconciles). Temporal adds the "why" and "who" that git commits alone don't capture
- **CRM flexibility**: because CRM is external, the platform can adopt any CRM system without refactoring the portal. The portal just reads from it

### Negative

- **Temporal operational overhead**: four Temporal services to run and monitor (portal component, PostgreSQL provided by infra). Mitigated by Helm chart and low starting resource requirements
- **PostgreSQL becomes critical path**: Temporal and KillBill depend on PostgreSQL availability. Mitigated by standard HA (streaming replication, automated failover)
- **Learning curve**: Temporal's programming model (deterministic workflow code, activity separation) requires developer education
- **CRM choice is unresolved**: the portal's timeline display, live updates, and workflow-to-CRM integration all depend on a CRM decision that hasn't been made yet. This is the most significant open dependency
- **Monorepo CI complexity**: CI must detect which paths changed to avoid rebuilding everything on every commit. Mitigated by path-based build triggers (standard pattern in GitHub Actions / Forgejo CI)

### Risks

| Risk | Mitigation |
|---|---|
| Temporal cluster failure halts all workflows | Temporal is designed for HA (multiple replicas per service). In-flight workflows resume automatically on recovery. Start with single replicas, scale to HA before production. |
| PostgreSQL overload from shared usage | Logical separation (separate databases) prevents query interference. Monitor connection pools and query performance. Split into separate clusters if needed. |
| Flagger canary analysis produces false positives/negatives | Start with conservative thresholds. Temporal workflow allows manual override via signal. Iterate on metrics and thresholds over time. |
| CRM choice blocks portal timeline feature | The BFF's timeline display and live updates depend on CRM choice. Mitigate by defining a minimal CRM read interface early; swap implementations behind it. |
| Reference-based payload discipline slips | Enforce via code review + linting rules that flag string fields matching PII patterns in workflow/activity input structs. CI check prevents merge. |

## Impact on Related Decisions

- **ADR-001 (Frontend)**: Refine's `liveProvider` connects to BFF WebSocket for real-time workflow state updates. CRM timeline display depends on CRM choice (separate decision). Workflow status components use Refine's `useCustom` hooks to query/signal Temporal through the BFF
- **ADR-002 (Architecture)**: BFF role changes from "GitOps commit gateway" to "read gateway + Temporal trigger". BFF no longer commits to GitOps directly -- Temporal worker activities do. Keycloak groups extended with `platform-workflow-*` roles
- **Issue #6 (Onboarding)**: Onboarding is a Temporal workflow. Wizard submission starts the workflow; portal shows real-time progress via workflow queries
- **Future: Billing Integration**: KillBill webhook events trigger Temporal workflows (payment escalation, plan changes). The billing worker translates KillBill events into workflow signals
- **Future: CRM Choice**: Separate decision required. Determines how the portal displays customer timeline, how Temporal workers record events, and whether live updates use LISTEN/NOTIFY or polling

## References

- [Workflow Catalog](./workflow-catalog.md)
- [Temporal Documentation](https://docs.temporal.io/)
- [Temporal Go SDK](https://github.com/temporalio/sdk-go)
- [Flagger Documentation](https://docs.flagger.app/)

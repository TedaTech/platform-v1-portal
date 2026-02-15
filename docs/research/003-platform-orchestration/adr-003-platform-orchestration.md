# ADR-003: Platform Orchestration, Workflow Engine, and CRM

## Status

Proposed

## Date

2026-02-15

## Context

The Customer Portal needs to orchestrate complex, multi-step business processes that span seconds to weeks:

- **Customer onboarding**: tenant creation, billing setup, welcome communication, initial app provisioning
- **Payment escalation**: failed payment detection, retry logic, customer notification, grace period, service suspension
- **App lifecycle**: backup, progressive rollout (canary/blue-green), health verification, automatic rollback on failure
- **Service disabling/re-enabling**: triggered by billing events or admin action, with notification workflows
- **Customer communication tracking**: every interaction (email, support, billing event, status change) recorded as a CRM timeline from day one

These workflows must survive infrastructure restarts, support human-in-the-loop decisions (approval gates, manual overrides), and provide full audit trails for GDPR, SOC 2, and ISO 27001 compliance.

ADR-002 established that the BFF commits resources to a GitOps repo and Flux reconciles them. This ADR extends that model: the BFF now also starts and signals Temporal workflows, which orchestrate the full lifecycle around those GitOps commits.

## Decision Drivers

- **CRM from day one**: every customer-facing interaction must be tracked and visible in the portal. Retrofitting communication tracking later is expensive and loses historical data
- **Full app lifecycle visibility**: users and platform operators must see exactly where an upgrade is (backup in progress, canary at 10%, health check passed, etc.)
- **Compliance by design**: reference-based payloads, immutable audit logs, and RBAC extensions are architectural decisions that cannot be retrofitted into a workflow engine after the fact
- **PostgreSQL as shared database**: already needed for KillBill billing. Adding Temporal, CRM state, and audit logs to the same PostgreSQL cluster avoids operational overhead of multiple database systems
- **No etcd/CRD pressure**: workflow state must live in PostgreSQL, not as Kubernetes custom resources. At 1,000 tenants with frequent operations, CRD-based workflow engines (Tekton, Argo Workflows) would pressure the etcd store that the entire cluster depends on
- **Go ecosystem alignment**: the Kubernetes ecosystem, Temporal's most mature SDK, and the Cozystack platform are all Go. A single language for BFF, workers, and infrastructure tooling reduces cognitive overhead and dependency sprawl
- **Monorepo for shared types**: the BFF starts workflows and workers execute them -- both need the same Go types (workflow inputs, activity interfaces). Splitting across repos creates version drift and import cycle headaches. One repo, one `go.mod`, one CI pipeline

## Decision

### 1. Temporal (Self-Hosted) as Central Workflow Orchestrator

All multi-step business processes run as Temporal workflows. Temporal is deployed self-hosted on the same Kubernetes cluster, using PostgreSQL as its persistence backend.

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
  bff-worker        -- handles portal-triggered workflows
  lifecycle-worker  -- handles app upgrade, backup, rollout activities
  billing-worker    -- handles payment escalation, KillBill integration
  comms-worker      -- handles email dispatch, CRM event recording
```

**Resource estimate (low-concurrency starting point):**

| Component | CPU Request | Memory Request | Replicas |
|---|---|---|---|
| Frontend | 0.5 core | 512 Mi | 1 |
| History | 0.5 core | 512 Mi | 1 |
| Matching | 0.25 core | 256 Mi | 1 |
| Worker | 0.25 core | 256 Mi | 1 |
| **Total Temporal** | **1.5 cores** | **1.5 Gi** | **4 pods** |
| Application workers | 0.5 core each | 256 Mi each | 4 pods |

Scale replicas as concurrency grows. Temporal's history service can be sharded across replicas when needed.

### 2. Go for BFF and Temporal Workers

The BFF (from ADR-002) and all Temporal worker processes are written in Go.

**BFF responsibilities expand from ADR-002:**

```
ADR-002 BFF:
  - Validate Keycloak tokens
  - Proxy K8s API requests
  - Commit tenant resources to GitOps repo
  - Connect OpenCost to KillBill

ADR-003 BFF additions:
  - Start Temporal workflows (onboarding, upgrade, etc.)
  - Signal workflows (approve/reject, manual override)
  - Query workflow state (for portal real-time display)
  - Expose CRM timeline API (read from audit/event store)
  - Validate workflow RBAC (Keycloak groups → workflow permissions)
```

The BFF does NOT contain workflow logic. It is a thin gateway that translates HTTP/WebSocket requests from the portal into Temporal workflow operations. All business logic lives in Temporal workflow and activity definitions.

### 3. PostgreSQL as Shared Persistence

A single PostgreSQL cluster (with logical separation) serves all platform data needs:

| Database / Schema | Owner | Purpose |
|---|---|---|
| `temporal` | Temporal cluster | Workflow execution state |
| `temporal_visibility` | Temporal cluster | Workflow search and filtering |
| `killbill` | KillBill | Billing, subscriptions, invoices |
| `portal` | BFF | CRM timeline, audit log, portal metadata |

**Why shared PostgreSQL:**

- Temporal requires PostgreSQL (or MySQL/Cassandra). PostgreSQL is already needed for KillBill
- Operational overhead of one database cluster vs. three-four is significant at small team scale
- Logical separation (separate databases within one cluster) provides isolation without operational cost
- Backup and HA is configured once for the cluster
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
  6. Activity: if Flagger promotes → record success in CRM timeline
  7. Activity: if Flagger rolls back → record failure, notify user
  8. Signal: user can manually promote or rollback at any step
```

Flagger adds one CRD (`Canary`) to the cluster. Traffic shifting is based on Kubernetes Gateway API or Istio/Linkerd service mesh metrics. Flagger handles the mechanics of canary analysis; Temporal orchestrates the full lifecycle around it.

### 5. SendGrid for Internal Platform Email

Platform-generated emails (welcome, payment reminders, service status changes) are sent via SendGrid API.

**Scope:** Internal platform communication only. Tenant-facing email (tenants sending email from their own apps) is out of scope for this ADR.

**Integration point:** The `comms-worker` Temporal worker calls the SendGrid API as a workflow activity. Email dispatch is never fire-and-forget -- it is a recorded step in a workflow with delivery status tracking.

```
Email Activity:
  Input:  { template_id, recipient_ref, tenant_id, context_data }
  Action: resolve recipient email from Keycloak (by ref, not stored in payload)
          call SendGrid API
          record delivery status
  Output: { message_id, status, timestamp }

  Retry:  Temporal activity retry policy (3 attempts, exponential backoff)
  Record: CRM timeline event created for every send attempt
```

### 6. CRM / Customer Communication Tracking

Every customer-facing interaction is recorded as a timeline event in the portal database. This is not a separate CRM product -- it is a first-class feature of the portal, powered by Temporal workflow events.

**CRM timeline event sources:**

| Source | Events Recorded |
|---|---|
| Onboarding workflow | Tenant created, billing setup, welcome email sent, first app provisioned |
| Payment workflow | Invoice generated, payment attempted, payment failed, reminder sent, service suspended |
| App lifecycle workflow | Upgrade started, backup created, canary at X%, promoted/rolled back, user notified |
| Support interaction | Ticket created, response sent, ticket resolved (future integration) |
| Admin action | Manual service override, plan change, account note added |
| Auth events | User invited, user joined, role changed |

**CRM data model (portal database):**

```sql
-- Timeline events (append-only, immutable)
CREATE TABLE crm_timeline (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       TEXT NOT NULL,
    actor_ref       TEXT NOT NULL,         -- Keycloak user ID or "system"
    event_type      TEXT NOT NULL,         -- e.g., "email.sent", "app.upgrade.started"
    event_source    TEXT NOT NULL,         -- e.g., "workflow:app_upgrade", "admin:manual"
    workflow_id     TEXT,                  -- Temporal workflow ID (nullable for non-workflow events)
    workflow_run_id TEXT,                  -- Temporal run ID
    subject_ref     TEXT,                  -- entity this event is about (user ref, app ref)
    summary         TEXT NOT NULL,         -- human-readable summary (no PII)
    metadata        JSONB,                -- structured event data (no PII -- references only)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_crm_timeline_tenant ON crm_timeline (tenant_id, created_at DESC);
CREATE INDEX idx_crm_timeline_subject ON crm_timeline (subject_ref, created_at DESC);
CREATE INDEX idx_crm_timeline_workflow ON crm_timeline (workflow_id);
```

**No PII in timeline events.** The `actor_ref` and `subject_ref` are Keycloak user IDs. The portal resolves these to names/emails at display time by querying Keycloak. If a user exercises GDPR right to erasure, the timeline events remain intact but the references become unresolvable -- effectively anonymized.

### 7. Compliance Architecture

Three-layer data separation enforced from day one:

```
Layer 1: Workflow Execution State (Temporal PostgreSQL)
  - Mutable: in-flight workflows can be signaled, cancelled, retried
  - Contains: workflow IDs, activity results, timer state
  - Contains NO PII: payloads use reference IDs only
  - Retention: 24 months (configurable per namespace)

Layer 2: Audit / CRM Log (Portal PostgreSQL)
  - Append-only: crm_timeline table, no UPDATE or DELETE permissions for application
  - Contains: who did what, when, with what outcome
  - Contains NO PII: actor_ref and subject_ref are Keycloak IDs
  - Retention: 36+ months (audit), 7 years (payment-related)

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
| `platform-workflow-viewer` | View workflow executions and CRM timeline across all tenants |
| `platform-workflow-operator` | Trigger, signal, retry, cancel workflows across all tenants |
| `platform-workflow-admin` | Deploy workflow definitions, manage schedules, configure retention |
| `{tenant}-view` (existing) | View CRM timeline and workflow status for own tenant |
| `{tenant}-admin` (existing) | Trigger tenant-scoped workflows (request upgrade, etc.) |

The BFF validates Keycloak token group claims before proxying any workflow operation to Temporal.

### 8. Portal Visibility into Workflows

The portal frontend displays workflow state using Refine's `liveProvider` (from ADR-001) connected to the BFF via WebSocket/SSE:

```
User Experience:

Tenant Dashboard:
  - CRM timeline: scrollable feed of all events for this tenant
  - Active workflows: cards showing current step, progress, duration
  - App status: current version, last upgrade result, next scheduled backup

App Upgrade View:
  - Step indicator: backup → canary 10% → canary 50% → canary 100% → promoted
  - Live metrics: error rate, latency (from Flagger's canary analysis)
  - Manual controls: "Promote now" / "Rollback" buttons (trigger Temporal signals)
  - History: previous upgrades with outcome and duration

BFF Implementation:
  - GET  /api/v1/tenants/{id}/timeline      → query crm_timeline table
  - GET  /api/v1/workflows/{id}/status       → Temporal QueryWorkflow
  - POST /api/v1/workflows/{id}/signal/{name} → Temporal SignalWorkflow
  - WS   /api/v1/tenants/{id}/events         → stream timeline + workflow updates
```

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
    worker-lifecycle/          # App upgrade, backup activities
      main.go
    worker-billing/            # Payment escalation, KillBill activities
      main.go
    worker-comms/              # Email dispatch, CRM recording
      main.go

  internal/                    # Shared Go packages (not importable externally)
    workflows/                 # Workflow definitions (imported by BFF to start, workers to execute)
      onboarding/
      payment_escalation/
      app_upgrade/
      service_lifecycle/
      backup/
      campaign/
      gdpr_erasure/
    activities/                # Activity implementations (imported by workers)
      keycloak/                # Keycloak user resolution
      killbill/                # KillBill billing operations
      gitops/                  # Git commit to GitOps repo
      sendgrid/                # Email dispatch
      flagger/                 # Canary status polling
      crm/                     # CRM timeline event recording
      k8s/                     # Kubernetes API operations
    auth/                      # Keycloak token validation, RBAC checks
    config/                    # Shared configuration loading
    db/                        # PostgreSQL connection, migrations

  migrations/                  # SQL migrations (portal database)
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
- `make worker-lifecycle` → builds lifecycle worker, produces `portal-worker-lifecycle:sha` image
- CI detects which paths changed and only builds/deploys affected images
- Flux watches each image independently (frontend, BFF, and workers can roll out at different times)

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

// cmd/worker-lifecycle/main.go — registers the workflow
w.RegisterWorkflow(app_upgrade.Workflow)
```

## Consequences

### Positive

- **Full visibility from day one**: every customer interaction is tracked. No retroactive data migration needed
- **Durable execution**: workflows survive pod restarts, node failures, and cluster upgrades. No lost state
- **Composable lifecycle**: app upgrade = backup activity + GitOps commit activity + Flagger polling activity. Each piece is independently testable and reusable
- **Compliance built-in**: reference-based payloads and three-layer separation are enforced at the architecture level, not bolted on
- **Single language, single repo**: Go for BFF, workers, and infrastructure tooling. One `go.mod`, one CI pipeline, one PR review for the full stack. Workflow type changes are compile-time errors, not runtime surprises
- **Progressive delivery without custom code**: Flagger handles canary analysis mechanics. Temporal handles the orchestration around it
- **GitOps audit trail preserved**: workflow-triggered changes still flow through git (Flux reconciles). Temporal adds the "why" and "who" that git commits alone don't capture

### Negative

- **Temporal operational overhead**: four additional services to run and monitor. Mitigated by Helm chart and low starting resource requirements
- **PostgreSQL becomes critical path**: all platform state depends on PostgreSQL availability. Mitigated by standard HA (streaming replication, automated failover)
- **Learning curve**: Temporal's programming model (deterministic workflow code, activity separation) requires developer education
- **SendGrid vendor dependency**: email delivery depends on external SaaS. Mitigated by activity-level abstraction (swap SendGrid activity for another provider without changing workflows)
- **Monorepo CI complexity**: CI must detect which paths changed to avoid rebuilding everything on every commit. Mitigated by path-based build triggers (standard pattern in GitHub Actions / Forgejo CI)

### Risks

| Risk | Mitigation |
|---|---|
| Temporal cluster failure halts all workflows | Temporal is designed for HA (multiple replicas per service). In-flight workflows resume automatically on recovery. Start with single replicas, scale to HA before production. |
| PostgreSQL overload from shared usage | Logical separation (separate databases) prevents query interference. Monitor connection pools and query performance. Split into separate clusters if needed. |
| Flagger canary analysis produces false positives/negatives | Start with conservative thresholds. Temporal workflow allows manual override via signal. Iterate on metrics and thresholds over time. |
| CRM timeline table grows unbounded | Partition by `created_at`. Archive partitions older than retention policy to cold storage. Implement read-through cache for recent events. |
| SendGrid deliverability or API outage | Temporal activity retry policy handles transient failures. For sustained outage, workflow pauses and resumes when SendGrid recovers. Alert on prolonged email delivery failures. |
| Reference-based payload discipline slips | Enforce via code review + linting rules that flag string fields matching PII patterns in workflow/activity input structs. CI check prevents merge. |

## Impact on Related Decisions

- **ADR-001 (Frontend)**: Refine's `liveProvider` connects to BFF WebSocket for real-time workflow state and CRM timeline updates. Workflow status components use Refine's `useCustom` hooks to query/signal Temporal through the BFF
- **ADR-002 (Architecture)**: BFF role expands from "GitOps commit gateway" to "workflow orchestration gateway". Tenant provisioning becomes a Temporal workflow (with the GitOps commit as one activity within it). Keycloak groups extended with `platform-workflow-*` roles
- **Issue #6 (Onboarding)**: Onboarding is now a Temporal workflow. Wizard submission starts the workflow; portal shows real-time progress via workflow queries
- **Future: Billing Integration**: KillBill webhook events trigger Temporal workflows (payment escalation, plan changes). The billing worker translates KillBill events into workflow signals

## References

- [Workflow Catalog](./workflow-catalog.md)
- [Compliance Architecture](./compliance-architecture.md)
- [Temporal Documentation](https://docs.temporal.io/)
- [Temporal Go SDK](https://github.com/temporalio/sdk-go)
- [Flagger Documentation](https://docs.flagger.app/)
- [SendGrid Go SDK](https://github.com/sendgrid/sendgrid-go)

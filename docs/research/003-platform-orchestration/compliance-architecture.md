# Compliance Architecture

Patterns and controls for GDPR, SOC 2 Type II, and ISO 27001 compliance in the context of Temporal workflow orchestration and the platform's GitOps architecture.

**Note:** CRM and email are external systems (choice TBD). This document covers compliance for the portal and Temporal. The CRM system will have its own compliance requirements depending on the choice made.

## Three-Layer Data Separation

The fundamental compliance pattern: separate mutable PII from immutable audit records and workflow state.

```
┌─────────────────────────────────────────────────────────────┐
│  Layer 3: PII Store (Mutable, Erasable)                     │
│  ┌──────────────┐  ┌──────────────┐                         │
│  │   Keycloak    │  │   KillBill   │                        │
│  │  email, name  │  │ billing addr │                        │
│  │  user profile │  │ payment info │                        │
│  └──────┬───────┘  └──────┬───────┘                         │
│         │ user_ref         │ account_ref                    │
├─────────┼──────────────────┼────────────────────────────────┤
│  Layer 1: Workflow State (Mutable, Reference-Based)         │
│  ┌──────────────────────────────────────────────┐           │
│  │          Temporal PostgreSQL                   │          │
│  │  workflow_id, tenant_id, user_ref, plan_ref   │          │
│  │  activity results, timer state                │          │
│  │  NO email, NO name, NO billing details        │          │
│  └──────────────────────────────────────────────┘           │
├─────────────────────────────────────────────────────────────┤
│  Layer 2: Audit / CRM Log (External CRM System)             │
│  ┌──────────────────────────────────────────────┐           │
│  │          External CRM (choice TBD)            │          │
│  │  who (actor_ref), what (event_type)           │          │
│  │  when (timestamp), outcome, workflow_id       │          │
│  │  Owned by CRM, read by portal BFF            │          │
│  │  NO email, NO name, NO billing details        │          │
│  └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
```

**GDPR erasure flow:** Delete PII from Layer 3 (Keycloak + KillBill). Layers 1 and 2 remain intact but the reference IDs (`user_ref`) can no longer be resolved to a person. The timeline is effectively anonymized.

## GDPR Controls

### Record of Processing Activities (ROPA)

GDPR Article 30 requires documenting each processing activity. Each workflow maps to a ROPA entry:

| Workflow | Purpose | Legal Basis | Data Subjects | Personal Data Categories | Retention |
|---|---|---|---|---|---|
| Customer Onboarding | Service delivery | Contract (Art. 6(1)(b)) | Customers | Email, name (resolved from Keycloak, not stored in workflow) | 24 months (workflow), indefinite (Keycloak, until erasure) |
| Payment Escalation | Contract enforcement | Contract + Legitimate Interest (Art. 6(1)(b)(f)) | Customers | Payment status, invoice references (amounts retained for tax) | 24 months (workflow), 7 years (payment records) |
| App Upgrade | Service delivery | Contract (Art. 6(1)(b)) | Customers | None (tenant/app references only) | 24 months |
| Service Disable | Contract enforcement | Contract (Art. 6(1)(b)) | Customers | Service status (no direct PII) | 24 months |
| Re-Engagement Campaign | Marketing | Consent (Art. 6(1)(a)) | Customers | Email (resolved at send time, not stored in workflow) | Campaign duration + 36 months (consent proof) |
| GDPR Data Erasure | Legal obligation | Legal obligation (Art. 6(1)(c)) | Data subjects | Erasure report (no PII, only refs + completion status) | 36 months (proof of compliance) |

### Consent Management (Marketing Workflows)

Marketing consent management is the CRM system's responsibility (choice TBD). The portal onboarding wizard collects initial consent preferences and passes them to the CRM. The CRM system manages the full consent lifecycle:

- **Collection**: explicit opt-in during onboarding (portal passes consent to CRM)
- **Verification**: CRM checks consent before every marketing email send
- **Withdrawal**: user unsubscribes via CRM mechanism; CRM halts active campaigns
- **Annual refresh**: CRM manages re-permission workflows

The portal's only role in consent is displaying the user's current consent status (read from CRM) and passing initial consent preferences during onboarding.

### Data Retention Policies

| Data Type | Store | Retention | Cleanup Mechanism |
|---|---|---|---|
| Temporal workflow history | Temporal PostgreSQL | 24 months | Temporal namespace retention policy (automatic) |
| CRM timeline events | External CRM | Governed by CRM system | CRM system's retention policy |
| Consent records | External CRM or dedicated store | Relationship duration + 36 months | Manual review (must prove consent was valid when used) |
| Keycloak user records | Keycloak | Until erasure request | GDPR erasure workflow |
| KillBill billing records | KillBill PostgreSQL | 7 years (tax/financial obligation) | Anonymize after retention period (keep amounts, remove identity) |
| Git commit history (GitOps) | Forgejo | Indefinite (audit trail) | Contains no PII (tenant IDs and resource specs only) |
| Email delivery logs | External CRM / email provider | Per provider retention | Provider DPA governs |

### Cross-Border Data Transfer

| System | Deployment | Data Residency | DPA Required |
|---|---|---|---|
| Temporal | Self-hosted (same cluster) | Same region as cluster | No (no third party) |
| PostgreSQL | Self-hosted (same cluster) | Same region as cluster | No |
| Keycloak | Self-hosted (Cozystack) | Same region as cluster | No |
| KillBill | Self-hosted (same cluster) | Same region as cluster | No |
| Forgejo | Self-hosted | Same region as cluster | No |
| CRM system | TBD (depends on choice) | TBD | **TBD** -- if external SaaS, DPA required |
| Email provider | TBD (depends on CRM choice) | TBD | **TBD** -- if external SaaS, DPA required |

**External processor DPAs depend on CRM choice.** If the CRM or email provider is SaaS-based, ensure:
- Data Processing Addendum (DPA) is signed
- EU data processing region is selected if available
- Standard Contractual Clauses (SCCs) are in place for any non-EU processing

## SOC 2 Type II Controls

### CC6: Logical and Physical Access Controls

**CC6.1 -- Logical access security:**

| System | Access Control Mechanism |
|---|---|
| Portal frontend | Keycloak OIDC authentication (oidc-spa) |
| BFF API | Keycloak JWT token validation, group claim checks |
| Temporal workflows | BFF validates Keycloak groups before starting/signaling workflows |
| Temporal admin (tctl) | Kubernetes RBAC, restricted to platform-workflow-admin |
| PostgreSQL | Database-level roles, connection restricted to K8s service accounts |
| GitOps repo | Forgejo authentication, branch protection rules |
| External CRM (read) | BFF validates tenant membership before proxying CRM reads |

**CC6.3 -- Role-based access:**

```
Authorization Matrix:

Operation                           | tenant-view | tenant-admin | wf-viewer | wf-operator | wf-admin | platform-admin
─────────────────────────────────────┼─────────────┼──────────────┼───────────┼─────────────┼──────────┼───────────────
View own tenant CRM timeline        |     Y       |      Y       |           |             |          |       Y
View all tenants CRM timeline       |             |              |     Y     |      Y      |    Y     |       Y
View own tenant workflow status     |     Y       |      Y       |           |             |          |       Y
View all workflow statuses          |             |              |     Y     |      Y      |    Y     |       Y
Start tenant-scoped workflow        |             |      Y       |           |             |          |       Y
Start platform workflow             |             |              |           |      Y      |    Y     |       Y
Signal workflow (approve/rollback)  |             |      Y *     |           |      Y      |    Y     |       Y
Cancel workflow                     |             |              |           |      Y      |    Y     |       Y
Deploy workflow definitions         |             |              |           |             |    Y     |       Y
Modify retention policies           |             |              |           |             |    Y     |       Y
Access Temporal admin CLI           |             |              |           |             |    Y     |       Y

* tenant-admin can signal only their own tenant's workflows

Note: CRM timeline access is governed by the BFF (validates tenant membership before
proxying reads from the external CRM). CRM write permissions are the CRM system's concern.
```

### CC7: System Operations and Monitoring

**CC7.1 -- Detection of unauthorized activity:**

Every workflow operation generates an audit event:

| Event | Logged Fields |
|---|---|
| Workflow started | actor_ref (Keycloak user ID), workflow_type, tenant_id, timestamp |
| Workflow signaled | actor_ref, signal_name, workflow_id, timestamp |
| Workflow completed | workflow_id, outcome (completed/failed/cancelled/timed_out), timestamp |
| Workflow definition deployed | git_commit_sha, author, reviewer, timestamp |
| Access denied | actor_ref, requested_operation, reason, timestamp |
| Retention policy changed | actor_ref, old_value, new_value, timestamp |

**CC7.2 -- Monitoring and alerting:**

| Alert | Condition | Severity | Channel |
|---|---|---|---|
| Workflow failure | Any workflow reaches terminal failed state | Warning | Platform ops |
| Payment escalation started | PaymentEscalation workflow begins | Info | Billing team |
| Service suspended | ServiceDisable workflow completes | High | Platform ops + tenant |
| Unauthorized access attempt | BFF rejects workflow operation due to RBAC | High | Security team |
| Workflow engine unhealthy | Temporal frontend health check fails | Critical | Platform ops |
| CRM integration failure | [CRM] activity fails after retries | High | Platform ops |

### CC8: Change Management

**CC8.1 -- Changes are authorized and documented:**

Workflow definitions are Go code stored in git. The change management flow:

```
1. Developer creates feature branch
2. Developer writes/modifies workflow or activity code
3. CI runs:
   - go vet, golangci-lint
   - Unit tests (workflow replay tests)
   - Integration tests (Temporal test environment)
   - OPA policy validation (if workflow policies defined)
4. Developer creates pull request
5. Different developer reviews and approves (four-eyes principle)
6. PR merged to main branch
7. CI builds worker container image, pushes to registry
8. Flux CD detects new image tag, updates worker deployment
9. Temporal workers restart with new code
10. Running workflows continue on previous version (Temporal versioning)
    New workflow starts use the new version

Audit trail:
  - Git commit: who wrote the code, what changed, when
  - PR review: who approved, review comments
  - Container image: SHA256 digest, build timestamp
  - Flux reconciliation: deployment timestamp, previous/new image
```

## ISO 27001 Controls

### 5.3 -- Segregation of Duties

| Role | Define | Review | Deploy | Trigger | Monitor | Audit |
|---|---|---|---|---|---|---|
| Developer | Y | | | | | |
| Reviewer (different person) | | Y | | | | |
| Flux CD (system) | | | Y | | | |
| Operator | | | | Y | Y | |
| Compliance | | | | | | Y |

**Enforced by:**
- Git branch protection: PRs require at least one approving review from a different user
- Flux CD: deployment is automated, no human can deploy directly
- Keycloak groups: operators cannot modify workflow code, developers cannot trigger production workflows (unless also assigned operator role)
- Temporal: workflow event history is immutable by design (event sourcing)

**For small teams:** If full segregation is impractical, compensating controls apply:
- Enhanced audit logging (all actions logged with full context)
- Mandatory PR reviews (no self-merge)
- Quarterly management review of workflow operation logs

### 8.15 -- Logging

Every log entry records:

| Field | Source | Example |
|---|---|---|
| **Who** | Keycloak JWT `sub` claim or service account identity | `user:a1b2c3d4` or `system:billing-worker` |
| **What** | Event type from application code | `workflow.started`, `activity.completed`, `access.denied` |
| **When** | UTC timestamp from NTP-synchronized clock | `2026-02-15T14:30:00.123Z` |
| **Where** | Kubernetes pod identity + namespace | `billing-worker-7d8f9-abc12.platform` |
| **How** | API endpoint or trigger mechanism | `POST /api/v1/workflows`, `killbill-webhook`, `temporal-schedule` |
| **Outcome** | Result of the action | `success`, `failed:insufficient_permissions`, `failed:timeout` |

**Log protection:**
- Temporal workflow history: immutable event sourcing (Temporal enforces this)
- Log aggregation pipeline (e.g., Loki): separate write and read credentials
- Platform operators: read access to logs, no delete capability
- Security team: full read access, delete only via documented retention policy execution
- Backup: Temporal PostgreSQL included in cluster backup schedule
- CRM audit logs: governed by the CRM system's own access controls

### 8.14 -- Business Continuity

| Failure | Impact | Recovery |
|---|---|---|
| Temporal frontend down | New workflows cannot start, signals queued | Restart pod (K8s liveness probe). In-flight workflows unaffected (history service is separate) |
| Temporal history down | In-flight workflows paused | Restart pod. Workflows resume exactly where they left off (durable execution) |
| PostgreSQL down | All state inaccessible | Failover to replica (streaming replication). Temporal reconnects automatically |
| Worker pod crash | Activities in progress retry | Temporal detects heartbeat timeout, reassigns activity to another worker. Idempotent activities safe to retry |
| Full cluster restart | Everything paused | On recovery: Temporal resumes all in-flight workflows from last checkpoint. No manual intervention needed |
| GitOps repo unavailable | Cannot deploy workflow changes or commit tenant resources | Flux retries. Existing workflows continue normally (they use already-deployed code). Only new deployments blocked |

**Recovery Time Objectives:**

| Component | RTO | RPO | Mechanism |
|---|---|---|---|
| Temporal cluster | < 5 minutes | 0 (PostgreSQL durability) | K8s self-healing + liveness probes |
| PostgreSQL | < 5 minutes | < 1 minute | Streaming replication + automated failover |
| Application workers | < 2 minutes | 0 (stateless, Temporal has the state) | K8s Deployment restart |
| External CRM | Depends on CRM choice | Depends on CRM choice | CRM system's HA mechanism |

### 8.17 -- Clock Synchronization

All components must use synchronized NTP for audit log consistency:

- Kubernetes nodes: NTP configured at OS level (cloud provider default or explicit NTP server)
- Temporal: uses host clock (synchronized via K8s node)
- PostgreSQL: `now()` uses host clock
- Application workers: use host clock for timestamps
- CRM system: clock sync depends on CRM deployment (self-hosted = same NTP, SaaS = provider's responsibility)

Verify: `SELECT now()` on PostgreSQL matches `date -u` on K8s nodes within 1 second.

## Audit Evidence Collection for SOC 2

During a SOC 2 Type II audit, auditors will request evidence for a 12-month period. The following sources satisfy their requirements:

| Evidence | Source | How to Extract |
|---|---|---|
| Access control configuration | Keycloak group memberships, K8s RoleBindings | Keycloak admin API export, `kubectl get rolebindings` |
| Access grant/revocation records | Keycloak audit log | Keycloak admin console |
| Change management records | Git history (PRs, reviews, merges) | `git log`, Forgejo PR export |
| Deployment history | Flux reconciliation logs, container image history | `flux get all`, container registry API |
| Incident detection records | Monitoring alerts, Temporal workflow failures | Alerting system export, Temporal visibility queries |
| Data processing records | Temporal workflow history, external CRM | Temporal visibility queries, CRM export (depends on choice) |
| Retention policy enforcement | Temporal namespace config | `tctl namespace describe` |

**Retention requirement:** All evidence sources must retain data for at least 12 months. Configure:
- Keycloak audit log retention: 12+ months
- Temporal namespace retention: 24 months
- CRM retention: governed by CRM system (ensure 36+ months for audit evidence)
- Git history: indefinite (default)
- Container image tags: 12+ months (do not prune old tags within audit window)

## Data Protection Impact Assessment (DPIA) Triggers

GDPR Article 35 requires a DPIA when processing is likely to result in high risk. The following workflows require DPIA documentation:

| Workflow | DPIA Required | Reason |
|---|---|---|
| Payment Escalation → Service Disable | **Yes** | Automated processing leading to service suspension (legal/significant effect on individual) |
| Re-Engagement Campaign | **Yes** | Profiling based on usage/engagement data for marketing purposes |
| GDPR Data Erasure | No | Processing serves the data subject's rights |
| Customer Onboarding | No | Standard contract performance, no automated decision-making |
| App Upgrade | No | Technical operation, no personal data decisions |
| Scheduled Backup | No | Technical operation, no personal data processing |

**DPIA template elements (for required workflows):**
1. Description of processing and purposes
2. Assessment of necessity and proportionality
3. Assessment of risks to data subjects
4. Measures to address risks (reference this document)
5. Consultation with Data Protection Officer (if appointed)

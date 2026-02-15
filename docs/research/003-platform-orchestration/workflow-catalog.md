# Workflow Catalog

High-level overview of Temporal workflows the platform orchestrates. All workflows follow the reference-based payload pattern (no PII in workflow inputs or event history).

**Detailed workflow designs will be developed separately** -- each workflow requires in-depth thinking about failure handling, signals, compensation, and edge cases. This catalog defines scope and intent only.

**CRM integration note:** Steps marked `[CRM]` represent integration points with the external CRM system. The specific API calls depend on the CRM choice (separate decision).

## 1. Customer Onboarding

**Trigger:** User submits onboarding wizard in portal (BFF starts workflow)
**Duration:** Minutes (mostly waiting for Flux reconciliation)
**Scope:** Validates billing plan, creates KillBill account, commits tenant resource to GitOps repo, waits for Flux reconciliation, verifies tenant readiness, records event and triggers welcome notification via [CRM].

## 2. App Upgrade

**Trigger:** User clicks "Upgrade" in portal OR scheduled maintenance window
**Duration:** Minutes to hours (backup + canary rollout)
**Scope:** Validates target version, creates backup, commits version bump to GitOps repo, monitors Flagger canary rollout, supports manual promote/rollback signals, records outcome via [CRM].

## 3. GDPR Data Erasure

**Trigger:** User requests data erasure via portal or support
**Duration:** Minutes (automated) + human verification
**Scope:** Verifies requester identity, halts active workflows for subject, erases PII from Keycloak and KillBill, requests erasure from [CRM], verifies completion, notifies compliance team. Supports human-in-the-loop for identity verification and incomplete erasure review.

---

## Future Workflows (Not Yet Scoped)

The following workflows are anticipated but will be designed when needed:

- **Payment Escalation** -- failed payment retry, notification, grace period, service suspension
- **Service Disable / Re-Enable** -- scale-to-zero, access restriction, restoration
- **Scheduled Backup** -- cron-triggered backup, verification, retention cleanup

## Temporal Namespace Strategy

| Namespace | Contains | Retention | Access |
|---|---|---|---|
| `platform` | Onboarding, GDPR erasure | 24 months | `platform-workflow-*` groups |
| `tenant-{id}` | App upgrade, tenant-specific workflows | 24 months | `{tenant}-*` groups + platform operators |

Per-tenant namespaces provide:
- **Isolation**: one tenant's workflows cannot interfere with another's
- **Per-tenant retention**: configurable per namespace
- **Access scoping**: tenant admins see only their workflows
- **Resource limits**: per-namespace rate limiting prevents one tenant from monopolizing workers

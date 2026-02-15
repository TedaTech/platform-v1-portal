# Workflow Catalog

Detailed definitions for each Temporal workflow the platform orchestrates. All workflows follow the reference-based payload pattern (no PII in workflow inputs or event history).

**CRM integration note:** Steps marked `[CRM]` represent integration points with the external CRM system. The specific API calls depend on the CRM choice (separate decision). These steps record events and/or trigger notifications -- the workflow orchestrates when they happen, the CRM system owns how.

## 1. Customer Onboarding

**Trigger:** User submits onboarding wizard in portal (BFF starts workflow)
**Duration:** Minutes (mostly waiting for Flux reconciliation)
**Namespace:** `platform` (cross-tenant, creates the tenant)

```
Workflow: CustomerOnboarding
Input:   { user_ref, plan_ref, tenant_name, region }
Output:  { tenant_id, status }

Steps:
  1. ValidatePlan          -- verify plan_ref exists in KillBill, check quota
  2. CreateBillingAccount  -- create KillBill account + subscription
  3. CommitTenantResource  -- worker activity commits tenant YAML to GitOps repo
  4. WaitForFluxReconcile  -- poll tenant namespace status (K8s watch)
  5. VerifyTenantReady     -- check namespace, RBAC, Keycloak groups exist
  6. [CRM] RecordAndNotify -- record "tenant.created" event + trigger welcome notification

Failure Handling:
  - Step 2 fails: workflow fails, no cleanup needed (nothing created yet)
  - Step 3 fails: revert KillBill subscription, workflow fails
  - Step 4 timeout (10 min): alert platform operator, wait for signal (manual fix or cancel)
  - Step 6 fails: retry 3x. CRM/notification failure should not block tenant creation

Signals:
  - cancel: operator can cancel at any point (triggers compensation activities)

Queries:
  - status: returns current step name + progress for portal display
```

**Portal display:** Step-by-step progress indicator. Each step shows pending/in-progress/completed/failed state. Real-time updates via WebSocket.

## 2. Payment Escalation

**Trigger:** KillBill webhook (payment failed event) starts workflow via billing worker
**Duration:** Days to weeks (grace period, retry cycles, escalation)
**Namespace:** `platform`

```
Workflow: PaymentEscalation
Input:   { tenant_id, invoice_ref, amount_ref, failure_reason }
Output:  { resolution: "paid" | "suspended" | "cancelled" }

Steps:
  1. [CRM] RecordEvent     -- "payment.failed" event
  2. RetryPayment          -- attempt payment retry via KillBill (after 24h)
     └─ If paid: [CRM] record "payment.succeeded", end workflow
  3. [CRM] SendReminder    -- trigger "payment failed, update payment method" notification
  4. WaitForPayment        -- sleep 72h, then check KillBill invoice status
     └─ If paid: [CRM] record "payment.succeeded", end workflow
  5. [CRM] SendWarning     -- trigger "service will be suspended in 48h" notification
  6. WaitForPayment        -- sleep 48h, then check KillBill invoice status
     └─ If paid: [CRM] record "payment.succeeded", end workflow
  7. SuspendService        -- start ServiceDisable child workflow
  8. [CRM] RecordEvent     -- "service.suspended.nonpayment"

Signals:
  - payment_received: KillBill webhook signals when payment succeeds (skips waiting)
  - extend_grace: operator can extend grace period
  - cancel_escalation: operator cancels escalation (e.g., invoice disputed)
  - override_suspend: operator forces immediate suspension

Queries:
  - status: current escalation step, days remaining in grace period
```

**Portal display:** Operator view shows escalation timeline with countdown. Tenant view shows "action required" banner with payment update link. Both update in real-time.

## 3. App Upgrade (Full Lifecycle)

**Trigger:** User clicks "Upgrade" in portal OR scheduled maintenance window
**Duration:** Minutes to hours (backup + canary rollout)
**Namespace:** Per-tenant Temporal namespace

```
Workflow: AppUpgrade
Input:   { tenant_id, app_ref, target_version, rollout_strategy: "canary" | "blue-green" }
Output:  { result: "promoted" | "rolled_back" | "cancelled", duration_ms }

Steps:
  1. [CRM] RecordEvent     -- "app.upgrade.started"
  2. ValidateVersion       -- check target version exists, compatibility
  3. CreateBackup          -- commit backup Job to GitOps repo, wait for completion
  4. CommitVersionBump     -- commit new version to GitOps repo (Flux reconciles)
  5. WaitForFlaggerCanary  -- poll Flagger Canary resource status
     ├─ Progressing: update portal (canary weight %, error rate, latency)
     ├─ Promoted: continue to step 6
     └─ Failed: jump to step 7
  6. [CRM] RecordEvent     -- "app.upgrade.promoted" (success path)
     └─ End workflow
  7. RollbackVersion       -- revert GitOps commit (git revert + push)
  8. [CRM] RecordAndNotify -- "app.upgrade.rolled_back" + notify user

Signals:
  - promote: user/operator forces immediate promotion (skip remaining canary steps)
  - rollback: user/operator forces immediate rollback
  - pause: pause canary progression (hold at current weight)
  - resume: resume paused canary

Queries:
  - status: current step, canary weight %, error rate, latency
  - backup_status: backup Job status (for step 3)
  - flagger_status: Flagger Canary analysis details
```

**Portal display:** Rich upgrade view with:
- Step progress indicator (backup → canary → promoted)
- Live canary metrics (error rate, p99 latency, success rate)
- Traffic split visualization (old vs. new version)
- Action buttons: Promote / Rollback / Pause
- History of previous upgrades with outcome and duration

## 4. Service Disable / Re-Enable

**Trigger:** Payment escalation workflow (automated) OR admin action (manual)
**Duration:** Minutes
**Namespace:** `platform`

```
Workflow: ServiceDisable
Input:   { tenant_id, reason: "nonpayment" | "tos_violation" | "admin_request", actor_ref }
Output:  { status: "disabled" | "failed" }

Steps:
  1. [CRM] RecordAndNotify -- "service.disable.started" + notify user (with reason)
  2. ScaleDownApps         -- commit scale-to-zero for tenant apps in GitOps repo
  3. WaitForReconcile      -- poll until Flux confirms apps scaled down
  4. RestrictAccess        -- update tenant Keycloak groups (remove use/admin, keep view)
  5. [CRM] RecordEvent     -- "service.disabled"

Signals:
  - cancel: abort disable (if payment arrives during the process)

Queries:
  - status: current step
```

```
Workflow: ServiceReEnable
Input:   { tenant_id, reason: "payment_received" | "admin_request", actor_ref }
Output:  { status: "enabled" | "failed" }

Steps:
  1. [CRM] RecordEvent     -- "service.reenable.started"
  2. RestoreAccess         -- restore tenant Keycloak groups
  3. ScaleUpApps           -- commit original scale for tenant apps in GitOps repo
  4. WaitForReconcile      -- poll until Flux confirms apps running
  5. VerifyHealth          -- check app health endpoints
  6. [CRM] RecordAndNotify -- "service.enabled" + notify user
```

## 5. Scheduled Backup

**Trigger:** Cron schedule (Temporal Schedule) per tenant/app
**Duration:** Minutes
**Namespace:** Per-tenant Temporal namespace

```
Workflow: ScheduledBackup
Input:   { tenant_id, app_ref, backup_type: "full" | "incremental" }
Output:  { backup_id, size_bytes, duration_ms }

Steps:
  1. CommitBackupJob       -- commit backup K8s Job to GitOps repo
  2. WaitForCompletion     -- poll Job status
  3. VerifyBackup          -- check backup integrity (checksum, restore test if configured)
  4. CleanupOldBackups     -- enforce retention policy (keep N backups)
  5. [CRM] RecordEvent     -- "app.backup.completed" with size and duration

Failure Handling:
  - Step 2 timeout: alert operator, [CRM] record "app.backup.timeout"
  - Step 3 fails: alert operator, [CRM] record "app.backup.verification_failed"
  - Consecutive failures (3+): alert platform-workflow-admin, pause schedule

Queries:
  - status: current step, progress percentage if available
  - last_backup: timestamp and status of most recent completed backup
```

## 6. Re-Engagement Email Campaign

**Status:** Out of portal scope. Email campaigns, drip sequences, consent management, and marketing automation belong to the CRM / comms system (choice TBD). The CRM may use Temporal internally for campaign orchestration, but that is the CRM's concern, not the portal's.

If the CRM exposes campaign status, the portal BFF can display it -- but the portal does not define, trigger, or manage campaigns.

## 7. GDPR Data Erasure

**Trigger:** User requests data erasure via portal or support
**Duration:** Minutes (automated) + human verification
**Namespace:** `platform`

```
Workflow: GDPRDataErasure
Input:   { requester_ref, subject_ref, verification_token }
Output:  { status: "completed" | "partial", erasure_report }

Steps:
  1. [CRM] RecordEvent     -- "gdpr.erasure.requested" (using refs, not PII)
  2. VerifyIdentity        -- confirm requester is the subject or authorized representative
  3. HaltActiveWorkflows   -- signal all active workflows for this subject to stop
  4. EraseFromKeycloak     -- delete user record from Keycloak
  5. EraseFromKillBill     -- anonymize billing records (retain transaction amounts for tax law)
  6. [CRM] EraseFromCRM    -- remove PII from CRM system (email, contact lists, suppression)
  7. VerifyErasure         -- confirm PII is gone from all stores
  8. [CRM] RecordEvent     -- "gdpr.erasure.completed" (with erasure report)
  9. NotifyCompliance      -- send erasure completion report to compliance team

Human-in-the-Loop:
  - Step 2 may require manual identity verification (signal: identity_verified)
  - Step 8 may flag incomplete erasure (signal: manual_review_completed)

Queries:
  - status: current step, which systems have been erased
  - blockers: any systems where erasure is incomplete
```

**CRM timeline events survive erasure.** If the CRM uses reference-based events (UUIDs, not PII), Keycloak deletion makes the references unresolvable -- effectively anonymizing the timeline. The timeline record serves as proof that the erasure was performed (required by GDPR Article 5(2) accountability principle). The specific behavior depends on CRM choice.

---

## Temporal Namespace Strategy

| Namespace | Contains | Retention | Access |
|---|---|---|---|
| `platform` | Onboarding, payment escalation, service disable, campaigns, GDPR erasure | 24 months | `platform-workflow-*` groups |
| `tenant-{id}` | App upgrade, scheduled backup, tenant-specific maintenance | 24 months | `{tenant}-*` groups + platform operators |

Per-tenant namespaces provide:
- **Isolation**: one tenant's workflows cannot interfere with another's
- **Per-tenant retention**: configurable per namespace
- **Access scoping**: tenant admins see only their workflows
- **Resource limits**: per-namespace rate limiting prevents one tenant from monopolizing workers

---

## Activity Design Principles

1. **Activities are idempotent.** Every activity can be safely retried without side effects. Use idempotency keys for external API calls (KillBill, CRM, Git commits)

2. **Activities resolve PII at point of use.** Never pass email addresses, names, or billing details as activity inputs. Pass reference IDs and resolve from Keycloak/KillBill within the activity

3. **[CRM] activities call external CRM API.** Steps marked `[CRM]` call the external CRM system's API to record events or trigger notifications. The specific API depends on CRM choice. These are standard Temporal activities with retry policies

4. **Activities have bounded timeouts.** Every activity specifies a `StartToClose` timeout appropriate to its work:
   - API calls (KillBill, CRM): 30 seconds
   - Git commits: 60 seconds
   - K8s Job completion polling: 30 minutes
   - Flagger canary polling: 2 hours

5. **Activities are separately deployable.** Each worker process registers the activities it handles. Workers can be scaled independently based on activity type (e.g., more lifecycle workers during maintenance windows)

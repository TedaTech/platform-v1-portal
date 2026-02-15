# Workflow Catalog

Detailed definitions for each Temporal workflow the platform orchestrates. All workflows follow the reference-based payload pattern (no PII in workflow inputs or event history).

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
  3. CommitTenantResource  -- BFF commits tenant YAML to GitOps repo
  4. WaitForFluxReconcile  -- poll tenant namespace status (K8s watch)
  5. VerifyTenantReady     -- check namespace, RBAC, Keycloak groups exist
  6. SendWelcomeEmail      -- resolve user email from Keycloak, send via SendGrid
  7. RecordCRMEvent        -- "tenant.created" event in CRM timeline

Failure Handling:
  - Step 2 fails: workflow fails, no cleanup needed (nothing created yet)
  - Step 3 fails: revert KillBill subscription, workflow fails
  - Step 4 timeout (10 min): alert platform operator, wait for signal (manual fix or cancel)
  - Step 6 fails: retry 3x. If still fails, record CRM event "welcome_email_failed", continue
                  (email failure should not block tenant creation)

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
  1. RecordCRMEvent        -- "payment.failed" event
  2. RetryPayment          -- attempt payment retry via KillBill (after 24h)
     └─ If paid: RecordCRMEvent("payment.succeeded"), end workflow
  3. SendPaymentReminder   -- email: "payment failed, please update payment method"
  4. RecordCRMEvent        -- "payment.reminder_sent"
  5. WaitForPayment        -- sleep 72h, then check KillBill invoice status
     └─ If paid: RecordCRMEvent("payment.succeeded"), end workflow
  6. SendFinalWarning      -- email: "service will be suspended in 48h"
  7. RecordCRMEvent        -- "payment.final_warning_sent"
  8. WaitForPayment        -- sleep 48h, then check KillBill invoice status
     └─ If paid: RecordCRMEvent("payment.succeeded"), end workflow
  9. SuspendService        -- start ServiceDisable child workflow
  10. RecordCRMEvent       -- "service.suspended.nonpayment"

Signals:
  - payment_received: KillBill webhook signals when payment succeeds (skips waiting)
  - extend_grace: operator can extend grace period
  - cancel_escalation: operator cancels escalation (e.g., invoice disputed)
  - override_suspend: operator forces immediate suspension

Queries:
  - status: current escalation step, days remaining in grace period
  - timeline: all CRM events for this escalation
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
  1. RecordCRMEvent        -- "app.upgrade.started"
  2. ValidateVersion       -- check target version exists, compatibility
  3. CreateBackup          -- commit backup Job to GitOps repo, wait for completion
  4. RecordCRMEvent        -- "app.backup.completed"
  5. CommitVersionBump     -- commit new version to GitOps repo (Flux reconciles)
  6. WaitForFlaggerCanary  -- poll Flagger Canary resource status
     ├─ Progressing: update portal (canary weight %, error rate, latency)
     ├─ Promoted: continue to step 7
     └─ Failed: jump to step 8
  7. RecordCRMEvent        -- "app.upgrade.promoted" (success path)
     └─ End workflow
  8. RollbackVersion       -- revert GitOps commit (git revert + push)
  9. RecordCRMEvent        -- "app.upgrade.rolled_back"
  10. NotifyUser           -- email: "upgrade rolled back, here's why"

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
  1. RecordCRMEvent        -- "service.disable.started" with reason
  2. NotifyUser            -- email: "your service is being suspended" (with reason)
  3. ScaleDownApps         -- commit scale-to-zero for tenant apps in GitOps repo
  4. WaitForReconcile      -- poll until Flux confirms apps scaled down
  5. RestrictAccess        -- update tenant Keycloak groups (remove use/admin, keep view)
  6. RecordCRMEvent        -- "service.disabled"

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
  1. RecordCRMEvent        -- "service.reenable.started"
  2. RestoreAccess         -- restore tenant Keycloak groups
  3. ScaleUpApps           -- commit original scale for tenant apps in GitOps repo
  4. WaitForReconcile      -- poll until Flux confirms apps running
  5. VerifyHealth          -- check app health endpoints
  6. NotifyUser            -- email: "your service has been restored"
  7. RecordCRMEvent        -- "service.enabled"
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
  4. RecordCRMEvent        -- "app.backup.completed" with size and duration
  5. CleanupOldBackups     -- enforce retention policy (keep N backups)

Failure Handling:
  - Step 2 timeout: alert operator, record CRM event "app.backup.timeout"
  - Step 3 fails: alert operator, record CRM event "app.backup.verification_failed"
  - Consecutive failures (3+): alert platform-workflow-admin, pause schedule

Queries:
  - status: current step, progress percentage if available
  - last_backup: timestamp and status of most recent completed backup
```

## 6. Re-Engagement Email Campaign

**Trigger:** Scheduled (Temporal cron) or manual campaign start
**Duration:** Days to weeks (drip sequence)
**Namespace:** `platform`

**NOTE:** This workflow requires explicit marketing consent. It is the strictest workflow from a GDPR perspective.

```
Workflow: ReEngagementCampaign
Input:   { campaign_ref, segment_criteria, drip_schedule }
Output:  { sent_count, unsubscribed_count, bounced_count }

Steps:
  1. ResolveSegment        -- query tenants matching criteria (by billing/usage refs, NOT PII)
  2. ForEachTenant:        -- child workflow per tenant
     a. CheckConsent       -- verify marketing consent is active (Keycloak attribute or consent DB)
        └─ No consent: skip, record "campaign.skipped.no_consent"
     b. CheckSuppression   -- verify not on suppression list
        └─ Suppressed: skip, record "campaign.skipped.suppressed"
     c. SendEmail          -- resolve email from Keycloak, send via SendGrid
     d. RecordCRMEvent     -- "campaign.email.sent"
     e. WaitForNextDrip    -- sleep until next drip date (if drip sequence)
     f. CheckConsent       -- re-verify consent before each drip (consent may be withdrawn)

Signals:
  - stop_campaign: immediately halt all in-flight drips
  - consent_withdrawn(user_ref): stop drips for specific user

Queries:
  - progress: sent/total, current drip step
  - stats: open rate, click rate (if tracking enabled)
```

**Consent withdrawal is immediate.** When a user withdraws marketing consent, the BFF signals all active campaign workflows for that user. The workflow stops before the next email send.

## 7. GDPR Data Erasure

**Trigger:** User requests data erasure via portal or support
**Duration:** Minutes (automated) + human verification
**Namespace:** `platform`

```
Workflow: GDPRDataErasure
Input:   { requester_ref, subject_ref, verification_token }
Output:  { status: "completed" | "partial", erasure_report }

Steps:
  1. RecordCRMEvent        -- "gdpr.erasure.requested" (using refs, not PII)
  2. VerifyIdentity        -- confirm requester is the subject or authorized representative
  3. HaltActiveWorkflows   -- signal all active workflows for this subject to stop
  4. EraseFromKeycloak     -- delete user record from Keycloak
  5. EraseFromKillBill     -- anonymize billing records (retain transaction amounts for tax law)
  6. EraseFromSendGrid     -- remove from all SendGrid contact lists
  7. AddToSuppression      -- add subject_ref to permanent suppression list
  8. VerifyErasure         -- confirm PII is gone from all stores
  9. RecordCRMEvent        -- "gdpr.erasure.completed" (with erasure report)
  10. NotifyCompliance     -- send erasure completion report to compliance team

Human-in-the-Loop:
  - Step 2 may require manual identity verification (signal: identity_verified)
  - Step 8 may flag incomplete erasure (signal: manual_review_completed)

Queries:
  - status: current step, which systems have been erased
  - blockers: any systems where erasure is incomplete
```

**CRM timeline events survive erasure.** They contain `subject_ref` (a UUID), not PII. After Keycloak deletion, the UUID cannot be resolved to a person, effectively anonymizing the timeline. The timeline record serves as proof that the erasure was performed (required by GDPR Article 5(2) accountability principle).

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

1. **Activities are idempotent.** Every activity can be safely retried without side effects. Use idempotency keys for external API calls (SendGrid, KillBill, Git commits)

2. **Activities resolve PII at point of use.** Never pass email addresses, names, or billing details as activity inputs. Pass reference IDs and resolve from Keycloak/KillBill within the activity

3. **Activities record CRM events.** Most activities end with a `RecordCRMEvent` call. This is part of the activity, not a separate step -- if the activity retries, it uses idempotency to avoid duplicate CRM events

4. **Activities have bounded timeouts.** Every activity specifies a `StartToClose` timeout appropriate to its work:
   - API calls (SendGrid, KillBill): 30 seconds
   - Git commits: 60 seconds
   - K8s Job completion polling: 30 minutes
   - Flagger canary polling: 2 hours

5. **Activities are separately deployable.** Each worker process registers the activities it handles. Workers can be scaled independently based on activity type (e.g., more lifecycle workers during maintenance windows)

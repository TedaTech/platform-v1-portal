# Research: Platform Orchestration and Workflow Engine

**Issue:** [platform-v1-portal#7](https://github.com/TedaTech/platform-v1-portal/issues/7)
**Date:** 2026-02-15
**Status:** Proposed

## Architecture Summary

All multi-step business processes (onboarding, payment escalation, app lifecycle) run as Temporal workflows. The BFF (Go) is a thin read-gateway that starts/signals Temporal workflows and reads from multiple backends (OpenCost, KillBill, Temporal, external CRM) for display. All workflow state lives in PostgreSQL -- zero etcd/CRD pressure on the Kubernetes cluster.

**CRM and email are external systems** (choice TBD). The portal reads from the CRM for timeline display but does not own CRM data, email templates, or delivery infrastructure. Temporal workers call the CRM API as activities -- the specific integration depends on the CRM choice.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Workflow engine | Temporal (self-hosted, MIT) | Code-first Go SDK, durable execution, PostgreSQL persistence, immutable audit trail, human-in-the-loop via signals |
| Language | Go | Aligns with K8s ecosystem, Temporal's most mature SDK, single language for BFF + workers |
| Database | PostgreSQL (shared cluster) | Already needed for KillBill. Temporal shares the same PostgreSQL cluster with logical separation |
| Progressive delivery | Flagger | Canary/blue-green rollouts with metrics-driven analysis. One CRD. Temporal orchestrates the full lifecycle around it |
| BFF role | Read-gateway + Temporal trigger | Reads from OpenCost, KillBill, Temporal, CRM. Writes only to Temporal. No direct GitOps commits, no email, no CRM writes |
| CRM / email | External (choice TBD) | Portal reads from CRM for display. Temporal workers call CRM API for event recording and notifications. CRM choice determines live update mechanism |
| Compliance model | Three-layer separation | PII (Keycloak/KillBill) separated from workflow state (Temporal) and audit log (external CRM) |
| Repository structure | Monorepo | Frontend, BFF, and all workers in one repo. Shared Go types, one CI pipeline, one PR for full-stack changes. Separate build targets per component |
| Temporal infrastructure | Installed via infra repo | Temporal cluster deployment is an infrastructure concern, not a portal concern |

### Workflow Catalog

| Workflow | Trigger | Duration | Key Capability |
|---|---|---|---|
| Customer Onboarding | Portal wizard | Minutes | Billing setup → GitOps commit → Flux reconcile → [CRM] notify |
| Payment Escalation | KillBill webhook | Days-weeks | Retry → [CRM] remind → warn → suspend (with grace period signals) |
| App Upgrade | User action / schedule | Minutes-hours | Backup → GitOps version bump → Flagger canary → promote/rollback |
| Service Disable/Enable | Billing event / admin | Minutes | Scale-to-zero → restrict access → [CRM] notify (reversible) |
| Scheduled Backup | Temporal cron schedule | Minutes | Backup Job → verify → retention cleanup |
| GDPR Data Erasure | User request | Minutes + human review | Halt workflows → erase from all stores → verify → report |

Steps marked `[CRM]` are integration points with the external CRM system (choice TBD).

### Open Dependency: CRM Choice

The CRM decision is **not resolved** in this ADR. It affects:
- How the portal displays customer timeline (REST API, direct DB query, GraphQL)
- How Temporal workers record events (CRM API activity)
- Whether live updates use PostgreSQL LISTEN/NOTIFY (if CRM is co-located) or polling/webhooks
- Email provider and template management

## Documents

| Document | Content |
|----------|---------|
| [ADR-003](./adr-003-platform-orchestration.md) | Architecture Decision Record (technology choices, integration model, consequences) |
| [Workflow Catalog](./workflow-catalog.md) | Detailed workflow definitions with inputs, steps, signals, queries, and failure handling |
| [Compliance Architecture](./compliance-architecture.md) | GDPR, SOC 2, ISO 27001 controls, data separation, retention, RBAC, DPIA requirements |

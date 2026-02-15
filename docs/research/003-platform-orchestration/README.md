# Research: Platform Orchestration, Workflow Engine, and CRM

**Issue:** [platform-v1-portal#7](https://github.com/TedaTech/platform-v1-portal/issues/7)
**Date:** 2026-02-15
**Status:** Proposed

## Architecture Summary

All multi-step business processes (onboarding, payment escalation, app lifecycle, CRM tracking) run as Temporal workflows. The BFF (Go) starts and signals workflows; Temporal workers execute activities. All workflow state lives in PostgreSQL -- zero etcd/CRD pressure on the Kubernetes cluster.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Workflow engine | Temporal (self-hosted, MIT) | Code-first Go SDK, durable execution, PostgreSQL persistence, immutable audit trail, human-in-the-loop via signals |
| Language | Go | Aligns with K8s ecosystem, Temporal's most mature SDK, single language for BFF + workers |
| Database | PostgreSQL (shared cluster) | Already needed for KillBill. Temporal, CRM timeline, and audit logs share the same PostgreSQL cluster with logical separation |
| Progressive delivery | Flagger | Canary/blue-green rollouts with metrics-driven analysis. One CRD. Temporal orchestrates the full lifecycle around it |
| Internal email | SendGrid | Managed deliverability. Activity-level abstraction allows future swap. Tenant email out of scope |
| CRM tracking | First-class from day one | Every customer interaction recorded as timeline event. Reference-based (no PII in events) |
| Compliance model | Three-layer separation | PII (Keycloak/KillBill) separated from workflow state (Temporal) and audit log (portal DB). GDPR erasure = delete PII, timeline auto-anonymizes |
| Repository structure | Monorepo | Frontend, BFF, and all workers in one repo. Shared Go types, one CI pipeline, one PR for full-stack changes. Separate build targets per component |

### Workflow Catalog

| Workflow | Trigger | Duration | Key Capability |
|---|---|---|---|
| Customer Onboarding | Portal wizard | Minutes | Billing setup → GitOps commit → Flux reconcile → welcome email |
| Payment Escalation | KillBill webhook | Days-weeks | Retry → remind → warn → suspend (with grace period signals) |
| App Upgrade | User action / schedule | Minutes-hours | Backup → GitOps version bump → Flagger canary → promote/rollback |
| Service Disable/Enable | Billing event / admin | Minutes | Scale-to-zero → restrict access → notify (reversible) |
| Scheduled Backup | Temporal cron schedule | Minutes | Backup Job → verify → retention cleanup |
| Re-Engagement Campaign | Schedule / manual | Days-weeks | Consent check → send → drip wait → re-check consent |
| GDPR Data Erasure | User request | Minutes + human review | Halt workflows → erase from all stores → verify → report |

## Documents

| Document | Content |
|----------|---------|
| [ADR-003](./adr-003-platform-orchestration.md) | Architecture Decision Record (technology choices, integration model, consequences) |
| [Workflow Catalog](./workflow-catalog.md) | Detailed workflow definitions with inputs, steps, signals, queries, and failure handling |
| [Compliance Architecture](./compliance-architecture.md) | GDPR, SOC 2, ISO 27001 controls, data separation, retention, RBAC, DPIA requirements |

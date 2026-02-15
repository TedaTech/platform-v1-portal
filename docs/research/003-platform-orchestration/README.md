# Research: Platform Orchestration and Workflow Engine

**Issue:** [platform-v1-portal#7](https://github.com/TedaTech/platform-v1-portal/issues/7)
**Date:** 2026-02-15
**Status:** Proposed

## Architecture Summary

Multi-step business processes (onboarding, app lifecycle, GDPR erasure) run as Temporal workflows. The BFF (Go) is a thin read-gateway that starts/signals Temporal workflows and reads from multiple backends (OpenCost, KillBill, Temporal, external CRM) for display. All workflow state lives in PostgreSQL -- zero etcd/CRD pressure on the Kubernetes cluster.

Temporal is a **portal component** (deployed as part of the portal). The PostgreSQL database it uses is provided by the **infra layer**.

**CRM and email are external systems** (choice TBD). The portal reads from the CRM for timeline display but does not own CRM data, email templates, or delivery infrastructure.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Workflow engine | Temporal (self-hosted, MIT) | Code-first Go SDK, durable execution, PostgreSQL persistence, immutable audit trail, human-in-the-loop via signals |
| Language | Go | Aligns with K8s ecosystem, Temporal's most mature SDK, single language for BFF + worker |
| Database | PostgreSQL (infra-provided, shared cluster) | Already needed for KillBill. Temporal shares the same PostgreSQL cluster with logical separation |
| Progressive delivery | Flagger | Canary/blue-green rollouts with metrics-driven analysis. One CRD. Temporal orchestrates the full lifecycle around it |
| BFF role | Read-gateway + Temporal trigger | Reads from OpenCost, KillBill, Temporal, CRM. Writes only to Temporal. No direct GitOps commits, no email, no CRM writes |
| CRM / email | External (choice TBD) | Portal reads from CRM for display. Temporal workers call CRM API for event recording and notifications. CRM choice determines live update mechanism |
| Compliance model | Three-layer separation | PII (Keycloak/KillBill) separated from workflow state (Temporal) and audit log (external CRM) |
| Repository structure | Monorepo | Frontend, BFF, and worker in one repo. Shared Go types, one CI pipeline, one PR for full-stack changes. Separate build targets per component |

### Workflow Catalog

| Workflow | Trigger | Duration | Key Capability |
|---|---|---|---|
| Customer Onboarding | Portal wizard | Minutes | Billing setup → GitOps commit → Flux reconcile → [CRM] notify |
| App Upgrade | User action / schedule | Minutes-hours | Backup → GitOps version bump → Flagger canary → promote/rollback |
| GDPR Data Erasure | User request | Minutes + human review | Halt workflows → erase from all stores → verify → report |

Future workflows (payment escalation, service disable/enable, scheduled backup) will be designed when needed.

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
| [Workflow Catalog](./workflow-catalog.md) | High-level workflow scope and Temporal namespace strategy |

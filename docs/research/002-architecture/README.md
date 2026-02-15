# Research: Portal Architecture and Tenant Model

**Issue:** [platform-v1-portal#2](https://github.com/TedaTech/platform-v1-portal/issues/2)
**Date:** 2026-02-15
**Status:** Proposed

## Architecture Summary

The portal is a single SPA that authenticates users via Cozystack's Keycloak realm (`cozy`) and commits tenant lifecycle resources to a GitOps repo. Flux reconciles those resources into Cozystack tenants.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Auth model | Cozystack single Keycloak realm + Kubernetes RBAC | No custom auth. Users are Keycloak users, permissions are Keycloak groups mapped to K8s RoleBindings. |
| Tenant provisioning | BFF commits tenant resource to gitops repo (one file per tenant) | Flux picks up changes, creates/deletes tenants. Portal never talks to K8s API directly for provisioning. |
| Per-tenant portal | Not for MVP | Users use Cozystack dashboard, Grafana, etc. for deeper management. Main portal links to these. |
| JWT token tuning | None for MVP | Accept default proxy header limits. Document scaling ceiling. |
| Scale target | < 1,000 tenants | Beyond this, adopt multi-cluster model for blast radius and reliability. |

## Documents

| Document | Content |
|----------|---------|
| [ADR-002](./adr-002-architecture.md) | Architecture Decision Record |
| [Scaling Limits](./scaling-limits.md) | Hard numbers for single-realm / single-cluster ceiling |

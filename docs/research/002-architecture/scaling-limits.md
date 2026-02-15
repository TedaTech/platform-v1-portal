# Scaling Limits: Single-Realm / Single-Cluster Model

**Last updated:** 2026-02-15

## Target Scale

< 1,000 tenants on a single management cluster. Beyond this, adopt a multi-cluster model.

## Expected Bottleneck Timeline

| Tenant Count | What Breaks | Severity | Action Required |
|---|---|---|---|
| **~50-100** | JWT token exceeds default nginx header limit (4-8 KB) for users spanning many tenants | Low -- only affects multi-tenant admins | Tune `large_client_header_buffers` on nginx ingress, or filter groups in Keycloak token mapper |
| **~250** | Platform admins reliably hit proxy header limits; Keycloak admin console slows on group listing | Medium -- operational friction | **Planned tuning point.** Increase proxy header limits. Consider lightweight access tokens (Keycloak 24+). |
| **~1,000** | Flux single-instance reconciliation reaches ~4-5 min cycle time | Medium -- provisioning latency | Shard Flux controllers by label |
| **~2,500** | Keycloak hits 10K groups (official scalability target) | High -- login latency, admin console degradation | Database index tuning, Keycloak horizontal scaling |
| **~4,000** | Cilium security identity default limit (16,384) | High -- network policy stops working for new tenants | Raise to 64K, use namespace-level identity labels |
| **~5,000** | Keycloak group operations start timing out (20K groups) | Critical -- provisioning breaks | Multi-cluster or realm-per-tenant |
| **~10,000** | Kubernetes namespace limit (SIG-Scalability) | Critical -- cluster-level | Multi-cluster required |

## Hard Numbers by Component

### Keycloak (Single Realm, Group-Based Isolation)

| Metric | Value | Source |
|---|---|---|
| Groups before degradation | ~1,000-5,000 | Keycloak GitHub #19923, #22261 |
| Official scalability target | 10,000 groups | Keycloak GitHub #30085 |
| Admin console unusable | ~18,000 groups (~5 min load) | Keycloak GitHub #20489 |
| Transaction timeouts | ~20,000 groups | Keycloak GitHub #22261 |
| Largest known deployment | 43,000 groups (flat, no perf data) | Keycloak Discussion #31122 |
| Login latency (warm cache) | 47ms p99 | Keycloak 26.4 benchmarks |
| Login latency (cold cache, 25K groups, user in 200 groups) | Seconds to minutes | Keycloak Forum |
| Users per realm | Millions (no hard limit) | Keycloak Discussion #15181 |

**Per-tenant cost:** 4 groups. At 1,000 tenants = 4,000 groups (within comfort zone).

### JWT Token Size

| Groups per user | Approx. token overhead | Hits default nginx limit? |
|---|---|---|
| 20 (5 tenants) | ~0.5-1 KB | No |
| 100 (25 tenants) | ~3-5 KB | Borderline |
| 200 (50 tenants) | ~6-10 KB | Yes (4 KB HTTP/2, 8 KB HTTP/1.1) |
| 500 (125 tenants) | ~15-25 KB | Yes, all default configs |

**Kubernetes API server:** 1 MB default header limit (Go net/http). Not the bottleneck.

**Mitigation when needed:** Keycloak 24+ lightweight access tokens, group prefix filtering in token mapper, or increase proxy buffer sizes.

### FluxCD Reconciliation

| HelmReleases | Full reconciliation time |
|---|---|
| 100 | ~27s |
| 500 | ~2m 9s |
| 1,000 | ~4m 35s |
| 3,000+ | Sharding required |

**Throughput:** ~220 HelmReleases/minute. Sharding is label-based -- each shard handles ~1,000 HelmReleases independently.

### Cilium Network Policy

| Metric | Default | Maximum |
|---|---|---|
| Security identities | 16,384 | 65,536 |

At 1,000 namespaces with ~10 workloads each = ~10,000 identities. Within default limit.

### Kubernetes

| Metric | Tested limit |
|---|---|
| Namespaces per cluster | 10,000 (SIG-Scalability) |
| Objects per namespace | ~3,000 |
| Total etcd object count | Millions (with appropriate compaction) |

Each tenant namespace creates ~10-15 objects (ServiceAccount, Secrets, RoleBindings, NetworkPolicies, HelmRelease).

## Decision

Accept these limits for MVP. No tuning until we approach ~250 tenants. At that point, evaluate:

1. Proxy header buffer increases (quick fix)
2. Keycloak token mapper group filtering (medium effort)
3. Flux sharding (if reconciliation lag is noticeable)

Beyond ~1,000 tenants, adopt multi-cluster model. This also provides better blast radius isolation and independent failure domains, which are desirable properties regardless of scaling pressure.

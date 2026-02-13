# Container Deployment Footprint

## Summary

All SPA frameworks produce static files served from nginx. The differences are in JS bundle size (affecting page load), not container resource usage. Container footprint is **not a differentiating factor** for framework selection.

---

## Framework Runtime Sizes (min+gzip)

| Framework | Runtime Size | Source |
|-----------|-------------|--------|
| React 19 (react + react-dom) | ~45 KB | bundlephobia, FrontendTools |
| Vue 3 (core runtime) | ~34 KB | Vue.js docs, FrontendTools |
| Svelte 5 (compiled runtime) | ~3-4 KB | Svelte discussions, Khromov blog |
| Remix/React Router v7 | ~50-55 KB | React 19 + React Router |
| HTMX (client library) | ~14-16 KB | htmx.org |

Svelte's compiler-based architecture produces 10-15x smaller runtime than React/Vue.

## Estimated Total JS Bundle (medium-complexity portal, gzipped)

| Framework | Estimated Total | Notes |
|-----------|----------------|-------|
| React + Vite | 150-250 KB | Largest runtime; good tree-shaking |
| Vue + Vite | 130-220 KB | Slightly smaller runtime |
| SvelteKit static | 80-150 KB | Compiler eliminates runtime overhead |
| Remix SPA | 160-260 KB | React + react-dom + React Router |
| Go + HTMX | 14-16 KB (client) | All logic server-side |

With code splitting, initial chunk can be 60-100 KB with rest lazy-loaded.

## Docker Image Sizes

### Base Images

| Base Image | Compressed | On-Disk |
|------------|-----------|---------|
| nginx:stable-alpine-slim | ~12 MB | ~18 MB |
| nginx:alpine | ~26 MB | ~43 MB |
| caddy:alpine | ~25-30 MB | ~40-45 MB |
| scratch (Go) | 0 MB | 0 MB |
| alpine:3.21 (Go) | ~3.5 MB | ~7 MB |

### Total Image Estimates

| Stack | Base | App | Total Compressed |
|-------|------|-----|-----------------|
| React + Vite (nginx-slim) | 12 MB | 1-3 MB | **13-15 MB** |
| Vue + Vite (nginx-slim) | 12 MB | 1-3 MB | **13-15 MB** |
| SvelteKit static (nginx-slim) | 12 MB | 0.5-2 MB | **12-14 MB** |
| Remix SPA (nginx-slim) | 12 MB | 1-3 MB | **13-15 MB** |
| Go + HTMX (scratch) | 0 MB | 8-15 MB | **8-15 MB** |

All options are 8-18 MB. Differences are negligible for Kubernetes scheduling.

## Cold Start

| Stack | Cold Start | Details |
|-------|-----------|---------|
| Any SPA on nginx | < 100 ms | nginx starts in milliseconds, just serves files |
| Any SPA on caddy | < 200 ms | Go binary, slightly slower |
| Go + HTMX | < 200 ms | Go binary, near-instant |

In Kubernetes, the dominant cold-start factor is image pull time + scheduler + CNI setup (~1-2 seconds), not process start.

## Runtime Resources

### Memory

| Stack | Idle | Under Load (50-100 users) |
|-------|------|--------------------------|
| nginx (1 worker) | 2-5 MB | 5-15 MB |
| caddy | 10-15 MB | 15-30 MB |
| Go + HTMX | 8-15 MB | 20-50 MB |

### CPU

| Stack | Idle | Under Load |
|-------|------|-----------|
| nginx | ~0% | < 1% core |
| caddy | ~0% | < 2% core |
| Go + HTMX | ~0% | 5-15% core |

## Recommended Kubernetes Resources

### SPA (any framework, nginx-based)

```yaml
resources:
  requests:
    cpu: "10m"
    memory: "16Mi"
  limits:
    cpu: "100m"
    memory: "64Mi"
```

### Go + HTMX

```yaml
resources:
  requests:
    cpu: "50m"
    memory: "32Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

## Caching Strategy

SPA containers + nginx are extremely efficient:
- `Cache-Control: max-age=31536000, immutable` for hashed static assets (Vite generates content hashes)
- CDN can offload 100% of traffic after first pull
- Service Worker caching enables offline support
- Kubernetes ingress caching eliminates pod hits for repeat visitors

## Conclusion

**Framework choice should be driven by developer productivity and ecosystem, not container footprint.** All SPA options produce ~13-15 MB images with < 100ms cold start and < 16Mi memory. If bundle size is a tiebreaker, SvelteKit wins (80-150 KB vs 150-260 KB for React).

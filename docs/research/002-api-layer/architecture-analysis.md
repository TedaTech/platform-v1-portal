# Customer Portal Backend Architecture Analysis

**Date:** 2026-02-13
**Status:** Draft
**Author:** PM/Scrum Master Agent
**Repo:** TedaTech/platform-v1-portal

---

## 1. Introduction

This document evaluates two backend architecture patterns for the TedaTech Customer Portal:

1. **BFF Monolith** -- a single Backend-for-Frontend service
2. **API Gateway + Microservices** -- multiple independently deployed services behind a gateway

The Customer Portal is a React SPA (Vite + TanStack Router + Refine headless + shadcn/ui) that communicates with exactly two backend systems:

- **Kubernetes API** -- all infrastructure operations (tenant provisioning, Forgejo instances, Cozystack resources, pipeline orchestration) abstracted behind Custom Resource Definitions (CRDs)
- **KillBill API** -- billing, subscriptions, invoices, and payments via its REST API (250+ endpoints)

Authentication is handled by Keycloak via OIDC (using `oidc-spa` on the frontend). The BFF runs in-cluster as a Kubernetes Deployment. Ingress is provided by Cilium Gateway API. Secrets are managed by OpenBao. GitOps is driven by Flux.

### Constraints

| Constraint | Value |
|---|---|
| Team size | 1-3 developers |
| Stage | MVP |
| Backend count | 2 (K8s API, KillBill) |
| Infrastructure | Cozystack tenant with resource quotas |
| Frontend framework | React + Refine (expects `dataProvider` + `liveProvider`) |
| Auth | Keycloak OIDC, tokens validated server-side |
| Ingress | Cilium Gateway API (HTTPRoute) |

---

## 2. Architecture Option A: BFF Monolith

### 2.1 Description

A single Go service that:

1. **Validates OIDC tokens** from the frontend (introspection or local JWT validation against Keycloak's JWKS endpoint).
2. **Serves REST/GraphQL/tRPC endpoints** for all portal CRUD operations that the Refine `dataProvider` consumes.
3. **Runs Kubernetes informers** (via `controller-runtime` or `client-go`) to watch CRDs across tenant namespaces and streams real-time status updates to the frontend via Server-Sent Events (SSE), consumed by the Refine `liveProvider`.
4. **Calls the KillBill REST API** for billing operations (account lookup, subscriptions, invoices, payment methods).
5. **Maps user identity** from the Keycloak JWT (`sub` claim, custom claims) to the correct tenant namespace and KillBill account ID.

### 2.2 Component Diagram

```
                     +------------------+
                     |   React SPA      |
                     | (Refine + oidc)  |
                     +--------+---------+
                              |
                     OIDC Bearer Token
                              |
                     +--------v---------+
           +---------+  Cilium Gateway  +---------+
           |         |  (HTTPRoute)     |         |
           |         +------------------+         |
           |                                      |
           v                                      v
+----------+----------+              +-----------+-----------+
|   BFF Monolith Pod  |              |  Keycloak (OIDC IdP) |
|                     |              +-----------------------+
|  +---------------+  |
|  | Auth Middleware|  |    JWKS validation
|  +-------+-------+  |
|          |           |
|  +-------v-------+  |         +-------------------+
|  | HTTP Router   |  +-------->| KillBill API      |
|  | (REST/tRPC)   |  |  HTTP   | (billing backend) |
|  +---+-------+---+  |         +-------------------+
|      |       |       |
|  +---v---+ +-v----+  |
|  | K8s   | |Billing|  |
|  | Module | |Module |  |
|  +---+---+ +-------+  |
|      |                 |
|  +---v-----------+     |
|  | Informers     |     |
|  | (CRD watches) |     |
|  +---+-----------+     |
|      | SSE stream      |
+------+-----------------+
       |
       v
  K8s API Server
  (in-cluster)
```

### 2.3 Advantages

**Single deployment, single binary.** One container image, one Deployment manifest, one rollout. The CI/CD pipeline is trivial: build, test, push, update image tag in Flux GitOps.

**One auth boundary.** The OIDC token is validated once in the auth middleware. Every downstream call (K8s API, KillBill) uses service-level credentials (ServiceAccount token, API key). No inter-service authentication is needed because there is only one service.

**One Kubernetes ServiceAccount.** A single ServiceAccount with a ClusterRole (read CRDs across namespaces) and per-tenant-namespace RoleBindings (write CRDs for provisioning). RBAC is straightforward to audit.

**No inter-service communication overhead.** No service mesh, no mTLS between services, no distributed tracing required, no service discovery. Function calls replace network calls.

**Simpler local development.** Developers run one process. They can use `kind` or `minikube` with CRDs installed and a local KillBill instance (or mock). No docker-compose with 5 containers.

**Lower resource usage.** A Go binary with informers and an HTTP server typically consumes 50-100 MB of memory at rest, scaling to 128-256 MB under load with active informers and SSE connections. One pod instead of three or more.

**Coherent real-time pipeline.** K8s informers fire events in-process. The SSE handler can directly subscribe to informer event channels without an external event bus. This eliminates Redis/NATS as infrastructure dependencies and removes the fan-out complexity of distributing events across multiple service instances.

**Refine integration simplicity.** Both `dataProvider` (CRUD) and `liveProvider` (real-time) point to the same origin. No CORS complexity, no multiple WebSocket/SSE connections to different services.

### 2.4 Disadvantages

**Coupled scaling.** K8s informers and HTTP handlers scale together. If informer memory grows (many namespaces with large CRDs), the HTTP handler pods scale too, even if HTTP load is low. Conversely, high HTTP traffic (many concurrent billing queries) forces scaling pods that each maintain informer caches.

**Shared blast radius.** A panic in the billing module crashes the entire process, taking down provisioning status streams. A memory leak in informers kills billing endpoints. This can be partially mitigated with goroutine isolation, circuit breakers, and graceful degradation, but a process-level fault affects everything.

**Monolithic deploy cadence.** Changes to billing logic require redeploying the entire BFF, including re-establishing informer watches (brief reconnection period). Changes to K8s operations deploy billing code too. For a small team this is acceptable; for a larger team it becomes a coordination bottleneck.

**Testing surface.** Integration tests must cover both K8s operations and KillBill operations in the same test environment. This is manageable with mocks/fakes but requires discipline to keep modules decoupled internally.

---

## 3. Architecture Option B: API Gateway + Microservices

### 3.1 Description

Three or more independently deployed services:

1. **Portal API Gateway** -- authenticates OIDC tokens, routes requests to downstream services, may serve the SSE/WebSocket connection.
2. **K8s Provisioning Service** -- runs informers, handles CRD CRUD, publishes events.
3. **Billing Service** -- integrates with KillBill REST API, manages account mapping.

An event bus (Redis Pub/Sub, NATS, or similar) fans out real-time events from the Provisioning Service to the Gateway for SSE delivery.

### 3.2 Component Diagram

```
                     +------------------+
                     |   React SPA      |
                     +--------+---------+
                              |
                     +--------v---------+
                     |  Cilium Gateway  |
                     +--------+---------+
                              |
                     +--------v---------+
                     | Portal API GW    |
                     | (auth + routing) |
                     +--+------+----+---+
                        |      |    |
              +---------+  +---+  +-+----------+
              v            v       v            v
    +---------+--+ +------+----+ ++----------+ +-------+
    | Provisioning| | Billing  | | Event Bus | |Keycloak|
    | Service     | | Service  | | (NATS/    | +-------+
    | (informers) | | (KillBill| | Redis)    |
    +------+------+ +----------+ +-----+-----+
           |                           |
    K8s API Server              fan-out events
```

### 3.3 Advantages

**Independent scaling.** The Provisioning Service can scale independently from the Billing Service. If informer memory grows, only provisioning pods scale. If billing traffic spikes (invoice generation day), only billing pods scale.

**Fault isolation.** A crash in the Billing Service does not affect provisioning status streams. Circuit breakers at the gateway level can return partial responses.

**Independent deployment.** Teams can deploy billing changes without redeploying provisioning. Different release cadences are possible.

**Team independence.** Different developers or teams can own different services with clear API contracts between them.

### 3.4 Disadvantages

**Operational complexity is significantly higher.** Three or more Deployments, Services, ServiceAccounts, ConfigMaps, Secrets, and health checks. Monitoring must cover inter-service latency, error rates between services, and event bus health. Debugging a user-facing error requires correlating logs across multiple services -- distributed tracing (OpenTelemetry) becomes essential rather than optional.

**Real-time event fan-out is the hardest problem.** The Provisioning Service watches K8s CRDs and generates events. These events must reach the API Gateway, which holds the SSE connections to browsers. This requires:

- An event bus (Redis Pub/Sub, NATS JetStream, or similar)
- Message serialization/deserialization
- Delivery guarantees (at-least-once, ordering)
- Handling of gateway pod restarts (client reconnection, event replay)

This is a substantial piece of infrastructure that does not exist in the monolith option. It adds 1-2 additional pods (event bus), operational burden, and latency to real-time updates.

**Inter-service authentication.** Services must authenticate calls from each other. Options include mTLS (via service mesh like Cilium's mutual authentication), shared secrets, or JWT propagation. Each adds configuration and operational overhead.

**Higher resource usage.** Three Go services at 50-80 MB each, plus an event bus at 50-100 MB, totals 200-340 MB minimum. With replicas for availability, this doubles. On a resource-constrained Cozystack tenant, this is significant.

**Development complexity.** Local development requires running 3+ services plus an event bus. Docker Compose or Tilt configurations become necessary. The feedback loop is slower. End-to-end testing requires orchestrating multiple services.

**Not justified for 2 backends.** The microservices pattern shines when there are many backends with different scaling characteristics, owned by different teams. With exactly 2 backends and 1-3 developers, the overhead outweighs the benefits.

---

## 4. Evaluation Matrix

### 4.1 Operational Complexity

**BFF Monolith:**
- 1 Deployment, 1 Service, 1 ServiceAccount
- Logs from a single process; grep by request ID
- Health checks: `/healthz` (liveness), `/readyz` (K8s API reachable + KillBill reachable)
- Monitoring: standard HTTP metrics (latency, error rate, active connections) + informer metrics (cache sync status, watch errors)
- Debugging: single process, standard Go profiling (pprof), single stack trace on panic

**API Gateway + Microservices:**
- 3+ Deployments, 3+ Services, 3+ ServiceAccounts, 1 event bus deployment
- Distributed logs require correlation IDs and a log aggregation system
- Health checks per service, plus event bus health
- Monitoring: inter-service latency, event bus lag, per-service error rates
- Debugging: distributed tracing (OpenTelemetry), cross-service correlation

**Verdict:** BFF Monolith is dramatically simpler. The operational overhead of microservices is not justified at MVP stage with a small team.

### 4.2 Resource Usage

| Resource | BFF Monolith | Microservices |
|---|---|---|
| Pods (no replicas) | 1 | 4+ (3 services + event bus) |
| Pods (with replicas) | 2 | 8+ |
| Memory (base) | 80-128 MB | 250-400 MB |
| Memory (with replicas) | 160-256 MB | 500-800 MB |
| CPU (idle) | 50-100m | 200-400m |
| CPU (peak) | 200-500m | 400-1000m |
| Network overhead | None (in-process) | Inter-service + event bus |
| Config artifacts | ~6 (Deploy, Svc, SA, CR, CM, Secret) | ~20+ |

**Verdict:** BFF Monolith uses 3-4x fewer resources. On a Cozystack tenant with resource quotas, this difference is material.

### 4.3 Development Velocity

**BFF Monolith:**
- Clone one repo, `go run ./cmd/bff`
- Tests: `go test ./...` covers everything
- CI: one pipeline, one image, one deploy
- New developer onboarding: understand one codebase
- Feature development: change one service, test locally, push

**API Gateway + Microservices:**
- Clone 3+ repos (or monorepo with 3+ services)
- Local dev: docker-compose with 4+ containers, or Tilt/Skaffold
- Tests: per-service unit tests + integration tests across services
- CI: 3+ pipelines, 3+ images, coordinated deploys
- New developer onboarding: understand service boundaries, event contracts, inter-service auth
- Feature development: may require changes across 2+ services for a single feature

**Verdict:** BFF Monolith enables significantly faster iteration. For an MVP with 1-3 developers, development velocity is the most important factor.

### 4.4 Scaling Characteristics

**BFF Monolith scaling model:**

The BFF Monolith scales horizontally by adding replicas. Each replica maintains its own informer cache (watch connections to the K8s API server) and handles a share of HTTP/SSE traffic via Cilium load balancing.

Scaling bottlenecks emerge at:
- **~1000 concurrent SSE connections per pod:** Go's goroutine-per-connection model handles this well, but memory for connection buffers accumulates. At 10 KB per connection, 1000 connections = 10 MB additional memory.
- **~50-100 watched namespaces per pod:** Each namespace watch adds a long-poll connection to the API server and an in-memory cache of CRD objects. At 1-5 MB per namespace (depending on CRD size and count), 100 namespaces = 100-500 MB cache.
- **API server watch limits:** The K8s API server has practical limits on concurrent watch connections. With `controller-runtime`'s shared informer pattern, watches are multiplexed efficiently, but at scale (hundreds of namespaces), the API server load becomes the bottleneck.

**Practical scaling curve:**

| Tenants | Namespaces | SSE Connections | BFF Pods | Memory/Pod | Total Memory |
|---|---|---|---|---|---|
| 10 | 10 | ~50 | 1 | 128 MB | 128 MB |
| 50 | 50 | ~250 | 2 | 200 MB | 400 MB |
| 100 | 100 | ~500 | 2-3 | 256 MB | 512-768 MB |
| 500 | 500 | ~2500 | 5-8 | 300 MB | 1.5-2.4 GB |
| 1000+ | 1000+ | ~5000 | Extract | -- | -- |

At the 500-1000 tenant mark, the monolith's coupled scaling (informers + HTTP) becomes inefficient. This is the extraction trigger point.

**Microservices scaling model:**

Provisioning pods scale independently from billing pods. The Provisioning Service can use fewer, larger pods with big informer caches, while the Billing Service uses more, smaller pods for HTTP throughput. However, the event bus becomes a new scaling bottleneck and single point of failure.

**Verdict:** The BFF Monolith comfortably handles the MVP scale (10-100 tenants) and likely the first 500 tenants. The microservices pattern only becomes advantageous beyond that threshold, which is a problem for the future, not for the MVP.

### 4.5 Fault Isolation

**BFF Monolith:**
- A panic in any module crashes the process. Mitigated by: recover() in goroutines, circuit breakers around external calls, graceful shutdown.
- KillBill outage: billing endpoints return errors, but K8s informers and SSE streams continue. The failure is contained at the module level if the billing module does not block shared resources.
- K8s API outage: informers stop receiving events (SSE streams stale), but billing endpoints continue.
- Memory leak in informers: affects the entire process. Mitigated by memory limits on the pod (OOM kill and restart).

**Microservices:**
- A crash in the Billing Service is completely isolated. Provisioning continues.
- A crash in the Provisioning Service is isolated. Billing continues.
- However: if the API Gateway crashes, everything is down. The gateway is a single point of failure unless replicated and health-checked.
- Event bus failure: real-time updates stop, but CRUD operations continue.

**Verdict:** Microservices provide better fault isolation in theory. In practice, the BFF Monolith's fault isolation is sufficient for the MVP if proper circuit breakers and module boundaries are implemented. The additional failure modes introduced by microservices (event bus failure, inter-service timeouts, partial failures) can actually reduce overall reliability for a small team that cannot invest in comprehensive resilience patterns.

### 4.6 Auth Model

**BFF Monolith:**
1. Frontend sends OIDC Bearer token in `Authorization` header.
2. BFF auth middleware validates the JWT against Keycloak's JWKS endpoint (cached).
3. Middleware extracts user identity (sub, email, custom claims like `tenant_id`).
4. K8s operations use the pod's ServiceAccount token (in-cluster config). Tenant namespace is derived from user identity.
5. KillBill operations use API key + API secret from OpenBao. KillBill account is looked up by external key (Keycloak user ID).
6. No inter-service auth needed.

**Microservices:**
1. Frontend sends OIDC Bearer token.
2. API Gateway validates JWT, extracts identity.
3. Gateway propagates identity to downstream services via:
   - JWT forwarding (each service validates again), or
   - Internal token (gateway issues a short-lived internal JWT), or
   - mTLS + header propagation (trust the gateway, pass identity as headers)
4. Each service has its own ServiceAccount with minimal RBAC.
5. Inter-service auth must be secured (compromised billing service should not access K8s API).

**Verdict:** BFF Monolith has a dramatically simpler auth model. One token validation, one trust boundary. Microservices introduce inter-service auth as a new attack surface and operational burden.

### 4.7 Real-Time Integration

This is the most architecturally significant difference between the two patterns.

**BFF Monolith -- in-process event pipeline:**

```go
// Informer event handler (runs in informer goroutine)
func (h *Handler) OnUpdate(oldObj, newObj interface{}) {
    tenant := extractTenant(newObj)
    event := mapToPortalEvent(newObj)
    h.eventBus.Publish(tenant, event) // in-process channel
}

// SSE handler (runs in HTTP goroutine)
func (h *Handler) StreamEvents(w http.ResponseWriter, r *http.Request) {
    tenant := r.Context().Value("tenant").(string)
    ch := h.eventBus.Subscribe(tenant)
    defer h.eventBus.Unsubscribe(tenant, ch)

    flusher := w.(http.Flusher)
    for event := range ch {
        fmt.Fprintf(w, "data: %s\n\n", event.JSON())
        flusher.Flush()
    }
}
```

The event pipeline is a Go channel. No serialization, no network hop, no delivery guarantees to worry about (the channel is the guarantee). Latency from CRD change to browser is bounded by K8s informer resync interval (typically <1 second) plus network latency to the browser.

With multiple BFF replicas, each replica watches the same CRDs and serves its own SSE connections. There is no cross-replica event routing needed because each replica has its own informer cache. The only cost is duplicated API server watches, which is the standard pattern for Kubernetes controllers and is well-supported by the API server.

**Microservices -- distributed event pipeline:**

```
K8s API Server
    |
    v (watch)
Provisioning Service (informers)
    |
    v (publish)
Event Bus (NATS/Redis)
    |
    v (subscribe)
API Gateway (SSE handler)
    |
    v (SSE)
Browser
```

The event pipeline requires:
1. Provisioning Service serializes events to JSON/protobuf.
2. Publishes to NATS subject or Redis channel (keyed by tenant).
3. API Gateway subscribes to relevant NATS subjects/Redis channels.
4. Deserializes events and writes to SSE stream.

Additional concerns:
- **Message ordering:** Must be guaranteed per tenant. NATS JetStream provides this; Redis Pub/Sub does not (use Redis Streams instead).
- **Delivery guarantees:** If the Gateway pod restarts, it must catch up on missed events. This requires persistent message storage (NATS JetStream, Redis Streams) and client-side reconnection with `Last-Event-ID`.
- **Backpressure:** If the browser is slow to consume SSE events, the Gateway buffers them. If the buffer fills, the Gateway must apply backpressure to the event bus subscription without affecting other subscribers.
- **Event schema contract:** A versioned schema must be defined between the Provisioning Service and the API Gateway. Schema evolution must be coordinated across services.

**Verdict:** The BFF Monolith's real-time pipeline is dramatically simpler and more reliable. The microservices pattern introduces an event bus as critical infrastructure, adds latency, and creates a new class of failure modes (event bus outage, message loss, ordering violations). For a small team building an MVP, this complexity is not justified.

---

## 5. The Hybrid Option: BFF Monolith Now, Extract Later

### 5.1 The Pattern

The "monolith-first" pattern is the dominant industry practice for new products. Start with a well-structured monolith that has clean internal module boundaries. Extract modules into microservices only when specific, measurable triggers are hit.

This pattern is not a compromise -- it is the recommended approach:

- **Shopify** ran a monolithic Rails application for years before introducing their "modular monolith" pattern, where internal boundaries (engines) enable future extraction without the operational cost of microservices. They explicitly chose _not_ to go full microservices, instead using internal modularity to achieve team independence.
- **GitHub** operated as a monolithic Rails application for over a decade. When they eventually extracted services, they started with the highest-value target (Authentication and Authorization) and used Twirp (gRPC-like) for service-to-service communication. The extraction was driven by specific organizational and scaling needs, not by architectural dogma.
- **Basecamp/37signals** runs a monolithic Ruby application serving millions of users, demonstrating that monoliths can scale far beyond what most teams will ever need.

### 5.2 Internal Module Structure for Future Extraction

The BFF Monolith should be structured with explicit internal boundaries from day one:

```
cmd/
  bff/
    main.go                    # Wires everything together

internal/
  auth/                        # OIDC token validation, identity extraction
    middleware.go
    claims.go
    keycloak_jwks.go

  tenant/                      # Tenant resolution: identity -> namespace + KillBill account
    resolver.go
    mapper.go

  k8sops/                      # Kubernetes operations module
    client.go                  # K8s client wrapper
    informers.go               # Informer setup and lifecycle
    crd_tenants.go             # Tenant CRD operations
    crd_forgejo.go             # Forgejo CRD operations
    crd_cozystack.go           # Cozystack CRD operations
    events.go                  # Event types and in-process event bus

  billing/                     # KillBill integration module
    client.go                  # KillBill HTTP client
    accounts.go                # Account operations
    subscriptions.go           # Subscription operations
    invoices.go                # Invoice operations
    payments.go                # Payment operations
    webhooks.go                # KillBill push notification handler

  api/                         # HTTP layer
    router.go                  # Route registration
    middleware.go              # Logging, recovery, CORS
    handlers_k8s.go            # Handlers that delegate to k8sops
    handlers_billing.go        # Handlers that delegate to billing
    handlers_sse.go            # SSE stream handler
    openapi.go                 # OpenAPI spec generation (if REST)

  events/                      # In-process event bus
    bus.go                     # Publish/Subscribe with Go channels
    types.go                   # Event types shared between modules
```

Key design principles:

1. **Modules communicate through Go interfaces, not concrete types.** The `api` package depends on interfaces like `k8sops.TenantService` and `billing.SubscriptionService`, not on their implementations.
2. **No cross-module database access.** Each module owns its data. (In this case, K8s is the database for provisioning, KillBill is the database for billing.)
3. **The event bus is an interface.** Today it is backed by Go channels. Tomorrow it can be backed by NATS or Redis without changing the modules that publish/subscribe.
4. **Tenant resolution is a shared concern.** The `tenant` package maps user identity to namespace and KillBill account. Both `k8sops` and `billing` depend on it.

### 5.3 Extraction Playbook

When the time comes to extract a module, the process is:

1. **Promote the module's interface to a protobuf/OpenAPI service definition.** The Go interface `billing.SubscriptionService` becomes a gRPC service definition or REST API spec.
2. **Implement the service definition in a new binary.** Move `internal/billing/` to a new `cmd/billing-service/` with an RPC server.
3. **Replace the in-process call with an RPC client.** In the BFF, replace the concrete `billing.Client` with a gRPC/HTTP client that calls the new service.
4. **Replace the in-process event bus with a distributed one.** The `events.Bus` interface implementation switches from Go channels to NATS/Redis.
5. **Deploy independently.** The billing service gets its own Deployment, Service, and CI/CD pipeline.

Because the interfaces are already defined, this extraction is a mechanical refactoring, not a redesign.

---

## 6. Deployment Pattern on Cozystack

### 6.1 BFF Monolith Kubernetes Resources

```yaml
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portal-bff
  namespace: portal-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: portal-bff
  template:
    metadata:
      labels:
        app: portal-bff
    spec:
      serviceAccountName: portal-bff
      containers:
      - name: bff
        image: registry.teda.tech/portal-bff:latest
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "250m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: KILLBILL_API_URL
          valueFrom:
            configMapKeyRef:
              name: portal-bff-config
              key: killbill-api-url
        - name: KILLBILL_API_KEY
          valueFrom:
            secretKeyRef:
              name: portal-bff-secrets
              key: killbill-api-key
        - name: KILLBILL_API_SECRET
          valueFrom:
            secretKeyRef:
              name: portal-bff-secrets
              key: killbill-api-secret
        - name: KEYCLOAK_ISSUER_URL
          valueFrom:
            configMapKeyRef:
              name: portal-bff-config
              key: keycloak-issuer-url
        volumeMounts:
        - name: secrets
          mountPath: /etc/portal-bff/secrets
          readOnly: true
      volumes:
      - name: secrets
        secret:
          secretName: portal-bff-secrets
---
# Service
apiVersion: v1
kind: Service
metadata:
  name: portal-bff
  namespace: portal-system
spec:
  selector:
    app: portal-bff
  ports:
  - port: 80
    targetPort: 8080
    name: http
---
# Cilium HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: portal-bff
  namespace: portal-system
spec:
  parentRefs:
  - name: platform-gateway
    namespace: gateway-system
  hostnames:
  - "portal-api.teda.tech"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: portal-bff
      port: 80
---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: portal-bff
  namespace: portal-system
---
# ClusterRole (read CRDs across namespaces)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portal-bff
rules:
- apiGroups: ["teda.tech"]
  resources: ["tenants", "forgejoes", "cozystacks", "pipelines"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["teda.tech"]
  resources: ["tenants", "forgejoes", "cozystacks", "pipelines"]
  verbs: ["create", "update", "patch", "delete"]
---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portal-bff
subjects:
- kind: ServiceAccount
  name: portal-bff
  namespace: portal-system
roleRef:
  kind: ClusterRole
  name: portal-bff
  apiGroup: rbac.authorization.k8s.io
---
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: portal-bff-config
  namespace: portal-system
data:
  killbill-api-url: "http://killbill.billing-system.svc.cluster.local:8080"
  keycloak-issuer-url: "https://auth.teda.tech/realms/platform"
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: portal-bff
  namespace: portal-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: portal-bff
  minReplicas: 2
  maxReplicas: 8
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        targetAverageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        targetAverageUtilization: 80
```

### 6.2 Health Check Design

**Liveness probe (`/healthz`):**
- Returns 200 if the process is alive and not deadlocked.
- Should NOT check external dependencies (K8s API, KillBill). A liveness probe failure causes a pod restart, which does not fix an external dependency outage.

**Readiness probe (`/readyz`):**
- Returns 200 only if:
  - K8s informer caches are synced (initial list completed)
  - KillBill API is reachable (lightweight health check call)
- A readiness probe failure removes the pod from the Service endpoints, so traffic is routed to healthy pods.

### 6.3 Resource Quotas

For a Cozystack tenant deployment, the BFF should fit within these resource quotas:

| Component | Requests | Limits |
|---|---|---|
| BFF Pod (each) | 128 Mi / 100m CPU | 256 Mi / 250m CPU |
| BFF (2 replicas) | 256 Mi / 200m CPU | 512 Mi / 500m CPU |

Compare with microservices (3 services x 2 replicas + event bus):

| Component | Requests | Limits |
|---|---|---|
| Microservices (8 pods) | 640 Mi / 400m CPU | 1280 Mi / 1000m CPU |

The BFF Monolith uses 2.5-3x fewer resources.

---

## 7. Scaling Trigger Analysis

The BFF Monolith should be split when any of the following thresholds are crossed. These are guidelines, not hard rules -- the decision should be based on observed pain points, not preemptive optimization.

### 7.1 Connection Count

**Trigger:** >1000 concurrent SSE/WebSocket connections per pod.

**Why:** Each SSE connection holds a goroutine and a TCP connection. At 1000 connections per pod with 8 pods (HPA max), that is 8000 concurrent connections, serving roughly 2000-4000 active users (multiple tabs, reconnections). Beyond this, the connection handling should be extracted into a dedicated "real-time gateway" service that does not carry informer cache weight.

**Mitigation before extraction:** Implement connection pooling, reduce SSE keep-alive frequency, use HTTP/2 multiplexing (fewer TCP connections).

### 7.2 Informer Load

**Trigger:** >100 namespaces watched per pod, or informer cache exceeding 500 MB per pod.

**Why:** Each namespace watch creates one long-poll connection to the API server. The informer cache stores all watched objects in memory. At 100+ namespaces with multiple CRD types each, the cache grows significantly, and the API server may throttle watch connections.

**Mitigation before extraction:** Use `controller-runtime`'s filtered cache (watch only relevant fields), implement namespace sharding (each BFF replica watches a subset of namespaces), or use server-side field selectors to reduce cache size.

### 7.3 Team Size

**Trigger:** >4 developers working on the BFF simultaneously.

**Why:** With 4+ developers, merge conflicts increase, the monolithic deploy cadence becomes a coordination bottleneck, and different developers may want to use different release schedules. This is the organizational trigger, not a technical one.

**Mitigation before extraction:** Strong internal module boundaries, feature flags, trunk-based development with short-lived branches.

### 7.4 Deploy Frequency Divergence

**Trigger:** One module (e.g., billing) changes 3+ times per week while the other (e.g., K8s ops) is stable, or vice versa.

**Why:** Deploying stable code alongside frequently changing code introduces unnecessary risk. If billing is changing daily but K8s operations have not changed in weeks, every billing deploy needlessly re-establishes informer watches.

**Mitigation before extraction:** Use feature flags to decouple deploy from release. Only extract when the deploy cadence divergence is persistent (>1 month) and causing actual friction.

### 7.5 Decision Framework

```
Is a specific, measurable threshold crossed?
  NO  --> Keep the monolith. Premature extraction is worse than no extraction.
  YES --> Which threshold?
    Connection count --> Extract real-time gateway first
    Informer load   --> Extract K8s provisioning service first
    Team size       --> Extract the module with the most active development
    Deploy cadence  --> Extract the frequently-changing module
```

---

## 8. Recommendation

### Recommendation: BFF Monolith

**For the TedaTech Customer Portal MVP and foreseeable growth (up to ~500 tenants), a BFF Monolith is the correct architecture.**

The rationale is unambiguous:

1. **Two backends, not twenty.** The microservices pattern is designed for systems that integrate many backends with divergent scaling characteristics. With exactly two backends (K8s API and KillBill), the overhead of service decomposition far exceeds any benefit.

2. **Small team, high velocity.** With 1-3 developers, every hour spent on inter-service auth, distributed tracing, event bus configuration, and multi-service local development is an hour not spent building the product. The MVP must ship fast.

3. **Resource constraints.** A Cozystack tenant has finite resources. The BFF Monolith uses 256-512 MB total memory. Microservices would use 640-1280 MB for the same functionality. That is memory and CPU that could serve actual tenant workloads.

4. **Real-time is the deciding factor.** The SSE/WebSocket integration with K8s informers is architecturally simple in a monolith (Go channels) and architecturally complex in microservices (event bus, delivery guarantees, ordering, replay). For an MVP, the monolith's in-process event pipeline is a massive simplification.

5. **The hybrid path is always available.** By structuring the monolith with clean module boundaries (interfaces, no cross-module data access, event bus abstraction), any module can be extracted into a standalone service when specific, measurable triggers are hit. This is not a one-way door.

### Conditions for Reconsideration

Revisit this decision when:

| Condition | Threshold | Action |
|---|---|---|
| Concurrent SSE connections | >1000 per pod | Extract real-time gateway |
| Informer cache memory | >500 MB per pod | Extract K8s provisioning service |
| Watched namespaces | >100 per pod | Implement namespace sharding or extract |
| Team size | >4 developers on BFF | Extract most active module |
| Deploy cadence divergence | Persistent >1 month | Extract frequently-changing module |
| Tenant count | >500 | Full architecture review |

### What NOT to do

- Do not introduce an event bus (NATS, Redis Streams) until a distributed event pipeline is actually needed.
- Do not introduce a service mesh for inter-service auth when there is only one service.
- Do not decompose into microservices preemptively to "prepare for scale." The modular monolith structure prepares for scale without the operational cost.
- Do not use GraphQL federation or API gateway patterns that assume multiple backend services.

---

## 9. Summary Table

| Criterion | BFF Monolith | API Gateway + Microservices |
|---|---|---|
| **Operational Complexity** | Low. 1 Deployment, standard monitoring. | High. 4+ Deployments, distributed tracing required. |
| **Resource Usage** | 256-512 MB (2 replicas) | 640-1280 MB (8+ pods) |
| **Development Velocity** | High. Single binary, fast feedback loop. | Low. Multi-service orchestration, slower iteration. |
| **Scaling** | Vertical + horizontal, sufficient to ~500 tenants. | Independent per service, beneficial at >500 tenants. |
| **Fault Isolation** | Module-level (circuit breakers, goroutine isolation). | Service-level (process boundary). |
| **Auth Model** | Simple. One token validation, no inter-service auth. | Complex. Token propagation, inter-service auth. |
| **Real-time (SSE)** | Trivial. In-process Go channels from informers. | Complex. Requires event bus infrastructure. |
| **Local Development** | One process, `go run`. | 4+ containers, docker-compose/Tilt. |
| **CI/CD** | One pipeline, one image, one deploy. | 3+ pipelines, coordinated deploys. |
| **Future Extraction** | Clean interfaces enable mechanical extraction. | Already decomposed (but prematurely). |
| **Recommendation** | **Use this for MVP and growth to ~500 tenants.** | Revisit when specific scaling triggers are hit. |

---

## 10. References

- [Shopify: Deconstructing the Monolith](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity) -- Shopify's modular monolith pattern
- [GitHub's Journey from Monolith to Microservices](https://www.infoq.com/articles/github-monolith-microservices/) -- GitHub's incremental extraction approach
- [Backend for Frontend (BFF) Architecture in 2025](https://devtechinsights.com/backend-for-frontend-bff-architecture-2025/) -- BFF pattern overview and best practices
- [BFF Pattern: Microservices for UX](https://goteleport.com/learn/cybersecurity-best-practices/backend-for-frontend-bff-pattern/) -- Security considerations for BFF
- [Kill Bill: Billing System Architecture](https://blog.killbill.io/blog/kill-bill-billing-system-architecture/) -- KillBill's layered service architecture
- [Kill Bill Subscription Billing APIs](https://blog.killbill.io/blog/all-about-the-kill-bill-subscription-billing-apis/) -- KillBill REST API design
- [Kubernetes Considerations for Large Clusters](https://kubernetes.io/docs/setup/best-practices/cluster-large/) -- K8s scaling limits
- [Cilium Gateway API Support](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/) -- Cilium HTTPRoute configuration
- [controller-runtime Package](https://pkg.go.dev/sigs.k8s.io/controller-runtime) -- Go library for K8s informers
- [Mastering Kubernetes Informers](https://www.plural.sh/blog/manage-kubernetes-events-informers/) -- Informer architecture deep dive
- [Minimizing Informer Memory Usage](https://github.com/kubernetes/client-go/issues/832) -- client-go memory optimization discussion

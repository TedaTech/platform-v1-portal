# K8s Interaction Patterns for Customer Portal BFF

Design document for the Go BFF monolith's interaction with Kubernetes resources. The BFF runs in-cluster, exposes a REST API (OpenAPI) with SSE for real-time updates, and uses a ServiceAccount with ClusterRole RBAC.

---

## Table of Contents

1. [Client Architecture](#1-client-architecture)
2. [Informer Setup for Real-Time](#2-informer-setup-for-real-time)
3. [CRD Operations (CRUD)](#3-crd-operations-crud)
4. [Real-Time Event Pipeline](#4-real-time-event-pipeline)
5. [Provisioning Status Pattern](#5-provisioning-status-pattern)
6. [Long-Running Operation Pattern](#6-long-running-operation-pattern)
7. [Custom Portal CRDs vs Direct CRD Manipulation](#7-custom-portal-crds-vs-direct-crd-manipulation)
8. [Namespace Strategy and RBAC](#8-namespace-strategy-and-rbac)
9. [Resource Versioning and Optimistic Concurrency](#9-resource-versioning-and-optimistic-concurrency)
10. [Testing Strategy](#10-testing-strategy)

---

## 1. Client Architecture

### Options Evaluated

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: controller-runtime `client.Client`** | Higher-level client with cache-backed reads via `manager.Manager` | Cache-backed reads (fast), typed access, handles informer lifecycle, consistent with hetzner-operators patterns | Designed around reconciler pattern; BFF is not a controller |
| **B: client-go dynamic client** | Lower-level `dynamic.Interface` for arbitrary CRDs | Works with any CRD without code-gen, flexible | No typed access (everything is `unstructured.Unstructured`), no built-in cache, verbose error-prone code |
| **C: Typed client-go with code-gen** | Generate typed clientsets per CRD group | Fully typed, IDE autocompletion, compile-time safety | Requires code-gen tooling for each CRD group, more build complexity, separate informer setup |

### Recommendation: Option A -- controller-runtime `client.Client` with `manager.Manager`

The controller-runtime `manager.Manager` is not limited to reconciler-based controllers. It manages informer lifecycle, provides a shared cache, and exposes a `client.Client` that reads from the informer cache and writes directly to the API server. This is the same pattern used in the hetzner-operators repo, providing consistency across the platform.

The BFF uses the manager purely for its cache and client infrastructure, without registering any reconcilers. The HTTP server and SSE handlers are registered as `manager.Runnable` instances so the manager coordinates their lifecycle (graceful shutdown, leader election if needed).

### In-Cluster Configuration

```go
package main

import (
    "os"

    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/cache"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/manager"

    platformv1alpha1 "github.com/tedatech/platform-v1-portal/api/v1alpha1"
)

func main() {
    // In-cluster config is automatic: reads the ServiceAccount token
    // mounted at /var/run/secrets/kubernetes.io/serviceaccount/
    mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), manager.Options{
        Scheme: scheme, // registered with all CRD types
        Cache:  cacheOptions(),
        // No controller registrations -- BFF uses cache + client only
    })
    if err != nil {
        os.Exit(1)
    }

    // Register the HTTP/SSE server as a Runnable
    httpServer := NewHTTPServer(mgr.GetClient(), mgr.GetCache())
    if err := mgr.Add(httpServer); err != nil {
        os.Exit(1)
    }

    // Start blocks until the context is cancelled
    if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
        os.Exit(1)
    }
}
```

### Scheme Registration

All CRD types the BFF interacts with must be registered in the runtime scheme. This allows the controller-runtime client and cache to work with typed objects.

```go
package main

import (
    "k8s.io/apimachinery/pkg/runtime"
    utilruntime "k8s.io/apimachinery/pkg/util/runtime"
    clientgoscheme "k8s.io/client-go/kubernetes/scheme"

    platformv1alpha1 "github.com/tedatech/platform-v1-portal/api/v1alpha1"
    cozystackv1alpha1 "github.com/aenix-io/cozystack/api/v1alpha1"
    edpv1 "github.com/epam/edp-keycloak-operator/api/v1"
    sourcev1 "github.com/fluxcd/source-controller/api/v1"
    kustomizev1 "github.com/fluxcd/kustomize-controller/api/v1"
)

var scheme = runtime.NewScheme()

func init() {
    utilruntime.Must(clientgoscheme.AddToScheme(scheme))
    utilruntime.Must(platformv1alpha1.AddToScheme(scheme))    // ForegejoTenant
    utilruntime.Must(cozystackv1alpha1.AddToScheme(scheme))   // Cozystack Tenant
    utilruntime.Must(edpv1.AddToScheme(scheme))               // KeycloakClient, KeycloakRealmUser, etc.
    utilruntime.Must(sourcev1.AddToScheme(scheme))             // GitRepository
    utilruntime.Must(kustomizev1.AddToScheme(scheme))          // Kustomization
}
```

### Direct API Reader

For operations that must bypass the cache (e.g., reading a resource immediately after creation to confirm it exists on the API server), use the manager's API reader:

```go
// Cache-backed read (fast, eventually consistent)
err := mgr.GetClient().Get(ctx, key, &obj)

// Direct API server read (slower, strongly consistent)
err := mgr.GetAPIReader().Get(ctx, key, &obj)
```

Use the direct reader sparingly. Every direct call results in a quorum read from etcd, which is costlier than reading from the informer cache.

---

## 2. Informer Setup for Real-Time

### Problem Statement

The BFF needs to watch CRD status changes across all tenant namespaces (`tenant-{id}`) and relay those changes to connected SSE clients. The informer cache strategy determines memory usage, API server load, and implementation complexity.

### Options Evaluated

| Option | Description | Memory at 10 Tenants | Memory at 100+ Tenants | Complexity |
|--------|-------------|----------------------|------------------------|------------|
| **A: Cluster-scoped cache** | Watch all namespaces, filter events in handler | Low (small total object count) | Medium-High (caches all objects including non-portal ones) | Low |
| **B: Namespace-scoped caches** | One cache per tenant namespace, dynamic lifecycle | Lowest (only portal objects) | Lowest (only portal objects) | High (add/remove caches dynamically) |
| **C: Label-selector cache** | Watch all namespaces, only objects with `tedatech.app/portal-managed: "true"` | Low | Low (only labeled objects) | Low-Medium |

### Recommendation: Option C -- Label-Selector Cache (with fallback plan)

Use a cluster-scoped cache with a label selector to reduce the cached object set. All CRDs created by the portal BFF are labeled with `tedatech.app/portal-managed: "true"`. The cache only watches objects with this label, regardless of namespace.

This balances simplicity (single cache, no dynamic lifecycle) with efficiency (only portal-relevant objects are cached). If scaling beyond hundreds of tenants becomes a concern, migrate to Option B with dynamic namespace caches.

For CRDs not created by the BFF (e.g., Flux `GitRepository` and `Kustomization` that are created by GitOps), use the `AllNamespaces` key with a field or label selector that matches only tenant-prefixed namespaces, or watch them with a broader cache and filter in the event handler.

### Cache Configuration

```go
func cacheOptions() cache.Options {
    portalManagedSelector := labels.SelectorFromSet(labels.Set{
        "tedatech.app/portal-managed": "true",
    })

    return cache.Options{
        // Default: only cache objects with the portal-managed label
        DefaultLabelSelector: portalManagedSelector,

        // Override for CRDs not labeled by the portal (Flux, Cozystack native)
        ByObject: map[client.Object]cache.ByObject{
            // Flux GitRepository: watch all in tenant namespaces, no label filter
            // The BFF does not create these, so they won't have the portal label.
            // Filter by namespace prefix in the event handler instead.
            &sourcev1.GitRepository{}: {
                Label: labels.Everything(),
            },
            &kustomizev1.Kustomization{}: {
                Label: labels.Everything(),
            },
        },

        // Strip managed fields from cached objects to reduce memory
        DefaultTransform: cache.TransformStripManagedFields(),
    }
}
```

### Informer Registration

The controller-runtime cache lazily creates informers when a type is first accessed via `Get`, `List`, or `GetInformer`. To ensure all informers are started at boot time (and to register event handlers for SSE), explicitly fetch informers during initialization:

```go
func setupInformers(ctx context.Context, c cache.Cache, bus *EventBus) error {
    types := []client.Object{
        &platformv1alpha1.ForegejoTenant{},
        &cozystackv1alpha1.Tenant{},
        &edpv1.KeycloakClient{},
        &edpv1.KeycloakRealmUser{},
        &sourcev1.GitRepository{},
        &kustomizev1.Kustomization{},
    }

    for _, obj := range types {
        informer, err := c.GetInformer(ctx, obj)
        if err != nil {
            return fmt.Errorf("failed to get informer for %T: %w", obj, err)
        }

        handler := &ResourceEventHandler{
            bus:          bus,
            resourceKind: reflect.TypeOf(obj).Elem().Name(),
        }
        if _, err := informer.AddEventHandler(handler); err != nil {
            return fmt.Errorf("failed to add event handler for %T: %w", obj, err)
        }
    }

    return nil
}
```

### Scaling Considerations

| Tenants | Strategy | Notes |
|---------|----------|-------|
| 1-50 | Label-selector cache (Option C) | Simple, low overhead |
| 50-200 | Label-selector cache + `TransformStripManagedFields` + monitor cache size | May need to tune `SyncPeriod` |
| 200+ | Evaluate namespace-scoped caches (Option B) or sharded BFF instances | Dynamic cache lifecycle adds complexity; consider whether the portal even needs real-time for all tenants simultaneously |

---

## 3. CRD Operations (CRUD)

All examples use `ForegejoTenant` as the representative CRD. The same patterns apply to all CRD types.

### Type Definitions

```go
// api/v1alpha1/foregejotenant_types.go
package v1alpha1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// ForegejoTenantSpec defines the desired state
type ForegejoTenantSpec struct {
    OrganizationName string `json:"organizationName"`
    AdminEmail       string `json:"adminEmail"`
    RepoTemplate     string `json:"repoTemplate,omitempty"`
}

// ForegejoTenantStatus defines the observed state
type ForegejoTenantStatus struct {
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    OrgURL     string             `json:"orgURL,omitempty"`
    RepoURL    string             `json:"repoURL,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type ForegejoTenant struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ForegejoTenantSpec   `json:"spec,omitempty"`
    Status ForegejoTenantStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type ForegejoTenantList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []ForegejoTenant `json:"items"`
}
```

### Service Layer

Use an interface-based service layer (consistent with hetzner-operators) to enable mocking in tests.

```go
// service/foregejo.go
package service

import (
    "context"

    "sigs.k8s.io/controller-runtime/pkg/client"
    "k8s.io/apimachinery/pkg/types"

    platformv1alpha1 "github.com/tedatech/platform-v1-portal/api/v1alpha1"
)

//go:generate mockgen -destination=mocks/foregejo_mock.go -package=mocks . ForegejoService

type ForegejoService interface {
    List(ctx context.Context, namespace string) ([]platformv1alpha1.ForegejoTenant, error)
    Get(ctx context.Context, namespace, name string) (*platformv1alpha1.ForegejoTenant, error)
    Create(ctx context.Context, tenant *platformv1alpha1.ForegejoTenant) error
    Update(ctx context.Context, tenant *platformv1alpha1.ForegejoTenant) error
    Delete(ctx context.Context, namespace, name string) error
}
```

### List

```go
func (s *foregejoService) List(ctx context.Context, namespace string) ([]platformv1alpha1.ForegejoTenant, error) {
    var list platformv1alpha1.ForegejoTenantList
    if err := s.client.List(ctx, &list, client.InNamespace(namespace)); err != nil {
        return nil, fmt.Errorf("listing ForegejoTenants in %s: %w", namespace, err)
    }
    return list.Items, nil
}
```

### Get

```go
func (s *foregejoService) Get(ctx context.Context, namespace, name string) (*platformv1alpha1.ForegejoTenant, error) {
    var tenant platformv1alpha1.ForegejoTenant
    key := types.NamespacedName{Namespace: namespace, Name: name}
    if err := s.client.Get(ctx, key, &tenant); err != nil {
        return nil, fmt.Errorf("getting ForegejoTenant %s/%s: %w", namespace, name, err)
    }
    return &tenant, nil
}
```

### Create

```go
func (s *foregejoService) Create(ctx context.Context, tenant *platformv1alpha1.ForegejoTenant) error {
    // Apply portal-managed label for cache filtering
    if tenant.Labels == nil {
        tenant.Labels = make(map[string]string)
    }
    tenant.Labels["tedatech.app/portal-managed"] = "true"

    if err := s.client.Create(ctx, tenant); err != nil {
        return fmt.Errorf("creating ForegejoTenant %s/%s: %w", tenant.Namespace, tenant.Name, err)
    }
    return nil
}
```

### Update (Patch)

Prefer `client.MergeFrom` patch over full `Update` to avoid conflicts when the operator has concurrently modified the status subresource.

```go
func (s *foregejoService) Update(ctx context.Context, tenant *platformv1alpha1.ForegejoTenant) error {
    // Fetch the current version
    var current platformv1alpha1.ForegejoTenant
    key := types.NamespacedName{Namespace: tenant.Namespace, Name: tenant.Name}
    if err := s.client.Get(ctx, key, &current); err != nil {
        return fmt.Errorf("fetching current ForegejoTenant %s/%s: %w", tenant.Namespace, tenant.Name, err)
    }

    // Apply spec changes via merge patch
    patch := client.MergeFrom(current.DeepCopy())
    current.Spec = tenant.Spec

    if err := s.client.Patch(ctx, &current, patch); err != nil {
        return fmt.Errorf("patching ForegejoTenant %s/%s: %w", tenant.Namespace, tenant.Name, err)
    }
    return nil
}
```

### Delete

```go
func (s *foregejoService) Delete(ctx context.Context, namespace, name string) error {
    tenant := &platformv1alpha1.ForegejoTenant{}
    tenant.Namespace = namespace
    tenant.Name = name

    if err := s.client.Delete(ctx, tenant); err != nil {
        return fmt.Errorf("deleting ForegejoTenant %s/%s: %w", namespace, name, err)
    }
    return nil
}
```

### Error Handling: K8s Errors to HTTP Status Codes

Centralized error mapping in the HTTP handler layer:

```go
package handler

import (
    "net/http"

    apierrors "k8s.io/apimachinery/pkg/api/errors"
)

// MapK8sError converts a Kubernetes API error to an HTTP status code and message.
func MapK8sError(err error) (int, string) {
    if err == nil {
        return http.StatusOK, ""
    }

    switch {
    case apierrors.IsNotFound(err):
        return http.StatusNotFound, "Resource not found"
    case apierrors.IsAlreadyExists(err):
        return http.StatusConflict, "Resource already exists"
    case apierrors.IsConflict(err):
        return http.StatusConflict, "Resource version conflict; retry with latest version"
    case apierrors.IsForbidden(err):
        return http.StatusForbidden, "Access denied"
    case apierrors.IsUnauthorized(err):
        return http.StatusUnauthorized, "Unauthorized"
    case apierrors.IsInvalid(err):
        return http.StatusUnprocessableEntity, "Validation failed: " + err.Error()
    case apierrors.IsServerTimeout(err), apierrors.IsTimeout(err):
        return http.StatusGatewayTimeout, "Kubernetes API server timeout"
    case apierrors.IsServiceUnavailable(err):
        return http.StatusServiceUnavailable, "Kubernetes API server unavailable"
    case apierrors.IsTooManyRequests(err):
        return http.StatusTooManyRequests, "Rate limited by Kubernetes API server"
    default:
        return http.StatusInternalServerError, "Internal error"
    }
}
```

---

## 4. Real-Time Event Pipeline

### Architecture

```
K8s API Server
      |
      v (watch events)
  Informer Cache
      |
      v (AddEventHandler callbacks)
  ResourceEventHandler
      |
      v (publish)
  EventBus (in-process, Go channels)
      |
      v (per-tenant fan-out)
  SSE Writer (per-connection goroutine)
      |
      v (HTTP chunked response)
  Frontend (EventSource API)
```

### Event Types

```go
package events

import "time"

// EventType represents the kind of resource change.
type EventType string

const (
    EventResourceCreated EventType = "resource.created"
    EventResourceUpdated EventType = "resource.updated"
    EventResourceDeleted EventType = "resource.deleted"
    EventStatusChanged   EventType = "status.changed"
)

// Event represents a change to a Kubernetes resource visible to the portal.
type Event struct {
    Type         EventType         `json:"type"`
    ResourceKind string            `json:"resourceKind"` // e.g. "ForegejoTenant"
    Namespace    string            `json:"namespace"`
    Name         string            `json:"name"`
    Status       *ResourceStatus   `json:"status,omitempty"`
    OldStatus    *ResourceStatus   `json:"oldStatus,omitempty"`
    Timestamp    time.Time         `json:"timestamp"`
    Generation   int64             `json:"generation,omitempty"`
    ResourceVersion string         `json:"resourceVersion,omitempty"`
}

// ResourceStatus is a portal-friendly representation of K8s conditions.
type ResourceStatus struct {
    Phase   string `json:"phase"`   // Pending, Provisioning, Active, Failed, Deleting
    Reason  string `json:"reason"`  // K8s condition reason
    Message string `json:"message"` // Human-readable message
}
```

### ResourceEventHandler

The handler receives raw informer callbacks and translates them into portal events, filtering by namespace prefix to ensure only tenant resources are published.

```go
package events

import (
    "strings"
    "time"

    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/tools/cache"
)

type ResourceEventHandler struct {
    bus          *EventBus
    resourceKind string
}

var _ cache.ResourceEventHandler = (*ResourceEventHandler)(nil)

func (h *ResourceEventHandler) OnAdd(obj interface{}, isInInitialList bool) {
    metaObj, ok := obj.(metav1.Object)
    if !ok || !isTenantNamespace(metaObj.GetNamespace()) {
        return
    }

    // During initial list sync (informer startup), do not emit creation events.
    // These are pre-existing objects, not new creations.
    if isInInitialList {
        return
    }

    h.bus.Publish(Event{
        Type:            EventResourceCreated,
        ResourceKind:    h.resourceKind,
        Namespace:       metaObj.GetNamespace(),
        Name:            metaObj.GetName(),
        Status:          extractStatus(obj),
        Timestamp:       time.Now(),
        Generation:      metaObj.GetGeneration(),
        ResourceVersion: metaObj.GetResourceVersion(),
    })
}

func (h *ResourceEventHandler) OnUpdate(oldObj, newObj interface{}) {
    metaObj, ok := newObj.(metav1.Object)
    if !ok || !isTenantNamespace(metaObj.GetNamespace()) {
        return
    }

    oldStatus := extractStatus(oldObj)
    newStatus := extractStatus(newObj)

    eventType := EventResourceUpdated
    if oldStatus != nil && newStatus != nil && oldStatus.Phase != newStatus.Phase {
        eventType = EventStatusChanged
    }

    h.bus.Publish(Event{
        Type:            eventType,
        ResourceKind:    h.resourceKind,
        Namespace:       metaObj.GetNamespace(),
        Name:            metaObj.GetName(),
        Status:          newStatus,
        OldStatus:       oldStatus,
        Timestamp:       time.Now(),
        Generation:      metaObj.GetGeneration(),
        ResourceVersion: metaObj.GetResourceVersion(),
    })
}

func (h *ResourceEventHandler) OnDelete(obj interface{}) {
    // Handle DeletedFinalStateUnknown (tombstone) objects
    if d, ok := obj.(cache.DeletedFinalStateUnknown); ok {
        obj = d.Obj
    }

    metaObj, ok := obj.(metav1.Object)
    if !ok || !isTenantNamespace(metaObj.GetNamespace()) {
        return
    }

    h.bus.Publish(Event{
        Type:         EventResourceDeleted,
        ResourceKind: h.resourceKind,
        Namespace:    metaObj.GetNamespace(),
        Name:         metaObj.GetName(),
        Timestamp:    time.Now(),
    })
}

func isTenantNamespace(ns string) bool {
    return strings.HasPrefix(ns, "tenant-")
}
```

### EventBus

In-process pub/sub with per-tenant fan-out. Uses Go channels and `sync.RWMutex` for thread safety.

```go
package events

import (
    "sync"
)

const defaultChannelBuffer = 64

// EventBus manages per-tenant event subscriptions.
type EventBus struct {
    mu          sync.RWMutex
    subscribers map[string]map[string]chan Event // namespace -> subscriberID -> channel
}

func NewEventBus() *EventBus {
    return &EventBus{
        subscribers: make(map[string]map[string]chan Event),
    }
}

// Subscribe creates a new subscription for events in the given namespace.
// Returns the event channel and a cleanup function.
func (b *EventBus) Subscribe(namespace, subscriberID string) (<-chan Event, func()) {
    ch := make(chan Event, defaultChannelBuffer)

    b.mu.Lock()
    if b.subscribers[namespace] == nil {
        b.subscribers[namespace] = make(map[string]chan Event)
    }
    b.subscribers[namespace][subscriberID] = ch
    b.mu.Unlock()

    cleanup := func() {
        b.mu.Lock()
        delete(b.subscribers[namespace], subscriberID)
        if len(b.subscribers[namespace]) == 0 {
            delete(b.subscribers, namespace)
        }
        b.mu.Unlock()
        close(ch)
    }

    return ch, cleanup
}

// Publish sends an event to all subscribers of the event's namespace.
// Non-blocking: if a subscriber's channel is full, the event is dropped
// (the subscriber is too slow and will receive subsequent events).
func (b *EventBus) Publish(event Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()

    subs, ok := b.subscribers[event.Namespace]
    if !ok {
        return
    }

    for _, ch := range subs {
        select {
        case ch <- event:
        default:
            // Channel full; drop event to avoid blocking the informer goroutine.
            // The subscriber will see the next event and can reconcile state
            // by re-fetching the resource.
        }
    }
}
```

### SSE Handler

```go
package handler

import (
    "encoding/json"
    "fmt"
    "net/http"

    "github.com/google/uuid"
    "github.com/tedatech/platform-v1-portal/internal/events"
)

type SSEHandler struct {
    bus *events.EventBus
}

func (h *SSEHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // Extract tenant namespace from authenticated context
    namespace := tenantNamespaceFromContext(r.Context())
    if namespace == "" {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }

    // Optional filters from query params
    resourceKind := r.URL.Query().Get("resourceKind")
    resourceName := r.URL.Query().Get("name")

    // Set SSE headers
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no") // Disable nginx buffering

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming not supported", http.StatusInternalServerError)
        return
    }

    // Subscribe to tenant events
    subscriberID := uuid.New().String()
    eventCh, cleanup := h.bus.Subscribe(namespace, subscriberID)
    defer cleanup()

    // Send initial connection event
    fmt.Fprintf(w, "event: connected\ndata: {\"subscriberId\":%q}\n\n", subscriberID)
    flusher.Flush()

    // Stream events until client disconnects
    for {
        select {
        case <-r.Context().Done():
            return
        case event, ok := <-eventCh:
            if !ok {
                return
            }

            // Apply optional filters
            if resourceKind != "" && event.ResourceKind != resourceKind {
                continue
            }
            if resourceName != "" && event.Name != resourceName {
                continue
            }

            data, err := json.Marshal(event)
            if err != nil {
                continue
            }

            fmt.Fprintf(w, "event: %s\ndata: %s\n\n", event.Type, data)
            flusher.Flush()
        }
    }
}
```

---

## 5. Provisioning Status Pattern

### K8s Status Conditions Convention

All portal-managed CRDs should follow the standard Kubernetes conditions pattern. The BFF translates these conditions into portal-friendly statuses.

```yaml
# Example: ForegejoTenant in progress
apiVersion: platform.tedatech.app/v1alpha1
kind: ForegejoTenant
metadata:
  name: acme-corp
  namespace: tenant-abc123
  labels:
    tedatech.app/portal-managed: "true"
spec:
  organizationName: acme-corp
  adminEmail: admin@acme.example.com
status:
  conditions:
  - type: Ready
    status: "False"
    reason: Provisioning
    message: "Creating Forgejo organization"
    lastTransitionTime: "2026-01-15T10:30:00Z"
  - type: OrgCreated
    status: "True"
    reason: Created
    message: "Organization acme-corp created"
    lastTransitionTime: "2026-01-15T10:30:05Z"
```

### Status Mapping

```go
package status

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/api/meta"
)

// Phase represents the portal-facing lifecycle phase.
type Phase string

const (
    PhasePending      Phase = "Pending"
    PhaseProvisioning Phase = "Provisioning"
    PhaseActive       Phase = "Active"
    PhaseFailed       Phase = "Failed"
    PhaseDeleting     Phase = "Deleting"
    PhaseUnknown      Phase = "Unknown"
)

// MapConditionsToPhase determines the portal phase from K8s conditions.
func MapConditionsToPhase(conditions []metav1.Condition, deletionTimestamp *metav1.Time) (Phase, string, string) {
    // Deletion in progress
    if deletionTimestamp != nil {
        return PhaseDeleting, "Deleting", "Resource is being removed"
    }

    // No conditions yet means the operator hasn't observed the resource
    if len(conditions) == 0 {
        return PhasePending, "Pending", "Waiting for operator to process"
    }

    readyCond := meta.FindStatusCondition(conditions, "Ready")
    if readyCond == nil {
        return PhasePending, "Pending", "Waiting for readiness check"
    }

    switch {
    case readyCond.Status == metav1.ConditionTrue:
        return PhaseActive, readyCond.Reason, readyCond.Message

    case readyCond.Reason == "Provisioning" || readyCond.Reason == "Reconciling":
        return PhaseProvisioning, readyCond.Reason, readyCond.Message

    case readyCond.Reason == "Error" || readyCond.Reason == "Failed" ||
         readyCond.Reason == "ReconciliationFailed":
        return PhaseFailed, readyCond.Reason, readyCond.Message

    default:
        return PhaseProvisioning, readyCond.Reason, readyCond.Message
    }
}
```

### Status Extraction from Informer Objects

```go
package events

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "github.com/tedatech/platform-v1-portal/internal/status"
)

// conditionAccessor is implemented by CRD types that expose status conditions.
type conditionAccessor interface {
    GetConditions() []metav1.Condition
}

func extractStatus(obj interface{}) *ResourceStatus {
    metaObj, ok := obj.(metav1.Object)
    if !ok {
        return nil
    }

    var conditions []metav1.Condition
    if ca, ok := obj.(conditionAccessor); ok {
        conditions = ca.GetConditions()
    }

    phase, reason, message := status.MapConditionsToPhase(conditions, metaObj.GetDeletionTimestamp())

    return &ResourceStatus{
        Phase:   string(phase),
        Reason:  reason,
        Message: message,
    }
}
```

---

## 6. Long-Running Operation Pattern

### Flow

```
1. POST /api/tenants                     Frontend -> BFF
2. BFF creates ForegejoTenant CR         BFF -> K8s API Server
3. Return 201 Created (with resource ID) BFF -> Frontend
4. GET /api/events?namespace=...&name=.. Frontend -> BFF (SSE subscribe)
5. Operator reconciles ForegejoTenant    Operator <-> K8s API Server
6. Status condition updates              K8s API Server -> Informer -> EventBus -> SSE
7. Frontend shows progress               SSE -> Frontend
8. Ready=True                            Final SSE event -> Frontend shows completion
```

### Create Endpoint (Non-Blocking)

```go
func (h *TenantHandler) Create(w http.ResponseWriter, r *http.Request) {
    // Parse and validate request body
    var req CreateTenantRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // Derive tenant namespace from authenticated user
    namespace := tenantNamespaceFromContext(r.Context())

    tenant := &platformv1alpha1.ForegejoTenant{
        ObjectMeta: metav1.ObjectMeta{
            Name:      req.Name,
            Namespace: namespace,
            Labels: map[string]string{
                "tedatech.app/portal-managed": "true",
            },
        },
        Spec: platformv1alpha1.ForegejoTenantSpec{
            OrganizationName: req.OrganizationName,
            AdminEmail:       req.AdminEmail,
        },
    }

    if err := h.service.Create(r.Context(), tenant); err != nil {
        statusCode, msg := MapK8sError(err)
        http.Error(w, msg, statusCode)
        return
    }

    // Return immediately with 201 and the resource details.
    // The client subscribes to SSE for status updates.
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(CreateTenantResponse{
        Name:            tenant.Name,
        Namespace:       tenant.Namespace,
        ResourceVersion: tenant.ResourceVersion,
        Status: ResourceStatusResponse{
            Phase:   "Pending",
            Message: "Resource created, waiting for operator to begin provisioning",
        },
        Links: LinksResponse{
            Self:   fmt.Sprintf("/api/tenants/%s", tenant.Name),
            Events: fmt.Sprintf("/api/events?namespace=%s&resourceKind=ForegejoTenant&name=%s", namespace, tenant.Name),
        },
    })
}
```

### Timeout Handling

The BFF does not implement server-side provisioning timeouts. Instead, the frontend uses the SSE event timestamps to determine if provisioning is stale. If no status update is received within a configurable window (default: 5 minutes), the frontend shows a warning.

For cases where the operator is genuinely stuck, the BFF exposes a retry endpoint:

```go
// POST /api/tenants/{name}/retry
func (h *TenantHandler) Retry(w http.ResponseWriter, r *http.Request) {
    namespace := tenantNamespaceFromContext(r.Context())
    name := chi.URLParam(r, "name")

    // Fetch current resource
    tenant, err := h.service.Get(r.Context(), namespace, name)
    if err != nil {
        statusCode, msg := MapK8sError(err)
        http.Error(w, msg, statusCode)
        return
    }

    // Patch an annotation to trigger re-reconciliation.
    // Most operators watch for annotation changes and will re-enter the reconcile loop.
    patch := client.MergeFrom(tenant.DeepCopy())
    if tenant.Annotations == nil {
        tenant.Annotations = make(map[string]string)
    }
    tenant.Annotations["tedatech.app/retry-requested"] = time.Now().Format(time.RFC3339)

    if err := h.client.Patch(r.Context(), tenant, patch); err != nil {
        statusCode, msg := MapK8sError(err)
        http.Error(w, msg, statusCode)
        return
    }

    w.WriteHeader(http.StatusAccepted)
    json.NewEncoder(w).Encode(map[string]string{
        "message": "Retry requested; watch the events stream for status updates",
    })
}
```

---

## 7. Custom Portal CRDs vs Direct CRD Manipulation

### Options

| Option | Description | MVP Suitable | Complexity |
|--------|-------------|--------------|------------|
| **A: Direct manipulation** | BFF creates ForegejoTenant, Cozystack Tenant, Keycloak CRDs directly | Yes | Low |
| **B: Custom `PortalTenant` CRD** | Single CRD abstracts all provisioning; a portal operator handles orchestration | No (requires operator) | High |
| **C: Hybrid bundle** | BFF creates multiple CRDs in sequence, tracks composite status | Maybe | Medium |

### Recommendation: Option A for MVP, Migrate to Option B When Needed

For the MVP, the BFF directly creates individual CRDs. The orchestration logic (which CRDs to create, in what order, with what parameters) lives in the BFF's service layer.

#### MVP: Direct Manipulation

```go
// service/provisioning.go
func (s *provisioningService) ProvisionTenant(ctx context.Context, req ProvisionRequest) error {
    ns := fmt.Sprintf("tenant-%s", req.TenantID)

    // 1. Create namespace (if not exists)
    if err := s.ensureNamespace(ctx, ns); err != nil {
        return fmt.Errorf("ensuring namespace %s: %w", ns, err)
    }

    // 2. Create Cozystack Tenant (resource quotas, ingress)
    if err := s.createCozystackTenant(ctx, ns, req); err != nil {
        return fmt.Errorf("creating Cozystack tenant: %w", err)
    }

    // 3. Create Keycloak resources (auth)
    if err := s.createKeycloakResources(ctx, ns, req); err != nil {
        return fmt.Errorf("creating Keycloak resources: %w", err)
    }

    // 4. Create ForegejoTenant (source code hosting)
    if err := s.createForegejoTenant(ctx, ns, req); err != nil {
        return fmt.Errorf("creating ForegejoTenant: %w", err)
    }

    return nil
}
```

#### When to Migrate to Option B

Introduce a custom `PortalTenant` CRD and operator when:

- **3+ CRDs need coordinated rollback** -- if creating Keycloak resources fails, the BFF must clean up the already-created Cozystack Tenant. This logic gets complex in the BFF.
- **Multiple consumers need to create tenants** -- if a CLI tool, internal admin panel, or GitOps pipeline also provisions tenants, the orchestration logic must not be duplicated.
- **Provisioning requires background retries** -- an operator with a reconcile loop is naturally suited for retrying failed steps, while the BFF is request-scoped.
- **Cross-cluster provisioning** -- if tenants span multiple clusters, an operator can coordinate across clusters more cleanly than an HTTP handler.

#### Option B Design Sketch (Future)

```yaml
apiVersion: platform.tedatech.app/v1alpha1
kind: PortalTenant
metadata:
  name: acme-corp
  namespace: portal-system
spec:
  tenantID: abc123
  displayName: "Acme Corp"
  adminEmail: admin@acme.example.com
  features:
    forgejo: true
    cicd: true
  resourceQuota:
    cpu: "4"
    memory: "8Gi"
status:
  phase: Provisioning
  conditions:
  - type: NamespaceReady
    status: "True"
  - type: CozystackTenantReady
    status: "True"
  - type: KeycloakReady
    status: "False"
    reason: Provisioning
  - type: ForegejoReady
    status: "False"
    reason: Pending
  - type: Ready
    status: "False"
    reason: Provisioning
    message: "3 of 4 components provisioned"
```

---

## 8. Namespace Strategy and RBAC

### Namespace Convention

```
tenant-{tenant_id}
```

Where `tenant_id` is a lowercase alphanumeric identifier (e.g., `tenant-abc123`). The ID is generated by the portal, not by K8s. It must match `^[a-z0-9]{1,48}$` to stay within the 63-character namespace name limit after the `tenant-` prefix.

### BFF ServiceAccount

The BFF runs in the `portal-system` namespace with a dedicated ServiceAccount.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: portal-bff
  namespace: portal-system
```

### RBAC Strategy: ClusterRole with ClusterRoleBinding

The BFF needs access to CRDs across all tenant namespaces. A ClusterRole with ClusterRoleBinding is the simplest approach. The BFF enforces tenant isolation at the application level (every request is scoped to the authenticated user's namespace).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portal-bff
rules:
  # Namespace management
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "create", "watch"]

  # ForegejoTenant CRDs
  - apiGroups: ["platform.tedatech.app"]
    resources: ["foregejotenant", "foregejotenant/status"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Cozystack Tenant
  - apiGroups: ["apps.cozystack.io"]
    resources: ["tenants", "tenants/status"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # EDP Keycloak CRDs
  - apiGroups: ["v1.edp.epam.com"]
    resources: ["keycloakclients", "keycloakrealmusers", "keycloakrealmgroups"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["v1.edp.epam.com"]
    resources: ["keycloakclients/status", "keycloakrealmusers/status", "keycloakrealmgroups/status"]
    verbs: ["get", "list", "watch"]

  # Flux CRDs (read-only: BFF monitors but does not create these)
  - apiGroups: ["source.toolkit.fluxcd.io"]
    resources: ["gitrepositories", "gitrepositories/status"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["kustomize.toolkit.fluxcd.io"]
    resources: ["kustomizations", "kustomizations/status"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portal-bff
subjects:
  - kind: ServiceAccount
    name: portal-bff
    namespace: portal-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: portal-bff
```

### Why Not Per-Namespace RoleBindings?

Per-namespace RoleBindings (one per tenant namespace) would provide more restrictive RBAC, but they create a chicken-and-egg problem: the BFF needs to create the namespace and the RoleBinding, but it cannot access the namespace until the RoleBinding exists. Solutions exist (a bootstrapper job, or a ClusterRole for namespace creation only) but add unnecessary complexity for the MVP.

**Migrate to per-namespace RoleBindings when:**
- The BFF is no longer the only consumer of tenant namespaces
- Compliance requires proving that the BFF cannot access namespaces it has no business accessing
- A dedicated namespace provisioning operator exists to create RoleBindings

### Application-Level Tenant Scoping

Every HTTP request to the BFF is authenticated and associated with a tenant. The BFF middleware extracts the tenant ID from the JWT and injects the namespace into the request context. All service-layer operations use this namespace, never a user-supplied namespace.

```go
func tenantMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        claims := claimsFromContext(r.Context())
        if claims == nil || claims.TenantID == "" {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        namespace := fmt.Sprintf("tenant-%s", claims.TenantID)
        ctx := context.WithValue(r.Context(), tenantNamespaceKey, namespace)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

---

## 9. Resource Versioning and Optimistic Concurrency

### How K8s Optimistic Concurrency Works

Every K8s resource has a `metadata.resourceVersion` field, which is an opaque string backed by etcd's `mod_revision`. When updating a resource, the API server compares the `resourceVersion` in the request with the current value in etcd. If they differ, the update is rejected with a `409 Conflict`.

### Mapping to REST: ETag / If-Match

The BFF exposes `resourceVersion` as an HTTP `ETag` header, following standard HTTP semantics:

```
GET /api/tenants/acme-corp
→ 200 OK
→ ETag: "12345"
→ { "name": "acme-corp", "spec": {...}, "resourceVersion": "12345" }

PUT /api/tenants/acme-corp
← If-Match: "12345"
← { "spec": {...} }
→ 200 OK (if resourceVersion matches)
→ 409 Conflict (if resourceVersion has changed)
```

### Implementation

```go
func (h *TenantHandler) Get(w http.ResponseWriter, r *http.Request) {
    namespace := tenantNamespaceFromContext(r.Context())
    name := chi.URLParam(r, "name")

    tenant, err := h.service.Get(r.Context(), namespace, name)
    if err != nil {
        statusCode, msg := MapK8sError(err)
        http.Error(w, msg, statusCode)
        return
    }

    // Set ETag from resourceVersion
    w.Header().Set("ETag", fmt.Sprintf("%q", tenant.ResourceVersion))
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(toTenantResponse(tenant))
}

func (h *TenantHandler) Update(w http.ResponseWriter, r *http.Request) {
    namespace := tenantNamespaceFromContext(r.Context())
    name := chi.URLParam(r, "name")

    // Require If-Match header for optimistic concurrency
    ifMatch := r.Header.Get("If-Match")
    if ifMatch == "" {
        http.Error(w, "If-Match header required for updates", http.StatusPreconditionRequired)
        return
    }
    // Remove quotes from ETag value
    expectedVersion := strings.Trim(ifMatch, "\"")

    var req UpdateTenantRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }

    // Fetch current resource
    tenant, err := h.service.Get(r.Context(), namespace, name)
    if err != nil {
        statusCode, msg := MapK8sError(err)
        http.Error(w, msg, statusCode)
        return
    }

    // Verify the client's version matches the current version
    if tenant.ResourceVersion != expectedVersion {
        http.Error(w, "Resource has been modified; fetch the latest version and retry",
            http.StatusPreconditionFailed)
        return
    }

    // Apply updates via patch
    patch := client.MergeFrom(tenant.DeepCopy())
    tenant.Spec = applyUpdateRequest(tenant.Spec, req)

    if err := h.client.Patch(r.Context(), tenant, patch); err != nil {
        statusCode, msg := MapK8sError(err)
        http.Error(w, msg, statusCode)
        return
    }

    w.Header().Set("ETag", fmt.Sprintf("%q", tenant.ResourceVersion))
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(toTenantResponse(tenant))
}
```

### Server-Side Retry for Internal Operations

For BFF-internal operations (not triggered by a user request with an ETag), use a retry loop:

```go
package k8sutil

import (
    "context"
    "fmt"

    apierrors "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/types"
    "sigs.k8s.io/controller-runtime/pkg/client"
)

const maxRetries = 3

// RetryOnConflict reads the resource, applies the mutate function, and patches.
// On conflict, it re-reads and retries up to maxRetries times.
func RetryOnConflict[T client.Object](
    ctx context.Context,
    c client.Client,
    key types.NamespacedName,
    obj T,
    mutate func(T),
) error {
    for attempt := 0; attempt < maxRetries; attempt++ {
        if err := c.Get(ctx, key, obj); err != nil {
            return fmt.Errorf("get for retry: %w", err)
        }

        patch := client.MergeFrom(obj.DeepCopy().(client.Object))
        mutate(obj)

        err := c.Patch(ctx, obj, patch)
        if err == nil {
            return nil
        }
        if !apierrors.IsConflict(err) {
            return fmt.Errorf("patch: %w", err)
        }
        // Conflict: retry with fresh resource
    }
    return fmt.Errorf("conflict after %d retries for %s", maxRetries, key)
}
```

---

## 10. Testing Strategy

### Layers

| Layer | Tool | What It Tests |
|-------|------|---------------|
| Unit tests | `fake.NewClientBuilder()` | Service logic, error mapping, status mapping |
| Integration tests | `envtest.Environment` | Real K8s API server behavior, CRD validation, RBAC |
| Event pipeline tests | In-process EventBus | Subscribe/publish/unsubscribe, fan-out, backpressure |
| SSE handler tests | `httptest.Server` + SSE client | End-to-end event streaming |

### Unit Tests with Fake Client

The `controller-runtime/pkg/client/fake` package provides an in-memory client that implements `client.Client`. It supports scheme-aware typed access and basic CRUD operations.

```go
package service_test

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "sigs.k8s.io/controller-runtime/pkg/client/fake"

    platformv1alpha1 "github.com/tedatech/platform-v1-portal/api/v1alpha1"
    "github.com/tedatech/platform-v1-portal/internal/service"
)

func TestForegejoService_Get(t *testing.T) {
    scheme := runtime.NewScheme()
    require.NoError(t, platformv1alpha1.AddToScheme(scheme))

    existing := &platformv1alpha1.ForegejoTenant{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "acme",
            Namespace: "tenant-abc123",
        },
        Spec: platformv1alpha1.ForegejoTenantSpec{
            OrganizationName: "acme-corp",
            AdminEmail:       "admin@acme.example.com",
        },
    }

    fakeClient := fake.NewClientBuilder().
        WithScheme(scheme).
        WithObjects(existing).
        WithStatusSubresource(&platformv1alpha1.ForegejoTenant{}).
        Build()

    svc := service.NewForegejoService(fakeClient)

    t.Run("found", func(t *testing.T) {
        tenant, err := svc.Get(context.Background(), "tenant-abc123", "acme")
        require.NoError(t, err)
        assert.Equal(t, "acme-corp", tenant.Spec.OrganizationName)
    })

    t.Run("not found", func(t *testing.T) {
        _, err := svc.Get(context.Background(), "tenant-abc123", "nonexistent")
        require.Error(t, err)
        // Verify the service returns a K8s NotFound error that can be mapped to HTTP 404
        assert.True(t, apierrors.IsNotFound(err) || strings.Contains(err.Error(), "not found"))
    })
}
```

### Unit Tests with MockGen

For testing HTTP handlers in isolation, mock the service interface:

```go
package handler_test

import (
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/stretchr/testify/assert"
    "go.uber.org/mock/gomock"

    "github.com/tedatech/platform-v1-portal/internal/handler"
    "github.com/tedatech/platform-v1-portal/internal/service/mocks"
)

func TestTenantHandler_Get_NotFound(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockSvc := mocks.NewMockForegejoService(ctrl)
    mockSvc.EXPECT().
        Get(gomock.Any(), "tenant-abc123", "nonexistent").
        Return(nil, apierrors.NewNotFound(schema.GroupResource{}, "nonexistent"))

    h := handler.NewTenantHandler(mockSvc)

    req := httptest.NewRequest("GET", "/api/tenants/nonexistent", nil)
    req = req.WithContext(withTenantNamespace(req.Context(), "tenant-abc123"))
    rec := httptest.NewRecorder()

    h.Get(rec, req)

    assert.Equal(t, http.StatusNotFound, rec.Code)
}
```

### Integration Tests with envtest

`envtest` starts a real API server (etcd + kube-apiserver) without kubelet or controller-manager. CRD schemas are loaded from YAML files, enabling real validation and RBAC testing.

```go
package integration_test

import (
    "context"
    "path/filepath"
    "testing"

    "github.com/stretchr/testify/require"
    "k8s.io/client-go/rest"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"

    platformv1alpha1 "github.com/tedatech/platform-v1-portal/api/v1alpha1"
)

var (
    testEnv   *envtest.Environment
    k8sClient client.Client
    cfg       *rest.Config
)

func TestMain(m *testing.M) {
    testEnv = &envtest.Environment{
        CRDDirectoryPaths: []string{
            filepath.Join("..", "..", "config", "crd", "bases"),
            // Include external CRDs for integration testing
            filepath.Join("..", "..", "config", "crd", "external"),
        },
        ErrorIfCRDPathMissing: true,
    }

    var err error
    cfg, err = testEnv.Start()
    if err != nil {
        panic(err)
    }

    scheme := runtime.NewScheme()
    _ = platformv1alpha1.AddToScheme(scheme)
    _ = clientgoscheme.AddToScheme(scheme)

    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme})
    if err != nil {
        panic(err)
    }

    code := m.Run()
    _ = testEnv.Stop()
    os.Exit(code)
}

func TestCreateForegejoTenant_Integration(t *testing.T) {
    ctx := context.Background()
    ns := "tenant-integration-test"

    // Create namespace
    namespace := &corev1.Namespace{
        ObjectMeta: metav1.ObjectMeta{Name: ns},
    }
    require.NoError(t, k8sClient.Create(ctx, namespace))

    // Create ForegejoTenant
    tenant := &platformv1alpha1.ForegejoTenant{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "test-tenant",
            Namespace: ns,
            Labels: map[string]string{
                "tedatech.app/portal-managed": "true",
            },
        },
        Spec: platformv1alpha1.ForegejoTenantSpec{
            OrganizationName: "test-org",
            AdminEmail:       "test@example.com",
        },
    }
    require.NoError(t, k8sClient.Create(ctx, tenant))

    // Verify it was created
    var fetched platformv1alpha1.ForegejoTenant
    require.NoError(t, k8sClient.Get(ctx, client.ObjectKeyFromObject(tenant), &fetched))
    require.Equal(t, "test-org", fetched.Spec.OrganizationName)

    // Verify listing with namespace filter
    var list platformv1alpha1.ForegejoTenantList
    require.NoError(t, k8sClient.List(ctx, &list, client.InNamespace(ns)))
    require.Len(t, list.Items, 1)

    // Cleanup
    require.NoError(t, k8sClient.Delete(ctx, tenant))
    require.NoError(t, k8sClient.Delete(ctx, namespace))
}
```

### EventBus Tests

```go
package events_test

import (
    "testing"
    "time"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"

    "github.com/tedatech/platform-v1-portal/internal/events"
)

func TestEventBus_SubscribePublish(t *testing.T) {
    bus := events.NewEventBus()

    // Subscribe to a tenant namespace
    ch, cleanup := bus.Subscribe("tenant-abc123", "sub-1")
    defer cleanup()

    // Publish an event
    event := events.Event{
        Type:         events.EventResourceCreated,
        ResourceKind: "ForegejoTenant",
        Namespace:    "tenant-abc123",
        Name:         "acme",
        Timestamp:    time.Now(),
    }
    bus.Publish(event)

    // Verify receipt
    select {
    case received := <-ch:
        assert.Equal(t, events.EventResourceCreated, received.Type)
        assert.Equal(t, "acme", received.Name)
    case <-time.After(time.Second):
        t.Fatal("timed out waiting for event")
    }
}

func TestEventBus_NamespaceIsolation(t *testing.T) {
    bus := events.NewEventBus()

    ch1, cleanup1 := bus.Subscribe("tenant-aaa", "sub-1")
    defer cleanup1()
    ch2, cleanup2 := bus.Subscribe("tenant-bbb", "sub-2")
    defer cleanup2()

    // Publish to tenant-aaa only
    bus.Publish(events.Event{
        Type:      events.EventResourceCreated,
        Namespace: "tenant-aaa",
        Name:      "resource-1",
        Timestamp: time.Now(),
    })

    // tenant-aaa subscriber receives the event
    select {
    case <-ch1:
        // expected
    case <-time.After(time.Second):
        t.Fatal("tenant-aaa subscriber did not receive event")
    }

    // tenant-bbb subscriber does NOT receive it
    select {
    case <-ch2:
        t.Fatal("tenant-bbb subscriber should not receive events for tenant-aaa")
    case <-time.After(100 * time.Millisecond):
        // expected
    }
}

func TestEventBus_Cleanup(t *testing.T) {
    bus := events.NewEventBus()

    _, cleanup := bus.Subscribe("tenant-abc", "sub-1")

    // Unsubscribe
    cleanup()

    // Publishing should not panic or block
    bus.Publish(events.Event{
        Namespace: "tenant-abc",
        Timestamp: time.Now(),
    })
}
```

### Test Organization

```
internal/
  events/
    bus.go              # EventBus implementation
    bus_test.go         # EventBus unit tests
    handler.go          # ResourceEventHandler
    types.go            # Event, ResourceStatus types
  handler/
    tenant.go           # HTTP handlers
    tenant_test.go      # Handler unit tests (with mocked services)
    errors.go           # MapK8sError
    sse.go              # SSE handler
  service/
    foregejo.go         # ForegejoService interface + implementation
    foregejo_test.go    # Service unit tests (with fake client)
    mocks/
      foregejo_mock.go  # Generated mock
  status/
    mapper.go           # MapConditionsToPhase
    mapper_test.go      # Status mapping unit tests
  k8sutil/
    retry.go            # RetryOnConflict
    retry_test.go       # Retry logic tests
test/
  integration/
    suite_test.go       # envtest setup (TestMain)
    tenant_test.go      # Integration tests
config/
  crd/
    bases/              # Portal CRD YAML definitions
    external/           # External CRD YAMLs for envtest
  rbac/
    clusterrole.yaml    # portal-bff ClusterRole
```

---

## References

- [controller-runtime cache package API docs](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/cache)
- [controller-runtime cache options design](https://github.com/kubernetes-sigs/controller-runtime/blob/main/designs/cache_options.md)
- [Kubernetes Controllers at Scale: Clients, Caches, Conflicts, Patches Explained](https://medium.com/@timebertt/kubernetes-controllers-at-scale-clients-caches-conflicts-patches-explained-aa0f7a8b4332)
- [Dynamic multi-namespace cache discussion](https://github.com/kubernetes-sigs/controller-runtime/issues/1590)
- [Kubebuilder: Writing controller tests](https://book.kubebuilder.io/cronjob-tutorial/writing-tests)
- [envtest package docs](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/envtest)
- [Configuring envtest for integration tests](https://book.kubebuilder.io/reference/envtest.html)
- [Operator SDK: Operator Scope](https://sdk.operatorframework.io/docs/building-operators/golang/operator-scope/)
- [Kubernetes Optimistic Concurrency](https://kyungho.me/en/posts/kubernetes-concurrency-control)
- [Go SSE real-time applications](https://oneuptime.com/blog/post/2026-02-01-go-realtime-applications-sse/view)
- [r3labs/sse - Go SSE library](https://github.com/r3labs/sse)

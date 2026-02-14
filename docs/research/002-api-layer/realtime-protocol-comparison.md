# Real-Time Protocol Comparison: WebSocket vs SSE

**Context:** Customer Portal BFF (Go, in-cluster) relaying K8s CRD status changes to a React SPA (Refine `liveProvider`).
**Infrastructure:** Cilium Gateway API (Envoy-based, per-node proxy), HTTP/2 termination at gateway.
**Date:** 2026-02-14

---

## Table of Contents

1. [Cilium Gateway API Compatibility](#1-cilium-gateway-api-compatibility)
2. [K8s Informer Relay Pattern](#2-k8s-informer-relay-pattern)
3. [Refine liveProvider Integration](#3-refine-liveprovider-integration)
4. [Connection Management](#4-connection-management)
5. [Scalability with Multiple BFF Replicas](#5-scalability-with-multiple-bff-replicas)
6. [HTTP/2 Considerations](#6-http2-considerations)
7. [Error Handling & Recovery](#7-error-handling--recovery)
8. [Auth Header Support](#8-auth-header-support)
9. [Go Server Implementation](#9-go-server-implementation)
10. [Recommendation Table](#10-recommendation-table)
11. [Verdict](#11-verdict)

---

## 1. Cilium Gateway API Compatibility

### WebSocket

**Protocol upgrade required.** WebSocket connections begin as an HTTP/1.1 `Upgrade: websocket` request, then switch to the WebSocket protocol. This upgrade negotiation must be explicitly supported by every proxy in the chain.

**Gateway API v1.2+ support (appProtocol):** Kubernetes Gateway API v1.2 introduced backend protocol selection via the `appProtocol` field on Service ports. When a Service is annotated with `appProtocol: kubernetes.io/ws`, an HTTPRoute referencing that Service will automatically upgrade connections to WebSocket. Cilium merged appProtocol support in [issue #30452](https://github.com/cilium/cilium/issues/30452), shipping in Cilium 1.16.x. Cilium 1.19 passes Gateway API v1.4.0 Core conformance tests.

**Envoy configuration:** Under the hood, Cilium's Envoy proxy uses a `websocket_lib` filter and requires `upgrade_configs` with `upgrade_type: websocket` in the `HttpConnectionManager`. The Gateway API `appProtocol` mechanism auto-generates this configuration, so no manual CiliumEnvoyConfig is needed for basic WebSocket routing.

**Known issues (active bugs):**
- [cilium/cilium#40822](https://github.com/cilium/cilium/issues/40822) — WebSocket connections through Gateway API (HTTPRoute) intermittently fail with close code 1006 (abnormal closure). Reported on Cilium 1.17.6 with a Blazor Server app. Connections work fine via NodePort/port-forward but break through the gateway.
- [cilium/cilium#41123](https://github.com/cilium/cilium/issues/41123) — WebSocket connections always fail through Gateway API for an Appflowy Server deployment. Auth tokens are not being forwarded correctly through the WebSocket upgrade path.

These are **open bugs as of 2026-02** that demonstrate real-world instability with WebSocket upgrades through Cilium Gateway API. The fact that multiple independent applications report WebSocket failures through the gateway while working fine with direct connections is a significant concern.

**Configuration required:**
- Service must set `appProtocol: kubernetes.io/ws` on the relevant port
- HTTPRoute backendRef must reference that Service
- May need to tune idle timeout (see below)

### SSE

**No protocol upgrade needed.** SSE is standard HTTP with `Content-Type: text/event-stream`. It uses chunked transfer encoding over a long-lived HTTP connection. No special proxy configuration, annotations, or appProtocol settings are required. Any proxy that handles HTTP/1.1 or HTTP/2 natively supports SSE.

**Cilium/Envoy compatibility:** SSE traffic is indistinguishable from any other HTTP response stream. Envoy handles it through its standard HTTP connection manager. No `upgrade_configs`, no appProtocol, no special filter chain needed.

**Idle timeout concern:** Envoy's default `stream_idle_timeout` is 5 minutes ([cilium/cilium#34582](https://github.com/cilium/cilium/issues/34582)). If no data flows on the SSE connection for 5 minutes, Envoy will terminate it. This is easily mitigated:

- **Server-side:** Send SSE keepalive comments (`: keepalive\n\n`) every 15-30 seconds. This is standard practice and resets the idle timer.
- **Cilium Helm:** Set `envoy.streamIdleTimeoutDurationSeconds` to increase the timeout (though there is [a bug](https://github.com/cilium/cilium/issues/43814) where this may not propagate correctly to all generated Envoy configs).
- **Fallback:** The `--http-stream-idle-timeout` cilium-agent flag can override the default.

With keepalive comments, the idle timeout is a non-issue for SSE.

**Verdict:** SSE has **zero compatibility risk** with Cilium Gateway API. WebSocket has **known open bugs** causing connection failures.

---

## 2. K8s Informer Relay Pattern

The BFF runs Go `client-go` informers watching CRDs (e.g., `ForgejaTenant`, `AgencyTenant`). When a CRD's `.status` changes, the BFF must push that event to connected frontend clients.

### WebSocket Pattern

```
Client                        BFF                           K8s API
  |                            |                               |
  |--- WS Upgrade ----------->|                               |
  |<-- WS Connection ---------|                               |
  |                            |--- informer.Watch() -------->|
  |--- {"subscribe":          |                               |
  |     "ForgejaTenant/foo",  |                               |
  |     "ns": "tenant-123"} ->|                               |
  |                            |  [registers subscription]    |
  |                            |                               |
  |                            |<-- CRD status changed -------|
  |<-- {"type": "updated",    |                               |
  |     "resource": "...",    |                               |
  |     "payload": {...}} ----|                               |
  |                            |                               |
  |--- {"unsubscribe":        |                               |
  |     "ForgejaTenant/foo"}->|                               |
```

- **Pros:** Clean subscription management over a single connection. Client can dynamically subscribe/unsubscribe to specific resources without new HTTP requests. Natural fit for "watch this resource" semantics.
- **Cons:** Server must maintain per-connection subscription state. Client-side subscription protocol must be designed and documented. More complex message framing (JSON envelopes for subscribe/unsubscribe/event messages).

### SSE Pattern

```
Client                        BFF                           K8s API
  |                            |                               |
  |--- GET /events?           |                               |
  |    resource=ForgejaTenant |                               |
  |    &name=foo              |                               |
  |    &ns=tenant-123 ------->|                               |
  |                            |--- informer.Watch() -------->|
  |<-- 200 OK                 |                               |
  |    Content-Type:           |                               |
  |    text/event-stream       |                               |
  |                            |                               |
  |                            |<-- CRD status changed -------|
  |<-- event: updated         |                               |
  |    id: 42                  |                               |
  |    data: {"resource":...}  |                               |
  |                            |                               |
  | [to change subscription:  |                               |
  |  close stream, open new]  |                               |
```

- **Pros:** Subscription encoded in the URL (REST-like, stateless). Server logic is simpler -- filter informer events by URL params, write matching events. No custom message protocol needed. Each stream is independently reconnectable with `Last-Event-ID`.
- **Cons:** Changing subscriptions requires closing the current stream and opening a new one (or opening multiple parallel streams). Less flexible for dynamic, fine-grained subscription changes.

### Analysis

For the Customer Portal use case, the data flow is **inherently unidirectional**: K8s CRD status changes flow from server to client. The client's "subscriptions" are determined by which portal page they are viewing (e.g., viewing tenant "foo" means watching `ForgejaTenant/foo`). This maps naturally to SSE:

- Navigate to tenant detail page -> open `GET /events?resource=ForgejaTenant&name=foo&ns=tenant-123`
- Navigate away -> close the EventSource
- Navigate to a list page -> open `GET /events?resource=ForgejaTenant&ns=tenant-123` (all tenants in namespace)

Multiple SSE streams can coexist (especially over HTTP/2, see section 6). The "open stream per view" pattern is simpler than maintaining a WebSocket subscription protocol.

WebSocket's bidirectional advantage (subscribe/unsubscribe messages) is **not needed** because subscription changes are tied to navigation, which naturally opens/closes connections.

---

## 3. Refine liveProvider Integration

### liveProvider Interface

Refine's `liveProvider` is **protocol-agnostic**. The interface is:

```typescript
interface LiveProvider {
  subscribe: (params: {
    channel: string;           // e.g., "resources/forgejo-tenants"
    types: string[];           // ["created", "updated", "deleted"] or ["*"]
    params?: {
      ids?: string[];          // filter by specific resource IDs
      [key: string]: any;
    };
    callback: (event: LiveEvent) => void;
    meta?: Record<string, any>;
  }) => Subscription;

  unsubscribe: (subscription: Subscription) => void;

  publish?: (event: {
    channel: string;
    type: string;
    payload: { ids: string[] };
    date: Date;
    meta?: Record<string, any>;
  }) => void;
}
```

The callback receives `LiveEvent` objects:
```typescript
interface LiveEvent {
  channel: string;
  type: "created" | "updated" | "deleted";
  payload: { ids: string[] };
  date: Date;
}
```

When a live event is received, Refine **automatically invalidates and refetches** the relevant queries (e.g., `useList`, `useOne`). It does not apply granular patches; it triggers full refetches. This means the live event payload only needs to contain the resource type and IDs, not the full data.

### Built-in Adapters

Refine ships adapters for:
- **Ably** (`@refinedev/ably`) -- WebSocket-based pub/sub
- **Supabase** (`@refinedev/supabase`) -- PostgreSQL LISTEN/NOTIFY over WebSocket
- **Appwrite** (`@refinedev/appwrite`) -- WebSocket-based
- **Hasura** (`@refinedev/hasura`) -- GraphQL subscriptions over WebSocket

All built-in adapters use WebSocket under the hood. There is **no built-in SSE adapter**.

### Custom Implementation Effort

Since the interface is protocol-agnostic, a custom implementation for either WebSocket or SSE is straightforward:

**SSE implementation (~40 lines):**
```typescript
import { LiveProvider } from "@refinedev/core";
import { fetchEventSource } from "@microsoft/fetch-event-source";

export const sseProvider = (baseUrl: string, getToken: () => string): LiveProvider => {
  const subscriptions = new Map<string, AbortController>();

  return {
    subscribe: ({ channel, types, params, callback }) => {
      const controller = new AbortController();
      const id = crypto.randomUUID();
      const [, resource] = channel.split("/"); // "resources/forgejo-tenants" -> "forgejo-tenants"

      const url = new URL(`${baseUrl}/events`);
      url.searchParams.set("resource", resource);
      if (params?.ids) url.searchParams.set("ids", params.ids.join(","));

      fetchEventSource(url.toString(), {
        headers: { Authorization: `Bearer ${getToken()}` },
        signal: controller.signal,
        onmessage(event) {
          const data = JSON.parse(event.data);
          if (types.includes("*") || types.includes(data.type)) {
            callback({ channel, type: data.type, payload: data.payload, date: new Date() });
          }
        },
      });

      subscriptions.set(id, controller);
      return { id };
    },
    unsubscribe: ({ id }) => {
      subscriptions.get(id)?.abort();
      subscriptions.delete(id);
    },
  };
};
```

**WebSocket implementation (~60 lines):**
```typescript
import { LiveProvider } from "@refinedev/core";

export const wsProvider = (wsUrl: string, getToken: () => string): LiveProvider => {
  let ws: WebSocket | null = null;
  const callbacks = new Map<string, { channel: string; types: string[]; callback: Function }>();

  const connect = () => {
    ws = new WebSocket(`${wsUrl}?token=${getToken()}`);
    ws.onmessage = (msg) => {
      const event = JSON.parse(msg.data);
      callbacks.forEach(({ channel, types, callback }) => {
        if (event.channel === channel && (types.includes("*") || types.includes(event.type))) {
          callback(event);
        }
      });
    };
    ws.onclose = () => setTimeout(connect, 1000); // manual reconnect
  };
  connect();

  return {
    subscribe: ({ channel, types, params, callback }) => {
      const id = crypto.randomUUID();
      callbacks.set(id, { channel, types, callback });
      ws?.send(JSON.stringify({ action: "subscribe", channel, params }));
      return { id };
    },
    unsubscribe: ({ id }) => {
      const sub = callbacks.get(id);
      if (sub) ws?.send(JSON.stringify({ action: "unsubscribe", channel: sub.channel }));
      callbacks.delete(id);
    },
  };
};
```

**Effort comparison:** Both are roughly equivalent in implementation effort. The SSE version is slightly simpler because `@microsoft/fetch-event-source` handles reconnection and the EventSource protocol automatically. The WebSocket version requires manual reconnection logic and a custom subscription message protocol.

### Verdict

Refine's liveProvider is **fully protocol-agnostic**. Neither WebSocket nor SSE has a built-in advantage. A custom SSE adapter is marginally simpler to implement due to `@microsoft/fetch-event-source` handling reconnection.

---

## 4. Connection Management

### WebSocket

| Aspect | Details |
|--------|---------|
| **Heartbeat** | Requires application-level ping/pong or WebSocket-level PING frames. If the server does not send pings, proxies may close idle connections. |
| **Reconnection** | Manual. `onclose`/`onerror` handlers must implement reconnect with exponential backoff. |
| **State on reconnect** | All subscriptions are lost. Client must re-send subscribe messages after reconnecting. |
| **Protocol upgrade** | HTTP -> WS upgrade handshake. Adds complexity at every proxy layer. |
| **Connection state** | Server must track: which client watches which resources, subscription filters, authentication state. |
| **Graceful shutdown** | Server must send close frames, wait for client acknowledgment, then close. During rolling updates, in-flight connections need drain handling. |

### SSE

| Aspect | Details |
|--------|---------|
| **Keepalive** | Server sends `: keepalive\n\n` comments every 15-30 seconds. These are SSE comments (lines starting with `:`) that the browser ignores but keep the connection alive through proxies. |
| **Reconnection** | **Automatic.** The `EventSource` API (and `@microsoft/fetch-event-source`) automatically reconnects on disconnection. The `retry:` field in the event stream controls the reconnection interval. |
| **State on reconnect** | `Last-Event-ID` header is sent automatically. Server can resume from the last event, providing seamless continuity. |
| **Protocol** | Standard HTTP. No upgrade, no special proxy configuration. |
| **Connection state** | Minimal. URL params encode the subscription. Server only needs to filter informer events by the request's query params. |
| **Graceful shutdown** | Server closes the response writer. Client auto-reconnects to another pod. No special drain logic needed. |

### Comparison

SSE has **dramatically simpler** connection management:
- Auto-reconnect vs manual reconnect with backoff
- `Last-Event-ID` resume vs re-subscribe from scratch
- Standard HTTP vs protocol upgrade negotiation
- Stateless URL-based subscriptions vs stateful per-connection subscription tracking

---

## 5. Scalability with Multiple BFF Replicas

Both protocols create **long-lived connections** that are bound to a specific pod. Neither can be load-balanced per-request once established.

### Shared Characteristics

- Each BFF pod runs its own `client-go` informers, so each pod has a complete, independent view of CRD state. No shared state or message bus is needed between pods.
- When a pod restarts during a rolling update, all connections to that pod drop.
- Cilium Gateway API distributes new connections across pods (standard L7 load balancing).

### WebSocket Specifics

- Connection is sticky to one pod for its entire lifetime.
- Pod restart = connection drops + client must reconnect + client must re-subscribe.
- During rolling updates, the old pod should drain WebSocket connections gracefully (send close frame, wait).
- If the client reconnects to a different pod, subscription state is lost and must be rebuilt.

### SSE Specifics

- Connection is also sticky to one pod for its lifetime.
- Pod restart = connection drops + **automatic reconnect by EventSource** + `Last-Event-ID` enables resume.
- During rolling updates, the old pod simply stops writing to the response. EventSource reconnects transparently.
- Reconnection to a different pod is seamless if the server implements `Last-Event-ID` properly. Since each pod runs its own informers, any pod can serve events for any resource.

### Event ID Strategy for SSE

To support `Last-Event-ID` across pods, event IDs should be derived from the resource itself rather than per-pod counters:

```
id: {resource}/{namespace}/{name}@{resourceVersion}
```

Example: `ForgejaTenant/tenant-123/my-tenant@12345`

Any pod receiving a reconnection with this `Last-Event-ID` can check its informer cache for the current resource version and send any events newer than the specified version.

### Verdict

SSE handles pod restarts and rolling updates **transparently** due to automatic reconnection and `Last-Event-ID` resume. WebSocket requires explicit handling and loses subscription state.

---

## 6. HTTP/2 Considerations

### SSE over HTTP/2

SSE over HTTP/2 is **excellent**:
- Multiple SSE streams are **multiplexed** over a single TCP connection. Each stream is an HTTP/2 stream.
- The HTTP/1.1 limitation of 6 connections per origin is **eliminated** with HTTP/2. HTTP/2 negotiates a maximum of 100 concurrent streams by default.
- This means a user viewing a dashboard with 5 different resource watches only uses 5 HTTP/2 streams on 1 TCP connection, not 5 separate TCP connections.
- SSE over HTTP/2 is the most efficient configuration for multiple concurrent server-push streams.

### WebSocket over HTTP/2

WebSocket over HTTP/2 is defined in [RFC 8441](https://www.rfc-editor.org/rfc/rfc8441) using the Extended CONNECT method. However:

- **Envoy issue [#37020](https://github.com/envoyproxy/envoy/issues/37020):** When a backend supports HTTP/2 but does not support RFC 8441 (`SETTINGS_ENABLE_CONNECT_PROTOCOL=1`), Envoy does **not** gracefully fall back to HTTP/1.1 for WebSocket. The connection fails. The only workaround is to disable HTTP/2 for the entire cluster.
- **Browser support:** Browser support for WebSocket over HTTP/2 is inconsistent. Most browsers still use HTTP/1.1 for WebSocket upgrades.
- **Practical result:** WebSocket connections typically fall back to HTTP/1.1, using one TCP connection per WebSocket. This is fine for a single connection but less efficient than SSE over HTTP/2 when multiple streams are needed.

### Browser Connection Limits

| Protocol | HTTP/1.1 | HTTP/2 |
|----------|----------|--------|
| SSE | 6 connections per origin (shared across tabs) | Effectively unlimited (100 streams default) |
| WebSocket | 1 connection per WebSocket (no origin limit) | 1 connection per WebSocket (RFC 8441 rare) |

For the Customer Portal, where a user might have multiple browser tabs open, the HTTP/1.1 SSE limit of 6 connections could be problematic. However, since Cilium Gateway terminates TLS and negotiates HTTP/2, this is a non-issue. **The gateway must serve HTTP/2 to browsers** (which is the default for HTTPS via Cilium Gateway).

### Verdict

SSE over HTTP/2 is **strictly superior** to WebSocket for this use case. Multiple streams multiplex efficiently. WebSocket over HTTP/2 has known Envoy issues and poor browser adoption.

---

## 7. Error Handling & Recovery

### WebSocket

```javascript
const ws = new WebSocket(url);

ws.onclose = (event) => {
  // Manual reconnect with backoff
  if (!event.wasClean) {
    setTimeout(() => reconnect(), backoffMs);
    backoffMs = Math.min(backoffMs * 2, 30000);
  }
};

ws.onerror = (error) => {
  // Error details are intentionally limited in browsers (security)
  console.error("WebSocket error:", error);
};

// After reconnecting, must re-subscribe to all resources:
ws.onopen = () => {
  backoffMs = 1000;
  activeSubscriptions.forEach(sub => {
    ws.send(JSON.stringify({ action: "subscribe", ...sub }));
  });
};
```

**Failure modes:**
- Connection drops silently (no close frame) -- detected only by ping/pong timeout
- Server restart -- abrupt close, client must reconnect and re-subscribe
- Network change (e.g., mobile switching from WiFi to cellular) -- connection drops
- Proxy timeout (Envoy 5-minute `stream_idle_timeout`) -- connection closed if no ping/pong

### SSE

```javascript
// Using @microsoft/fetch-event-source
fetchEventSource("/events?resource=ForgejaTenant&ns=tenant-123", {
  headers: { Authorization: `Bearer ${token}` },
  onmessage(event) {
    // Process event
  },
  onclose() {
    // Auto-reconnects with retry interval from server
  },
  onerror(err) {
    // Return retry interval or throw to stop
    return 3000; // retry in 3 seconds
  },
});

// Server sends:
// retry: 3000\n
// id: ForgejaTenant/tenant-123/my-tenant@12345\n
// event: updated\n
// data: {"ids": ["my-tenant"]}\n\n
```

**Failure modes (all handled automatically):**
- Connection drops -- EventSource reconnects with `Last-Event-ID`
- Server restart -- EventSource reconnects to another pod, resumes from last event ID
- Network change -- EventSource reconnects when network is restored
- Proxy timeout -- prevented by keepalive comments; if it still occurs, EventSource reconnects

**Key advantage:** The `Last-Event-ID` mechanism means no events are missed during reconnection if the server implements resume correctly. With K8s informers maintaining a local cache with resource versions, the BFF can always serve events from the last known state.

### Verdict

SSE's built-in reconnection and `Last-Event-ID` resume mechanism is **fundamentally more resilient** than WebSocket's manual reconnection. For a portal showing live provisioning status, SSE provides reliable event delivery with minimal client-side complexity.

---

## 8. Auth Header Support

### Native EventSource API

The browser's native `EventSource` API does **not** support custom headers. It can only set:
- URL (including query parameters)
- `withCredentials` (for cookies)

This means you **cannot** send `Authorization: Bearer <token>` with native `EventSource`.

### Solutions for SSE Auth

1. **`@microsoft/fetch-event-source`** (recommended): Drop-in replacement that uses the Fetch API under the hood. Supports all fetch options including custom headers, POST method, request body, and `AbortController`.

   ```typescript
   import { fetchEventSource } from "@microsoft/fetch-event-source";

   fetchEventSource("/events", {
     method: "GET",
     headers: {
       Authorization: `Bearer ${accessToken}`,
       Accept: "text/event-stream",
     },
     signal: abortController.signal,
     onmessage(event) { /* ... */ },
   });
   ```

2. **Cookie-based auth**: If the BFF issues HTTP-only cookies, native `EventSource` works with `withCredentials: true`. However, this conflicts with token-based auth patterns and complicates CORS.

3. **Query parameter token**: `EventSource("/events?token=xxx")`. **Not recommended** -- tokens in URLs end up in server logs, proxy logs, browser history, and Referer headers.

### WebSocket Auth

WebSocket auth options during the handshake:

1. **Query parameter**: `new WebSocket("wss://host/ws?token=xxx")`. Same log-exposure risk as SSE query params.
2. **Sec-WebSocket-Protocol header**: Smuggle the token as a "subprotocol". Non-standard, may be logged.
3. **Cookie-based**: Works if using HTTP-only cookies.
4. **Post-connect auth message**: Open the WebSocket, then send `{"type": "auth", "token": "xxx"}` as the first message. Requires server-side logic to hold the connection unauthenticated until the auth message arrives, with a timeout to prevent resource exhaustion.

Neither protocol has a clean, native solution for Bearer token auth. Both require workarounds. The `@microsoft/fetch-event-source` library provides the cleanest solution for SSE, making auth headers as simple as a standard fetch request.

### Verdict

**Tie**, but SSE with `@microsoft/fetch-event-source` is slightly cleaner because auth headers work exactly like any other HTTP request. WebSocket requires either query params (insecure), protocol header smuggling (non-standard), or post-connect auth messages (complex).

---

## 9. Go Server Implementation

### SSE Server (Manual Implementation)

```go
func (s *Server) HandleSSE(w http.ResponseWriter, r *http.Request) {
    // Validate auth
    claims, err := s.validateToken(r.Header.Get("Authorization"))
    if err != nil {
        http.Error(w, "Unauthorized", http.StatusUnauthorized)
        return
    }

    // Parse subscription from URL params
    resource := r.URL.Query().Get("resource")
    namespace := r.URL.Query().Get("namespace")
    name := r.URL.Query().Get("name")
    lastEventID := r.Header.Get("Last-Event-ID")

    // Set SSE headers
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("X-Accel-Buffering", "no")

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "Streaming not supported", http.StatusInternalServerError)
        return
    }

    // Create event channel
    events := make(chan Event, 64)
    sub := s.informerHub.Subscribe(resource, namespace, name, lastEventID, events)
    defer s.informerHub.Unsubscribe(sub)

    // Keepalive ticker
    keepalive := time.NewTicker(15 * time.Second)
    defer keepalive.Stop()

    for {
        select {
        case <-r.Context().Done():
            return
        case event := <-events:
            fmt.Fprintf(w, "id: %s\nevent: %s\ndata: %s\n\n",
                event.ID, event.Type, event.JSON())
            flusher.Flush()
        case <-keepalive.C:
            fmt.Fprintf(w, ": keepalive\n\n")
            flusher.Flush()
        }
    }
}
```

**Complexity:** ~50 lines. Uses standard `net/http`. No external dependencies required. The `http.Flusher` interface is the only non-obvious part.

**Libraries:**
- **`r3labs/sse`** -- Full-featured SSE server library with stream management, event broadcasting, and client tracking. Adds abstraction but may be overkill for a simple informer relay.
- **Manual** -- Recommended for this use case. SSE is simple enough that a library adds more complexity than it saves.

### WebSocket Server

```go
import "github.com/coder/websocket" // formerly nhooyr.io/websocket

func (s *Server) HandleWebSocket(w http.ResponseWriter, r *http.Request) {
    conn, err := websocket.Accept(w, r, &websocket.AcceptOptions{
        InsecureSkipVerify: false,
    })
    if err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    defer conn.CloseNow()

    // Auth: read first message or check query param
    // ...

    ctx := r.Context()
    subscriptions := make(map[string]func()) // resource -> cancel

    // Read loop: handle subscribe/unsubscribe messages
    go func() {
        for {
            _, msg, err := conn.Read(ctx)
            if err != nil {
                return
            }
            var req SubscriptionRequest
            json.Unmarshal(msg, &req)
            switch req.Action {
            case "subscribe":
                events := make(chan Event, 64)
                cancel := s.informerHub.Subscribe(req.Resource, req.Namespace, req.Name, "", events)
                subscriptions[req.Channel] = cancel
                go func() {
                    for event := range events {
                        conn.Write(ctx, websocket.MessageText,
                            []byte(event.JSON()))
                    }
                }()
            case "unsubscribe":
                if cancel, ok := subscriptions[req.Channel]; ok {
                    cancel()
                    delete(subscriptions, req.Channel)
                }
            }
        }
    }()

    // Ping loop
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            conn.Ping(ctx)
        }
    }
}
```

**Complexity:** ~80 lines. Requires external library. Concurrent read/write goroutines. Subscription state management. Custom message protocol (subscribe/unsubscribe/event JSON envelopes).

**Libraries:**
- **`github.com/coder/websocket`** (recommended if using WebSocket): Maintained by Coder (formerly nhooyr.io/websocket). Context-aware, zero-alloc reads/writes, proper close handshake, idiomatic Go API. Actively maintained.
- **`github.com/gorilla/websocket`**: Was archived in 2022, partially restored by new maintainers, but had trust issues (v1.5.1 introduced problematic changes, reverted in v1.5.3). Not recommended for new projects.

### Comparison

| Aspect | SSE | WebSocket |
|--------|-----|-----------|
| Lines of code | ~50 | ~80 |
| External deps | None (or `r3labs/sse`) | `github.com/coder/websocket` |
| Goroutines per connection | 1 (the handler) | 2-3 (reader + writer + pinger) |
| Protocol design | None needed (HTTP) | Custom JSON subscribe/unsubscribe protocol |
| Connection cleanup | `r.Context().Done()` | Close handshake + subscription cleanup |
| Testing | Standard HTTP test utilities | WebSocket test client needed |

### Verdict

SSE is **significantly simpler** to implement in Go. It uses standard library primitives, requires no external dependencies, and avoids the complexity of bidirectional message handling, concurrent goroutines, and custom protocols.

---

## 10. Recommendation Table

| Criterion | WebSocket | SSE | Winner |
|-----------|-----------|-----|--------|
| **Cilium Gateway Compat** | Supported via appProtocol, but **open bugs** (#40822, #41123) causing connection failures | Standard HTTP, zero configuration needed, no known issues | **SSE** |
| **K8s Watch Relay** | Bidirectional subscribe/unsubscribe over single connection | URL-param subscriptions, one stream per resource scope | **SSE** (unidirectional matches the data flow) |
| **Refine liveProvider** | No built-in custom adapter; ~60 lines to implement | No built-in adapter; ~40 lines to implement | **SSE** (marginally simpler) |
| **Connection Mgmt** | Manual reconnect, re-subscribe, ping/pong heartbeat | Auto-reconnect, Last-Event-ID resume, keepalive comments | **SSE** |
| **Scalability** | Pod restart = reconnect + re-subscribe | Pod restart = auto-reconnect + resume from Last-Event-ID | **SSE** |
| **HTTP/2** | RFC 8441 has Envoy issues (#37020); falls back to HTTP/1.1 | Multiplexed streams on single TCP connection; efficient | **SSE** |
| **Error Recovery** | Manual reconnect with backoff; lost subscriptions | Built-in reconnect; Last-Event-ID resume; no lost events | **SSE** |
| **Auth Header Support** | Query param (insecure) or post-connect message | `@microsoft/fetch-event-source` with standard headers | **Tie** (slight SSE edge) |
| **Go Implementation** | ~80 lines, external dep, concurrent goroutines, custom protocol | ~50 lines, stdlib only, single goroutine, standard HTTP | **SSE** |

**Score: SSE 8 -- WebSocket 0 -- Tie 1**

---

## 11. Verdict

### Working Hypothesis: Validated

> "SSE is likely simpler than WebSocket: unidirectional (server->client status updates), built-in reconnection, standard HTTP (no upgrade negotiation issues with Cilium Gateway), stateless."

This hypothesis is **validated with strong evidence**.

### The Decisive Factor: Cilium Gateway API

The most compelling argument for SSE is **Cilium Gateway API compatibility**. There are active, unresolved bugs ([#40822](https://github.com/cilium/cilium/issues/40822), [#41123](https://github.com/cilium/cilium/issues/41123)) where WebSocket connections fail through Cilium's Gateway API despite working with direct connections. SSE, being standard HTTP, has **zero compatibility risk**.

### Counter-Argument Addressed

> "Counter-argument: WebSocket enables bidirectional communication (subscribe/unsubscribe without separate REST calls)."

This is true but **irrelevant for this use case**:

1. **Data flow is unidirectional.** K8s CRD status changes flow server -> client. The client never sends data that the server needs to process in real-time.
2. **Subscriptions are navigation-driven.** The user's page determines what they watch. Opening/closing SSE streams on navigation is natural and simple.
3. **Multiple SSE streams over HTTP/2 are efficient.** The "one connection" advantage of WebSocket is negated by HTTP/2 multiplexing.

### Recommendation

**Use SSE (Server-Sent Events)** for the Customer Portal BFF real-time layer.

**Implementation plan:**
1. BFF exposes `GET /api/events?resource={crd}&namespace={ns}&name={name}` endpoint
2. Server filters informer events by query parameters, writes matching events as SSE
3. Server sends `id:` with each event using `{resource}/{namespace}/{name}@{resourceVersion}` format
4. Server sends `: keepalive\n\n` every 15 seconds
5. Server checks `Last-Event-ID` header on reconnection and replays missed events from informer cache
6. Frontend uses `@microsoft/fetch-event-source` for auth header support
7. Custom Refine `liveProvider` wraps `fetchEventSource`, ~40 lines of code
8. No special Cilium Gateway API configuration needed

### Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Envoy `stream_idle_timeout` (5 min default) | Keepalive comments every 15s; optionally increase timeout via Cilium Helm |
| `@microsoft/fetch-event-source` maintenance | Library is simple (~300 lines); can vendor or rewrite if abandoned |
| HTTP/1.1 fallback (6 connection limit) | Ensure Cilium Gateway serves HTTP/2 (default for HTTPS) |
| Last-Event-ID replay complexity | Use K8s resourceVersion as event ID; informer cache provides replay |

---

## References

- [Gateway API v1.2: WebSockets, Timeouts, Retries, and More](https://kubernetes.io/blog/2024/11/21/gateway-api-v1-2/)
- [Gateway API Backend Protocol Selection](https://gateway-api.sigs.k8s.io/guides/backend-protocol/)
- [Cilium Gateway API Support (v1.19)](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/gateway-api/)
- [Cilium #40822 -- WebSocket breaks through Gateway API](https://github.com/cilium/cilium/issues/40822)
- [Cilium #41123 -- WebSocket fails through Gateway API](https://github.com/cilium/cilium/issues/41123)
- [Cilium #34582 -- Configure stream_idle_timeout for Envoy](https://github.com/cilium/cilium/issues/34582)
- [Cilium #30452 -- appProtocol support (merged, shipped in 1.16)](https://github.com/cilium/cilium/issues/30452)
- [Envoy #37020 -- WebSocket HTTP/2 fallback issue](https://github.com/envoyproxy/envoy/issues/37020)
- [Refine liveProvider Documentation](https://refine.dev/docs/realtime/live-provider/)
- [Refine liveProvider Discussion #5217](https://github.com/refinedev/refine/discussions/5217)
- [@microsoft/fetch-event-source (npm)](https://www.npmjs.com/package/@microsoft/fetch-event-source)
- [coder/websocket (GitHub)](https://github.com/coder/websocket)
- [r3labs/sse (GitHub)](https://github.com/r3labs/sse)
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [Ably: WebSockets vs SSE](https://ably.com/blog/websockets-vs-sse)
- [SSE over HTTP/2 and Envoy](https://medium.com/@kaitmore/server-sent-events-http-2-and-envoy-6927c70368bb)
- [Gateway API Implementations List](https://gateway-api.sigs.k8s.io/implementations/)

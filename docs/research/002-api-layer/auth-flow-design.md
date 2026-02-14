# Customer Portal BFF — Authentication & Authorization Design

**Status:** Draft
**Date:** 2026-02-14
**Repository:** `TedaTech/platform-v1-portal`
**Milestone:** Customer Platform MVP

---

## Table of Contents

1. [Frontend OIDC Flow](#1-frontend-oidc-flow)
2. [BFF JWT Validation](#2-bff-jwt-validation)
3. [Tenant Resolution](#3-tenant-resolution)
4. [Kubernetes Access Model](#4-kubernetes-access-model)
5. [KillBill Access Model](#5-killbill-access-model)
6. [Multi-Tenancy Authorization Matrix](#6-multi-tenancy-authorization-matrix)
7. [SSE Authentication](#7-sse-authentication)
8. [CORS Configuration](#8-cors-configuration)
9. [Security Considerations](#9-security-considerations)
10. [Keycloak Client Configuration](#10-keycloak-client-configuration)

---

## 1. Frontend OIDC Flow

### 1.1 Library Choice

The frontend uses [`oidc-spa`](https://github.com/keycloakify/oidc-spa), a browser-centric OpenID Connect library purpose-built for SPAs. It wraps the full Authorization Code + PKCE flow in a high-level API and handles the difficult edge cases (third-party cookie blocks, multi-tab session sync, silent refresh, session restore on reload) that most OIDC libraries leave to the developer.

### 1.2 OIDC Configuration

```typescript
// src/oidc.ts
import { createReactOidc } from "oidc-spa/react";

export const {
  OidcProvider,
  useOidc,
  getOidc,
  withLoginEnforced,
  enforceLogin,
} = createReactOidc({
  issuerUri: "https://keycloak.platform-v1.tedatech.app/realms/cozy",
  clientId: "portal-spa",
  // Scopes requested from Keycloak
  // "openid" is always implied
  // "groups" is a custom scope that includes the groups claim
  scopes: ["profile", "email", "groups"],
  // oidc-spa stores tokens in-memory by default (NOT localStorage)
  // This is the most secure option for SPAs
});
```

### 1.3 Keycloak Client Parameters

| Parameter | Value |
|-----------|-------|
| Client ID | `portal-spa` |
| Client Type | Public (no client secret) |
| Response Type | `code` (Authorization Code flow) |
| PKCE | Required (S256) |
| Scopes | `openid profile email groups` |
| Redirect URIs | `https://portal.platform-v1.tedatech.app/*` |
| Web Origins | `https://portal.platform-v1.tedatech.app` |

### 1.4 Token Lifecycle

| Token | Lifetime | Storage | Refresh Strategy |
|-------|----------|---------|------------------|
| Access Token | 5 minutes | In-memory (JS variable) | Silent refresh via iframe / refresh token |
| Refresh Token | 30 minutes | In-memory (JS variable) | Exchanged for new access token before expiry |
| ID Token | 5 minutes | In-memory (JS variable) | Renewed alongside access token |

`oidc-spa` manages the full lifecycle automatically:

1. **Initial login:** User is redirected to Keycloak's authorization endpoint. After authentication, Keycloak redirects back with an authorization code. `oidc-spa` exchanges the code for tokens (Authorization Code + PKCE flow).
2. **Silent refresh:** Before the access token expires, `oidc-spa` attempts a silent refresh using the refresh token. This happens transparently without user interaction.
3. **Session restore:** On page reload, `oidc-spa` restores the session from Keycloak's SSO session (via a hidden iframe or refresh token, depending on browser support).
4. **Multi-tab sync:** Login/logout events are synchronized across tabs using the BroadcastChannel API.
5. **Idle timeout:** After configurable inactivity, the user is logged out. Keycloak's SSO Session Idle setting (recommended: 5 minutes) controls this.

### 1.5 API Request Integration

Every API request includes the access token as a Bearer token:

```typescript
// src/api/client.ts
import { getOidc } from "../oidc";

export async function apiFetch(path: string, init?: RequestInit): Promise<Response> {
  const oidc = await getOidc();

  if (!oidc.isUserLoggedIn) {
    // This should not happen on protected routes
    throw new Error("User is not logged in");
  }

  const { accessToken } = oidc.getTokens();

  const response = await fetch(`https://portal-api.platform-v1.tedatech.app${path}`, {
    ...init,
    headers: {
      ...init?.headers,
      Authorization: `Bearer ${accessToken}`,
      "Content-Type": "application/json",
    },
  });

  if (response.status === 401) {
    // Token may have expired between getTokens() and the request arriving.
    // Trigger re-login. oidc-spa will attempt silent refresh first.
    await oidc.login({ doesCurrentHrefRequiresAuth: true });
  }

  return response;
}
```

### 1.6 TanStack Router Integration

Protected routes use `oidc-spa`'s `enforceLogin` helper in TanStack Router's `beforeLoad` hook:

```typescript
// src/routes/_authenticated.tsx
import { createFileRoute, Outlet } from "@tanstack/react-router";
import { enforceLogin } from "../oidc";

export const Route = createFileRoute("/_authenticated")({
  // enforceLogin redirects to Keycloak if user is not logged in.
  // On return, oidc-spa restores the original URL.
  beforeLoad: () => enforceLogin(),
  component: () => <Outlet />,
});
```

All portal routes are nested under `/_authenticated`, ensuring that unauthenticated users are always redirected to Keycloak:

```
src/routes/
  _authenticated.tsx          # Layout route with enforceLogin
  _authenticated/
    dashboard.tsx             # /dashboard
    billing.tsx               # /billing
    tenants/
      $tenantId.tsx           # /tenants/:tenantId
    settings.tsx              # /settings
```

---

## 2. BFF JWT Validation

### 2.1 Library Selection

**Recommended:** `github.com/MicahParks/keyfunc/v3` + `github.com/golang-jwt/jwt/v5`

This combination is preferred over `github.com/coreos/go-oidc/v3` because:

- `go-oidc` is designed primarily for ID token verification in the context of an OAuth2 code exchange flow. It validates ID tokens, not access tokens presented as Bearer tokens.
- `keyfunc/v3` + `golang-jwt/v5` provide direct access token validation with JWKS auto-refresh, which is exactly what a BFF needs.
- `keyfunc/v3` wraps `github.com/MicahParks/jwkset` and launches a background goroutine that automatically refreshes the JWKS from Keycloak's endpoint (default: every hour, with on-demand refresh for unknown key IDs, rate-limited to once per 5 minutes).

### 2.2 JWKS Initialization

```go
package auth

import (
    "context"
    "fmt"

    "github.com/MicahParks/keyfunc/v3"
    "github.com/golang-jwt/jwt/v5"
)

const (
    keycloakIssuer = "https://keycloak.platform-v1.tedatech.app/realms/cozy"
    jwksURL        = keycloakIssuer + "/protocol/openid-connect/certs"
    expectedAud    = "portal-spa"
)

type JWTValidator struct {
    keyfunc keyfunc.Keyfunc
}

// NewJWTValidator creates a validator with background JWKS refresh.
// Cancel the context to stop the refresh goroutine on shutdown.
func NewJWTValidator(ctx context.Context) (*JWTValidator, error) {
    k, err := keyfunc.NewDefaultCtx(ctx, []string{jwksURL})
    if err != nil {
        return nil, fmt.Errorf("failed to create JWKS keyfunc: %w", err)
    }
    return &JWTValidator{keyfunc: k}, nil
}
```

### 2.3 Token Validation Pipeline

```go
// Claims represents the JWT claims extracted from Keycloak access tokens.
type Claims struct {
    jwt.RegisteredClaims
    Email            string   `json:"email"`
    Name             string   `json:"name"`
    PreferredUsername string   `json:"preferred_username"`
    Groups           []string `json:"groups"`
    TenantID         string   `json:"tenant_id"`
    RealmAccess      struct {
        Roles []string `json:"roles"`
    } `json:"realm_access"`
}

// ValidateToken validates a JWT access token and returns the parsed claims.
func (v *JWTValidator) ValidateToken(tokenString string) (*Claims, error) {
    claims := &Claims{}

    token, err := jwt.ParseWithClaims(tokenString, claims, v.keyfunc.Keyfunc,
        jwt.WithValidMethods([]string{"RS256", "ES256"}),
        jwt.WithIssuer(keycloakIssuer),
        jwt.WithExpirationRequired(),
        // Note: Keycloak access tokens may use "account" as audience
        // rather than the client_id. Configure audience mapper if needed.
    )
    if err != nil {
        return nil, fmt.Errorf("token validation failed: %w", err)
    }

    if !token.Valid {
        return nil, fmt.Errorf("token is not valid")
    }

    return claims, nil
}
```

### 2.4 HTTP Middleware

```go
package middleware

import (
    "context"
    "net/http"
    "strings"

    "github.com/TedaTech/platform-v1-portal/internal/auth"
)

type contextKey string

const ClaimsContextKey contextKey = "claims"

// AuthMiddleware validates JWT tokens on every request.
func AuthMiddleware(validator *auth.JWTValidator) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 1. Extract Bearer token from Authorization header
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, `{"error":"missing authorization header"}`, http.StatusUnauthorized)
                return
            }

            parts := strings.SplitN(authHeader, " ", 2)
            if len(parts) != 2 || !strings.EqualFold(parts[0], "Bearer") {
                http.Error(w, `{"error":"invalid authorization header format"}`, http.StatusUnauthorized)
                return
            }

            tokenString := parts[1]

            // 2. Validate token (signature, issuer, expiry, not-before)
            claims, err := validator.ValidateToken(tokenString)
            if err != nil {
                http.Error(w, `{"error":"invalid token"}`, http.StatusUnauthorized)
                return
            }

            // 3. Resolve tenant from claims (see Section 3)
            tenantID, err := resolveTenant(claims)
            if err != nil {
                http.Error(w, `{"error":"unable to resolve tenant"}`, http.StatusForbidden)
                return
            }

            // 4. Inject claims and tenant into request context
            ctx := context.WithValue(r.Context(), ClaimsContextKey, claims)
            ctx = context.WithValue(ctx, TenantContextKey, tenantID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

### 2.5 Validation Steps Summary

| Step | Check | Failure Response |
|------|-------|-----------------|
| 1 | Authorization header present and well-formed | `401 Unauthorized` |
| 2 | JWT signature valid (via JWKS) | `401 Unauthorized` |
| 3 | Issuer matches `keycloakIssuer` | `401 Unauthorized` |
| 4 | Token not expired (`exp` claim) | `401 Unauthorized` |
| 5 | Token not used before valid (`nbf` claim) | `401 Unauthorized` |
| 6 | Algorithm is `RS256` or `ES256` (allowlist) | `401 Unauthorized` |
| 7 | Tenant resolvable from claims | `403 Forbidden` |
| 8 | User has required role for operation | `403 Forbidden` |

### 2.6 Error Response Format

All auth error responses use a consistent JSON format:

```json
{
  "error": "short_error_code",
  "message": "Human-readable description"
}
```

| HTTP Status | Error Code | When |
|-------------|-----------|------|
| 401 | `missing_token` | No Authorization header |
| 401 | `invalid_token` | Malformed, expired, or signature mismatch |
| 403 | `tenant_not_found` | Cannot resolve tenant from claims |
| 403 | `insufficient_permissions` | User role insufficient for the operation |

---

## 3. Tenant Resolution

### 3.1 Options Analysis

| Option | Mechanism | Pros | Cons |
|--------|-----------|------|------|
| **(a) Custom claim `tenant_id`** | Keycloak protocol mapper adds `tenant_id` to access token | Clean, explicit, fast lookup in BFF. Single source of truth. | Requires Keycloak protocol mapper configuration. Must keep mapper in sync. |
| **(b) Group membership** | User belongs to `/tenants/{tenant-id}` group. BFF parses `groups` claim. | Works with standard Keycloak groups. No custom mapper needed. Supports multi-tenant users natively (user in multiple groups). | Requires parsing/filtering group paths. Slightly more complex BFF logic. |
| **(c) Realm per tenant** | Each tenant gets its own Keycloak realm. BFF reads realm from `iss` claim. | Hard isolation between tenants. | Operational nightmare: N realms to manage, N OIDC configurations, N JWKS endpoints. Does not scale for MVP. |

### 3.2 Recommendation: Option (b) Group Membership with Custom Claim

Use **both (a) and (b)** in a layered approach:

1. **Primary: Group membership** (`/tenants/{tenant-id}`) as the authoritative source of tenant membership. This integrates naturally with the EDP Keycloak Operator's `KeycloakRealmGroup` CRD and supports multi-tenant users in the future.

2. **Convenience: Custom claim `tenant_id`** added via a Keycloak protocol mapper that extracts the tenant ID from the user's primary group. This makes BFF parsing simpler for the common single-tenant case.

The BFF resolves tenants in the following order:

```go
// resolveTenant extracts the tenant ID from JWT claims.
// Priority: explicit tenant_id claim > groups claim parsing.
func resolveTenant(claims *auth.Claims) (string, error) {
    // 1. Check explicit tenant_id claim (set by protocol mapper)
    if claims.TenantID != "" {
        return claims.TenantID, nil
    }

    // 2. Fall back to parsing groups claim
    for _, group := range claims.Groups {
        // Groups are in format "/tenants/{tenant-id}"
        if strings.HasPrefix(group, "/tenants/") {
            tenantID := strings.TrimPrefix(group, "/tenants/")
            if tenantID != "" {
                return tenantID, nil
            }
        }
    }

    return "", fmt.Errorf("no tenant membership found in token claims")
}
```

### 3.3 Tenant Mapping

| Domain | Mapping | Example |
|--------|---------|---------|
| Keycloak Group | `/tenants/{tenant-id}` | `/tenants/acme-corp` |
| K8s Namespace | `tenant-{tenant-id}` | `tenant-acme-corp` |
| KillBill Tenant | External key: `tenant:{tenant-id}` | `tenant:acme-corp` |
| KillBill Account | External key: `tenant:{tenant-id}:user:{sub}` | `tenant:acme-corp:user:a1b2c3` |

### 3.4 Multi-Tenant User Handling (Future)

For the MVP, each user belongs to exactly one tenant. The BFF uses the first matching `/tenants/{id}` group. In the future, multi-tenant users (e.g., platform admins) will need:

- A tenant selector in the frontend (stored in session/localStorage)
- A `X-Tenant-ID` request header sent by the frontend
- BFF validation that the requested tenant ID is in the user's groups

This is out of scope for MVP but the group-based design accommodates it without breaking changes.

---

## 4. Kubernetes Access Model

### 4.1 Architecture

```
Frontend (SPA)
    |
    | Bearer token (Keycloak JWT)
    v
BFF (Go monolith)
    |
    | in-cluster ServiceAccount
    | (no user impersonation)
    v
Kubernetes API
```

The BFF runs as a Pod in the cluster with a ServiceAccount (`portal-bff`). It does **not** impersonate the end user when talking to the Kubernetes API. Instead, it enforces authorization internally.

### 4.2 ServiceAccount RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: portal-bff
  namespace: platform-portal
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: portal-bff
rules:
  # Tenant namespace management
  - apiGroups: [""]
    resources: ["namespaces"]
    verbs: ["get", "list", "watch"]
  # Portal CRDs (tenant resources)
  - apiGroups: ["portal.tedatech.app"]
    resources: ["*"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Cozystack tenant resources
  - apiGroups: ["apps.cozystack.io"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  # Secrets (for KillBill API keys per tenant)
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
    # Restricted to specific secret names via resourceNames or
    # namespace-scoped RoleBindings per tenant namespace
  # Keycloak CRDs (for user/group management)
  - apiGroups: ["v1.edp.epam.com"]
    resources: ["keycloakclients", "keycloakrealmusers", "keycloakrealmgroups"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: portal-bff
subjects:
  - kind: ServiceAccount
    name: portal-bff
    namespace: platform-portal
roleRef:
  kind: ClusterRole
  name: portal-bff
  apiGroup: rbac.authorization.k8s.io
```

### 4.3 Internal Authorization Flow

```
Incoming request
    |
    v
[Auth Middleware] -- validates JWT, extracts claims, resolves tenant
    |
    v
[Namespace Guard] -- maps tenant_id -> "tenant-{id}" namespace
    |               -- rejects if namespace doesn't exist
    v
[Role Guard] -- checks user's role (admin/member/viewer)
    |          -- rejects if insufficient for the operation
    v
[K8s Client] -- performs operation scoped to tenant namespace
```

```go
// NamespaceForTenant returns the K8s namespace for a tenant.
func NamespaceForTenant(tenantID string) string {
    return "tenant-" + tenantID
}

// Example: listing Forgejo instances for a tenant
func (h *Handler) ListForgejoInstances(w http.ResponseWriter, r *http.Request) {
    claims := auth.ClaimsFromContext(r.Context())
    tenantID := auth.TenantFromContext(r.Context())
    namespace := NamespaceForTenant(tenantID)

    // Role check: any member can view
    if !claims.HasRole("member") {
        http.Error(w, `{"error":"insufficient_permissions"}`, http.StatusForbidden)
        return
    }

    // K8s operation scoped to tenant namespace
    instances, err := h.k8sClient.ForgejoInstances(namespace).List(r.Context())
    if err != nil {
        http.Error(w, `{"error":"internal_error"}`, http.StatusInternalServerError)
        return
    }

    json.NewEncoder(w).Encode(instances)
}
```

### 4.4 Why Not K8s Impersonation?

K8s impersonation (`Impersonate-User`, `Impersonate-Group` headers) was considered and rejected for the MVP:

| Factor | ServiceAccount (chosen) | Impersonation |
|--------|------------------------|---------------|
| Complexity | BFF holds one SA token. Authorization logic is in Go code. | BFF needs `impersonate` verb on ClusterRole. Must create K8s User/Group for every portal user. |
| Audit trail | Application-level audit logs with full user context. | K8s audit logs show impersonated user, but requires K8s audit policy configuration. |
| Flexibility | BFF can implement fine-grained RBAC logic (e.g., billing-only access) that doesn't map to K8s RBAC. | Constrained to what K8s RBAC can express. |
| Multi-backend | Same authorization model for K8s and KillBill. | Only works for K8s. KillBill still needs application-level auth. |
| Performance | Single informer cache shared across all users. | Impersonation may bypass informer cache depending on client-go usage. |

**Decision:** Use ServiceAccount with application-level authorization. Revisit impersonation if we need K8s-native audit compliance.

---

## 5. KillBill Access Model

### 5.1 KillBill Multi-Tenant Architecture

KillBill uses a per-tenant API authentication model. Each tenant in KillBill has a unique `apiKey` and `apiSecret` pair. Every API request must include these in headers to identify and authenticate the target tenant:

```
X-Killbill-ApiKey: {tenant-specific-api-key}
X-Killbill-ApiSecret: {tenant-specific-api-secret}
```

This is separate from user-level authentication (which KillBill handles via its own user management, but we bypass entirely since the BFF acts as a trusted intermediary).

### 5.2 API Key Storage

KillBill tenant credentials are stored as K8s Secrets, one per tenant:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: killbill-tenant-creds
  namespace: tenant-acme-corp
  labels:
    app.kubernetes.io/managed-by: portal-bff
    portal.tedatech.app/type: killbill-credentials
type: Opaque
data:
  api-key: <base64-encoded-api-key>
  api-secret: <base64-encoded-api-secret>
```

### 5.3 Credential Loading Strategy

The BFF loads KillBill credentials lazily with caching:

```go
type KillBillCredentials struct {
    APIKey    string
    APISecret string
}

type KillBillCredentialStore struct {
    k8sClient kubernetes.Interface
    cache     sync.Map // map[string]*KillBillCredentials
}

// GetCredentials returns cached credentials for a tenant,
// loading from K8s Secret on first access.
func (s *KillBillCredentialStore) GetCredentials(
    ctx context.Context, tenantID string,
) (*KillBillCredentials, error) {
    // Check cache first
    if cached, ok := s.cache.Load(tenantID); ok {
        return cached.(*KillBillCredentials), nil
    }

    // Load from K8s Secret
    namespace := NamespaceForTenant(tenantID)
    secret, err := s.k8sClient.CoreV1().Secrets(namespace).Get(
        ctx, "killbill-tenant-creds", metav1.GetOptions{},
    )
    if err != nil {
        return nil, fmt.Errorf("failed to load KillBill credentials for tenant %s: %w", tenantID, err)
    }

    creds := &KillBillCredentials{
        APIKey:    string(secret.Data["api-key"]),
        APISecret: string(secret.Data["api-secret"]),
    }

    s.cache.Store(tenantID, creds)
    return creds, nil
}
```

### 5.4 Request Enrichment

Every request from the BFF to KillBill includes tenant authentication and audit headers:

```go
func (c *KillBillClient) do(
    ctx context.Context, method, path string, body io.Reader,
    tenantID string, claims *auth.Claims,
) (*http.Response, error) {
    creds, err := c.credStore.GetCredentials(ctx, tenantID)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequestWithContext(ctx, method,
        c.baseURL+path, body)
    if err != nil {
        return nil, err
    }

    // Tenant authentication
    req.Header.Set("X-Killbill-ApiKey", creds.APIKey)
    req.Header.Set("X-Killbill-ApiSecret", creds.APISecret)

    // Audit trail headers
    req.Header.Set("X-Killbill-CreatedBy", claims.Email)
    req.Header.Set("X-Killbill-Reason", "portal-api")
    req.Header.Set("X-Killbill-Comment",
        fmt.Sprintf("user:%s tenant:%s", claims.Subject, tenantID))

    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Accept", "application/json")

    return c.httpClient.Do(req)
}
```

### 5.5 User-to-Account Mapping

Each portal user maps to a KillBill account via an external key:

```
External Key Format: tenant:{tenant-id}:user:{keycloak-sub}
Example:             tenant:acme-corp:user:a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

On first interaction with billing, the BFF checks if a KillBill account exists for the external key. If not, it creates one:

```go
func (c *KillBillClient) EnsureAccount(
    ctx context.Context, tenantID string, claims *auth.Claims,
) (*Account, error) {
    externalKey := fmt.Sprintf("tenant:%s:user:%s", tenantID, claims.Subject)

    // Try to fetch existing account
    account, err := c.GetAccountByExternalKey(ctx, tenantID, claims, externalKey)
    if err == nil {
        return account, nil
    }

    // Create new account
    return c.CreateAccount(ctx, tenantID, claims, &CreateAccountRequest{
        ExternalKey: externalKey,
        Name:        claims.Name,
        Email:       claims.Email,
        Currency:    "EUR",
    })
}
```

---

## 6. Multi-Tenancy Authorization Matrix

### 6.1 Role Definitions

| Role | Description | Assignment |
|------|-------------|-----------|
| `viewer` | Read-only access to tenant resources | Keycloak group: `/tenants/{id}/viewers` |
| `member` | Standard access: view + limited write | Keycloak group: `/tenants/{id}/members` |
| `admin` | Full access: all operations | Keycloak group: `/tenants/{id}/admins` |

Roles are hierarchical: `admin` implies `member`, which implies `viewer`.

```go
func (c *Claims) HasRole(required string) bool {
    roleHierarchy := map[string]int{
        "viewer": 1,
        "member": 2,
        "admin":  3,
    }

    userLevel := 0
    for _, group := range c.Groups {
        // Parse role from group path: /tenants/{id}/{role}s
        parts := strings.Split(strings.TrimPrefix(group, "/"), "/")
        if len(parts) == 3 && parts[0] == "tenants" {
            role := strings.TrimSuffix(parts[2], "s") // "admins" -> "admin"
            if level, ok := roleHierarchy[role]; ok && level > userLevel {
                userLevel = level
            }
        }
    }

    return userLevel >= roleHierarchy[required]
}
```

### 6.2 Authorization Matrix

| Operation | Endpoint | Min. Role | K8s Scoped | KillBill Scoped |
|-----------|----------|-----------|------------|-----------------|
| View tenant dashboard | `GET /api/tenants/{id}` | `viewer` | Yes | No |
| View tenant status | `GET /api/tenants/{id}/status` | `viewer` | Yes | No |
| List Forgejo instances | `GET /api/tenants/{id}/forgejo` | `viewer` | Yes | No |
| Create Forgejo instance | `POST /api/tenants/{id}/forgejo` | `admin` | Yes | No |
| Delete Forgejo instance | `DELETE /api/tenants/{id}/forgejo/{name}` | `admin` | Yes | No |
| View billing overview | `GET /api/tenants/{id}/billing` | `member` | No | Yes |
| View invoices | `GET /api/tenants/{id}/billing/invoices` | `member` | No | Yes |
| Change subscription | `POST /api/tenants/{id}/billing/subscription` | `admin` | No | Yes |
| Update payment method | `PUT /api/tenants/{id}/billing/payment-method` | `admin` | No | Yes |
| View pipeline status | `GET /api/tenants/{id}/pipelines` | `viewer` | Yes | No |
| Trigger pipeline run | `POST /api/tenants/{id}/pipelines/{name}/run` | `admin` | Yes | No |
| View pipeline logs | `GET /api/tenants/{id}/pipelines/{name}/logs` | `member` | Yes | No |
| Manage tenant members | `POST /api/tenants/{id}/members` | `admin` | No (Keycloak) | No |
| Watch tenant events (SSE) | `GET /api/tenants/{id}/events` | `viewer` | Yes | No |

### 6.3 Middleware Chain

Every request passes through the following middleware in order:

```
Request
  -> CORS Middleware
  -> Rate Limit Middleware
  -> Auth Middleware (JWT validation + tenant resolution)
  -> Tenant Guard Middleware (ensures tenant ID in URL matches token)
  -> Role Guard Middleware (checks minimum role for the operation)
  -> Handler
```

---

## 7. SSE Authentication

### 7.1 Connection Flow

The frontend uses `@microsoft/fetch-event-source` for SSE, which (unlike the native `EventSource` API) supports custom headers:

```typescript
// src/api/events.ts
import { fetchEventSource } from "@microsoft/fetch-event-source";
import { getOidc } from "../oidc";

export function watchTenantEvents(
  tenantId: string,
  onEvent: (event: TenantEvent) => void,
  signal: AbortSignal,
) {
  const connect = async () => {
    const oidc = await getOidc();
    if (!oidc.isUserLoggedIn) return;

    const { accessToken } = oidc.getTokens();

    await fetchEventSource(
      `https://portal-api.platform-v1.tedatech.app/api/tenants/${tenantId}/events`,
      {
        method: "GET",
        headers: {
          Authorization: `Bearer ${accessToken}`,
        },
        signal,
        onmessage(ev) {
          const event = JSON.parse(ev.data) as TenantEvent;
          onEvent(event);
        },
        onclose() {
          // Server closed the connection. Reconnect with fresh token.
          if (!signal.aborted) {
            setTimeout(() => connect(), 1000);
          }
        },
        onerror(err) {
          // On 401, reconnect with refreshed token.
          // fetchEventSource will call this before reconnecting.
          // Return undefined to let it reconnect, or throw to stop.
          if (!signal.aborted) {
            return; // Allow reconnect
          }
          throw err; // Stop reconnecting
        },
      },
    );
  };

  connect();
}
```

### 7.2 Token Expiry During SSE Connections

**Recommendation: Option A -- Close SSE on token expiry, frontend reconnects with refreshed token.**

| Aspect | Option A: Close on expiry (recommended) | Option B: Validate only on connect |
|--------|----------------------------------------|-----------------------------------|
| Security | Connection lifetime bounded by token lifetime. Revoked users are disconnected within ~5 min. | Connection can live indefinitely after initial auth. Revoked users remain connected. |
| Complexity | BFF must track token expiry per connection. | Simpler BFF logic. |
| UX | Brief reconnection gap (~100ms). `@microsoft/fetch-event-source` handles this transparently with automatic retry. | No reconnection interruption. |
| Implementation | BFF sets a timer per SSE connection based on token `exp` claim. | BFF validates once and forgets. |

**Implementation:**

```go
func (h *Handler) HandleSSE(w http.ResponseWriter, r *http.Request) {
    claims := auth.ClaimsFromContext(r.Context())
    tenantID := auth.TenantFromContext(r.Context())
    namespace := NamespaceForTenant(tenantID)

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    // Calculate connection deadline from token expiry
    tokenExpiry := claims.ExpiresAt.Time
    ctx, cancel := context.WithDeadline(r.Context(), tokenExpiry)
    defer cancel()

    // Start K8s watch on tenant namespace
    watcher, err := h.k8sClient.Watch(ctx, namespace)
    if err != nil {
        http.Error(w, "failed to start watch", http.StatusInternalServerError)
        return
    }
    defer watcher.Stop()

    for {
        select {
        case <-ctx.Done():
            // Token expired or client disconnected.
            // Client will reconnect with a fresh token.
            return
        case event, ok := <-watcher.ResultChan():
            if !ok {
                return
            }
            data, _ := json.Marshal(event)
            fmt.Fprintf(w, "data: %s\n\n", data)
            flusher.Flush()
        }
    }
}
```

The frontend's `@microsoft/fetch-event-source` detects the connection close and automatically reconnects. At that point, `oidc-spa` will have already silently refreshed the access token, so the reconnection carries a valid token.

### 7.3 SSE Last-Event-ID

To avoid missing events during reconnection, the BFF should support the `Last-Event-ID` header:

1. Each SSE event includes an `id` field (e.g., the K8s resource version).
2. On reconnect, `@microsoft/fetch-event-source` sends the `Last-Event-ID` header.
3. The BFF uses this to replay events since the last received event.

```
id: 12345
event: resource_updated
data: {"kind":"ForgejoInstance","name":"main","status":"ready"}
```

---

## 8. CORS Configuration

### 8.1 Domain Layout

| Component | Domain |
|-----------|--------|
| Frontend (SPA) | `https://portal.platform-v1.tedatech.app` |
| BFF API | `https://portal-api.platform-v1.tedatech.app` |
| Keycloak | `https://keycloak.platform-v1.tedatech.app` |

Since the frontend and BFF are on different subdomains, CORS is required.

### 8.2 CORS Middleware Configuration

```go
package middleware

import "net/http"

func CORSMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        origin := r.Header.Get("Origin")

        // Only allow the portal frontend origin
        allowedOrigin := "https://portal.platform-v1.tedatech.app"
        if origin == allowedOrigin {
            w.Header().Set("Access-Control-Allow-Origin", allowedOrigin)
        }

        w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
        w.Header().Set("Access-Control-Allow-Headers", "Authorization, Content-Type, Accept, Last-Event-ID")
        w.Header().Set("Access-Control-Expose-Headers", "X-Request-ID")
        w.Header().Set("Access-Control-Max-Age", "86400") // 24 hours preflight cache
        w.Header().Set("Access-Control-Allow-Credentials", "false") // No cookies needed

        // Handle preflight
        if r.Method == http.MethodOptions {
            w.WriteHeader(http.StatusNoContent)
            return
        }

        next.ServeHTTP(w, r)
    })
}
```

### 8.3 Key Decisions

| Header | Value | Rationale |
|--------|-------|-----------|
| `Access-Control-Allow-Origin` | `https://portal.platform-v1.tedatech.app` | Single allowed origin. No wildcard. |
| `Access-Control-Allow-Credentials` | `false` | We use Bearer tokens, not cookies. No need for credentials. |
| `Access-Control-Allow-Headers` | `Authorization, Content-Type, Accept, Last-Event-ID` | Authorization for Bearer token. Last-Event-ID for SSE reconnect. |
| `Access-Control-Max-Age` | `86400` | Cache preflight for 24 hours to reduce OPTIONS requests. |

### 8.4 Cilium Gateway API Configuration

CORS is handled at the BFF application level, not at the gateway. The Cilium Gateway API terminates TLS and forwards traffic:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: portal-api
  namespace: platform-portal
spec:
  parentRefs:
    - name: platform-gateway
      namespace: platform-system
  hostnames:
    - "portal-api.platform-v1.tedatech.app"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: portal-bff
          port: 8080
```

---

## 9. Security Considerations

### 9.1 CSRF Protection

**Not required.** CSRF attacks exploit browser-managed credentials (cookies, HTTP Basic Auth) that are automatically attached to requests. Bearer tokens in the `Authorization` header are not auto-sent by browsers. Since we use `Authorization: Bearer {token}` exclusively and do not use cookies for auth, CSRF is not a vector.

### 9.2 XSS Mitigation

| Control | Implementation |
|---------|---------------|
| Token storage | `oidc-spa` stores tokens in JavaScript variables (in-memory). They are not persisted to `localStorage`, `sessionStorage`, or cookies. A page refresh requires re-authentication via Keycloak's SSO session. |
| Content Security Policy | BFF sets `Content-Security-Policy` headers. Frontend build produces hashes for inline scripts. |
| Input sanitization | React's JSX auto-escapes rendered content. `dangerouslySetInnerHTML` is prohibited by linting rules. |
| HTTP-only cookies | Not applicable (no cookies used). |

Even though in-memory token storage means XSS-injected code running in the page context could theoretically read the token via the `getTokens()` API, this is the best available trade-off for SPAs. The short token lifetime (5 minutes) limits the blast radius.

### 9.3 Token Leakage Prevention

| Vector | Mitigation |
|--------|-----------|
| URL query parameters | Tokens are never placed in URLs. Authorization Code flow uses POST for token exchange. `oidc-spa` handles redirect callbacks securely. |
| Referrer headers | `Referrer-Policy: strict-origin-when-cross-origin` header set by BFF. |
| Browser history | Authorization codes are removed from the URL after exchange by `oidc-spa`. |
| Network interception | All traffic is HTTPS (TLS terminated at Cilium Gateway). HSTS headers enforced. |
| Server logs | BFF must never log the `Authorization` header value. Log the `sub` claim instead. |

### 9.4 Rate Limiting

| Endpoint Category | Limit | Key |
|-------------------|-------|-----|
| REST API (authenticated) | 100 req/min | Per user (`sub` claim) |
| SSE connections | 5 concurrent | Per user (`sub` claim) |
| Unauthenticated endpoints (health, readiness) | 60 req/min | Per IP |

Implementation uses a token bucket algorithm per user, stored in-memory (BFF is a single replica for MVP):

```go
// Rate limiter keyed by user subject
type RateLimiter struct {
    limiters sync.Map // map[string]*rate.Limiter
}

func (rl *RateLimiter) Allow(subject string) bool {
    limiter, _ := rl.limiters.LoadOrStore(subject,
        rate.NewLimiter(rate.Every(600*time.Millisecond), 10)) // ~100/min with burst of 10
    return limiter.(*rate.Limiter).Allow()
}
```

### 9.5 Audit Logging

All state-changing operations are logged with structured fields:

```go
type AuditEntry struct {
    Timestamp  time.Time `json:"timestamp"`
    Action     string    `json:"action"`      // "create", "update", "delete"
    Resource   string    `json:"resource"`     // "forgejo_instance", "subscription"
    ResourceID string    `json:"resource_id"`
    TenantID   string    `json:"tenant_id"`
    UserSub    string    `json:"user_sub"`
    UserEmail  string    `json:"user_email"`
    SourceIP   string    `json:"source_ip"`
    StatusCode int       `json:"status_code"`
}
```

Audit logs are written to stdout in JSON format and collected by the cluster's logging stack. For KillBill operations, audit context is additionally passed via `X-Killbill-CreatedBy` and `X-Killbill-Comment` headers (see Section 5.4).

### 9.6 Secret Management

| Secret | Storage | Access |
|--------|---------|--------|
| Keycloak JWKS | Fetched over HTTPS from Keycloak, cached in-memory | BFF auto-refreshes via `keyfunc` |
| KillBill API keys | K8s Secrets (`killbill-tenant-creds`) per tenant namespace | BFF ServiceAccount RBAC |
| KillBill admin password | Not used. BFF authenticates via tenant API keys only. | N/A |

Future consideration: Migrate KillBill API keys to OpenBao/Vault with dynamic secrets. The BFF would use Vault Agent sidecar or the Vault CSI provider to inject secrets.

---

## 10. Keycloak Client Configuration

### 10.1 KeycloakClient CRD (EDP Keycloak Operator)

```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakClient
metadata:
  name: portal-spa
  namespace: platform-keycloak
spec:
  # Reference to the Keycloak realm
  realmRef:
    name: cozy
    kind: KeycloakRealm

  # Client configuration
  clientId: portal-spa
  public: true                    # Public client (no client secret, uses PKCE)
  directAccess: false             # No Resource Owner Password Credentials grant
  implicitFlowEnabled: false      # Only Authorization Code + PKCE
  standardFlowEnabled: true       # Authorization Code flow enabled

  # URIs
  webUrl: "https://portal.platform-v1.tedatech.app"
  redirectUris:
    - "https://portal.platform-v1.tedatech.app/*"
  webOrigins:
    - "https://portal.platform-v1.tedatech.app"

  # Token settings
  attributes:
    "access.token.lifespan": "300"           # 5 minutes
    "client.session.idle.timeout": "1800"    # 30 minutes
    "pkce.code.challenge.method": "S256"     # Enforce PKCE with S256

  # Default scopes (automatically included in every token request)
  defaultClientScopes:
    - "openid"
    - "profile"
    - "email"
    - "groups"      # Custom scope for group membership claim

  # Protocol mappers for custom claims
  advancedProtocolMappers: true
  protocolMappers:
    # Include group membership in the access token
    - name: "group-membership"
      protocol: "openid-connect"
      protocolMapper: "oidc-group-membership-mapper"
      config:
        "full.path": "true"
        "id.token.claim": "true"
        "access.token.claim": "true"
        "claim.name": "groups"
        "userinfo.token.claim": "true"

    # Custom tenant_id claim extracted from user attribute
    - name: "tenant-id"
      protocol: "openid-connect"
      protocolMapper: "oidc-usermodel-attribute-mapper"
      config:
        "user.attribute": "tenant_id"
        "claim.name": "tenant_id"
        "id.token.claim": "true"
        "access.token.claim": "true"
        "userinfo.token.claim": "true"
        "jsonType.label": "String"

    # Audience mapper to include client_id in aud claim
    - name: "audience"
      protocol: "openid-connect"
      protocolMapper: "oidc-audience-mapper"
      config:
        "included.client.audience": "portal-spa"
        "id.token.claim": "true"
        "access.token.claim": "true"
```

### 10.2 KeycloakRealmGroup CRDs (Tenant Groups)

Each tenant gets a group hierarchy in Keycloak, managed by the EDP Keycloak Operator:

```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakRealmGroup
metadata:
  name: tenant-acme-corp
  namespace: platform-keycloak
spec:
  realmRef:
    name: cozy
    kind: KeycloakRealm
  name: "tenants/acme-corp"
  subGroups:
    - "admins"
    - "members"
    - "viewers"
```

### 10.3 Custom Client Scope for Groups

The `groups` scope must be created in Keycloak to include the group membership mapper:

```yaml
apiVersion: v1.edp.epam.com/v1
kind: KeycloakClientScope
metadata:
  name: groups
  namespace: platform-keycloak
spec:
  realmRef:
    name: cozy
    kind: KeycloakRealm
  name: "groups"
  description: "Include user group memberships in tokens"
  protocol: "openid-connect"
  protocolMappers:
    - name: "groups"
      protocol: "openid-connect"
      protocolMapper: "oidc-group-membership-mapper"
      config:
        "full.path": "true"
        "id.token.claim": "true"
        "access.token.claim": "true"
        "claim.name": "groups"
```

### 10.4 Keycloak Realm Settings

These settings should be applied to the `cozy` realm:

| Setting | Value | Rationale |
|---------|-------|-----------|
| SSO Session Idle | 5 minutes | Log out inactive users quickly (security-first) |
| SSO Session Max | 8 hours | Maximum session length for active users |
| Access Token Lifespan | 5 minutes | Short-lived tokens reduce exposure window |
| Client Session Idle | 30 minutes | Refresh token idle timeout |
| Client Session Max | 8 hours | Refresh token absolute timeout |
| Require PKCE | Yes (S256) | Prevent authorization code interception |
| Allowed Grant Types | Authorization Code only | No implicit, no ROPC, no client credentials |

### 10.5 Example JWT Access Token (Decoded)

```json
{
  "exp": 1739538600,
  "iat": 1739538300,
  "jti": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "iss": "https://keycloak.platform-v1.tedatech.app/realms/cozy",
  "aud": ["portal-spa", "account"],
  "sub": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "typ": "Bearer",
  "azp": "portal-spa",
  "scope": "openid profile email groups",
  "email": "jane@acme-corp.com",
  "name": "Jane Admin",
  "preferred_username": "jane",
  "email_verified": true,
  "tenant_id": "acme-corp",
  "groups": [
    "/tenants/acme-corp",
    "/tenants/acme-corp/admins"
  ],
  "realm_access": {
    "roles": ["default-roles-cozy", "offline_access", "uma_authorization"]
  }
}
```

---

## Appendix A: Sequence Diagrams

### A.1 Initial Login Flow

```
User            SPA (oidc-spa)         Keycloak                BFF
 |                   |                    |                      |
 |  Visit portal     |                    |                      |
 |------------------>|                    |                      |
 |                   |                    |                      |
 |                   | Redirect to /auth  |                      |
 |                   | (code_challenge,   |                      |
 |                   |  client_id, scope) |                      |
 |<------------------+                    |                      |
 |                   |                    |                      |
 |  Authenticate     |                    |                      |
 |--------------------------------------->|                      |
 |                   |                    |                      |
 |  Redirect back with authorization code |                      |
 |<----------------------------------------                      |
 |                   |                    |                      |
 |  Exchange code    |                    |                      |
 |------------------>|                    |                      |
 |                   | POST /token        |                      |
 |                   | (code, verifier)   |                      |
 |                   |------------------->|                      |
 |                   |                    |                      |
 |                   | {access_token,     |                      |
 |                   |  refresh_token,    |                      |
 |                   |  id_token}         |                      |
 |                   |<-------------------|                      |
 |                   |                    |                      |
 |  Render dashboard |                    |                      |
 |<------------------|                    |                      |
 |                   |                    |                      |
 |                   | GET /api/tenants/me                       |
 |                   | Authorization: Bearer {access_token}      |
 |                   |---------------------------------------------->|
 |                   |                    |                      |
 |                   |                    |     Validate JWT     |
 |                   |                    |     (JWKS cached)    |
 |                   |                    |     Extract tenant   |
 |                   |                    |     Query K8s        |
 |                   |                    |                      |
 |                   | 200 OK {tenant data}                     |
 |                   |<----------------------------------------------|
 |                   |                    |                      |
 |  Display data     |                    |                      |
 |<------------------|                    |                      |
```

### A.2 Token Refresh + SSE Reconnect

```
SPA (oidc-spa)             BFF                  Keycloak
 |                          |                      |
 | SSE: GET /events         |                      |
 | Bearer: token_v1         |                      |
 |------------------------->|                      |
 |                          |                      |
 |  event: resource_updated |                      |
 |<-------------------------|                      |
 |                          |                      |
 |  ... (streaming) ...     |                      |
 |                          |                      |
 |  [token_v1 approaches    |                      |
 |   expiry]                |                      |
 |                          |                      |
 | Silent refresh           |                      |
 |----------------------------------------------------->|
 |                          |                      |
 | New tokens (token_v2)    |                      |
 |<-----------------------------------------------------|
 |                          |                      |
 |  [BFF closes SSE:        |                      |
 |   token_v1 expired]      |                      |
 |<--------(connection closed)                     |
 |                          |                      |
 | SSE: GET /events         |                      |
 | Bearer: token_v2         |                      |
 | Last-Event-ID: 12345     |                      |
 |------------------------->|                      |
 |                          |                      |
 |  event: (replayed)       |                      |
 |<-------------------------|                      |
 |                          |                      |
 |  event: (new events)     |                      |
 |<-------------------------|                      |
```

---

## Appendix B: Dependency Summary

### Frontend

| Package | Version | Purpose |
|---------|---------|---------|
| `oidc-spa` | ^6.x | OIDC client (Authorization Code + PKCE, token management) |
| `@microsoft/fetch-event-source` | ^2.x | SSE with custom headers (Authorization) |
| `@tanstack/react-router` | ^1.x | Routing with `beforeLoad` auth guards |
| `@refinedev/core` | ^4.x | Data provider framework (wraps API calls) |

### Backend (Go)

| Module | Purpose |
|--------|---------|
| `github.com/MicahParks/keyfunc/v3` | JWKS fetching + background refresh + `jwt.Keyfunc` |
| `github.com/golang-jwt/jwt/v5` | JWT parsing and claim extraction |
| `k8s.io/client-go` | K8s API client (informers, watchers) |
| `github.com/oapi-codegen/oapi-codegen/v2` | OpenAPI code generation for REST API |

---

## Appendix C: Open Questions

1. **Audience claim:** Keycloak access tokens may include `"account"` in the `aud` array by default. Should we validate that `portal-spa` is in `aud`, or skip audience validation for access tokens? (ID tokens always have the client as audience, but access tokens may not.) **Recommendation:** Add an audience mapper (Section 10.1) to ensure `portal-spa` is always in `aud`, then validate it.

2. **Platform admin role:** How should TedaTech internal admins access all tenants? Options: (a) a Keycloak realm role `platform-admin` that bypasses tenant scoping, (b) add platform admins to all tenant groups. **Recommendation:** Option (a) with a dedicated realm role, checked in the BFF's tenant resolution logic.

3. **KillBill credential rotation:** When a KillBill API key is rotated, the BFF's in-memory cache becomes stale. Options: (a) watch the K8s Secret for changes, (b) use a short TTL on the cache, (c) use Vault dynamic secrets. **Recommendation:** Option (a) for MVP using a K8s informer on Secrets with the `portal.tedatech.app/type: killbill-credentials` label.

4. **Offline access:** Should the portal support offline access (refresh tokens that survive browser close)? `oidc-spa` supports this but it weakens security. **Recommendation:** No. Require re-authentication after browser close. Set `SSO Session Idle` to 5 minutes.

5. **Multiple tenants per user:** The design accommodates this (group-based membership) but the BFF only uses the first tenant for MVP. When should multi-tenant user support be implemented? **Recommendation:** Defer to post-MVP. Track as a separate issue.

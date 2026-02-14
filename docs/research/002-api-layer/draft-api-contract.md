# Draft API Contract: Customer Portal BFF

**Status:** Draft
**Date:** 2026-02-14
**API Style:** REST (OpenAPI 3.0)
**Base URL:** `https://portal-api.platform-v1.tedatech.app`
**Related ADR:** [ADR-002: API Layer Architecture](adr-002-api-layer.md)

---

## Overview

This document defines the draft OpenAPI 3.0 contract for the TedaTech Customer Portal BFF. The BFF is a Go monolith serving REST endpoints for CRUD operations and SSE for real-time status updates. All endpoints (except health checks) require a valid Keycloak JWT in the `Authorization` header.

The BFF proxies two backend systems:
- **Kubernetes API** -- tenant provisioning, Forgejo instances, pipelines (via CRDs)
- **KillBill API** -- billing, subscriptions, invoices (REST 1:1 proxy)

---

## OpenAPI 3.0 Specification

### Info and Servers

```yaml
openapi: "3.0.3"
info:
  title: TedaTech Customer Portal API
  description: |
    Backend-for-Frontend API for the TedaTech Customer Portal.
    Proxies Kubernetes CRD operations and KillBill billing API.
  version: 0.1.0
  contact:
    name: TedaTech Platform Team
    url: https://github.com/TedaTech/platform-v1-portal

servers:
  - url: https://portal-api.platform-v1.tedatech.app
    description: Production
  - url: http://localhost:8080
    description: Local development
```

### Security Schemes

```yaml
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: |
        Keycloak-issued JWT access token. Obtained via Authorization Code + PKCE
        flow from the oidc-spa frontend library. Token contains `sub`, `email`,
        `groups` (tenant membership), and `tenant_id` claims.

security:
  - bearerAuth: []
```

### Common Schemas

```yaml
components:
  schemas:
    Error:
      type: object
      required:
        - error
      properties:
        error:
          type: object
          required:
            - code
            - message
          properties:
            code:
              type: string
              description: Machine-readable error code
              example: TENANT_NOT_FOUND
            message:
              type: string
              description: Human-readable error description
              example: "Tenant 'acme-corp' does not exist or you do not have access"
            details:
              type: object
              additionalProperties: true
              description: Additional context for debugging

    ResourceStatus:
      type: object
      required:
        - phase
      properties:
        phase:
          type: string
          enum: [Pending, Provisioning, Active, Failed, Deleting, Unknown]
          description: Portal-friendly lifecycle phase derived from K8s conditions
        reason:
          type: string
          description: K8s condition reason
          example: Provisioning
        message:
          type: string
          description: Human-readable status message
          example: "Creating Forgejo organization"

    PaginationMeta:
      type: object
      properties:
        total:
          type: integer
          description: Total number of items
          example: 42
        page:
          type: integer
          description: Current page number (1-indexed)
          example: 1
        pageSize:
          type: integer
          description: Items per page
          example: 25
```

---

## Endpoints

### 1. Auth / Profile

#### `GET /api/users/me`

Returns the current user's profile, resolved from the JWT claims and backend systems.

```yaml
paths:
  /api/users/me:
    get:
      operationId: getCurrentUser
      summary: Get current user profile
      tags: [Auth]
      responses:
        "200":
          description: Current user profile
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserProfile"
        "401":
          description: Invalid or expired JWT
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    UserProfile:
      type: object
      required:
        - id
        - email
        - name
        - tenants
      properties:
        id:
          type: string
          format: uuid
          description: Keycloak user ID (sub claim)
          example: f47ac10b-58cc-4372-a567-0e02b2c3d479
        email:
          type: string
          format: email
          example: jane@acme-corp.com
        name:
          type: string
          example: Jane Admin
        tenants:
          type: array
          items:
            $ref: "#/components/schemas/TenantMembership"
          description: Tenants the user has access to
        billingAccountId:
          type: string
          description: KillBill account ID (null if not yet initialized)
          nullable: true
          example: a1b2c3d4-e5f6-7890-abcd-ef1234567890

    TenantMembership:
      type: object
      required:
        - id
        - name
        - role
      properties:
        id:
          type: string
          description: Tenant identifier
          example: acme-corp
        name:
          type: string
          description: Display name
          example: Acme Corp
        role:
          type: string
          enum: [viewer, member, admin]
          description: User's role in this tenant
```

#### `POST /api/users/me/initialize`

First-login initialization. Creates a KillBill billing account if one does not exist for this user, and resolves tenant membership. Idempotent -- safe to call on every login.

```yaml
paths:
  /api/users/me/initialize:
    post:
      operationId: initializeUser
      summary: Initialize user on first login
      description: |
        Creates a KillBill billing account (if not exists) and resolves
        tenant membership from Keycloak groups. Idempotent.
      tags: [Auth]
      responses:
        "200":
          description: User initialized (or already initialized)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserProfile"
        "401":
          description: Invalid or expired JWT
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "502":
          description: KillBill or Keycloak unavailable
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
```

---

### 2. Tenants

#### `GET /api/tenants`

List tenants accessible to the current user.

```yaml
paths:
  /api/tenants:
    get:
      operationId: listTenants
      summary: List user's tenants
      tags: [Tenants]
      responses:
        "200":
          description: List of tenants
          content:
            application/json:
              schema:
                type: object
                required:
                  - data
                properties:
                  data:
                    type: array
                    items:
                      $ref: "#/components/schemas/Tenant"
        "401":
          description: Invalid or expired JWT
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
```

#### `POST /api/tenants`

Create a new tenant. Triggers provisioning of Cozystack namespace, Forgejo organization, and Keycloak group. Returns immediately with status `Provisioning`; subscribe to SSE for real-time progress.

```yaml
    post:
      operationId: createTenant
      summary: Create a new tenant
      description: |
        Creates a new tenant by provisioning Cozystack namespace, ForgejaTenant CR,
        and KeycloakRealmGroup CR. Returns immediately with status "Provisioning".
        Use the SSE events endpoint to track provisioning progress.
      tags: [Tenants]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateTenantRequest"
      responses:
        "201":
          description: Tenant created, provisioning started
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Tenant"
          headers:
            Location:
              schema:
                type: string
              description: URL to the created tenant
              example: /api/tenants/acme-corp
        "400":
          description: Invalid request body
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "403":
          description: Insufficient permissions (admin role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "409":
          description: Tenant with this name already exists
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    CreateTenantRequest:
      type: object
      required:
        - name
        - displayName
      properties:
        name:
          type: string
          pattern: "^[a-z0-9][a-z0-9-]{1,46}[a-z0-9]$"
          description: |
            Unique tenant identifier. Must be lowercase alphanumeric with hyphens.
            Used as K8s namespace suffix (tenant-{name}) and Keycloak group name.
          example: acme-corp
        displayName:
          type: string
          maxLength: 128
          description: Human-readable tenant name
          example: Acme Corporation
        plan:
          type: string
          enum: [starter, professional, enterprise]
          default: starter
          description: Billing plan (determines resource quotas)
        adminEmail:
          type: string
          format: email
          description: Admin email for Forgejo organization (defaults to current user's email)
```

#### `GET /api/tenants/{id}`

Get tenant details including current provisioning status.

```yaml
paths:
  /api/tenants/{id}:
    get:
      operationId: getTenant
      summary: Get tenant details
      tags: [Tenants]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
          example: acme-corp
      responses:
        "200":
          description: Tenant details
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Tenant"
          headers:
            ETag:
              schema:
                type: string
              description: K8s resourceVersion for optimistic concurrency
              example: '"12345"'
        "403":
          description: User does not have access to this tenant
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "404":
          description: Tenant not found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    Tenant:
      type: object
      required:
        - id
        - displayName
        - status
        - plan
        - createdAt
      properties:
        id:
          type: string
          example: acme-corp
        displayName:
          type: string
          example: Acme Corporation
        plan:
          type: string
          enum: [starter, professional, enterprise]
          example: starter
        status:
          $ref: "#/components/schemas/ResourceStatus"
        resourceVersion:
          type: string
          description: K8s resourceVersion (used as ETag for optimistic concurrency)
          example: "12345"
        namespace:
          type: string
          description: K8s namespace for this tenant
          example: tenant-acme-corp
        quotas:
          $ref: "#/components/schemas/ResourceQuotas"
        createdAt:
          type: string
          format: date-time
        links:
          type: object
          properties:
            self:
              type: string
              example: /api/tenants/acme-corp
            events:
              type: string
              example: /api/tenants/acme-corp/events
            forgejo:
              type: string
              example: /api/tenants/acme-corp/forgejo
            billing:
              type: string
              example: /api/tenants/acme-corp/billing/subscription

    ResourceQuotas:
      type: object
      properties:
        cpu:
          type: string
          description: CPU quota (K8s resource format)
          example: "4"
        memory:
          type: string
          description: Memory quota (K8s resource format)
          example: 8Gi
        storage:
          type: string
          description: Storage quota
          example: 50Gi
```

#### `DELETE /api/tenants/{id}`

Delete a tenant and all associated resources.

```yaml
    delete:
      operationId: deleteTenant
      summary: Delete a tenant
      description: |
        Initiates tenant deletion. Removes Cozystack Tenant CR, ForgejaTenant CR,
        KeycloakRealmGroup CR, and the tenant namespace. Deletion is asynchronous;
        the tenant status transitions to "Deleting" and eventually disappears.
      tags: [Tenants]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        "202":
          description: Deletion initiated
          content:
            application/json:
              schema:
                type: object
                properties:
                  message:
                    type: string
                    example: "Tenant deletion initiated"
        "403":
          description: Insufficient permissions (admin role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "404":
          description: Tenant not found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
```

#### `POST /api/tenants/{id}/retry`

Retry failed provisioning by patching the retry annotation on the underlying CRDs.

```yaml
paths:
  /api/tenants/{id}/retry:
    post:
      operationId: retryTenantProvisioning
      summary: Retry failed tenant provisioning
      description: |
        Patches a retry annotation on the tenant's CRDs to trigger
        re-reconciliation by the respective operators. Only applicable
        when tenant status is "Failed".
      tags: [Tenants]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        "202":
          description: Retry requested
          content:
            application/json:
              schema:
                type: object
                properties:
                  message:
                    type: string
                    example: "Retry requested; watch events for status updates"
        "403":
          description: Insufficient permissions (admin role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "404":
          description: Tenant not found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "409":
          description: Tenant is not in a failed state
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
```

---

### 3. Forgejo Instances

#### `GET /api/tenants/{id}/forgejo`

List Forgejo instances for a tenant.

```yaml
paths:
  /api/tenants/{id}/forgejo:
    get:
      operationId: listForgejoInstances
      summary: List Forgejo instances
      tags: [Forgejo]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
          description: Tenant ID
      responses:
        "200":
          description: List of Forgejo instances
          content:
            application/json:
              schema:
                type: object
                required:
                  - data
                properties:
                  data:
                    type: array
                    items:
                      $ref: "#/components/schemas/ForgejoInstance"
        "403":
          description: User does not have access to this tenant
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    ForgejoInstance:
      type: object
      required:
        - name
        - status
        - createdAt
      properties:
        name:
          type: string
          description: Instance name (K8s resource name)
          example: main
        organizationName:
          type: string
          description: Forgejo organization name
          example: acme-corp
        orgUrl:
          type: string
          format: uri
          description: URL to the Forgejo organization (available when Active)
          example: https://forgejo.platform-v1.tedatech.app/acme-corp
        repoUrl:
          type: string
          format: uri
          description: URL to the GitOps repository (available when Active)
          example: https://forgejo.platform-v1.tedatech.app/acme-corp/gitops
        status:
          $ref: "#/components/schemas/ResourceStatus"
        resourceVersion:
          type: string
        createdAt:
          type: string
          format: date-time
```

#### `POST /api/tenants/{id}/forgejo`

Create a new Forgejo instance (ForgejaTenant CR).

```yaml
    post:
      operationId: createForgejoInstance
      summary: Create a Forgejo instance
      description: |
        Creates a ForgejaTenant CR in the tenant namespace. The Crossplane
        composition reconciles this into a Forgejo organization, GitOps
        repository, and webhook configuration.
      tags: [Forgejo]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateForgejoRequest"
      responses:
        "201":
          description: Forgejo instance created, provisioning started
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ForgejoInstance"
        "400":
          description: Invalid request
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "403":
          description: Insufficient permissions (admin role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "409":
          description: Forgejo instance with this name already exists
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    CreateForgejoRequest:
      type: object
      required:
        - organizationName
      properties:
        organizationName:
          type: string
          pattern: "^[a-z0-9][a-z0-9-]{1,38}[a-z0-9]$"
          description: Forgejo organization name
          example: acme-corp
        adminEmail:
          type: string
          format: email
          description: Organization admin email (defaults to current user)
        repoTemplate:
          type: string
          description: Template repository to clone for GitOps repo
          example: tedatech/gitops-template
```

---

### 4. Pipelines

#### `GET /api/tenants/{id}/pipelines`

List pipelines for a tenant.

```yaml
paths:
  /api/tenants/{id}/pipelines:
    get:
      operationId: listPipelines
      summary: List pipelines
      tags: [Pipelines]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
        - name: page
          in: query
          schema:
            type: integer
            default: 1
            minimum: 1
        - name: pageSize
          in: query
          schema:
            type: integer
            default: 25
            minimum: 1
            maximum: 100
      responses:
        "200":
          description: List of pipelines
          content:
            application/json:
              schema:
                type: object
                required:
                  - data
                properties:
                  data:
                    type: array
                    items:
                      $ref: "#/components/schemas/Pipeline"
                  meta:
                    $ref: "#/components/schemas/PaginationMeta"
```

#### `POST /api/tenants/{id}/pipelines`

Trigger a new pipeline from a template.

```yaml
    post:
      operationId: triggerPipeline
      summary: Trigger a pipeline
      description: |
        Creates a Pipeline CR in the tenant namespace. The pipeline operator
        reconciles this by applying a template, pushing a branch, and creating
        a PR in the tenant's Forgejo GitOps repository. Returns 202 Accepted
        immediately; track progress via SSE.
      tags: [Pipelines]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/TriggerPipelineRequest"
      responses:
        "202":
          description: Pipeline triggered
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Pipeline"
        "400":
          description: Invalid request
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "403":
          description: Insufficient permissions (admin role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "422":
          description: Template parameter validation failed
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    TriggerPipelineRequest:
      type: object
      required:
        - template
        - params
      properties:
        template:
          type: string
          description: Template name (ConfigMap name with template label)
          example: nextjs-app
        params:
          type: object
          additionalProperties:
            type: string
          description: Template parameters (validated against template schema)
          example:
            name: my-app
            domain: my-app.acme.teda.tech

    Pipeline:
      type: object
      required:
        - id
        - template
        - status
        - createdAt
      properties:
        id:
          type: string
          description: Pipeline CR name
          example: pipeline-abc123
        template:
          type: string
          description: Template used
          example: nextjs-app
        params:
          type: object
          additionalProperties:
            type: string
        status:
          $ref: "#/components/schemas/PipelineStatus"
        createdAt:
          type: string
          format: date-time
        completedAt:
          type: string
          format: date-time
          nullable: true

    PipelineStatus:
      type: object
      required:
        - phase
      properties:
        phase:
          type: string
          enum: [Pending, Running, BranchPushed, PRCreated, Completed, Failed]
        reason:
          type: string
        message:
          type: string
        prUrl:
          type: string
          format: uri
          description: Forgejo PR URL (available after PRCreated phase)
          example: https://forgejo.platform-v1.tedatech.app/acme-corp/gitops/pulls/42
        prNumber:
          type: integer
          description: PR number in Forgejo
          example: 42
```

#### `GET /api/tenants/{id}/pipelines/{pipelineId}`

Get pipeline details and current status.

```yaml
paths:
  /api/tenants/{id}/pipelines/{pipelineId}:
    get:
      operationId: getPipeline
      summary: Get pipeline status
      tags: [Pipelines]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
        - name: pipelineId
          in: path
          required: true
          schema:
            type: string
      responses:
        "200":
          description: Pipeline details
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Pipeline"
        "404":
          description: Pipeline not found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
```

---

### 5. Billing

#### `GET /api/tenants/{id}/billing/subscription`

Get the current subscription for a tenant. Proxied from KillBill.

```yaml
paths:
  /api/tenants/{id}/billing/subscription:
    get:
      operationId: getSubscription
      summary: Get current subscription
      description: |
        Retrieves the tenant's current billing subscription from KillBill.
        Includes available plans for upgrade/downgrade.
      tags: [Billing]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        "200":
          description: Current subscription
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Subscription"
        "403":
          description: Insufficient permissions (member role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "502":
          description: KillBill unavailable
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    Subscription:
      type: object
      required:
        - id
        - planName
        - status
      properties:
        id:
          type: string
          format: uuid
          description: KillBill subscription ID
        planName:
          type: string
          enum: [starter, professional, enterprise]
          example: starter
        status:
          type: string
          enum: [ACTIVE, CANCELLED, PENDING, BLOCKED]
          example: ACTIVE
        phase:
          type: string
          description: KillBill billing phase
          example: EVERGREEN
        startDate:
          type: string
          format: date
        currentPeriodStart:
          type: string
          format: date
        currentPeriodEnd:
          type: string
          format: date
        pricePerMonth:
          type: string
          description: Monthly price in EUR
          example: "29.00"
        currency:
          type: string
          default: EUR
        quotas:
          $ref: "#/components/schemas/ResourceQuotas"
          description: Resource quotas associated with this plan
        availablePlans:
          type: array
          items:
            $ref: "#/components/schemas/AvailablePlan"
          description: Plans available for upgrade/downgrade

    AvailablePlan:
      type: object
      required:
        - name
        - pricePerMonth
        - quotas
      properties:
        name:
          type: string
          example: professional
        displayName:
          type: string
          example: Professional
        pricePerMonth:
          type: string
          example: "49.00"
        currency:
          type: string
          default: EUR
        quotas:
          $ref: "#/components/schemas/ResourceQuotas"
        features:
          type: array
          items:
            type: string
          description: Features included in this plan
          example: ["8 vCPU", "16 GB RAM", "200 GB Storage", "3 Namespaces"]
```

#### `POST /api/tenants/{id}/billing/subscription/change`

Change the tenant's billing plan.

```yaml
paths:
  /api/tenants/{id}/billing/subscription/change:
    post:
      operationId: changeSubscription
      summary: Change billing plan
      description: |
        Changes the tenant's subscription plan in KillBill. The plan change
        may be immediate or effective at end of the current billing period.
        Resource quota updates in K8s are triggered asynchronously via KillBill
        webhook after the plan change is confirmed.
      tags: [Billing]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/ChangeSubscriptionRequest"
      responses:
        "200":
          description: Plan change applied
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Subscription"
        "403":
          description: Insufficient permissions (admin role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "422":
          description: |
            Plan change not allowed. Possible reasons:
            - Cannot downgrade during trial period
            - Current resource usage exceeds new plan quotas
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "502":
          description: KillBill unavailable
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    ChangeSubscriptionRequest:
      type: object
      required:
        - planName
      properties:
        planName:
          type: string
          enum: [starter, professional, enterprise]
          description: Target plan
          example: professional
        effectiveDate:
          type: string
          enum: [IMMEDIATE, END_OF_TERM]
          default: IMMEDIATE
          description: |
            When the plan change takes effect.
            IMMEDIATE: change now, prorate current period.
            END_OF_TERM: change at next billing cycle.
```

#### `GET /api/tenants/{id}/billing/invoices`

List invoices for a tenant.

```yaml
paths:
  /api/tenants/{id}/billing/invoices:
    get:
      operationId: listInvoices
      summary: List invoices
      tags: [Billing]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
        - name: page
          in: query
          schema:
            type: integer
            default: 1
            minimum: 1
        - name: pageSize
          in: query
          schema:
            type: integer
            default: 25
            minimum: 1
            maximum: 100
      responses:
        "200":
          description: List of invoices
          content:
            application/json:
              schema:
                type: object
                required:
                  - data
                properties:
                  data:
                    type: array
                    items:
                      $ref: "#/components/schemas/Invoice"
                  meta:
                    $ref: "#/components/schemas/PaginationMeta"
        "403":
          description: Insufficient permissions (member role required)
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    Invoice:
      type: object
      required:
        - id
        - invoiceNumber
        - status
        - amount
        - currency
        - invoiceDate
      properties:
        id:
          type: string
          format: uuid
          description: KillBill invoice ID
        invoiceNumber:
          type: string
          example: INV-0042
        status:
          type: string
          enum: [DRAFT, COMMITTED, VOID, PAID]
          example: COMMITTED
        amount:
          type: string
          description: Total amount
          example: "49.00"
        balance:
          type: string
          description: Outstanding balance
          example: "0.00"
        currency:
          type: string
          example: EUR
        invoiceDate:
          type: string
          format: date
          example: "2026-02-01"
        targetDate:
          type: string
          format: date
        items:
          type: array
          items:
            $ref: "#/components/schemas/InvoiceItem"

    InvoiceItem:
      type: object
      properties:
        description:
          type: string
          example: "Professional Plan (Feb 2026)"
        amount:
          type: string
          example: "49.00"
        startDate:
          type: string
          format: date
        endDate:
          type: string
          format: date
        planName:
          type: string
          example: professional
```

---

### 6. Events (SSE)

#### `GET /api/tenants/{id}/events`

Server-Sent Events stream for real-time tenant resource updates.

```yaml
paths:
  /api/tenants/{id}/events:
    get:
      operationId: streamTenantEvents
      summary: SSE stream for tenant events
      description: |
        Opens a Server-Sent Events (text/event-stream) connection that pushes
        real-time updates for resources in the tenant's namespace. Events are
        generated from K8s informer callbacks via the in-process event bus.

        **Connection lifecycle:**
        - Connection is bounded by the JWT expiry time. The BFF closes the
          connection when the token expires; the client reconnects with a
          refreshed token.
        - Keepalive comments (`: keepalive`) are sent every 15 seconds to
          prevent Envoy idle timeout.
        - On reconnection, send `Last-Event-ID` header to resume from the
          last received event.

        **Event ID format:** `{CRDKind}/{namespace}/{name}@{resourceVersion}`

        **Frontend usage:** Use `@microsoft/fetch-event-source` (supports
        Authorization headers, unlike native EventSource API).
      tags: [Events]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
          description: Tenant ID
        - name: resourceKind
          in: query
          schema:
            type: string
          description: Filter events by CRD kind (e.g., ForgejaTenant, Pipeline)
        - name: name
          in: query
          schema:
            type: string
          description: Filter events by resource name
      responses:
        "200":
          description: SSE event stream
          content:
            text/event-stream:
              schema:
                type: string
                description: |
                  SSE stream. Each event has the format:

                  ```
                  id: ForgejaTenant/tenant-acme-corp/main@rv5678
                  event: updated
                  data: {"type":"updated","resourceKind":"ForgejaTenant","name":"main","status":{"phase":"Active","reason":"Ready","message":"Forgejo organization ready"},"timestamp":"2026-02-14T10:30:00Z"}
                  ```

                  Event types: `created`, `updated`, `deleted`, `status.changed`

                  Keepalive (sent every 15s, ignored by EventSource):
                  ```
                  : keepalive
                  ```
          headers:
            Cache-Control:
              schema:
                type: string
                default: no-cache
            Connection:
              schema:
                type: string
                default: keep-alive
            X-Accel-Buffering:
              schema:
                type: string
                default: "no"
        "401":
          description: Invalid or expired JWT
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"
        "403":
          description: User does not have access to this tenant
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Error"

components:
  schemas:
    TenantEvent:
      type: object
      required:
        - type
        - resourceKind
        - name
        - timestamp
      properties:
        type:
          type: string
          enum: [resource.created, resource.updated, resource.deleted, status.changed]
          description: Event type
        resourceKind:
          type: string
          description: K8s CRD kind
          example: ForgejaTenant
        namespace:
          type: string
          example: tenant-acme-corp
        name:
          type: string
          description: Resource name
          example: main
        status:
          $ref: "#/components/schemas/ResourceStatus"
        oldStatus:
          $ref: "#/components/schemas/ResourceStatus"
          description: Previous status (only for status.changed events)
        timestamp:
          type: string
          format: date-time
        generation:
          type: integer
          format: int64
        resourceVersion:
          type: string
```

---

### 7. Health Checks

Health endpoints do not require authentication.

```yaml
paths:
  /healthz:
    get:
      operationId: healthLiveness
      summary: Liveness probe
      description: |
        Returns 200 if the process is alive and not deadlocked.
        Does NOT check external dependencies.
        Used by Kubernetes liveness probe.
      tags: [Health]
      security: []
      responses:
        "200":
          description: Process is alive
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok

  /readyz:
    get:
      operationId: healthReadiness
      summary: Readiness probe
      description: |
        Returns 200 only when the BFF is ready to serve traffic:
        - K8s informer caches are synced (initial list completed)
        - KillBill API is reachable (lightweight health check)

        A readiness probe failure removes the pod from Service endpoints.
      tags: [Health]
      security: []
      responses:
        "200":
          description: BFF is ready to serve traffic
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ReadinessStatus"
        "503":
          description: BFF is not ready
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/ReadinessStatus"

components:
  schemas:
    ReadinessStatus:
      type: object
      required:
        - status
        - checks
      properties:
        status:
          type: string
          enum: [ready, not_ready]
        checks:
          type: object
          properties:
            informerCachesSynced:
              type: boolean
              description: Whether all K8s informer caches have completed initial sync
            killbillReachable:
              type: boolean
              description: Whether the KillBill API responded to a health check
```

---

## Authorization Matrix

Summary of required roles per endpoint:

| Endpoint | Method | Min. Role | Notes |
|----------|--------|-----------|-------|
| `/api/users/me` | GET | viewer | Any authenticated user |
| `/api/users/me/initialize` | POST | viewer | Idempotent, called on every login |
| `/api/tenants` | GET | viewer | Returns only tenants user has access to |
| `/api/tenants` | POST | admin | Creates tenant, triggers provisioning |
| `/api/tenants/{id}` | GET | viewer | Scoped to user's tenants |
| `/api/tenants/{id}` | DELETE | admin | Initiates async deletion |
| `/api/tenants/{id}/retry` | POST | admin | Retry failed provisioning |
| `/api/tenants/{id}/forgejo` | GET | viewer | List Forgejo instances |
| `/api/tenants/{id}/forgejo` | POST | admin | Create Forgejo instance |
| `/api/tenants/{id}/pipelines` | GET | viewer | List pipelines |
| `/api/tenants/{id}/pipelines` | POST | admin | Trigger pipeline |
| `/api/tenants/{id}/pipelines/{pipelineId}` | GET | viewer | Get pipeline status |
| `/api/tenants/{id}/billing/subscription` | GET | member | View billing |
| `/api/tenants/{id}/billing/subscription/change` | POST | admin | Change plan |
| `/api/tenants/{id}/billing/invoices` | GET | member | View invoices |
| `/api/tenants/{id}/events` | GET | viewer | SSE stream |
| `/healthz` | GET | none | No auth required |
| `/readyz` | GET | none | No auth required |

Roles are hierarchical: `admin` implies `member`, which implies `viewer`.

---

## Conventions

### Response Format

All list endpoints return data wrapped in a `data` array with optional `meta` for pagination:

```json
{
  "data": [...],
  "meta": {
    "total": 42,
    "page": 1,
    "pageSize": 25
  }
}
```

Single-resource endpoints return the resource directly (not wrapped).

### Optimistic Concurrency

Resources backed by K8s CRDs support optimistic concurrency via HTTP ETag/If-Match:

- `GET` responses include an `ETag` header containing the K8s `resourceVersion`
- `PUT`/`PATCH` requests must include an `If-Match` header with the ETag value
- If the resource was modified since the ETag, the server returns `412 Precondition Failed`

### Error Format

All errors use the standard `Error` schema:

```json
{
  "error": {
    "code": "TENANT_NOT_FOUND",
    "message": "Tenant 'acme-corp' does not exist or you do not have access",
    "details": { "tenantId": "acme-corp" }
  }
}
```

### SSE Event Format

SSE events follow the [SSE specification](https://html.spec.whatwg.org/multipage/server-sent-events.html):

```
id: {CRDKind}/{namespace}/{name}@{resourceVersion}
event: {created|updated|deleted|status.changed}
retry: 3000
data: {"type":"...","resourceKind":"...","name":"...","status":{...}}

```

The `retry` field (in milliseconds) is sent on initial connection to configure the client's reconnection interval.

### Rate Limits

| Endpoint Category | Limit | Key |
|-------------------|-------|-----|
| REST API (authenticated) | 100 req/min | Per user (`sub` claim) |
| SSE connections | 5 concurrent | Per user (`sub` claim) |
| Health endpoints | 60 req/min | Per IP |

Rate limit responses use `429 Too Many Requests` with `Retry-After` header.

---

## References

- [ADR-002: API Layer Architecture](adr-002-api-layer.md)
- [Auth Flow Design](auth-flow-design.md) -- OIDC flow, JWT validation, tenant resolution
- [K8s Interaction Patterns](k8s-interaction-patterns.md) -- CRD operations, informer setup, event pipeline
- [Sequence Diagrams](sequence-diagrams.md) -- Full interaction flows for registration, provisioning, pipelines, billing
- [oapi-codegen Documentation](https://github.com/oapi-codegen/oapi-codegen) -- Go server/client generation from OpenAPI
- [orval Documentation](https://github.com/orval-labs/orval) -- TypeScript TanStack Query hooks from OpenAPI
- [Refine simple-rest Data Provider](https://refine.dev/docs/data/data-provider/) -- REST conventions expected by Refine
- [KillBill API Documentation](https://docs.killbill.io/) -- Billing API endpoints

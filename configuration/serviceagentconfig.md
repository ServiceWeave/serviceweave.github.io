# ServiceAgentConfig

The `ServiceAgentConfig` custom resource defines per-service configuration for ServiceWeave agents. It is a **namespaced** resource that configures API schema discovery, risk tiers, caching, and per-service LLM overrides.

## Quick Reference

```yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: order-service
  namespace: production
spec:
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
    description: "Order management service for creating, updating, and tracking customer orders"
  targetService:
    name: order-service
    port: 8080
  riskTier: 1
  llmOverrides:
    model: gpt-4o
    baseURL: "https://api.openai.com/v1"
    apiKeySecretRef:
      name: premium-openai-key
      key: api-key
  sessionAffinity:
    enabled: true
    maxTurns: 20
    timeoutSeconds: 600
  cache:
    enabled: true
    intentTTL: 3600
    responseTTL: 60
```

## Spec Fields

### schema

API schema discovery configuration.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `schemaPath` | string | Yes | Path to fetch schema from the service |
| `schemaType` | SchemaType | Yes | Type of schema: `openapi`, `graphql`, or `grpc` |
| `description` | string | Yes | Human-readable service description (10-500 chars) |

**Schema Types:**

| Type | Description | Example Path |
|------|-------------|--------------|
| `openapi` | OpenAPI 3.x specification | `/openapi.json`, `/swagger.json`, `/api-docs` |
| `graphql` | GraphQL schema (introspection) | `/graphql` |
| `grpc` | gRPC reflection | N/A (uses reflection) |

**Example - OpenAPI:**
```yaml
schema:
  schemaPath: "/openapi.json"
  schemaType: openapi
  description: "RESTful API for managing customer orders including creation, updates, and status tracking"
```

**Example - GraphQL:**
```yaml
schema:
  schemaPath: "/graphql"
  schemaType: graphql
  description: "GraphQL API for product catalog queries and mutations"
```

**Example - gRPC:**
```yaml
schema:
  schemaPath: ""  # Uses reflection
  schemaType: grpc
  description: "gRPC service for real-time inventory updates"
```

**Best Practices for Description:**
- Be specific about what the service does
- Include key operations (e.g., "creating", "updating", "querying")
- Mention the domain (e.g., "orders", "payments", "users")
- Keep between 50-200 characters for best results

### targetService

Optional reference to the Kubernetes Service.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the Kubernetes Service |
| `port` | int32 | No | Port to use (defaults to first port) |

**Example:**
```yaml
targetService:
  name: order-service
  port: 8080
```

### riskTier

Risk classification for the service operations.

| Value | Name | Behavior |
|-------|------|----------|
| `0` | Autonomous | Execute immediately without approval |
| `1` | Notify | Execute and notify after completion |
| `2` | Approve | Request approval before execution |
| `3` | Delegate | Route to human operator |

**Default:** `0` (Autonomous)

**Example:**
```yaml
riskTier: 2  # Require approval before execution
```

**Guidelines for Risk Tier Selection:**

| Tier | Use When |
|------|----------|
| 0 | Read-only operations, non-sensitive data |
| 1 | Creates/updates with low impact, internal services |
| 2 | Financial transactions, PII modifications |
| 3 | Deletions, security-sensitive operations |

### llmOverrides

Optional LLM configuration overrides for this service.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | No | Override model name |
| `baseURL` | string | No | Override LLM endpoint |
| `apiKeySecretRef` | SecretKeyRef | No | Override API key secret |

**Note:** Override secrets must be in the same namespace as the ServiceAgentConfig.

**Example - Model Override Only:**
```yaml
llmOverrides:
  model: gpt-4o  # Use more capable model
```

**Example - Full Override:**
```yaml
llmOverrides:
  model: gpt-4o
  baseURL: "https://premium-api.openai.com/v1"
  apiKeySecretRef:
    name: premium-openai-key
    key: api-key
```

### sessionAffinity

Optional session affinity configuration for multi-turn conversations.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `enabled` | bool | No | `false` | Enable session tracking |
| `maxTurns` | int32 | No | `10` | Maximum conversation turns (1-100) |
| `timeoutSeconds` | int32 | No | `300` | Session timeout (60-3600) |

**Example:**
```yaml
sessionAffinity:
  enabled: true
  maxTurns: 20
  timeoutSeconds: 600  # 10 minutes
```

### cache

Optional caching configuration.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `enabled` | bool | No | `true` | Enable caching |
| `intentTTL` | int32 | No | `3600` | Intent-to-tool mapping cache TTL (seconds) |
| `responseTTL` | int32 | No | `60` | API response cache TTL (seconds) |

**Example:**
```yaml
cache:
  enabled: true
  intentTTL: 7200    # Cache intent mappings for 2 hours
  responseTTL: 30    # Cache responses for 30 seconds
```

**Caching Strategy:**

| Data Type | Recommended TTL | Notes |
|-----------|-----------------|-------|
| Intent mappings | 1-24 hours | Stable, rarely changes |
| GET responses | 30-300 seconds | Depends on data freshness needs |
| POST/PUT/DELETE | 0 (no cache) | Mutations should not be cached |

## Status Fields

The ServiceAgentConfig status is automatically updated by the controller.

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Current state conditions |
| `discoveredEndpoints` | int32 | Count of API endpoints discovered |
| `schemaVersion` | string | Hash of the discovered schema |
| `lastSchemaFetch` | Time | Timestamp of last schema fetch |
| `observedGeneration` | int64 | Generation observed by controller |

### Condition Types

| Type | Description |
|------|-------------|
| `Ready` | Overall readiness of the configuration |
| `SchemaDiscovered` | API schema has been successfully parsed |
| `Registered` | Service is registered with the mesh |

**Example Status:**
```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    reason: ConfigurationValid
    message: "ServiceAgentConfig is ready"
    lastTransitionTime: "2024-01-15T10:30:00Z"
  - type: SchemaDiscovered
    status: "True"
    reason: SchemaFetched
    message: "OpenAPI schema discovered with 15 endpoints"
    lastTransitionTime: "2024-01-15T10:30:00Z"
  - type: Registered
    status: "True"
    reason: Registered
    message: "Service registered with vector store"
    lastTransitionTime: "2024-01-15T10:30:00Z"
  discoveredEndpoints: 15
  schemaVersion: "sha256:abc123..."
  lastSchemaFetch: "2024-01-15T10:30:00Z"
  observedGeneration: 2
```

## Complete Examples

### Basic REST Service

```yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: user-service
  namespace: production
spec:
  schema:
    schemaPath: "/api/v1/openapi.json"
    schemaType: openapi
    description: "User management service for registration, authentication, and profile management"
  targetService:
    name: user-service
    port: 8080
```

### High-Risk Financial Service

```yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: payment-service
  namespace: production
spec:
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
    description: "Payment processing service for charges, refunds, and transaction history"
  targetService:
    name: payment-service
    port: 443
  riskTier: 2  # Require approval
  llmOverrides:
    model: gpt-4o  # Use most capable model
  cache:
    enabled: true
    intentTTL: 7200
    responseTTL: 0  # Don't cache payment responses
```

### GraphQL API with Sessions

```yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: product-catalog
  namespace: production
spec:
  schema:
    schemaPath: "/graphql"
    schemaType: graphql
    description: "Product catalog GraphQL API for browsing, searching, and filtering products"
  targetService:
    name: product-catalog
    port: 4000
  sessionAffinity:
    enabled: true
    maxTurns: 50
    timeoutSeconds: 1800  # 30 minutes for shopping sessions
  cache:
    enabled: true
    intentTTL: 3600
    responseTTL: 300  # Cache product data for 5 minutes
```

### gRPC Service

```yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: inventory-service
  namespace: production
spec:
  schema:
    schemaPath: ""  # gRPC uses reflection
    schemaType: grpc
    description: "Real-time inventory gRPC service for stock levels, reservations, and updates"
  targetService:
    name: inventory-service
    port: 50051
  riskTier: 1  # Notify on inventory changes
```

### Multiple Services Example

Deploy multiple ServiceAgentConfigs for a microservices architecture:

```yaml
# orders.yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: order-service
  namespace: ecommerce
spec:
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
    description: "Order service for creating and managing customer orders"
  riskTier: 1
---
# inventory.yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: inventory-service
  namespace: ecommerce
spec:
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
    description: "Inventory service for stock management and availability checks"
  riskTier: 0
---
# payments.yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: payment-service
  namespace: ecommerce
spec:
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
    description: "Payment service for processing transactions and refunds"
  riskTier: 2
```

## Validation Rules

The controller validates the following:

1. **schemaPath**: Must be a valid URL path (starts with `/` for HTTP schemas)
2. **schemaType**: Must be one of `openapi`, `graphql`, `grpc`
3. **description**: Must be 10-500 characters
4. **riskTier**: Must be 0, 1, 2, or 3
5. **maxTurns**: Must be 1-100
6. **timeoutSeconds**: Must be 60-3600
7. **targetService**: Service must exist in the same namespace
8. **apiKeySecretRef**: Secret must exist in the same namespace

## See Also

- [MeshConfig](meshconfig.md) - Global mesh configuration
- [Agentic Mesh](../concepts/agentic-mesh.md) - Understanding risk tiers
- [Architecture](../concepts/architecture.md) - System architecture

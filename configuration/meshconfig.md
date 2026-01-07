# MeshConfig

The `MeshConfig` custom resource defines global configuration for the ServiceWeave mesh. It is a **cluster-scoped** singleton resource that configures LLM providers, embedding models, vector stores, and injection settings.

## Quick Reference

```yaml
apiVersion: serviceweave.ai/v1
kind: MeshConfig
metadata:
  name: serviceweave
spec:
  llm:
    baseURL: "https://api.openai.com/v1"
    defaultModel: "gpt-4o-mini"
    apiKeySecretRef:
      name: openai-credentials
      key: api-key
  embedding:
    baseURL: "https://api.openai.com/v1"
    model: "text-embedding-3-small"
    apiKeySecretRef:
      name: openai-credentials
      key: api-key
  vectorStore:
    type: qdrant
    endpoint: "qdrant.serviceweave-system.svc:6333"
  observability:
    endpoint: "otel-collector.monitoring:4317"
  modelOverrides:
  - selector:
      matchLabels:
        app: critical-service
    model: gpt-4o
  injection:
    namespaceSelector:
      matchLabels:
        serviceweave.ai/inject: "enabled"
    image: ghcr.io/serviceweave/service-agent:latest
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

## Spec Fields

### llm

LLM (Large Language Model) provider configuration.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `baseURL` | string | Yes | OpenAI-compatible API endpoint. Examples: `https://api.openai.com/v1`, `http://ollama.local:11434/v1` |
| `defaultModel` | string | Yes | Default model name. Examples: `gpt-4o-mini`, `gpt-4o`, `llama3.1` |
| `apiKeySecretRef` | SecretKeyRef | Yes | Reference to secret containing the API key |

**Supported Providers:**
- OpenAI
- Azure OpenAI
- Ollama
- vLLM
- LiteLLM
- Any OpenAI-compatible endpoint

**Example - OpenAI:**
```yaml
llm:
  baseURL: "https://api.openai.com/v1"
  defaultModel: "gpt-4o-mini"
  apiKeySecretRef:
    name: openai-credentials
    key: api-key
```

**Example - Azure OpenAI:**
```yaml
llm:
  baseURL: "https://my-resource.openai.azure.com/openai/deployments/my-deployment"
  defaultModel: "gpt-4o"
  apiKeySecretRef:
    name: azure-openai-credentials
    key: api-key
```

**Example - Ollama (local):**
```yaml
llm:
  baseURL: "http://ollama.default.svc:11434/v1"
  defaultModel: "llama3.1"
  apiKeySecretRef:
    name: dummy-secret  # Ollama doesn't require auth
    key: api-key
```

### embedding

Embedding model configuration for semantic search.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `baseURL` | string | Yes | Embedding API endpoint |
| `model` | string | Yes | Embedding model name |
| `apiKeySecretRef` | SecretKeyRef | Yes | Reference to secret containing the API key |

**Example:**
```yaml
embedding:
  baseURL: "https://api.openai.com/v1"
  model: "text-embedding-3-small"
  apiKeySecretRef:
    name: openai-credentials
    key: api-key
```

**Recommended Models:**
| Provider | Model | Dimensions | Notes |
|----------|-------|------------|-------|
| OpenAI | text-embedding-3-small | 1536 | Cost-effective |
| OpenAI | text-embedding-3-large | 3072 | Higher quality |
| Ollama | nomic-embed-text | 768 | Local/free |

### vectorStore

Vector store configuration for semantic search.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Vector store type. Currently only `qdrant` is supported |
| `endpoint` | string | Yes | Vector store endpoint (host:port) |

**Example:**
```yaml
vectorStore:
  type: qdrant
  endpoint: "qdrant.serviceweave-system.svc:6333"
```

### observability

Optional observability configuration for distributed tracing.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `endpoint` | string | No | OpenTelemetry collector endpoint |

**Example:**
```yaml
observability:
  endpoint: "otel-collector.monitoring.svc:4317"
```

### modelOverrides

Optional list of model overrides for specific services based on pod labels.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `selector` | LabelSelector | Yes | Kubernetes label selector |
| `model` | string | Yes | Model to use for matching pods |

**Example:**
```yaml
modelOverrides:
- selector:
    matchLabels:
      app: payment-gateway
  model: gpt-4o  # Use more capable model for payments
- selector:
    matchLabels:
      tier: critical
  model: gpt-4o
```

### injection

Sidecar injection configuration.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `namespaceSelector` | LabelSelector | Yes | - | Selector for namespaces to enable injection |
| `image` | string | No | `ghcr.io/serviceweave/service-agent:latest` | Agent sidecar image |
| `imagePullPolicy` | string | No | `IfNotPresent` | Image pull policy |
| `resources` | ResourceRequirements | No | See below | Sidecar resource limits |

**Default Resources:**
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

**Example:**
```yaml
injection:
  namespaceSelector:
    matchLabels:
      serviceweave.ai/inject: "enabled"
    matchExpressions:
    - key: environment
      operator: In
      values:
      - production
      - staging
  image: ghcr.io/serviceweave/service-agent:v1.2.3
  imagePullPolicy: Always
  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: "1"
      memory: 1Gi
```

## Status Fields

The MeshConfig status is automatically updated by the controller.

| Field | Type | Description |
|-------|------|-------------|
| `conditions` | []Condition | Current state conditions |
| `activeAgents` | int32 | Count of ready ServiceAgentConfigs |
| `injectedNamespaces` | []string | List of namespaces with injection enabled |
| `observedGeneration` | int64 | Generation observed by controller |

### Condition Types

| Type | Description |
|------|-------------|
| `Ready` | Overall readiness of the mesh configuration |
| `LLMConnected` | LLM API key secret exists and is valid |
| `VectorStoreConnected` | Vector store is reachable |

**Example Status:**
```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    reason: AllComponentsReady
    message: "All mesh components are configured correctly"
    lastTransitionTime: "2024-01-15T10:30:00Z"
  - type: LLMConnected
    status: "True"
    reason: SecretFound
    message: "LLM API key secret found"
    lastTransitionTime: "2024-01-15T10:30:00Z"
  - type: VectorStoreConnected
    status: "True"
    reason: Connected
    message: "Vector store is reachable"
    lastTransitionTime: "2024-01-15T10:30:00Z"
  activeAgents: 5
  injectedNamespaces:
  - production
  - staging
  observedGeneration: 3
```

## SecretKeyRef

Reference to a key within a Kubernetes Secret.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Name of the Secret |
| `key` | string | Yes | Key within the Secret |

**Important:** Secrets referenced by MeshConfig must be in the `serviceweave-system` namespace.

## Complete Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: openai-credentials
  namespace: serviceweave-system
type: Opaque
stringData:
  api-key: sk-your-openai-api-key-here
---
apiVersion: serviceweave.ai/v1
kind: MeshConfig
metadata:
  name: serviceweave
spec:
  # LLM Configuration
  llm:
    baseURL: "https://api.openai.com/v1"
    defaultModel: "gpt-4o-mini"
    apiKeySecretRef:
      name: openai-credentials
      key: api-key

  # Embedding Configuration
  embedding:
    baseURL: "https://api.openai.com/v1"
    model: "text-embedding-3-small"
    apiKeySecretRef:
      name: openai-credentials
      key: api-key

  # Vector Store Configuration
  vectorStore:
    type: qdrant
    endpoint: "qdrant.serviceweave-system.svc:6333"

  # Observability (optional)
  observability:
    endpoint: "otel-collector.monitoring.svc:4317"

  # Model Overrides (optional)
  modelOverrides:
  - selector:
      matchLabels:
        app: payment-service
    model: gpt-4o
  - selector:
      matchLabels:
        app: analytics
    model: gpt-4o-mini

  # Injection Settings
  injection:
    namespaceSelector:
      matchLabels:
        serviceweave.ai/inject: "enabled"
    image: ghcr.io/serviceweave/service-agent:v1.0.0
    imagePullPolicy: IfNotPresent
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi
```

## See Also

- [ServiceAgentConfig](serviceagentconfig.md) - Per-service configuration
- [Sidecar Injection](../concepts/sidecar-injection.md) - How injection works
- [Architecture](../concepts/architecture.md) - System architecture

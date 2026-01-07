# ServiceWeave

**ServiceWeave** is an AI-driven Kubernetes operator that implements an **Agentic Mesh** architecture. It enables microservices to be augmented with LLM-powered Service Agent sidecars, allowing services to understand natural language requests and automatically invoke appropriate APIs.

## What is ServiceWeave?

ServiceWeave works similarly to how Istio injects Envoy proxies into your pods, but instead injects intelligent agents capable of semantic understanding. These agents can:

- **Understand natural language requests** and translate them into API calls
- **Discover API schemas** automatically (OpenAPI, GraphQL, gRPC)
- **Cache intent-to-tool mappings** for improved performance
- **Implement risk-based approval workflows** for sensitive operations

## Key Features

| Feature | Description |
|---------|-------------|
| **Transparent Injection** | Automatic sidecar injection via Kubernetes mutating webhooks |
| **Multi-Schema Support** | Works with OpenAPI 3.x, GraphQL, and gRPC services |
| **Flexible LLM Integration** | Compatible with any OpenAI-compatible API provider |
| **Risk Management** | Four-tier risk classification (Autonomous, Notify, Approve, Delegate) |
| **Semantic Caching** | Intent and response caching with configurable TTLs |
| **Observability** | Built-in OpenTelemetry integration for distributed tracing |

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                 ServiceWeave Operator                │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │    │
│  │  │ MeshConfig  │  │ServiceAgent │  │    Pod      │  │    │
│  │  │ Controller  │  │  Config     │  │  Mutator    │  │    │
│  │  │             │  │ Controller  │  │  (Webhook)  │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Your Application Pod                    │    │
│  │  ┌─────────────────┐    ┌─────────────────────────┐ │    │
│  │  │   Your App      │◄──►│  ServiceWeave Agent     │ │    │
│  │  │   Container     │    │  (Injected Sidecar)     │ │    │
│  │  └─────────────────┘    └─────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

1. **MeshConfig** defines global LLM, embedding, and vector store settings
2. **ServiceAgentConfig** configures per-service API schema discovery
3. **Pod Mutator** automatically injects the agent sidecar into labeled namespaces
4. **ServiceWeave Agent** handles natural language requests and API orchestration

## Quick Example

Enable ServiceWeave in a namespace:

```bash
kubectl label namespace my-app serviceweave.ai/inject=enabled
```

Create a global mesh configuration:

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
  injection:
    namespaceSelector:
      matchLabels:
        serviceweave.ai/inject: "enabled"
```

Configure a service for agent discovery:

```yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: order-service
  namespace: my-app
spec:
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
    description: "Order management service for creating, updating, and tracking orders"
  riskTier: 0  # Autonomous execution
```

## Next Steps

- [Quick Start](getting-started/quick-start.md) - Get up and running in minutes
- [Architecture](concepts/architecture.md) - Understand the system design
- [MeshConfig Reference](configuration/meshconfig.md) - Full configuration options

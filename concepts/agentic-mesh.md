# Agentic Mesh

ServiceWeave introduces the concept of an **Agentic Mesh** - a service mesh architecture where intelligent AI agents augment your microservices with natural language understanding and autonomous decision-making capabilities.

## What is an Agentic Mesh?

An Agentic Mesh extends the traditional service mesh pattern (like Istio or Linkerd) by replacing simple proxies with intelligent agents that can:

- **Understand Intent**: Parse natural language requests and determine user intent
- **Discover Capabilities**: Automatically learn what APIs your services expose
- **Make Decisions**: Autonomously select and invoke the right APIs
- **Manage Risk**: Escalate sensitive operations for human approval

### Traditional Service Mesh vs Agentic Mesh

| Aspect | Traditional Service Mesh | Agentic Mesh |
|--------|-------------------------|--------------|
| Sidecar Type | Proxy (Envoy, etc.) | AI Agent |
| Request Handling | Route based on headers/paths | Route based on semantic intent |
| Intelligence | Rule-based | LLM-powered |
| API Discovery | Manual configuration | Automatic schema parsing |
| Decision Making | Static rules | Dynamic reasoning |

## Core Concepts

### 1. Service Agents

Each pod in an enabled namespace receives a **Service Agent** sidecar. This agent:

- Exposes a gRPC interface for natural language queries
- Maintains a connection to the configured LLM provider
- Caches API schemas and intent mappings
- Handles risk assessment and approval workflows

```
┌─────────────────────────────────────────────────────┐
│                    Application Pod                   │
│  ┌─────────────────┐    ┌─────────────────────────┐ │
│  │                 │    │    Service Agent        │ │
│  │   Your App      │    │  ┌───────────────────┐  │ │
│  │   Container     │◄──►│  │ Intent Detection  │  │ │
│  │                 │    │  ├───────────────────┤  │ │
│  │   REST API      │    │  │ Schema Cache      │  │ │
│  │   GraphQL       │    │  ├───────────────────┤  │ │
│  │   gRPC          │    │  │ Risk Assessment   │  │ │
│  │                 │    │  ├───────────────────┤  │ │
│  │                 │    │  │ API Orchestration │  │ │
│  │                 │    │  └───────────────────┘  │ │
│  └─────────────────┘    └─────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 2. Semantic Discovery

Service Agents automatically discover what your services can do by parsing their API schemas:

- **OpenAPI 3.x**: REST API specifications
- **GraphQL**: Introspection queries
- **gRPC**: Service reflection

The discovered schemas are embedded into a vector store (Qdrant) for semantic search. When a natural language request arrives, the agent performs similarity search to find the most relevant API operations.

### 3. Intent-to-Tool Mapping

The Agentic Mesh maintains a mapping between user intents and API operations:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Intent-to-Tool Pipeline                        │
│                                                                   │
│  "Create an order for      ┌─────────────┐    POST /orders       │
│   2 widgets"          ───► │   LLM +     │ ──► {                 │
│                             │   Vector    │     "items": [       │
│                             │   Search    │       {"name":       │
│                             └─────────────┘        "widget",     │
│                                                    "qty": 2}     │
│                                                   ]              │
│                                                  }               │
└──────────────────────────────────────────────────────────────────┘
```

### 4. Risk Tiers

Not all operations are created equal. ServiceWeave implements a four-tier risk classification system:

| Tier | Name | Behavior | Example |
|------|------|----------|---------|
| 0 | Autonomous | Execute immediately | GET requests, read-only operations |
| 1 | Notify | Execute, then notify | Creating non-critical resources |
| 2 | Approve | Request approval before execution | Financial transactions |
| 3 | Delegate | Route to human operator | Account deletion, security changes |

Risk tiers are configured per-service in the ServiceAgentConfig:

```yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: payment-service
spec:
  riskTier: 2  # Require approval for payment operations
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
```

## How It Works

### Request Flow

1. **Receive Request**: Agent receives natural language query via gRPC
2. **Parse Intent**: LLM extracts the user's intent from the query
3. **Search Tools**: Vector similarity search finds relevant API operations
4. **Select Tool**: LLM selects the best matching operation
5. **Extract Parameters**: LLM extracts required parameters from the query
6. **Assess Risk**: Check operation against risk tier configuration
7. **Execute or Escalate**: Either execute the API call or escalate based on risk
8. **Return Response**: Format and return the result to the caller

### Multi-Service Orchestration

The Agentic Mesh can orchestrate requests across multiple services:

```
User: "Show me orders for customer John that are pending"
                    │
                    ▼
         ┌─────────────────┐
         │  Agent Router   │
         └─────────────────┘
                    │
         ┌──────────┼──────────┐
         │          │          │
         ▼          ▼          ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Customer   │ │   Order     │ │  Aggregate  │
│  Service    │ │  Service    │ │  & Return   │
│             │ │             │ │             │
│ GET /search │ │ GET /orders │ │  Results    │
│  ?name=John │ │ ?customer=  │ │             │
│             │ │  123&status │ │             │
│             │ │  =pending   │ │             │
└─────────────┘ └─────────────┘ └─────────────┘
```

## Benefits

### For Developers

- **Reduced Boilerplate**: No need to write intent parsing code
- **Automatic API Exposure**: Services are automatically discoverable
- **Flexible Integration**: Works with existing REST, GraphQL, and gRPC services

### For Operations

- **Centralized Configuration**: MeshConfig controls all agents
- **Observability**: OpenTelemetry integration for tracing
- **Gradual Rollout**: Enable per-namespace with labels

### For Security

- **Risk-Based Controls**: Sensitive operations require approval
- **Audit Trail**: All agent decisions can be logged
- **Least Privilege**: Agents only access configured services

## Use Cases

### 1. Natural Language APIs

Expose your services via natural language:

```
User: "What's the status of order 12345?"
Agent: "Order 12345 is currently 'shipped' and expected to arrive on Jan 15th."
```

### 2. Chatbots and Assistants

Power conversational interfaces:

```
User: "I need to update my shipping address to 123 Main St"
Agent: [Calls PATCH /users/me/address with extracted data]
Agent: "I've updated your shipping address to 123 Main St."
```

### 3. Internal Tools

Build AI-powered internal tools:

```
Support Agent: "Refund the last order for customer support@example.com"
Agent: [Risk Tier 2 - Requires Approval]
Agent: "This operation requires manager approval. Request sent to #approvals."
```

### 4. API Gateway Intelligence

Add semantic routing to your API gateway:

```
Request: "Find all products under $50 with good reviews"
Agent: [Translates to: GET /products?max_price=50&min_rating=4]
```

## Best Practices

### 1. Schema Quality

High-quality API schemas lead to better intent matching:

- Use descriptive operation summaries
- Include parameter descriptions
- Provide example values

### 2. Risk Classification

Carefully consider risk tiers:

- Start with higher tiers and lower as confidence grows
- Group related operations in the same service/tier
- Document what each tier means for your organization

### 3. Caching Strategy

Configure appropriate cache TTLs:

```yaml
spec:
  cache:
    enabled: true
    intentTTL: 3600    # Cache intent mappings for 1 hour
    responseTTL: 60    # Cache API responses for 1 minute
```

### 4. Observability

Enable tracing to understand agent behavior:

```yaml
spec:
  observability:
    endpoint: "otel-collector.monitoring:4317"
```

## Next Steps

- [Sidecar Injection](sidecar-injection.md) - How sidecars are injected
- [Architecture](architecture.md) - System architecture details
- [ServiceAgentConfig](../configuration/serviceagentconfig.md) - Per-service configuration

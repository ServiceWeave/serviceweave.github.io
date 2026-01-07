# Quick Start

This guide will help you get ServiceWeave running in your Kubernetes cluster in under 10 minutes.

## Prerequisites

- Kubernetes cluster (v1.26+)
- `kubectl` configured to access your cluster
- An OpenAI API key (or compatible LLM provider)
- Helm 3.x (optional, for Helm-based installation)

## Step 1: Install ServiceWeave Operator

Apply the ServiceWeave operator manifests:

```bash
kubectl apply -k github.com/serviceweave/serviceweave/config/default
```

Verify the operator is running:

```bash
kubectl get pods -n serviceweave-system
```

Expected output:
```
NAME                                           READY   STATUS    RESTARTS   AGE
serviceweave-controller-manager-xxxxx-xxxxx    1/1     Running   0          30s
```

## Step 2: Create LLM Credentials Secret

Create a secret containing your OpenAI API key:

```bash
kubectl create secret generic openai-credentials \
  -n serviceweave-system \
  --from-literal=api-key=sk-your-openai-api-key
```

## Step 3: Deploy MeshConfig

Create the global mesh configuration:

```yaml
# meshconfig.yaml
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

Apply it:

```bash
kubectl apply -f meshconfig.yaml
```

## Step 4: Enable Injection for a Namespace

Label your application namespace to enable sidecar injection:

```bash
kubectl label namespace default serviceweave.ai/inject=enabled
```

## Step 5: Configure Your Service

Create a ServiceAgentConfig for your application:

```yaml
# serviceagentconfig.yaml
apiVersion: serviceweave.ai/v1
kind: ServiceAgentConfig
metadata:
  name: my-api
  namespace: default
spec:
  schema:
    schemaPath: "/openapi.json"
    schemaType: openapi
    description: "My application API for managing resources"
  targetService:
    name: my-api-service
    port: 8080
```

Apply it:

```bash
kubectl apply -f serviceagentconfig.yaml
```

## Step 6: Deploy Your Application

Deploy (or restart) your application. ServiceWeave will automatically inject the agent sidecar:

```bash
kubectl rollout restart deployment my-api -n default
```

Verify the sidecar is injected:

```bash
kubectl get pods -n default -l app=my-api -o jsonpath='{.items[0].spec.containers[*].name}'
```

Expected output should include `serviceweave-agent`:
```
my-api serviceweave-agent
```

## Verify Installation

Check that all ServiceWeave components are healthy:

```bash
# Check MeshConfig status
kubectl get meshconfig serviceweave -o yaml

# Check ServiceAgentConfig status
kubectl get serviceagentconfig -n default

# View agent logs
kubectl logs -n default deployment/my-api -c serviceweave-agent
```

## Next Steps

- [Installation Guide](installation.md) - Detailed installation options
- [MeshConfig Reference](../configuration/meshconfig.md) - Full configuration options
- [Architecture](../concepts/architecture.md) - Understand how ServiceWeave works

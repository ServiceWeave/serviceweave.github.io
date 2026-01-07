# Prerequisites

Before deploying ServiceWeave, ensure your environment meets the following requirements.

## Kubernetes Cluster

### Version Requirements

ServiceWeave requires Kubernetes 1.26 or later.

```bash
# Check your cluster version
kubectl version --short
```

### Supported Distributions

| Distribution | Minimum Version | Status |
|--------------|-----------------|--------|
| Kubernetes (upstream) | 1.26+ | Fully supported |
| Amazon EKS | 1.26+ | Fully supported |
| Google GKE | 1.26+ | Fully supported |
| Azure AKS | 1.26+ | Fully supported |
| Red Hat OpenShift | 4.13+ | Supported with SCC configuration |
| Rancher RKE2 | 1.26+ | Fully supported |
| k3s | 1.26+ | Fully supported |
| kind | 1.26+ | Development only |
| minikube | 1.26+ | Development only |

### Required Features

The following Kubernetes features must be enabled:

- **Admission Webhooks**: For sidecar injection
- **Custom Resource Definitions**: For MeshConfig and ServiceAgentConfig
- **RBAC**: For access control

Most distributions have these enabled by default.

## External Services

### LLM Provider

ServiceWeave requires access to an LLM provider with an OpenAI-compatible API.

**Supported Providers:**

| Provider | API Endpoint | Notes |
|----------|--------------|-------|
| OpenAI | `https://api.openai.com/v1` | Recommended |
| Azure OpenAI | `https://<resource>.openai.azure.com/...` | Enterprise option |
| Ollama | `http://<host>:11434/v1` | Self-hosted, free |
| vLLM | `http://<host>:8000/v1` | High-performance self-hosted |
| LiteLLM | `http://<host>:8000` | Multi-provider proxy |
| Anthropic (via proxy) | Via LiteLLM or similar | Requires adapter |

**Required Capabilities:**
- Chat completions endpoint (`/chat/completions`)
- JSON mode support (recommended)
- Function calling support (recommended)

### Vector Store

ServiceWeave requires a vector database for semantic search.

**Currently Supported:**

| Vector Store | Version | Endpoint Format |
|--------------|---------|-----------------|
| Qdrant | 1.0+ | `<host>:6333` |

**Quick Qdrant Setup:**

```bash
# Using Docker
docker run -p 6333:6333 -p 6334:6334 qdrant/qdrant

# Using Kubernetes
kubectl apply -f https://raw.githubusercontent.com/qdrant/qdrant-helm/main/examples/simple-deployment.yaml
```

### Embedding Service

An embedding model is required for semantic search.

**Recommended Models:**

| Provider | Model | Dimensions | Cost |
|----------|-------|------------|------|
| OpenAI | text-embedding-3-small | 1536 | $0.02/1M tokens |
| OpenAI | text-embedding-3-large | 3072 | $0.13/1M tokens |
| Ollama | nomic-embed-text | 768 | Free (self-hosted) |
| Ollama | mxbai-embed-large | 1024 | Free (self-hosted) |

## Resource Requirements

### Operator Resources

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-----------|-------------|-----------|----------------|--------------|
| Controller Manager | 100m | 500m | 128Mi | 256Mi |

### Per-Pod Agent Sidecar

| Configuration | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------------|-------------|-----------|----------------|--------------|
| Default | 100m | 500m | 128Mi | 512Mi |
| Minimal | 50m | 200m | 64Mi | 256Mi |
| Production | 200m | 1000m | 256Mi | 1Gi |

### Cluster Capacity

Plan for the following additional resources per pod:

```
Per injected pod:
  CPU: ~100-500m additional
  Memory: ~128-512Mi additional
```

**Example Calculation:**
- 50 pods with sidecars
- Default resources
- Additional cluster capacity needed:
  - CPU: 50 × 500m = 25 cores (limit)
  - Memory: 50 × 512Mi = 25Gi (limit)

## Network Requirements

### Outbound Connectivity

The operator and agents need outbound access to:

| Destination | Port | Purpose |
|-------------|------|---------|
| LLM Provider | 443 | API calls |
| Vector Store | 6333 | Schema storage |
| OpenTelemetry Collector | 4317 | Tracing (optional) |

### Internal Connectivity

| Source | Destination | Port | Purpose |
|--------|-------------|------|---------|
| API Server | Webhook Service | 9443 | Admission webhooks |
| Agents | Target Services | Various | API calls |
| Agents | Vector Store | 6333 | Semantic search |

### Network Policies

If using NetworkPolicies, ensure the following are allowed:

```yaml
# Allow webhook traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-webhook
  namespace: serviceweave-system
spec:
  podSelector:
    matchLabels:
      app: serviceweave-controller-manager
  ingress:
  - from:
    - namespaceSelector: {}  # Allow from all namespaces
    ports:
    - port: 9443
      protocol: TCP
```

## TLS Certificates

### Webhook Certificates

The admission webhook requires TLS certificates. Options:

1. **cert-manager (Required)**

   The operator uses cert-manager to manage TLS certificates for the admission webhook. Install it before deploying ServiceWeave:

   ```bash
   # Install cert-manager
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

   # Wait for cert-manager to be ready
   kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=60s
   kubectl wait --for=condition=Available deployment/cert-manager-webhook -n cert-manager --timeout=60s
   ```

2. **Manual Certificates**
   - Generate a CA and server certificate
   - Create a Secret named `serviceweave-webhook-server-cert`
   - Update MutatingWebhookConfiguration with CA bundle

3. **Kubernetes CA (Development)**
   - Use Kubernetes CSR API
   - Limited certificate validity

## Permissions

### Cluster Admin Access

Installation requires cluster-admin permissions for:

- Creating CRDs
- Creating ClusterRole and ClusterRoleBinding
- Creating MutatingWebhookConfiguration

### Runtime Permissions

The operator ServiceAccount needs:

| Resource | Verbs | Scope |
|----------|-------|-------|
| meshconfigs | * | Cluster |
| serviceagentconfigs | * | Cluster |
| secrets | get, list, watch | Namespaced |
| services | get, list, watch | Namespaced |
| namespaces | get, list, watch | Cluster |
| pods | get, list, watch | Cluster |

## Pre-Flight Checklist

Run this checklist before installation:

```bash
# 1. Check Kubernetes version
kubectl version --short

# 2. Check cluster connectivity
kubectl cluster-info

# 3. Check admin access
kubectl auth can-i create crd

# 4. Check webhook support
kubectl api-resources | grep mutatingwebhookconfigurations

# 5. Test LLM connectivity (from within cluster)
kubectl run test-llm --rm -it --image=curlimages/curl -- \
  curl -s https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"

# 6. Check available resources
kubectl describe nodes | grep -A 5 "Allocatable"
```

## Next Steps

Once prerequisites are met:

- [Quick Start](../getting-started/quick-start.md) - Get started quickly
- [Installation](../getting-started/installation.md) - Detailed installation
- [Production Deployment](production.md) - Production best practices

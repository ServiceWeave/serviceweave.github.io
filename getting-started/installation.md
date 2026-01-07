# Installation

This guide covers detailed installation options for ServiceWeave.

## Requirements

### Kubernetes Version

ServiceWeave requires Kubernetes 1.26 or later. The following distributions are supported:

| Distribution | Minimum Version | Notes |
|--------------|-----------------|-------|
| Kubernetes (upstream) | 1.26+ | Fully supported |
| Amazon EKS | 1.26+ | Fully supported |
| Google GKE | 1.26+ | Fully supported |
| Azure AKS | 1.26+ | Fully supported |
| OpenShift | 4.13+ | Requires additional SCCs |
| Rancher RKE2 | 1.26+ | Fully supported |

### Resource Requirements

**Operator:**
- CPU: 100m request, 500m limit
- Memory: 128Mi request, 256Mi limit

**Per-Pod Agent Sidecar (default):**
- CPU: 100m request, 500m limit
- Memory: 128Mi request, 512Mi limit

### External Dependencies

ServiceWeave requires the following external services:

1. **LLM Provider** - Any OpenAI-compatible API:
   - OpenAI API
   - Azure OpenAI
   - Ollama
   - vLLM
   - LiteLLM
   - Any OpenAI-compatible endpoint

2. **Vector Store** - For semantic search:
   - Qdrant (currently supported)

3. **cert-manager** (required) - For webhook TLS certificates

## Prerequisites

### Install cert-manager

The operator requires cert-manager for webhook TLS certificates. Install it first:

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=Available deployment/cert-manager -n cert-manager --timeout=60s
kubectl wait --for=condition=Available deployment/cert-manager-webhook -n cert-manager --timeout=60s
```

## Installation Methods

### Method 1: Kustomize (Recommended)

Install using Kustomize directly from the repository:

```bash
# Install with default configuration
kubectl apply -k github.com/serviceweave/serviceweave/config/default

# Or clone and customize
git clone https://github.com/serviceweave/serviceweave.git
cd serviceweave
kubectl apply -k config/default
```

### Method 2: Raw Manifests

Apply individual manifest files:

```bash
# Install CRDs
kubectl apply -f config/crd/bases/

# Install RBAC
kubectl apply -f config/rbac/

# Install webhook configuration
kubectl apply -f config/webhook/

# Install operator deployment
kubectl apply -f config/manager/
```

### Method 3: Build from Source

Build and deploy from source:

```bash
# Clone repository
git clone https://github.com/serviceweave/serviceweave.git
cd serviceweave

# Build operator image
make docker-build IMG=your-registry/serviceweave-operator:latest

# Push to registry
make docker-push IMG=your-registry/serviceweave-operator:latest

# Deploy to cluster
make deploy IMG=your-registry/serviceweave-operator:latest
```

## Post-Installation Configuration

### 1. Create LLM Credentials

Create a Kubernetes secret with your LLM API credentials:

```bash
kubectl create secret generic openai-credentials \
  -n serviceweave-system \
  --from-literal=api-key=<your-api-key>
```

For Azure OpenAI:

```bash
kubectl create secret generic azure-openai-credentials \
  -n serviceweave-system \
  --from-literal=api-key=<your-azure-api-key>
```

### 2. Deploy Vector Store

If you don't have Qdrant deployed, you can use the following quick-start deployment:

```yaml
# qdrant.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qdrant
  namespace: serviceweave-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
      - name: qdrant
        image: qdrant/qdrant:latest
        ports:
        - containerPort: 6333
        - containerPort: 6334
        volumeMounts:
        - name: storage
          mountPath: /qdrant/storage
      volumes:
      - name: storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: qdrant
  namespace: serviceweave-system
spec:
  selector:
    app: qdrant
  ports:
  - name: http
    port: 6333
    targetPort: 6333
  - name: grpc
    port: 6334
    targetPort: 6334
```

Apply it:

```bash
kubectl apply -f qdrant.yaml
```

### 3. Configure TLS (Production)

For production deployments, configure cert-manager to manage webhook certificates:

```yaml
# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: serviceweave-webhook-cert
  namespace: serviceweave-system
spec:
  secretName: serviceweave-webhook-server-cert
  dnsNames:
  - serviceweave-webhook-service.serviceweave-system.svc
  - serviceweave-webhook-service.serviceweave-system.svc.cluster.local
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
```

## Verifying Installation

### Check Operator Status

```bash
# Verify operator pod is running
kubectl get pods -n serviceweave-system

# Check operator logs
kubectl logs -n serviceweave-system deployment/serviceweave-controller-manager

# Verify CRDs are installed
kubectl get crd | grep serviceweave
```

Expected CRDs:
```
meshconfigs.serviceweave.ai
serviceagentconfigs.serviceweave.ai
```

### Check Webhook Configuration

```bash
kubectl get mutatingwebhookconfiguration | grep serviceweave
```

### Test Injection

Create a test namespace and verify injection:

```bash
# Create and label namespace
kubectl create namespace serviceweave-test
kubectl label namespace serviceweave-test serviceweave.ai/inject=enabled

# Deploy a test pod
kubectl run test-pod --image=nginx -n serviceweave-test

# Verify sidecar injection
kubectl get pod test-pod -n serviceweave-test -o jsonpath='{.spec.containers[*].name}'
# Should output: nginx serviceweave-agent

# Cleanup
kubectl delete namespace serviceweave-test
```

## Uninstallation

To remove ServiceWeave from your cluster:

```bash
# Remove all ServiceAgentConfigs
kubectl delete serviceagentconfigs --all-namespaces --all

# Remove MeshConfig
kubectl delete meshconfig serviceweave

# Remove operator
kubectl delete -k github.com/serviceweave/serviceweave/config/default

# Remove CRDs (optional - will delete all custom resources)
kubectl delete crd meshconfigs.serviceweave.ai
kubectl delete crd serviceagentconfigs.serviceweave.ai
```

## Troubleshooting

### Operator Not Starting

Check for RBAC issues:
```bash
kubectl describe clusterrolebinding serviceweave-manager-rolebinding
```

### Webhook Not Injecting

1. Verify namespace label:
   ```bash
   kubectl get namespace <namespace> --show-labels
   ```

2. Check webhook logs:
   ```bash
   kubectl logs -n serviceweave-system deployment/serviceweave-controller-manager
   ```

3. Verify MeshConfig exists:
   ```bash
   kubectl get meshconfig serviceweave
   ```

### Certificate Issues

If using cert-manager, check certificate status:
```bash
kubectl get certificate -n serviceweave-system
kubectl describe certificate serviceweave-webhook-cert -n serviceweave-system
```

## Next Steps

- [Quick Start](quick-start.md) - Get started with a simple example
- [MeshConfig Reference](../configuration/meshconfig.md) - Configure global settings
- [Production Deployment](../deployment/production.md) - Production best practices

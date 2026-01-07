# Sidecar Injection

ServiceWeave uses Kubernetes mutating admission webhooks to automatically inject Service Agent sidecars into pods. This page explains how the injection mechanism works.

## How Injection Works

### Admission Webhook Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Pod Creation Request                               │
│                                                                           │
│  kubectl apply -f pod.yaml                                               │
│         │                                                                 │
│         ▼                                                                 │
│  ┌─────────────────────┐                                                 │
│  │  Kubernetes API     │                                                 │
│  │  Server             │                                                 │
│  └─────────────────────┘                                                 │
│         │                                                                 │
│         │ 1. AdmissionReview Request                                     │
│         ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │              ServiceWeave Pod Mutator Webhook                        │ │
│  │                                                                      │ │
│  │  ┌────────────────┐    ┌────────────────┐    ┌────────────────────┐ │ │
│  │  │ 2. Check       │───►│ 3. Check       │───►│ 4. Inject          │ │ │
│  │  │ Namespace      │    │ Pod            │    │ Sidecar            │ │ │
│  │  │ Labels         │    │ Annotations    │    │                    │ │ │
│  │  └────────────────┘    └────────────────┘    └────────────────────┘ │ │
│  │         │                     │                      │              │ │
│  │         ▼                     ▼                      ▼              │ │
│  │  serviceweave.ai/     serviceweave.ai/       - Add container      │ │
│  │  inject: enabled      inject: false?         - Add volume         │ │
│  │  label present?       (skip if true)         - Add env vars       │ │
│  │                                              - Add annotations    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│         │                                                                 │
│         │ 5. AdmissionReview Response (with patch)                       │
│         ▼                                                                 │
│  ┌─────────────────────┐                                                 │
│  │  Pod Created with   │                                                 │
│  │  Injected Sidecar   │                                                 │
│  └─────────────────────┘                                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

### Injection Decision Logic

The webhook follows this decision tree:

```
Pod Creation Request
        │
        ▼
┌───────────────────────┐
│ Already has           │     YES
│ inject-status:        │──────────► Skip (already injected)
│ injected annotation?  │
└───────────────────────┘
        │ NO
        ▼
┌───────────────────────┐
│ Has annotation        │     YES
│ serviceweave.ai/      │──────────► Skip (explicitly disabled)
│ inject: false?        │
└───────────────────────┘
        │ NO
        ▼
┌───────────────────────┐
│ MeshConfig exists?    │     NO
│                       │──────────► Skip (no global config)
└───────────────────────┘
        │ YES
        ▼
┌───────────────────────┐
│ Namespace matches     │     NO
│ injection selector?   │──────────► Skip (namespace not enabled)
└───────────────────────┘
        │ YES
        ▼
    INJECT SIDECAR
```

## Enabling Injection

### Namespace-Level

Enable injection for an entire namespace:

```bash
kubectl label namespace my-app serviceweave.ai/inject=enabled
```

This works with the default MeshConfig namespace selector:

```yaml
spec:
  injection:
    namespaceSelector:
      matchLabels:
        serviceweave.ai/inject: "enabled"
```

### Custom Selectors

You can customize the namespace selector in MeshConfig:

```yaml
spec:
  injection:
    namespaceSelector:
      matchLabels:
        environment: production
        team: platform
```

### Disabling for Specific Pods

Disable injection for a specific pod using annotations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  annotations:
    serviceweave.ai/inject: "false"
spec:
  containers:
  - name: my-app
    image: my-app:latest
```

## What Gets Injected

### Sidecar Container

The webhook injects a container named `serviceweave-agent`:

```yaml
containers:
- name: serviceweave-agent
  image: ghcr.io/serviceweave/service-agent:latest
  ports:
  - name: grpc
    containerPort: 9090
    protocol: TCP
  - name: health
    containerPort: 9091
    protocol: TCP
  env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
  - name: LLM_BASE_URL
    value: "https://api.openai.com/v1"
  - name: LLM_MODEL
    value: "gpt-4o-mini"
  - name: LLM_API_KEY
    valueFrom:
      secretKeyRef:
        name: openai-credentials
        key: api-key
  # ... additional env vars
  readinessProbe:
    httpGet:
      path: /ready
      port: 9091
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /health
      port: 9091
    initialDelaySeconds: 10
    periodSeconds: 30
  securityContext:
    runAsNonRoot: true
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop:
      - ALL
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  volumeMounts:
  - name: serviceweave-shared
    mountPath: /var/run/serviceweave
```

### Shared Volume

A shared emptyDir volume is added for inter-container communication:

```yaml
volumes:
- name: serviceweave-shared
  emptyDir:
    medium: Memory
```

### Annotations

The webhook adds annotations to track injection:

```yaml
metadata:
  annotations:
    serviceweave.ai/inject-status: "injected"
    serviceweave.ai/agent-image: "ghcr.io/serviceweave/service-agent:latest"
```

## Configuration Options

### Agent Image

Configure the agent image in MeshConfig:

```yaml
spec:
  injection:
    image: ghcr.io/serviceweave/service-agent:v1.2.3
    imagePullPolicy: IfNotPresent
```

### Resource Limits

Configure sidecar resource limits:

```yaml
spec:
  injection:
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: "1"
        memory: 1Gi
```

### Model Overrides

Override the LLM model for specific services:

```yaml
spec:
  modelOverrides:
  - selector:
      matchLabels:
        app: payment-gateway
    model: gpt-4o  # Use more capable model
```

## Environment Variables

The following environment variables are injected into the sidecar:

| Variable | Source | Description |
|----------|--------|-------------|
| `POD_NAME` | Field ref | Name of the pod |
| `POD_NAMESPACE` | Field ref | Namespace of the pod |
| `POD_IP` | Field ref | IP address of the pod |
| `LLM_BASE_URL` | MeshConfig | LLM provider base URL |
| `LLM_MODEL` | MeshConfig | Default LLM model |
| `LLM_API_KEY` | Secret ref | LLM API key |
| `EMBEDDING_BASE_URL` | MeshConfig | Embedding API base URL |
| `EMBEDDING_MODEL` | MeshConfig | Embedding model name |
| `EMBEDDING_API_KEY` | Secret ref | Embedding API key |
| `VECTOR_STORE_TYPE` | MeshConfig | Vector store type (qdrant) |
| `VECTOR_STORE_ENDPOINT` | MeshConfig | Vector store endpoint |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | MeshConfig | OpenTelemetry endpoint (optional) |

## Troubleshooting

### Pod Not Getting Sidecar

1. **Check namespace label:**
   ```bash
   kubectl get namespace my-app --show-labels
   ```
   Ensure `serviceweave.ai/inject=enabled` is present.

2. **Check MeshConfig exists:**
   ```bash
   kubectl get meshconfig serviceweave
   ```

3. **Check pod annotations:**
   ```bash
   kubectl get pod my-pod -o jsonpath='{.metadata.annotations}'
   ```
   Ensure `serviceweave.ai/inject: false` is not set.

4. **Check webhook logs:**
   ```bash
   kubectl logs -n serviceweave-system deployment/serviceweave-controller-manager
   ```

### Sidecar Failing to Start

1. **Check sidecar logs:**
   ```bash
   kubectl logs my-pod -c serviceweave-agent
   ```

2. **Verify secrets exist:**
   ```bash
   kubectl get secret openai-credentials -n serviceweave-system
   ```

3. **Check readiness probe:**
   ```bash
   kubectl describe pod my-pod
   ```

### Webhook Not Working

1. **Check webhook configuration:**
   ```bash
   kubectl get mutatingwebhookconfiguration serviceweave-mutating-webhook-configuration
   ```

2. **Check webhook service:**
   ```bash
   kubectl get svc -n serviceweave-system serviceweave-webhook-service
   ```

3. **Check certificate:**
   ```bash
   kubectl get secret -n serviceweave-system serviceweave-webhook-server-cert
   ```

## Best Practices

### 1. Use Namespace Isolation

Enable injection only for namespaces that need it:

```bash
# Only enable for specific namespaces
kubectl label namespace api-services serviceweave.ai/inject=enabled
kubectl label namespace web-frontend serviceweave.ai/inject=enabled
```

### 2. Test in Non-Production First

Enable injection in development/staging before production:

```yaml
spec:
  injection:
    namespaceSelector:
      matchLabels:
        serviceweave.ai/inject: "enabled"
        environment: staging  # Additional selector
```

### 3. Monitor Resource Usage

Start with default resources and adjust based on metrics:

```bash
# Check sidecar resource usage
kubectl top pod -n my-app --containers
```

### 4. Use Pod Disruption Budgets

Ensure availability during rollouts:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: my-app
```

## Next Steps

- [Architecture](architecture.md) - Overall system architecture
- [MeshConfig Reference](../configuration/meshconfig.md) - Global configuration
- [Production Deployment](../deployment/production.md) - Production best practices

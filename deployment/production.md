# Production Deployment

This guide covers best practices for deploying ServiceWeave in production environments.

## High Availability

### Operator HA

For production, deploy the operator with leader election for high availability:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: serviceweave-controller-manager
  namespace: serviceweave-system
spec:
  replicas: 2  # Multiple replicas for HA
  selector:
    matchLabels:
      app: serviceweave-controller-manager
  template:
    spec:
      containers:
      - name: manager
        args:
        - --leader-elect=true  # Enable leader election
        - --leader-election-id=serviceweave-controller
```

### Vector Store HA

Deploy Qdrant in cluster mode for high availability:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: serviceweave-system
spec:
  replicas: 3
  serviceName: qdrant
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
        env:
        - name: QDRANT__CLUSTER__ENABLED
          value: "true"
        ports:
        - containerPort: 6333
        - containerPort: 6334
        - containerPort: 6335  # Internal cluster port
        volumeMounts:
        - name: storage
          mountPath: /qdrant/storage
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
```

## Security Hardening

### Secret Management

Use external secret management for production:

**AWS Secrets Manager with External Secrets Operator:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: openai-credentials
  namespace: serviceweave-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: openai-credentials
  data:
  - secretKey: api-key
    remoteRef:
      key: serviceweave/openai-api-key
```

**HashiCorp Vault:**

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: openai-credentials
  namespace: serviceweave-system
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: openai-credentials
  data:
  - secretKey: api-key
    remoteRef:
      key: secret/data/serviceweave
      property: openai-api-key
```

### Network Policies

Implement strict network policies:

```yaml
# Operator network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: serviceweave-operator
  namespace: serviceweave-system
spec:
  podSelector:
    matchLabels:
      app: serviceweave-controller-manager
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow webhook traffic from API server
  - from:
    - namespaceSelector: {}
    ports:
    - port: 9443
      protocol: TCP
  # Allow metrics scraping
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 8080
      protocol: TCP
  egress:
  # Allow Kubernetes API
  - to:
    - namespaceSelector: {}
    ports:
    - port: 443
      protocol: TCP
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
---
# Agent sidecar network policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: serviceweave-agent
  namespace: production
spec:
  podSelector:
    matchLabels:
      serviceweave.ai/inject-status: injected
  policyTypes:
  - Egress
  egress:
  # Allow LLM provider
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
    ports:
    - port: 443
      protocol: TCP
  # Allow vector store
  - to:
    - namespaceSelector:
        matchLabels:
          name: serviceweave-system
    ports:
    - port: 6333
      protocol: TCP
```

### Pod Security Standards

Apply pod security standards:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: serviceweave-system
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### RBAC Best Practices

Use namespace-scoped roles where possible:

```yaml
# Per-namespace role for ServiceAgentConfig management
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: serviceweave-config-admin
  namespace: production
rules:
- apiGroups: ["serviceweave.ai"]
  resources: ["serviceagentconfigs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: serviceweave-config-admin
  namespace: production
subjects:
- kind: Group
  name: platform-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: serviceweave-config-admin
  apiGroup: rbac.authorization.k8s.io
```

## Resource Management

### Production Resource Limits

Configure appropriate resource limits for production:

```yaml
apiVersion: serviceweave.ai/v1
kind: MeshConfig
metadata:
  name: serviceweave
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

### Pod Disruption Budgets

Ensure availability during maintenance:

```yaml
# Operator PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: serviceweave-operator-pdb
  namespace: serviceweave-system
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: serviceweave-controller-manager
---
# Vector store PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: qdrant-pdb
  namespace: serviceweave-system
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: qdrant
```

### Resource Quotas

Set resource quotas for the ServiceWeave namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: serviceweave-quota
  namespace: serviceweave-system
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

## Observability

### Prometheus Metrics

Configure Prometheus to scrape ServiceWeave metrics:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: serviceweave-operator
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: serviceweave-controller-manager
  namespaceSelector:
    matchNames:
    - serviceweave-system
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Key Metrics to Monitor

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| `controller_runtime_reconcile_total` | Total reconciliations | N/A |
| `controller_runtime_reconcile_errors_total` | Reconciliation errors | > 0 |
| `workqueue_depth` | Work queue depth | > 100 |
| `workqueue_longest_running_processor_seconds` | Longest running reconcile | > 60s |

### Alerting Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: serviceweave-alerts
  namespace: monitoring
spec:
  groups:
  - name: serviceweave
    rules:
    - alert: ServiceWeaveOperatorDown
      expr: absent(up{job="serviceweave-operator"}) == 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: ServiceWeave operator is down

    - alert: ServiceWeaveReconcileErrors
      expr: rate(controller_runtime_reconcile_errors_total[5m]) > 0
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: ServiceWeave reconciliation errors detected

    - alert: ServiceWeaveWebhookLatencyHigh
      expr: histogram_quantile(0.99, rate(admission_webhook_latency_seconds_bucket[5m])) > 1
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: ServiceWeave webhook latency is high
```

### Distributed Tracing

Configure OpenTelemetry for distributed tracing:

```yaml
apiVersion: serviceweave.ai/v1
kind: MeshConfig
metadata:
  name: serviceweave
spec:
  observability:
    endpoint: "otel-collector.monitoring.svc:4317"
```

**OpenTelemetry Collector Configuration:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
  namespace: monitoring
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    processors:
      batch:
        timeout: 10s
    exporters:
      jaeger:
        endpoint: jaeger-collector:14250
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [jaeger]
```

## Backup and Recovery

### MeshConfig Backup

Backup the MeshConfig resource:

```bash
# Backup MeshConfig
kubectl get meshconfig serviceweave -o yaml > meshconfig-backup.yaml

# Backup all ServiceAgentConfigs
kubectl get serviceagentconfigs --all-namespaces -o yaml > serviceagentconfigs-backup.yaml
```

### Vector Store Backup

For Qdrant, configure snapshots:

```bash
# Create snapshot
curl -X POST "http://qdrant:6333/collections/serviceweave/snapshots"

# List snapshots
curl "http://qdrant:6333/collections/serviceweave/snapshots"
```

### Disaster Recovery

Document your disaster recovery procedure:

1. **Restore CRDs** (if cluster is new)
2. **Restore operator deployment**
3. **Restore secrets**
4. **Restore MeshConfig**
5. **Restore vector store data**
6. **Restore ServiceAgentConfigs**
7. **Restart application pods** to trigger re-injection

## Upgrade Strategy

### Rolling Upgrades

Use rolling updates for the operator:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

### Upgrade Procedure

1. **Backup current state**
   ```bash
   kubectl get meshconfig,serviceagentconfigs -A -o yaml > backup.yaml
   ```

2. **Update CRDs** (if changed)
   ```bash
   kubectl apply -f config/crd/bases/
   ```

3. **Update operator**
   ```bash
   kubectl set image deployment/serviceweave-controller-manager \
     manager=ghcr.io/serviceweave/operator:v1.2.0 \
     -n serviceweave-system
   ```

4. **Update agent image** (triggers pod restarts)
   ```bash
   kubectl patch meshconfig serviceweave --type merge -p '
     {"spec":{"injection":{"image":"ghcr.io/serviceweave/service-agent:v1.2.0"}}}'
   ```

5. **Rolling restart applications**
   ```bash
   kubectl rollout restart deployment -n production
   ```

### Rollback Procedure

```bash
# Rollback operator
kubectl rollout undo deployment/serviceweave-controller-manager -n serviceweave-system

# Rollback agent image
kubectl patch meshconfig serviceweave --type merge -p '
  {"spec":{"injection":{"image":"ghcr.io/serviceweave/service-agent:v1.1.0"}}}'

# Restart applications
kubectl rollout restart deployment -n production
```

## Production Checklist

Before going to production, verify:

- [ ] **High Availability**: Operator replicas > 1, leader election enabled
- [ ] **Security**: Network policies, RBAC, pod security standards applied
- [ ] **Secrets**: External secret management configured
- [ ] **Resources**: Appropriate limits set, PDBs configured
- [ ] **Monitoring**: Prometheus metrics, alerts configured
- [ ] **Tracing**: OpenTelemetry configured
- [ ] **Backup**: Backup procedure documented and tested
- [ ] **Upgrade**: Upgrade procedure documented
- [ ] **Documentation**: Runbooks created for common operations

## Next Steps

- [Prerequisites](prerequisites.md) - Environment requirements
- [Installation](../getting-started/installation.md) - Installation guide
- [MeshConfig Reference](../configuration/meshconfig.md) - Configuration options

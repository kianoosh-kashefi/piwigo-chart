# Piwigo Helm Chart

A comprehensive third-party Helm chart for deploying Piwigo photo gallery application on Kubernetes with full persistence support and extensibility.

## Features

- **Piwigo Docker Image**: Uses `linuxserver/piwigo` Docker image
- **Persistent Storage**: Separate persistent volumes for configuration and gallery data
- **Flexible Ingress**: Configurable ingress with TLS/cert-manager support
- **Health Checks**: Liveness and readiness probes for robust operations
- **Resource Management**: Configurable CPU and memory requests/limits
- **Service Account**: Optional service account creation with RBAC support
- **Extra Objects**: Ability to create additional Kubernetes resources (ConfigMaps, Secrets, NetworkPolicies, etc.) via values file
- **Security Context**: Support for pod and container security contexts
- **Node Affinity**: Support for node selectors, tolerations, and affinity rules

## Installation

### Basic Installation

```bash
helm repo add piwigo /path/to/chart
helm install my-piwigo piwigo/piwigo
```

### Using Custom Values

```bash
helm install my-piwigo piwigo/piwigo -f custom-values.yaml
```

## Values Configuration

### Core Piwigo Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| `piwigo.replicas` | `1` | Number of Piwigo deployment replicas |
| `piwigo.image.repository` | `linuxserver/piwigo` | Docker image repository |
| `piwigo.image.tag` | `latest` | Docker image tag |
| `piwigo.image.pullPolicy` | `IfNotPresent` | Image pull policy |
| `piwigo.service.type` | `ClusterIP` | Service type (ClusterIP, NodePort, LoadBalancer) |
| `piwigo.service.port` | `80` | Service port |

### Persistence Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| `persistence.enabled` | `true` | Enable persistent storage |
| `persistence.config.enabled` | `true` | Enable config volume |
| `persistence.config.size` | `500Mi` | Config volume size |
| `persistence.config.storageClass` | `""` | Storage class name (empty uses default) |
| `persistence.config.mountPath` | `/var/www/html` | Config mount path in container |
| `persistence.config.existingClaim` | `""` | Use existing PVC instead of creating new |
| `persistence.gallery.enabled` | `true` | Enable gallery volume |
| `persistence.gallery.size` | `5Gi` | Gallery volume size |
| `persistence.gallery.storageClass` | `""` | Storage class name |
| `persistence.gallery.mountPath` | `/var/www/html/galleries` | Gallery mount path in container |
| `persistence.gallery.existingClaim` | `""` | Use existing PVC instead of creating new |

### Ingress Settings

| Parameter | Default | Description |
|-----------|---------|-------------|
| `ingress.enabled` | `true` | Enable ingress |
| `ingress.className` | `traefik` | Ingress class name |
| `ingress.host` | `piwigo.example.com` | Ingress hostname |
| `ingress.tls.enabled` | `true` | Enable TLS |
| `ingress.tls.secretName` | `piwigo-tls` | TLS secret name |
| `ingress.annotations` | `{cert-manager.io/cluster-issuer: letsencrypt-prod}` | Ingress annotations |

### Resource Management

| Parameter | Default | Description |
|-----------|---------|-------------|
| `piwigo.resources.requests.cpu` | `100m` | CPU request |
| `piwigo.resources.requests.memory` | `256Mi` | Memory request |
| `piwigo.resources.limits.cpu` | `1000m` | CPU limit |
| `piwigo.resources.limits.memory` | `1Gi` | Memory limit |

### Probes Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `piwigo.livenessProbe.initialDelaySeconds` | `30` | Initial delay for liveness probe |
| `piwigo.readinessProbe.initialDelaySeconds` | `5` | Initial delay for readiness probe |

## Using Extra Objects

The chart supports creating additional Kubernetes objects by specifying them in the `extraObjects` array in values.yaml. This allows you to:

- Create ConfigMaps for application configuration
- Create Secrets for sensitive data
- Deploy NetworkPolicies for network segmentation
- Define PodDisruptionBudgets for high availability
- Deploy any other Kubernetes resource

### Example: Adding a ConfigMap

```yaml
extraObjects:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: piwigo-custom-config
    data:
      custom-setting: "value"
      another-setting: "another-value"
```

### Example: Adding a Secret

```yaml
extraObjects:
  - apiVersion: v1
    kind: Secret
    metadata:
      name: piwigo-db-secret
    type: Opaque
    data:
      # Note: values must be base64 encoded
      username: dXNlcm5hbWU=  # "username"
      password: cGFzc3dvcmQ=  # "password"
```

### Example: Adding a NetworkPolicy

```yaml
extraObjects:
  - apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: piwigo-network-policy
    spec:
      podSelector:
        matchLabels:
          app.kubernetes.io/name: piwigo
      policyTypes:
        - Ingress
      ingress:
        - from:
            - podSelector:
                matchLabels:
                  role: frontend
```

### Example: Adding a PodDisruptionBudget

```yaml
extraObjects:
  - apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: piwigo-pdb
    spec:
      maxUnavailable: 1
      selector:
        matchLabels:
          app.kubernetes.io/name: piwigo
```

### Example: Using Templating in Extra Objects

Extra objects support Helm templating, allowing you to reference chart values:

```yaml
extraObjects:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: {{ include "piwigo.fullname" . }}-config
      namespace: {{ .Release.Namespace }}
    data:
      app-name: "{{ .Chart.Name }}"
      version: "{{ .Chart.Version }}"
```

## Advanced Configurations

### Using Existing PersistentVolumeClaims

If you have existing PVCs, you can reuse them:

```yaml
persistence:
  config:
    existingClaim: my-existing-config-pvc
  gallery:
    existingClaim: my-existing-gallery-pvc
```

### Custom Storage Class

```yaml
persistence:
  config:
    storageClass: fast-ssd
  gallery:
    storageClass: slow-hdd
```

### Disabling Persistence

```yaml
persistence:
  enabled: false
```

### Using NodePort Service

```yaml
piwigo:
  service:
    type: NodePort
    port: 80
    targetPort: 80
```

### Adding Environment Variables

```yaml
piwigo:
  env:
    - name: TZ
      value: "Europe/London"
    - name: LOG_LEVEL
      value: "debug"
```

### Pod Security Context

```yaml
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  capabilities:
    drop:
      - ALL
```

### Node Affinity and Tolerations

```yaml
piwigo:
  nodeSelector:
    node-type: compute
  
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-type
                operator: In
                values:
                  - compute
  
  tolerations:
    - key: dedicated
      operator: Equal
      value: piwigo
      effect: NoSchedule
```

## Troubleshooting

### Check Deployment Status

```bash
kubectl get deployment -l app.kubernetes.io/name=piwigo
kubectl describe pod -l app.kubernetes.io/name=piwigo
kubectl logs -l app.kubernetes.io/name=piwigo
```

### Verify PersistentVolumeClaims

```bash
kubectl get pvc -l app.kubernetes.io/name=piwigo
kubectl describe pvc <pvc-name>
```

### Check Ingress

```bash
kubectl get ingress -l app.kubernetes.io/name=piwigo
kubectl describe ingress <ingress-name>
```

### View Events

```bash
kubectl get events --sort-by='.lastTimestamp'
```

## Upgrading

To upgrade an existing Piwigo deployment:

```bash
helm upgrade my-piwigo piwigo/piwigo -f custom-values.yaml
```

## Uninstalling

To remove Piwigo deployment:

```bash
helm uninstall my-piwigo
```

**Note**: Persistent volumes and claims are not automatically deleted when the chart is uninstalled. To clean up:

```bash
kubectl delete pvc -l app.kubernetes.io/name=piwigo
```

## Development

### Validating the Chart

```bash
helm lint piwigo/
```

### Dry Run

```bash
helm install my-piwigo piwigo/piwigo --dry-run --debug
```

### Generating Manifests

```bash
helm template my-piwigo piwigo/piwigo -f values.yaml
```

## License

See LICENSE file


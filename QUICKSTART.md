# Piwigo Helm Chart - Quick Start Guide

## Overview

This Helm chart deploys Piwigo, a free and open source photo gallery/management application, on Kubernetes with:
- ✅ Official Piwigo Docker image
- ✅ Persistent storage for configuration and gallery data
- ✅ Configurable ingress with TLS support
- ✅ Health checks and resource management
- ✅ Support for additional Kubernetes objects via `extraObjects`

## Directory Structure

```
helm/piwigo/
├── Chart.yaml                 # Chart metadata
├── values.yaml               # Default configuration values
├── README.md                 # Comprehensive documentation
├── EXAMPLES.md              # Practical deployment examples
├── QUICKSTART.md            # This file
├── .helmignore              # Files to ignore
└── templates/
    ├── _helpers.tpl         # Template helpers
    ├── deployment.yaml      # Piwigo deployment
    ├── piwigo.yaml         # Service definition
    ├── serviceaccount.yaml  # Service account
    ├── ingress.yaml        # Ingress configuration
    ├── config.yaml         # Config PersistentVolumeClaim
    ├── gallery.yaml        # Gallery PersistentVolumeClaim
    └── extra-objects.yaml  # Extra Kubernetes objects
```

## Quick Start (5 minutes)

### 1. Install with defaults
```bash
cd /home/kia/Git
helm install piwigo helm/piwigo \
  --namespace piwigo \
  --create-namespace
```

### 2. Check deployment status
```bash
kubectl get pods -n piwigo
kubectl get svc -n piwigo
kubectl get ingress -n piwigo
```

### 3. Get service IP
```bash
kubectl get svc -n piwigo piwigo
```

### 4. Port forward (if not using ingress)
```bash
kubectl port-forward -n piwigo svc/piwigo 8080:80
# Access at http://localhost:8080
```

## Common Configuration Scenarios

### Scenario 1: Change Host and Storage Size

Create `custom-values.yaml`:
```yaml
ingress:
  host: photos.yourdomain.com

persistence:
  config:
    size: 1Gi
  gallery:
    size: 100Gi
```

Deploy:
```bash
helm install piwigo helm/piwigo -f custom-values.yaml -n piwigo --create-namespace
```

### Scenario 2: Add ConfigMap via Extra Objects

Create `values-with-config.yaml`:
```yaml
extraObjects:
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: piwigo-custom-settings
    data:
      setting1: "value1"
      setting2: "value2"
```

Deploy:
```bash
helm install piwigo helm/piwigo -f values-with-config.yaml -n piwigo --create-namespace
```

### Scenario 3: High Availability with 3 Replicas

Create `values-ha.yaml`:
```yaml
piwigo:
  replicas: 3
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi

extraObjects:
  - apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: piwigo-pdb
    spec:
      minAvailable: 2
      selector:
        matchLabels:
          app.kubernetes.io/name: piwigo
```

Deploy:
```bash
helm install piwigo helm/piwigo -f values-ha.yaml -n piwigo --create-namespace
```

## Key Features Explained

### Persistence
Two separate storage volumes:
- **Config**: Stores Piwigo configuration, plugins, themes (500Mi default)
- **Gallery**: Stores photos and media files (5Gi default)

```yaml
persistence:
  config:
    enabled: true
    size: 500Mi
    storageClass: ""  # Uses default if empty
  gallery:
    enabled: true
    size: 5Gi
```

### Extra Objects
Create any Kubernetes resource (ConfigMap, Secret, NetworkPolicy, etc.) through the values file:

```yaml
extraObjects:
  - apiVersion: v1
    kind: Secret
    metadata:
      name: my-secret
    type: Opaque
    data:
      key: dGVzdA==  # base64 encoded
```

### Health Checks
Automatic liveness and readiness probes:

```yaml
piwigo:
  livenessProbe:
    httpGet:
      path: /
      port: http
    initialDelaySeconds: 30
    periodSeconds: 10
  readinessProbe:
    httpGet:
      path: /
      port: http
    initialDelaySeconds: 5
    periodSeconds: 5
```

### Resource Management
CPU and memory requests/limits:

```yaml
piwigo:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi
```

## Upgrade

Update your deployment with new configuration:
```bash
helm upgrade piwigo helm/piwigo -f custom-values.yaml -n piwigo
```

## Uninstall

Remove the deployment:
```bash
helm uninstall piwigo -n piwigo
```

Clean up PersistentVolumeClaims (optional):
```bash
kubectl delete pvc -n piwigo -l app.kubernetes.io/name=piwigo
```

## Validation

Before installing, validate the chart:
```bash
helm lint helm/piwigo/
```

Dry run to see what will be installed:
```bash
helm install piwigo helm/piwigo --dry-run --debug -n piwigo
```

Generate manifests to review:
```bash
helm template piwigo helm/piwigo > manifests.yaml
```

## Troubleshooting

### Pod not starting?
```bash
kubectl describe pod -n piwigo -l app.kubernetes.io/name=piwigo
kubectl logs -n piwigo -l app.kubernetes.io/name=piwigo
```

### Storage not mounting?
```bash
kubectl get pvc -n piwigo
kubectl describe pvc -n piwigo piwigo-gallery
```

### Ingress not working?
```bash
kubectl get ingress -n piwigo
kubectl describe ingress -n piwigo piwigo
```

## Documentation

- **README.md**: Full feature list and parameter reference
- **EXAMPLES.md**: Real-world deployment examples
- **Chart.yaml**: Chart metadata and versioning
- **values.yaml**: Complete configuration reference with comments

## Support & Next Steps

1. For detailed documentation, see [README.md](README.md)
2. For deployment examples, see [EXAMPLES.md](EXAMPLES.md)
3. For full values reference, see [values.yaml](values.yaml)

## What's New in This Chart

✨ **Key Improvements**:
- Uses official `piwigo/piwigo` Docker image
- Full persistence support with flexible storage options
- **New**: `extraObjects` feature to create additional Kubernetes resources
- Comprehensive health checks (liveness & readiness probes)
- Configurable resource limits
- Service account and RBAC support
- Security context options
- Pod affinity and node selector support
- Detailed documentation and examples

## Next Steps

1. Customize `values.yaml` for your environment
2. Review [EXAMPLES.md](EXAMPLES.md) for your use case
3. Use `helm install` or `helm upgrade` to deploy
4. Access Piwigo through configured ingress or port-forward

Happy photo managing! 📸

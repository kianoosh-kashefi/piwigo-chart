# Piwigo Helm Chart - Examples

This file contains practical examples of how to use the Piwigo Helm chart with various configurations and extra objects.

## Example 1: Basic Installation with Custom Host

```yaml
# values-basic.yaml
piwigo:
  replicas: 1
  image:
    repository: piwigo/piwigo
    tag: "latest"

ingress:
  enabled: true
  host: "photos.mycompany.com"
  tls:
    enabled: true
    secretName: "photos-tls"

persistence:
  config:
    size: 1Gi
  gallery:
    size: 50Gi
```

Deploy with:
```bash
helm install piwigo ./piwigo -f values-basic.yaml
```

---

## Example 2: High Availability Setup with PodDisruptionBudget

```yaml
# values-ha.yaml
piwigo:
  replicas: 3
  strategy:
    type: RollingUpdate
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 500m
      memory: 1Gi
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app.kubernetes.io/name
                  operator: In
                  values:
                    - piwigo
            topologyKey: kubernetes.io/hostname

persistence:
  config:
    storageClass: "fast-ssd"
    size: 2Gi
  gallery:
    storageClass: "standard"
    size: 100Gi

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

---

## Example 3: Piwigo with Database Configuration

```yaml
# values-with-db.yaml
piwigo:
  replicas: 1
  env:
    - name: MYSQL_HOST
      value: "mariadb.default.svc.cluster.local"
    - name: MYSQL_DB
      value: "piwigo"
    - name: MYSQL_USER
      value: "piwigo"
    - name: TZ
      value: "UTC"

persistence:
  config:
    size: 500Mi
  gallery:
    size: 50Gi

ingress:
  enabled: true
  host: "piwigo.example.com"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

extraObjects:
  # Secret for database credentials
  - apiVersion: v1
    kind: Secret
    metadata:
      name: piwigo-db-credentials
    type: Opaque
    stringData:
      password: "your-secure-password-here"
  
  # ConfigMap for Piwigo local settings
  - apiVersion: v1
    kind: ConfigMap
    metadata:
      name: piwigo-local-config
    data:
      config.php: |
        <?php
        define('SHOW_QUERIES', false);
        define('PWG_CHARSET', 'utf-8');
        define('PWG_ALLOW_SYNCHRONIZATION', true);
        ?>
```

---

## Example 4: Network Policy and RBAC Configuration

```yaml
# values-secure.yaml
piwigo:
  replicas: 1

serviceAccount:
  create: true
  name: piwigo
  annotations:
    {}

podSecurityContext:
  runAsNonRoot: true
  runAsUser: 33  # www-data
  fsGroup: 33

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: false
  capabilities:
    drop:
      - ALL

persistence:
  config:
    size: 500Mi
  gallery:
    size: 50Gi

extraObjects:
  # NetworkPolicy to restrict traffic
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
        - Egress
      ingress:
        - from:
            - namespaceSelector:
                matchLabels:
                  name: ingress-nginx
          ports:
            - protocol: TCP
              port: 80
      egress:
        # Allow DNS
        - to:
            - namespaceSelector: {}
          ports:
            - protocol: UDP
              port: 53
        # Allow internet access
        - to:
            - ipBlock:
                cidr: 0.0.0.0/0
                except:
                  - 169.254.169.254/32  # Block metadata service
          ports:
            - protocol: TCP
              port: 80
            - protocol: TCP
              port: 443
```

---

## Example 5: Using Existing PersistentVolumeClaims

```yaml
# values-existing-pvc.yaml
piwigo:
  replicas: 1

persistence:
  config:
    enabled: true
    # Use existing PVC instead of creating new
    existingClaim: "my-existing-config-pvc"
  gallery:
    enabled: true
    existingClaim: "my-existing-gallery-pvc"

ingress:
  enabled: true
  host: "piwigo.example.com"
```

First, create the PVCs:
```bash
kubectl apply -f existing-pvc-manifest.yaml
helm install piwigo ./piwigo -f values-existing-pvc.yaml
```

---

## Example 6: Development Environment with Resource Minimization

```yaml
# values-dev.yaml
piwigo:
  replicas: 1
  image:
    tag: "latest"
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
  livenessProbe:
    initialDelaySeconds: 60
    periodSeconds: 20
  readinessProbe:
    initialDelaySeconds: 10
    periodSeconds: 10

persistence:
  config:
    size: 200Mi
  gallery:
    size: 5Gi

ingress:
  enabled: true
  host: "piwigo-dev.example.com"
  annotations: {}  # No cert-manager for dev
  tls:
    enabled: false
```

---

## Example 7: Multiple Environments Using Extra Objects

```yaml
# values-production.yaml
piwigo:
  replicas: 2
  image:
    repository: piwigo/piwigo
    tag: "latest"
  resources:
    requests:
      cpu: 250m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 2Gi

persistence:
  config:
    size: 2Gi
    storageClass: "fast-ssd"
  gallery:
    size: 100Gi
    storageClass: "standard"

ingress:
  enabled: true
  host: "photos.company.com"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/limit-rps: "10"

extraObjects:
  # ServiceMonitor for Prometheus (if using Prometheus Operator)
  - apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: piwigo
      labels:
        release: prometheus
    spec:
      selector:
        matchLabels:
          app.kubernetes.io/name: piwigo
      endpoints:
        - port: http
          interval: 30s

  # HorizontalPodAutoscaler
  - apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
      name: piwigo
    spec:
      scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: "{{ include \"piwigo.fullname\" . }}"
      minReplicas: 2
      maxReplicas: 5
      metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 70
        - type: Resource
          resource:
            name: memory
            target:
              type: Utilization
              averageUtilization: 80

  # PodDisruptionBudget
  - apiVersion: policy/v1
    kind: PodDisruptionBudget
    metadata:
      name: piwigo
    spec:
      minAvailable: 1
      selector:
        matchLabels:
          app.kubernetes.io/name: piwigo
```

---

## Example 8: Backup Configuration with Extra Objects

```yaml
# values-with-backup.yaml
piwigo:
  replicas: 1

persistence:
  config:
    size: 1Gi
  gallery:
    size: 50Gi

extraObjects:
  # Backup Job
  - apiVersion: batch/v1
    kind: CronJob
    metadata:
      name: piwigo-backup
    spec:
      schedule: "0 2 * * *"  # Daily at 2 AM
      jobTemplate:
        spec:
          template:
            spec:
              containers:
                - name: backup
                  image: busybox:latest
                  command:
                    - /bin/sh
                    - -c
                    - tar czf /backup/piwigo-$(date +\%Y\%m\%d-\%H\%M\%S).tar.gz -C /gallery .
                  volumeMounts:
                    - name: gallery
                      mountPath: /gallery
                    - name: backup
                      mountPath: /backup
              volumes:
                - name: gallery
                  persistentVolumeClaim:
                    claimName: "{{ include \"piwigo.fullname\" . }}-gallery"
                - name: backup
                  persistentVolumeClaim:
                    claimName: piwigo-backup
              restartPolicy: OnFailure
```

---

## Deployment Commands

### Example 1 - Basic
```bash
helm install piwigo ./helm/piwigo -f examples/values-basic.yaml --namespace piwigo --create-namespace
```

### Example 4 - Secure
```bash
helm install piwigo ./helm/piwigo -f examples/values-secure.yaml --namespace piwigo --create-namespace
```

### Example 7 - Production
```bash
helm install piwigo ./helm/piwigo -f examples/values-production.yaml --namespace piwigo-prod --create-namespace
```

### Upgrade
```bash
helm upgrade piwigo ./helm/piwigo -f values-updated.yaml
```

### Delete
```bash
helm uninstall piwigo -n piwigo
kubectl delete pvc -n piwigo -l app.kubernetes.io/name=piwigo
```

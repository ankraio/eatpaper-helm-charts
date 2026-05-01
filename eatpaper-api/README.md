# eatpaper-api

Helm chart for the Eatpaper REST API server.

| | |
|---|---|
| **Chart Version** | 0.1.0 |
| **App Version** | 1.0.0 |
| **Type** | application |

## Prerequisites

- Kubernetes 1.23+
- Helm 3.x
- NGINX Ingress Controller (if Ingress is enabled)
- cert-manager with a ClusterIssuer (if TLS is enabled)
- The following secrets must exist in the target namespace **before** installation:

| Secret | Keys | Purpose |
|--------|------|---------|
| `eatpaper-database-secret` | `username`, `password` | PostgreSQL credentials |
| `eatpaper-smtp-secret` | *(provider-specific)* | SMTP mail configuration |

The chart references these secrets by name and does not create them.

## Quick Start

```bash
helm install eatpaper-api ./deploy/charts/eatpaper-api \
  --set ingress.hostname=eatpaper.example.com \
  --set image.tag=1.0.0
```

## Kubernetes Resources

| Resource | Condition |
|----------|-----------|
| Deployment | Always |
| Service | Always |
| ConfigMap | Always |
| ServiceAccount | `serviceAccount.create: true` (default) |
| Ingress | `ingress.enabled: true` (default) |
| HorizontalPodAutoscaler | `autoscaling.enabled: true` |
| PersistentVolumeClaim | `storage.backend: local` and `storage.persistence.enabled: true` |
| Secret (S3 credentials) | `storage.backend: s3` and `storage.s3.existingSecret` is empty |

## Configuration

### Application

| Parameter | Description | Default |
|-----------|-------------|---------|
| `mode` | Application runtime mode | `"api"` |
| `log_level` | Log verbosity | `"INFO"` |
| `replicaCount` | Pod replicas (ignored when HPA is enabled) | `1` |
| `terminationGracePeriodSeconds` | Graceful shutdown timeout | `10` |
| `cors_allowed_origins` | CORS allowed origins | `""` |
| `currency_api_url` | External currency conversion API URL | `""` |

### Image

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Container image registry/repo | `registry.ankra.cloud/eatpaper/eatpaper-backend` |
| `image.tag` | Image tag (falls back to `appVersion`) | `""` |
| `image.pullPolicy` | Pull policy | `Always` |
| `image.pullSecrets` | Registry pull secrets, e.g. `[{name: my-secret}]` | `[]` |

### Database

| Parameter | Description | Default |
|-----------|-------------|---------|
| `database.name` | Database name | `"eatpaper"` |
| `database.port` | Database port | `5432` |
| `database.secretName` | Secret containing `username` and `password` | `"eatpaper-database-secret"` |
| `database.readWriteEndpoint` | Read-write DB host | `"eatpaper-rw"` |
| `database.readEndpoint` | Read-only DB host | `"eatpaper-r"` |

### SMTP

| Parameter | Description | Default |
|-----------|-------------|---------|
| `smtp.secretName` | Existing Secret with SMTP configuration | `"eatpaper-smtp-secret"` |

### Storage

| Parameter | Description | Default |
|-----------|-------------|---------|
| `storage.backend` | Storage backend type (`local` or `s3`) | `"local"` |
| `storage.localPath` | Mount path for receipt files (local backend) | `"/data/receipts"` |
| `storage.persistence.enabled` | Use a PVC for local storage | `true` |
| `storage.persistence.storageClass` | Storage class for the PVC | `""` |
| `storage.persistence.size` | PVC size | `5Gi` |
| `storage.persistence.accessModes` | PVC access modes | `[ReadWriteMany]` |
| `storage.persistence.existingClaim` | Use an existing PVC instead of creating one | `""` |
| `storage.s3.bucket` | S3 bucket name | `""` |
| `storage.s3.endpoint` | S3 endpoint URL | `""` |
| `storage.s3.region` | S3 region | `"us-east-1"` |
| `storage.s3.existingSecret` | Existing Secret with S3 credentials | `""` |
| `storage.s3.accessKey` | S3 access key (ignored if existingSecret is set) | `""` |
| `storage.s3.secretKey` | S3 secret key (ignored if existingSecret is set) | `""` |

When `backend: local` with `persistence.enabled: true`, a PersistentVolumeClaim is created. Set `persistence.enabled: false` for ephemeral `emptyDir` storage (dev only). When `backend: s3`, no volume is mounted and S3 credentials are injected from a Secret.

### Auth0

| Parameter | Description | Default |
|-----------|-------------|---------|
| `auth0.domain` | Auth0 tenant domain | `""` |
| `auth0.api_audience` | Auth0 API audience identifier | `""` |
| `auth0.mgmtSecretName` | Existing Secret with `AUTH0_MGMT_CLIENT_ID` and `AUTH0_MGMT_CLIENT_SECRET` keys (required for the delete-account feature) | `""` |

### Service

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `service.port` | Service / container port | `8000` |
| `service.name` | Port name | `http` |

### Ingress

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ingress.enabled` | Create an Ingress resource | `true` |
| `ingress.hostname` | Ingress host | `""` |
| `ingress.path` | API path prefix | `/api/` |
| `ingress.ingressClassName` | Ingress class | `nginx` |
| `ingress.annotations` | Additional Ingress annotations | `{nginx.ingress.kubernetes.io/proxy-body-size: "12m"}` |
| `ingress.tls` | Enable TLS (requires `hostname`) | `true` |

When `hostname` and `tls` are both set, the chart creates a TLS block with a secret named `<hostname>-tls`. Use a cert-manager ClusterIssuer annotation to auto-provision the certificate.

### Autoscaling

| Parameter | Description | Default |
|-----------|-------------|---------|
| `autoscaling.enabled` | Enable HPA | `false` |
| `autoscaling.minReplicas` | Minimum replicas | `1` |
| `autoscaling.maxReplicas` | Maximum replicas | `10` |
| `autoscaling.targetCPUUtilizationPercentage` | CPU target for scaling | `80` |

### Service Account

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceAccount.create` | Create a ServiceAccount | `true` |
| `serviceAccount.name` | ServiceAccount name | `"eatpaper-api-sa"` |
| `serviceAccount.annotations` | ServiceAccount annotations | `{}` |

### Resources

| Parameter | Description | Default |
|-----------|-------------|---------|
| `resources.limits.memory` | Memory limit | `512Mi` |
| `resources.requests.cpu` | CPU request | `100m` |
| `resources.requests.memory` | Memory request | `256Mi` |

### Security

| Parameter | Description | Default |
|-----------|-------------|---------|
| `podSecurityContext.runAsNonRoot` | Reject containers running as root | `true` |
| `podSecurityContext.fsGroup` | Filesystem group for mounted volumes | `1000` |
| `securityContext.allowPrivilegeEscalation` | Prevent privilege escalation | `false` |
| `securityContext.runAsUser` | Container UID | `1000` |
| `securityContext.capabilities.drop` | Dropped Linux capabilities | `["ALL"]` |

### Pod Scheduling

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nodeSelector` | Node selector labels | `{}` |
| `tolerations` | Pod tolerations | `[]` |
| `affinity` | Pod affinity rules | `{}` |
| `podAnnotations` | Additional pod annotations | `{}` |

## Health Checks

The Deployment configures three probes against `/health`:

- **Startup**: period 5s, 6 failures allowed (up to 30s for initial startup)
- **Liveness**: period 10s, 3 failures allowed
- **Readiness**: period 5s, 3 failures allowed

ConfigMap changes automatically trigger a rolling restart via a `checksum/config` pod annotation.

## Related Charts

- [eatpaper-scheduler](../eatpaper-scheduler) — background job scheduler using the same container image in `scheduler` mode

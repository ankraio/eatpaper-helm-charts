# eatpaper-scheduler

Helm chart for the Eatpaper background job scheduler. Deploys a Deployment running the application in `scheduler` mode. It has no Service, Ingress, or HPA — it is a background worker only.

| | |
|---|---|
| **Chart Version** | 0.1.0 |
| **Type** | application |

## Prerequisites

- Kubernetes 1.23+
- Helm 3.x
- The following secrets must exist in the target namespace **before** installation:

| Secret | Keys | Purpose |
|--------|------|---------|
| `eatpaper-database-secret` | `username`, `password` | PostgreSQL credentials |
| `eatpaper-smtp-secret` | *(provider-specific)* | SMTP mail configuration |

The chart references these secrets by name and does not create them.

## Quick Start

```bash
helm install eatpaper-scheduler ./deploy/charts/eatpaper-scheduler \
  --set image.tag=1.0.0
```

## Kubernetes Resources

| Resource | Condition |
|----------|-----------|
| Deployment | Always |
| ServiceAccount | `serviceAccount.create: true` (default) |
| PersistentVolumeClaim | `storage.backend: local` and `storage.persistence.enabled: true` |
| Secret (S3 credentials) | `storage.backend: s3` and `storage.s3.existingSecret` is empty |

## Configuration

### Application

| Parameter | Description | Default |
|-----------|-------------|---------|
| `mode` | Application runtime mode | `scheduler` |
| `replicaCount` | Pod replicas (should normally be 1) | `1` |
| `terminationGracePeriodSeconds` | Graceful shutdown timeout | `30` |

### Image

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Container image registry/repo | `registry.ankra.cloud/eatpaper/eatpaper-backend` |
| `image.tag` | Image tag (falls back to `appVersion`) | `""` |
| `image.pullPolicy` | Pull policy | `IfNotPresent` |
| `image.pullSecrets` | Registry pull secrets, e.g. `[{name: my-secret}]` | `[]` |

### Database

| Parameter | Description | Default |
|-----------|-------------|---------|
| `database.name` | Database name | `eatpaper` |
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
| `storage.backend` | Storage backend type (`local` or `s3`) | `local` |
| `storage.localPath` | Mount path for receipt files (local backend) | `/data/receipts` |
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

### Service Account

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceAccount.create` | Create a ServiceAccount | `true` |
| `serviceAccount.name` | ServiceAccount name | `"eatpaper-scheduler-sa"` |
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

The scheduler is a background worker with no HTTP endpoint. Probes use an exec command (`pgrep -f python`) to verify the process is alive:

- **Startup**: period 5s, 6 failures allowed (up to 30s for initial startup)
- **Liveness**: period 30s, 3 failures allowed

## Deployment Strategy

The chart uses `Recreate` strategy to ensure only one scheduler instance runs at a time, avoiding duplicate job execution.

## Related Charts

- [eatpaper-api](../eatpaper-api) — REST API server using the same container image in `api` mode

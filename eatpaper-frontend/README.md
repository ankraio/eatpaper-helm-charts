# eatpaper-frontend

Helm chart for the Eatpaper web frontend. Deploys an Nginx-based container serving the static frontend assets with runtime Auth0 configuration injected via ConfigMap.

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

## Quick Start

```bash
helm install eatpaper-frontend ./deploy/charts/eatpaper-frontend \
  --set ingress.hostname=eatpaper.example.com \
  --set auth0.domain=your-tenant.auth0.com \
  --set auth0.clientId=your-client-id \
  --set auth0.audience=your-api-audience \
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

## Configuration

### Application

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of pod replicas | `1` |
| `terminationGracePeriodSeconds` | Graceful shutdown timeout | `10` |

### Image

| Parameter | Description | Default |
|-----------|-------------|---------|
| `image.repository` | Container image registry/repo | `registry.ankra.cloud/eatpaper/eatpaper-frontend` |
| `image.tag` | Image tag (falls back to `appVersion`) | `""` |
| `image.pullPolicy` | Pull policy | `IfNotPresent` |
| `image.pullSecrets` | Registry pull secrets, e.g. `[{name: my-secret}]` | `[]` |

### Auth0

| Parameter | Description | Default |
|-----------|-------------|---------|
| `auth0.domain` | Auth0 tenant domain | `""` |
| `auth0.clientId` | Auth0 client ID | `""` |
| `auth0.audience` | Auth0 API audience | `""` |

These values are injected into `config.js` at runtime via a ConfigMap mount. ConfigMap changes automatically trigger a rolling restart via a `checksum/config` pod annotation.

### Service

| Parameter | Description | Default |
|-----------|-------------|---------|
| `service.type` | Kubernetes Service type | `ClusterIP` |
| `service.port` | Service port | `80` |
| `service.name` | Port name | `http` |

### Ingress

| Parameter | Description | Default |
|-----------|-------------|---------|
| `ingress.enabled` | Create an Ingress resource | `true` |
| `ingress.hostname` | Ingress host | `""` |
| `ingress.path` | URL path prefix | `/` |
| `ingress.ingressClassName` | Ingress class | `nginx` |
| `ingress.annotations` | Additional Ingress annotations | `{}` |
| `ingress.tls` | Enable TLS (requires `hostname`) | `true` |

When `hostname` and `tls` are both set, the chart creates a TLS block with a secret named `<hostname>-tls`. Use a cert-manager ClusterIssuer annotation to auto-provision the certificate.

### Service Account

| Parameter | Description | Default |
|-----------|-------------|---------|
| `serviceAccount.create` | Create a ServiceAccount | `true` |
| `serviceAccount.name` | ServiceAccount name | `"eatpaper-frontend-sa"` |
| `serviceAccount.annotations` | ServiceAccount annotations | `{}` |

### Resources

| Parameter | Description | Default |
|-----------|-------------|---------|
| `resources.limits.memory` | Memory limit | `128Mi` |
| `resources.requests.cpu` | CPU request | `50m` |
| `resources.requests.memory` | Memory request | `64Mi` |

### Security

| Parameter | Description | Default |
|-----------|-------------|---------|
| `podSecurityContext.runAsNonRoot` | Reject containers running as root | `true` |
| `podSecurityContext.fsGroup` | Filesystem group for mounted volumes | `101` |
| `securityContext.allowPrivilegeEscalation` | Prevent privilege escalation | `false` |
| `securityContext.runAsUser` | Container UID | `101` |
| `securityContext.capabilities.drop` | Dropped Linux capabilities | `["ALL"]` |

> **Note:** UID 101 is the `nginx` user. The container image must support running as non-root (e.g. `nginxinc/nginx-unprivileged`). Override these values if your image uses a different UID.

### Pod Scheduling

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nodeSelector` | Node selector labels | `{}` |
| `tolerations` | Pod tolerations | `[]` |
| `affinity` | Pod affinity rules | `{}` |
| `podAnnotations` | Additional pod annotations | `{}` |

## Health Checks

The Deployment configures three probes against `GET /` on the `http` port:

- **Startup**: period 5s, 6 failures allowed (up to 30s for initial startup)
- **Liveness**: period 10s, 3 failures allowed
- **Readiness**: period 5s, 3 failures allowed

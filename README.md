# linkr-helm

Helm Charts for [Linkr](https://github.com/VictorMadu/linkr) — a URL shortener platform deployed on Kubernetes with AWS-native secret management.

## Architecture

```
shortener-api  ──► RabbitMQ ──► analytics-worker ──► MongoDB
      │                                                   │
      │ (PostgreSQL + Redis)                              ▼
      │                                             stats-api
      │
  platform (ClusterSecretStore → AWS Secrets Manager)
```

| Chart | Role |
|---|---|
| `platform` | Cluster-level bootstrap: ClusterSecretStore backed by AWS Secrets Manager |
| `shortener-api` | Core URL shortening REST API (PostgreSQL + Redis + RabbitMQ publisher) |
| `analytics-worker` | Background worker consuming click events from RabbitMQ and writing to MongoDB |
| `stats-api` | Read-only API for link analytics queries against MongoDB |

All service charts use the [External Secrets Operator](https://external-secrets.io/) to pull credentials from AWS Secrets Manager and expose them as native Kubernetes Secrets.

## Prerequisites

- Kubernetes 1.25+
- Helm 3.10+
- [External Secrets Operator](https://external-secrets.io/latest/introduction/getting-started/) installed in the cluster
- IRSA configured: a service account in the `external-secrets` namespace with permissions to read from AWS Secrets Manager (region `ca-central-1`)
- RabbitMQ reachable at `amqp://rabbitmq:5672/` from within the cluster

## Charts

### platform

Installs a `ClusterSecretStore` that authenticates to AWS Secrets Manager via IRSA (JWT). **Must be deployed before any service chart.**

| Value | Default | Description |
|---|---|---|
| `aws.region` | `ca-central-1` | AWS region for Secrets Manager |
| `clusterSecretStore.enabled` | `true` | Create the ClusterSecretStore |
| `clusterSecretStore.name` | `aws-secrets-manager` | Name referenced by service ExternalSecrets |
| `clusterSecretStore.serviceAccountName` | `external-secrets` | IRSA service account name |
| `clusterSecretStore.serviceAccountNamespace` | `external-secrets` | Namespace of the service account |

### shortener-api

Handles URL creation and redirect. Reads from PostgreSQL and Redis; publishes click events to RabbitMQ.

Secrets pulled from `linkr/{env}/shortener-api` in AWS Secrets Manager: `DATABASE_URL`, `REDIS_URL`.

| Value | Default (prod) | Description |
|---|---|---|
| `image.repository` | `…/shortener-api` | ECR image repository |
| `image.tag` | `latest` | Image tag |
| `replicaCount` | `2` | Replicas (ignored when HPA is enabled) |
| `service.targetPort` | `8080` | Container port |
| `hpa.enabled` | `true` | Enable HorizontalPodAutoscaler |
| `hpa.minReplicas` | `2` | Minimum replicas |
| `hpa.maxReplicas` | `5` | Maximum replicas |
| `hpa.targetCPUUtilizationPercentage` | `70` | CPU target for scaling |
| `pdb.enabled` | `true` | Enable PodDisruptionBudget |
| `pdb.minAvailable` | `1` | Minimum pods available during disruption |
| `config.CACHE_TTL` | `24h` | Redirect cache TTL |
| `config.AMQP_URL` | `amqp://rabbitmq:5672/` | RabbitMQ connection URL |
| `externalSecret.secretsManagerKey` | `linkr/prod/shortener-api` | Secrets Manager key path |

### analytics-worker

Consumes click events from RabbitMQ and persists them to MongoDB.

Secrets pulled from `linkr/{env}/analytics-worker`: `MONGO_URI`, `AMQP_URL`.

| Value | Default (prod) | Description |
|---|---|---|
| `image.repository` | `…/analytics-worker` | ECR image repository |
| `replicaCount` | `2` | Replicas |
| `service.targetPort` | `8081` | Health check port |
| `config.AMQP_PREFETCH` | `10` | RabbitMQ prefetch count |
| `config.MONGO_DB` | `analytics` | Target MongoDB database |
| `config.SHUTDOWN_TIMEOUT` | `15s` | Graceful shutdown timeout |
| `hpa.enabled` | `true` | Enable HorizontalPodAutoscaler |
| `pdb.enabled` | `true` | Enable PodDisruptionBudget |
| `externalSecret.secretsManagerKey` | `linkr/prod/analytics-worker` | Secrets Manager key path |

### stats-api

Read-only API serving aggregated link analytics from MongoDB.

Secrets pulled from `linkr/{env}/stats-api`: `MONGO_URI`.

| Value | Default (prod) | Description |
|---|---|---|
| `image.repository` | `…/stats-api` | ECR image repository |
| `replicaCount` | `2` | Replicas |
| `service.targetPort` | `8083` | Container port |
| `config.MONGO_DB` | `analytics` | MongoDB database |
| `config.MONGO_COLLECTION` | `click_events` | MongoDB collection |
| `config.STATS_WINDOW_DAYS` | `30` | Rolling window for stats queries |
| `config.TOP_REFERRERS_LIMIT` | `10` | Max referrers returned per query |
| `hpa.enabled` | `true` | Enable HorizontalPodAutoscaler |
| `pdb.enabled` | `true` | Enable PodDisruptionBudget |
| `externalSecret.secretsManagerKey` | `linkr/prod/stats-api` | Secrets Manager key path |

## Installation

### 1. Bootstrap the platform

```bash
helm install platform ./charts/platform
```

### 2. Deploy the services

**Production:**

```bash
helm install shortener-api  ./charts/shortener-api
helm install analytics-worker ./charts/analytics-worker
helm install stats-api       ./charts/stats-api
```

**Development** (uses `values-dev.yaml` overrides — single replica, no HPA/PDB, lower resources):

```bash
helm install shortener-api  ./charts/shortener-api  -f ./charts/shortener-api/values-dev.yaml
helm install analytics-worker ./charts/analytics-worker -f ./charts/analytics-worker/values-dev.yaml
helm install stats-api       ./charts/stats-api       -f ./charts/stats-api/values-dev.yaml
```

### Upgrading

```bash
helm upgrade <release> ./charts/<chart> [-f values-dev.yaml]
```

### Uninstalling

```bash
helm uninstall shortener-api analytics-worker stats-api
helm uninstall platform   # remove last, after services are gone
```

## Environment differences (dev vs prod)

| Setting | prod | dev |
|---|---|---|
| Replicas | 2 | 1 |
| HPA | enabled | disabled |
| PDB | enabled | disabled |
| CPU request | 100m | 50m |
| Memory request | 128Mi | 64Mi |
| Image pull policy | `IfNotPresent` | `Always` |
| Secrets Manager path | `linkr/prod/…` | `linkr/dev/…` |
| `shortener-api` CACHE_TTL | `24h` | `5m` |
| `analytics-worker` AMQP_PREFETCH | `10` | `5` |
| `stats-api` STATS_WINDOW_DAYS | `30` | `7` |

# Mercury GitOps v2

Kubernetes manifests for multi-tenant AKS, managed by FluxCD. This repository serves as the single source of truth for all cluster resources across staging and production environments.

## Stack

| Component | Tool | Purpose |
|-----------|------|---------|
| GitOps | FluxCD | Automated sync every 60s |
| Ingress | Gateway API (Traefik) | Traffic routing + TLS termination |
| TLS | Cert-Manager + Let's Encrypt | Automatic certificate provisioning |
| Database | CNPG + Barman Cloud | PostgreSQL HA with automated backups |
| Monitoring | Prometheus + Grafana | Metrics, dashboards, Telegram alerts |
| Secrets | Azure Key Vault CSI Driver | Zero secrets in Git |
| Network | Cilium CNI + Network Policies | Tenant isolation |

## Repository Structure

```
mercury-gitops-v2/
├── infrastructure/
│   ├── controllers/              # What to INSTALL (Helm releases)
│   │   ├── base/
│   │   │   ├── gateway-api/      # Kubernetes Gateway API CRDs
│   │   │   ├── traefik/          # Traefik as Gateway API provider
│   │   │   ├── cert-manager/     # TLS certificate automation
│   │   │   └── cnpg/             # CloudNative PostgreSQL operator
│   │   ├── staging/
│   │   └── production/
│   └── configs/                  # How to CONFIGURE (post-install)
│       ├── base/
│       │   ├── cert-manager/     # ClusterIssuers (Let's Encrypt)
│       │   └── gateway/          # Gateway routes
│       ├── staging/
│       └── production/
│
├── cnpg-plugin/                  # Barman Cloud backup plugin
│   ├── base/                     # HelmRelease v0.3.1
│   ├── staging/
│   └── production/
│
├── monitoring/
│   ├── controllers/              # Prometheus + Grafana Operator
│   │   ├── base/
│   │   │   ├── prometheus/       # kube-prometheus-stack
│   │   │   └── grafana-operator/ # Grafana CRD management
│   │   ├── staging/
│   │   └── production/
│   └── configs/                  # Dashboards, datasources, alert rules
│       ├── base/
│       ├── staging/
│       └── production/
│
└── apps/                         # Customer workloads
    ├── base/                     # Shared policies
    │   ├── network-policy.yaml   # Tenant isolation
    │   ├── resource-quota.yaml   # CPU/Memory limits
    │   ├── rbac.yaml             # Access control
    │   └── pdb.yaml              # Pod Disruption Budgets
    ├── staging/
    │   └── cicero/               # 11 manifests per customer
    └── production/
        └── cicero/
```

## FluxCD Kustomization Order

Dependencies are synced in this order to ensure controllers are ready before workloads deploy:

```
1. controllers     Traefik, Cert-Manager, CNPG Operator
       ↓
2. configs         Gateway routes, ClusterIssuers
       ↓
3. cnpg-plugin     Barman Cloud (backup sidecar)
       ↓
4. monitoring      Prometheus, Grafana, alert rules
       ↓
5. apps            Customer namespaces + workloads
```

## Base / Overlay Pattern

Every component follows the Kustomize base/overlay pattern:

- **`base/`** contains shared configuration (Helm charts, common settings)
- **`staging/`** references base with staging-specific patches
- **`production/`** references base with production-specific patches (more replicas, HA, GRS backups)

## Per-Customer Resources (11 manifests)

Each customer in `apps/<env>/<customer>/` gets:

| File | Resource |
|------|----------|
| `namespace.yaml` | Isolated namespace |
| `secrets.yaml` | SecretProviderClass (Key Vault CSI) |
| `database.yaml` | CNPG PostgreSQL cluster |
| `objectstore.yaml` | Backup destination (Azure Blob) |
| `scheduled-backup.yaml` | Daily backup (03:00 UTC) |
| `configmap.yaml` | Application config |
| `deployment.yaml` | n8n workload |
| `service.yaml` | ClusterIP service |
| `storage.yaml` | PersistentVolumeClaim |
| `ingress.yaml` | Gateway API route |
| `grafana-secrets.yaml` | Monitoring credentials |

## Related

- **[infra-live](https://github.com/Tim275/infra-live)** - Terraform IaC that provisions Azure resources and generates manifests for this repo

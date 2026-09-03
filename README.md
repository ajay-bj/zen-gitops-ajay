# zen-gitops
# zen-gitops

GitOps configuration repository for the Zen Pharma platform.
ArgoCD watches this repo and syncs all changes to the EKS cluster automatically.

> **This fork (`ajay-bj`) — current status (2026-09-03):** all 9 dev ArgoCD Applications are
> `Synced`, and all application pods are `Running`. Image tags point at ECR account
> `304312474711`; `DB_HOST` points at RDS `pharma-dev-postgres.c2vs4a4kqea5.us-east-1.rds.amazonaws.com`.
> The personalisation described below has already been applied to this fork.

> **Companion forks:**
> - [`zen-infra-ajay`](https://github.com/ajay-bj/zen-infra-ajay) — Terraform for AWS infrastructure (EKS, RDS, ECR, IAM)
> - [`zen-pharma-backend-ajay`](https://github.com/ajay-bj/zen-pharma-backend-ajay) — Spring Boot microservices
> - [`zen-pharma-frontend-ajay`](https://github.com/ajay-bj/zen-pharma-frontend-ajay) — React frontend

> **Note:** all `repoURL` fields inside `argocd/` already point at `ajay-bj/zen-gitops-ajay`.

---

## Accessing the ArgoCD UI

ArgoCD is installed but **not exposed publicly** — `argocd-server` is a ClusterIP service and no
ArgoCD ingress is applied (this is the secure default for dev). Access it locally via port-forward:

```bash
# 1. Point kubectl at the cluster
aws eks update-kubeconfig --region us-east-1 --name pharma-dev-cluster

# 2. Port-forward the ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443

# 3. Open the UI (accept the self-signed cert warning)
#    https://localhost:8080
```

**Login:**
- Username: `admin`
- Password — read it from the cluster (do not commit the value to Git):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d ; echo
```

> The initial admin password lives only in the `argocd-initial-admin-secret` on the cluster.
> Retrieve it with the command above whenever you need it. Change it in the UI
> (User Info → Update Password) for anything beyond throwaway dev.

**Application entry point (NGINX Ingress NLB):**
the app itself is reached through the NGINX Ingress Controller's AWS NLB hostname:
```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath="{.status.loadBalancer.ingress[0].hostname}" ; echo
```

---

## After Forking — Required Personalisation (already applied to this fork)

Before ArgoCD can sync anything, two AWS-specific values that ship hardcoded to the instructor's
account must be replaced throughout `envs/dev/*.yaml`. For this fork they are already set to:
- **AWS Account ID:** `304312474711`
- **RDS instance identifier:** `c2vs4a4kqea5`

The steps below document how it was done, in case you re-fork from upstream.

### 1. AWS Account ID

Every `image.repository` field and the IAM role ARN in `values-api-gateway.yaml` ship with the
instructor's account ID (`516209541629`). Replace it with your own.

Find your account ID:
```bash
aws sts get-caller-identity --query Account --output text
```

Do the replacement (run from repo root):
```bash
# Replace YOUR_ACCOUNT_ID with the value from the command above
find envs/ -name "*.yaml" -exec sed -i '' 's/516209541629/YOUR_ACCOUNT_ID/g' {} +
```

After this, every `image.repository` points to your ECR registry, e.g.:
```yaml
image:
  repository: 304312474711.dkr.ecr.us-east-1.amazonaws.com/auth-service
```

### 2. RDS Endpoint

Every `DB_HOST` env var ships with the instructor's RDS instance identifier. Replace it with
your own (the subdomain prefix in the endpoint shown in AWS Console → RDS → your instance).

```bash
# Replace YOUR_RDS_ID with your RDS instance identifier (this fork: c2vs4a4kqea5)
find envs/ -name "*.yaml" -exec sed -i '' 's/cyrywaguk6v4/YOUR_RDS_ID/g' {} +
```

After this, `DB_HOST` looks like:
```yaml
DB_HOST: pharma-dev-postgres.c2vs4a4kqea5.us-east-1.rds.amazonaws.com
```

### 3. Verify

```bash
# Should show no instructor values
grep -r "516209541629\|cyrywaguk6v4" envs/
```

Commit and push these changes. CI then updates image tags to your ECR images on every backend
build, and ArgoCD syncs the dev namespace. On this fork all 9 dev pods are `Running`.

---

## What Lives Here

| Folder | Purpose |
|--------|---------|
| `helm-charts/` | Shared Helm chart used by all services |
| `envs/` | Per-environment Helm values files (dev = 9 services, qa/prod = 8) |
| `argocd/` | ArgoCD AppProject + per-service Application manifests |
| `external-secrets/` | ClusterSecretStore + ExternalSecrets (db-credentials, jwt-secret) per env |
| `db-init/` | PostgreSQL schema initialisation scripts |

---

## Repository Structure

```
zen-gitops/
├── helm-charts/                        # Shared Helm chart (one chart, all services)
│   ├── Chart.yaml
│   ├── values.yaml                     # Default values (overridden per service)
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── configmap.yaml
│       ├── serviceaccount.yaml
│       ├── hpa.yaml
│       └── _helpers.tpl
│
├── envs/                               # Per-environment Helm values
│   ├── dev/                            # 9 services (includes qc-service)
│   │   ├── values-api-gateway.yaml
│   │   ├── values-auth-service.yaml
│   │   ├── values-catalog-service.yaml
│   │   ├── values-inventory-service.yaml
│   │   ├── values-manufacturing-service.yaml
│   │   ├── values-notification-service.yaml
│   │   ├── values-pharma-ui.yaml
│   │   ├── values-qc-service.yaml
│   │   └── values-supplier-service.yaml
│   ├── qa/                             # 8 services, QA-specific values
│   └── prod/                           # 8 services, prod-specific values + podAntiAffinity
│
├── argocd/
│   ├── install/
│   │   ├── argocd-namespace.yaml       # argocd namespace definition
│   │   └── argocd-ingress.yaml         # ArgoCD UI ingress
│   ├── projects/
│   │   └── pharma-project.yaml         # ArgoCD AppProject (scopes allowed repos/namespaces)
│   └── apps/
│       ├── dev/                        # Individual Application per service (9 apps)
│       │   ├── api-gateway-app.yaml
│       │   ├── auth-service-app.yaml
│       │   ├── catalog-service-app.yaml
│       │   ├── inventory-service-app.yaml
│       │   ├── manufacturing-service-app.yaml
│       │   ├── notification-service-app.yaml
│       │   ├── pharma-ui-app.yaml
│       │   ├── qc-service-app.yaml
│       │   └── supplier-service-app.yaml
│       ├── qa/
│       │   └── pharma-qa-app.yaml      # Single app-of-apps pointing to envs/qa/
│       └── prod/
│           └── pharma-prod-app.yaml    # Single app-of-apps pointing to envs/prod/
│
├── external-secrets/                   # External Secrets Operator manifests
│   └── dev/
│       ├── cluster-secret-store.yaml   # ClusterSecretStore → AWS Secrets Manager (IRSA)
│       ├── db-credentials.yaml         # ExternalSecret → /pharma/dev/db-credentials
│       └── jwt-secret.yaml             # ExternalSecret → /pharma/dev/jwt-secret
│
└── db-init/
    └── 01-schemas.sql                  # Creates schemas: pharmacy, inventory, procurement, manufacturing
```

---

## How Helm Works Here

One chart (`helm-charts/`) is shared across all services.
Each service gets its own values file that overrides the defaults:

```
helm-charts/values.yaml                 <- defaults (replicas, probes, resources)
      +
envs/dev/values-auth-service.yaml       <- service-specific overrides (port, image tag, env vars)
      =
Final Kubernetes manifests for auth-service in the dev namespace
```

ArgoCD Application for a service:
```yaml
source:
  repoURL: https://github.com/ajay-bj/zen-gitops-ajay.git
  path: helm-charts
  helm:
    valueFiles:
      - ../envs/dev/values-auth-service.yaml
```

---

## ArgoCD Sync Policy per Environment

| Environment | App structure | Sync policy | Who triggers deploy |
|---|---|---|---|
| `dev` | 8 individual Applications | Automated + selfHeal | CI commits image tag → ArgoCD auto-syncs |
| `qa` | 1 `pharma-qa` app-of-apps | Automated + selfHeal | QA promotion PR merged → ArgoCD auto-syncs |
| `prod` | 1 `pharma-prod` app-of-apps | **Manual sync** | PROD PR merged → engineer triggers sync in ArgoCD UI |

---

## Updating an Image Tag (how CI does it)

CI workflow in `zen-pharma-backend` updates the image tag after a successful build:

```bash
# Example: update auth-service to sha-a1b2c3d in dev
yq e '.image.tag = "sha-a1b2c3d"' -i envs/dev/values-auth-service.yaml
git add envs/dev/values-auth-service.yaml
git commit -m "ci(dev): update auth-service -> sha-a1b2c3d"
git push
# ArgoCD detects the commit and syncs dev within 3 minutes
```

---

## Environment Differences

| Setting | dev | qa | prod |
|---|---|---|---|
| `replicaCount` | 1 | 1 | HPA-managed (min 2) |
| `autoscaling.minReplicas` | disabled | 1 | 2 |
| `autoscaling.maxReplicas` | disabled | 3 | 5 |
| `podDisruptionBudget` | disabled | disabled | enabled (`minAvailable: 1`) |
| `networkPolicy` | disabled | disabled | enabled (ingress from NGINX or api-gateway) |
| `LOG_LEVEL` | DEBUG | INFO | WARN |
| `podAntiAffinity` | no | no | yes (pods spread across nodes) |
| CPU request/limit | 100m / 500m | 150m / 500m | 250m / 1000m |
| Memory request/limit | 256Mi / 512Mi | 256Mi / 512Mi | 512Mi / 1Gi |

---

## Full Setup Guide

See [`zen-infra-ajay/docs/FULL-DEPLOYMENT-GUIDE.md`](https://github.com/ajay-bj/zen-infra-ajay/blob/main/docs/FULL-DEPLOYMENT-GUIDE.md)
for the complete step-by-step guide covering all 4 stages: infra → prerequisites → CI → ArgoCD CD.

---

## Required Steps — DEV-only (quick reference for the ajay-bj fork)

> Scope: **DEV only. Do NOT deploy qa/prod.** Assumes Stage 1 (infra) and Stage 2 (ArgoCD/ESO/NGINX
> installed) are done — see `zen-infra-ajay`.

**1. Personalize `envs/dev/*.yaml` (already applied; re-verify after any upstream re-fork)**
- AWS account ID → `304312474711` in every `image.repository` and the IAM role ARN.
- `DB_HOST` → the CURRENT RDS endpoint. **This changes on every infra rebuild** — set it to
  `pharma-dev-postgres.<NEW_ID>.us-east-1.rds.amazonaws.com` across all `envs/dev/values-*.yaml`.
- All `argocd/apps/dev/*.yaml` `repoURL` → `https://github.com/ajay-bj/zen-gitops-ajay.git`.
- `pharma-ui-app.yaml` uses `../envs/dev/values-pharma-ui.yaml` (typo fixed).

**2. Ingress must be `nginx`, not `alb` (critical)**
This cluster runs the **NGINX** Ingress Controller, not the AWS Load Balancer Controller. In
`envs/dev/values-api-gateway.yaml` and `envs/dev/values-pharma-ui.yaml` the ingress must be:
```yaml
ingress:
  enabled: true
  className: nginx
  annotations: {}
```
If left as `className: alb`, those two Ingresses never get a load-balancer ADDRESS and ArgoCD reports
the apps as **Progressing** forever (pods still run, but health never goes green). This is already fixed.

**3. How deploys happen**
App CI (backend/frontend) commits new `image.tag` values here → ArgoCD auto-syncs the `dev` namespace.
9 dev apps: api-gateway, auth-service, catalog-service, inventory-service, manufacturing-service,
notification-service, pharma-ui, qc-service, supplier-service.

**4. Verify**
```bash
kubectl get applications -n argocd     # all 9 Synced + Healthy
kubectl get pods -n dev                # all 9 Running
kubectl get ingress -n dev             # api-gateway + pharma-ui: CLASS=nginx, ADDRESS=<NLB>
```

**5. Access the ArgoCD UI** (see the "Accessing the ArgoCD UI" section above)
Port-forward `svc/argocd-server -n argocd 8080:443`, open **https**://localhost:8080, login `admin`.
Retrieve the password from the cluster (never commit it):
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

> **QA/PROD are off:** the app CI `open-qa-pr` jobs are disabled, so no QA promotion PRs are created.
> Keep it that way; deploy dev only.

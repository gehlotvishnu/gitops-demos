# gitops-demos  —  Helm Charts + Deploy Config  (Ops team repo)

## Architecture: 3-Repo GitOps Pattern

```
┌──────────────────────┐   push + tag   ┌───────────────────────────────────────┐
│  APP REPO            │ ─────────────► │  gitops-demos  (THIS REPO)            │
│  payment-service     │  CI updates    │  Helm charts + deploy-values          │
│  order-service       │  image tag     │  Owned by: Platform / Ops team        │
│  - Java source       │               └───────────┬───────────────────────────┘
│  - Dockerfile        │                           │  ArgoCD multi-source
│  - GitHub Actions CI │               ┌───────────┘  (both repos)
└──────────────────────┘               │
                                       │           ┌───────────────────────────────┐
                        ┌──────────────┘           │  spring-boot-config           │
                        │  ArgoCD watches both     │  application.properties       │
                        ▼                          │  Owned by: Dev / App team     │
             ┌──────────────────────┐              └───────────────────────────────┘
             │  OpenShift (CRC)     │
             │  payment-service-dev │
             │  order-service-dev   │
             │  payment-service-sit │
             │  order-service-sit   │
             │  *-prod (manual)     │
             └──────────────────────┘
```

## Repo Responsibilities

| Repo | Owner | Contains | Change triggers |
|------|-------|----------|-----------------|
| `gitops-demos` | Ops/Platform | Helm chart templates, deploy-values (replicas, image, resources) | Ops PRs |
| `spring-boot-config` | Dev/App team | `application.properties` per service per env | Dev PRs |
| `payment-service` | App team | Java source, Dockerfile, CI | Feature branches |

## Directory Structure

```
gitops-demos/
├── charts/
│   └── spring-boot-app/              ← Reusable Helm chart (all services use this)
│       ├── Chart.yaml
│       ├── values.yaml               ← Chart defaults only
│       └── templates/
│           ├── namespace.yaml
│           ├── configmap.yaml        ← Renders springConfig → application.properties
│           ├── deployment.yaml       ← Mounts ConfigMap at /config/
│           ├── service.yaml
│           └── route.yaml
├── envs/
│   ├── dev/
│   │   ├── payment-service/deploy-values.yaml   ← replicas, image, resources
│   │   └── order-service/deploy-values.yaml
│   ├── sit/
│   │   ├── payment-service/deploy-values.yaml
│   │   └── order-service/deploy-values.yaml
│   └── prod/
│       ├── payment-service/deploy-values.yaml
│       └── order-service/deploy-values.yaml
└── argocd/
    ├── dev/
    │   ├── payment-service.yaml      ← multi-source: chart + deploy-values + app-config
    │   └── order-service.yaml
    ├── sit/
    │   ├── payment-service.yaml
    │   └── order-service.yaml
    └── prod/
        ├── payment-service.yaml      ← manual sync (no auto-prune in prod)
        └── order-service.yaml

spring-boot-config/                   ← Separate repo
├── dev/
│   ├── payment-service/app-config.yaml   ← Spring properties as Helm values
│   └── order-service/app-config.yaml
├── sit/
│   ├── payment-service/app-config.yaml
│   └── order-service/app-config.yaml
└── prod/
    ├── payment-service/app-config.yaml
    └── order-service/app-config.yaml
```

## How spring-boot-config flows into the pod

```
spring-boot-config/dev/payment-service/app-config.yaml
    springConfig:
      server.port: "8080"
      payment.gateway.url: https://sandbox.payment-gateway.io
      ...
         │
         │  ArgoCD multi-source merges with deploy-values.yaml
         ▼
    Helm renders → ConfigMap (application.properties)
         │
         │  Deployment mounts ConfigMap as volume
         ▼
    /config/application.properties  (inside pod)
         │
         │  Spring Boot auto-discovers /config/ location
         ▼
    app reads server.port, gateway URL, feature flags, etc.
```

## Applying ArgoCD Applications

```bash
eval $(~/local/bin/crc oc-env)
oc login -u kubeadmin -p '5SkgS-KHtGB-Xp7zJ-9hnef' https://api.crc.testing:6443 --insecure-skip-tls-verify

# Dev (auto-sync)
oc apply -f argocd/dev/

# SIT (auto-sync)
oc apply -f argocd/sit/

# Prod (manual sync only)
oc apply -f argocd/prod/
```

## GitOps Workflows

### Dev team changes app config (e.g. enable a feature flag)
```bash
# In spring-boot-config repo:
vi dev/payment-service/app-config.yaml
# change: payment.feature.upi-enabled: "false" → "true"
git commit -am "enable UPI in dev" && git push
# ArgoCD detects diff → updates ConfigMap → pod restarts with new config
```

### Ops team scales up for load test
```bash
# In gitops-demos repo:
vi envs/sit/payment-service/deploy-values.yaml
# change: replicaCount: 1 → replicaCount: 3
git commit -am "scale payment-service sit to 3 for load test" && git push
# ArgoCD syncs → Deployment scales to 3 pods
```

### Promote dev image tag to SIT
```bash
# CI updates this automatically after a successful dev deploy:
yq -i '.image.tag = "1.2.0-rc1"' envs/sit/payment-service/deploy-values.yaml
git commit -am "chore(sit): promote payment-service to 1.2.0-rc1"
git push
```

### Promote SIT to prod (manual gate)
```bash
# Update prod tag
yq -i '.image.tag = "1.2.0"' envs/prod/payment-service/deploy-values.yaml
git commit -am "chore(prod): release payment-service 1.2.0"
git push
# ArgoCD shows prod app as OutOfSync — requires manual sync approval
# oc patch application payment-service-prod -n openshift-gitops \
#   --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```

## ArgoCD UI
- URL: https://openshift-gitops-server-openshift-gitops.apps-crc.testing
- Username: `admin`
- Password: `oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-`

# gitops-demo — Config Repo (ArgoCD Watches This)

## Two-Repo GitOps Pattern

```
┌─────────────────────────┐     CI/CD      ┌───────────────────────────┐
│   APP REPO              │   ─────────►   │   CONFIG REPO (this repo) │
│   spring-boot-demo      │   image push   │   gitops-demo             │
│   - Java source         │   + tag update │   - Helm chart            │
│   - Dockerfile          │               │   - env values (dev/prod) │
│   - GitHub Actions CI   │               │   - ArgoCD Applications   │
└─────────────────────────┘               └───────────┬───────────────┘
                                                       │ ArgoCD polls
                                                       ▼
                                          ┌────────────────────────┐
                                          │   OpenShift (CRC)      │
                                          │   spring-boot-dev ns   │
                                          │   spring-boot-prod ns  │
                                          └────────────────────────┘
```

## Repo Structure

```
gitops-demo/
├── charts/
│   └── spring-boot-app/          ← Reusable Helm chart
│       ├── Chart.yaml
│       ├── values.yaml           ← Defaults
│       └── templates/
│           ├── namespace.yaml
│           ├── configmap.yaml    ← Spring app.config via env vars
│           ├── deployment.yaml
│           ├── service.yaml
│           └── route.yaml        ← OpenShift Route (HTTPS)
├── envs/
│   ├── dev/values.yaml           ← Dev overrides (image tag updated by CI)
│   └── prod/values.yaml          ← Prod overrides (manual promotion)
├── argocd/
│   ├── app-dev.yaml              ← ArgoCD Application (auto-sync)
│   └── app-prod.yaml             ← ArgoCD Application (manual sync)
└── .github/workflows/
    └── update-image-tag.yml      ← Triggered by app repo CI
```

## Setup

### 1. Create GitHub Repo
```bash
cd ~/Desktop/gitops-demo
git init && git add . && git commit -m "initial gitops structure"
gh repo create gitops-demo --public --source=. --push
```

### 2. Update repo URLs
Replace `YOUR_ORG` in:
- `argocd/app-dev.yaml`
- `argocd/app-prod.yaml`
- `envs/dev/values.yaml`
- `envs/prod/values.yaml`

### 3. Apply ArgoCD Applications
```bash
eval $(~/local/bin/crc oc-env)
oc login -u kubeadmin -p '5SkgS-KHtGB-Xp7zJ-9hnef' https://api.crc.testing:6443 --insecure-skip-tls-verify
oc apply -f argocd/app-dev.yaml
# Prod apply after verifying dev:
# oc apply -f argocd/app-prod.yaml
```

### 4. Access ArgoCD UI
- URL: https://openshift-gitops-server-openshift-gitops.apps-crc.testing
- Username: `admin`
- Password: `oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-`

## GitOps Loop Demo

### Demo: Scale Up (shows self-healing)
```bash
# 1. Edit config repo
sed -i 's/replicaCount: 1/replicaCount: 2/' envs/dev/values.yaml
git commit -am "scale dev to 2 replicas"
git push

# 2. ArgoCD detects diff (within 3 min) and syncs
# 3. Try manually scaling down in cluster — ArgoCD will revert it (selfHeal: true)
oc scale deployment spring-boot-demo -n spring-boot-dev --replicas=0
# Watch ArgoCD bring it back to 2
```

### Demo: Image Update (simulates CI push)
```bash
yq -i '.image.tag = "1.2.0-abc1234"' envs/dev/values.yaml
git commit -am "chore(dev): update image tag to 1.2.0-abc1234"
git push
# ArgoCD rolls out new image with zero-downtime rolling update
```

### Demo: Config Change (shows app config management)
```bash
yq -i '.appConfig.LOG_LEVEL = "TRACE"' envs/dev/values.yaml
git commit -am "enable trace logging in dev"
git push
# ArgoCD updates ConfigMap and triggers pod restart
```

### Demo: Prod Promotion (shows manual gate)
```bash
# 1. Test passed in dev — promote same tag to prod
NEW_TAG=$(yq '.image.tag' envs/dev/values.yaml)
yq -i ".image.tag = \"$NEW_TAG\"" envs/prod/values.yaml
git commit -am "promote $NEW_TAG to prod"
git push
# 2. ArgoCD shows prod as OutOfSync (no auto-sync in prod)
# 3. Manual sync via UI or:
# argocd app sync spring-boot-demo-prod
```

## Key GitOps Principles Demonstrated
| Principle | How |
|-----------|-----|
| **Declarative** | All state in YAML, not imperative kubectl |
| **Versioned** | Git is the single source of truth |
| **Pull-based** | ArgoCD pulls from Git (vs CI push to cluster) |
| **Self-healing** | `selfHeal: true` reverts manual cluster changes |
| **Env parity** | Same chart, different values per env |
| **Separation of concerns** | App code ≠ deployment config |

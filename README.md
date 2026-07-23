# Bootstrap Demo — ArgoCD RollingSync Health-Gated Deployment

## What This Proves

ArgoCD ApplicationSet with `RollingSync` strategy deploys bootstrap components
one-at-a-time, waiting for each to reach **Healthy** before starting the next.

## The Pattern (per ArgoCD docs)

`RollingSync` groups generated Applications by labels and processes them in
sequential steps. Each step waits for all Applications in that step to become
**Healthy** before proceeding to the next step.

**Critical**: RollingSync DISABLES autosync on generated Applications — the
ApplicationSet controller triggers syncs itself when it's that step's turn.

## Demo Components

| Step | App Label | What It Deploys | Depends On |
|------|-----------|-----------------|------------|
| 0    | bootstrap-group: 00-lvi | Namespace + ConfigMap (simulates CRD) | nothing |
| 1    | bootstrap-group: 01-intents | Deployments in the namespace | step 0 Healthy |
| 2    | bootstrap-group: 02-rbac | ClusterRole + ClusterRoleBinding | step 1 Healthy |
| 3    | bootstrap-group: 03-service | Service + Ingress | step 2 Healthy |

## Setup Commands

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Enable progressive syncs on the ApplicationSet controller
kubectl patch configmap argocd-cmd-params-cm -n argocd --type merge -p '{"data":{"applicationsetcontroller.enable.progressive.syncs":"true"}}'
kubectl rollout restart deployment argocd-applicationset-controller -n argocd

# 3. Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=120s

# 4. Deploy the demo ApplicationSet
kubectl apply -f apps/appset.yaml -n argocd

# 5. Watch the rolling sync happen
watch -n 2 "kubectl get applications -n argocd -l app.kubernetes.io/part-of=bootstrap-demo -o custom-columns='NAME:.metadata.name,SYNC:.status.sync.status,HEALTH:.status.health.status'"
```

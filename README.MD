# shoplite-gitops

ArgoCD Applications and environment-specific Helm values for ShopLite.

The Helm chart lives in [shoplite-app](https://github.com/Swapno963/shoplite-app) (`helm/shoplite`). This repository supplies one ArgoCD Application per environment and the overlay values ArgoCD applies on top of the chart defaults.

```text
argocd/applications/     ArgoCD Application CRs
environments/dev/        Development Helm overlay
environments/prod/       Production Helm overlay
```

## Sync policy

| Environment | Application     | Namespace     | Git sync                         | Self-heal                          |
|-------------|-----------------|---------------|----------------------------------|------------------------------------|
| Development | `shoplite-dev`  | `shoplite-dev`  | Automatic (`prune` + `selfHeal`) | Yes (part of automated sync)     |
| Production  | `shoplite-prod` | `shoplite-prod` | Manual (UI or `argocd app sync`) | No (ArgoCD only self-heals with automated sync) |

## Bootstrap

Register both GitHub repositories in ArgoCD if they are private. Then apply the Applications:

```bash
kubectl apply -n argocd -f argocd/applications/
```

Check status:

```bash
kubectl -n argocd get applications
```

Development syncs itself once Git is reachable. Production stays `OutOfSync` until you sync it:

```bash
argocd app sync shoplite-prod
```

Secrets (`shoplite-secret`) are not managed here. Create them in each destination namespace before the workloads will become Healthy, matching the Helm `secrets.existingSecret` value.

## Self-healing demonstration (development)

ArgoCD reverts live cluster drift on `shoplite-dev` because `syncPolicy.automated.selfHeal` is enabled. Run this after the Application is `Synced` / `Healthy`.

1. Record the desired replica count from Git (dev `web` is `1`):

```bash
kubectl -n shoplite-dev get deploy web -o jsonpath='{.spec.replicas}{"\n"}'
```

2. Change the live resource so it no longer matches Git:

```bash
kubectl -n shoplite-dev scale deploy web --replicas=5
```

You can instead use `kubectl patch` or `kubectl edit` on the same Deployment.

3. Confirm drift:

```bash
kubectl -n shoplite-dev get deploy web
kubectl -n argocd get application shoplite-dev
```

The Deployment should show `5` replicas and the Application should briefly report `OutOfSync`.

4. Wait for the reconcile (ArgoCD typically refreshes within a few seconds). Re-check:

```bash
kubectl -n shoplite-dev get deploy web -o jsonpath='{.spec.replicas}{"\n"}'
kubectl -n argocd get application shoplite-dev
```

Replicas return to `1`. The Application is `Synced` and `Healthy` again.

### Contrast with production

The same `kubectl scale` against `shoplite-prod` stays drifted until someone syncs `shoplite-prod` manually. That is intentional: production does not auto-sync.

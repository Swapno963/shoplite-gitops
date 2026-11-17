# shoplite-gitops

ArgoCD Applications and environment-specific Helm values for ShopLite.

The Helm chart lives in [shoplite-app](https://github.com/Swapno963/shoplite-app) (`helm/shoplite`). This repository supplies one ArgoCD Application per environment and the overlay values ArgoCD applies on top of the chart defaults.

```text
shoplite-gitops/
├── apps/
│   └── argocd-applications
└── envs/
    ├── dev/
    │   └── values.yaml
    └── prod/
        └── values.yaml
```

## Sync policy

| Environment | Application     | Namespace       | Git sync                         | Self-heal                                      |
|-------------|-----------------|-----------------|----------------------------------|------------------------------------------------|
| Development | `shoplite-dev`  | `shoplite-dev`  | Automatic (`prune` + `selfHeal`) | Yes (part of automated sync)                   |
| Production  | `shoplite-prod` | `shoplite-prod` | Manual (UI or `argocd app sync`) | No (ArgoCD only self-heals with automated sync) |

## Bootstrap

Register both GitHub repositories in ArgoCD if they are private. Then apply the Applications:

```bash
kubectl apply -n argocd -f apps/argocd-applications/
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

## GitOps: page title change (no manual deploy)

This shows a UI change reaching the browser after Git pushes only. Do not run `helm upgrade`, `kubectl apply` on the app, or `argocd app sync shoplite-dev`. Development auto-syncs.

1. Change the document title in the web app (`shoplite-app`):

```bash
cd /path/to/shoplite-app
```

Edit `application/web/index.html` and set a new title, for example:

```html
<title>ShopLite GitOps</title>
```

2. Push the application change:

```bash
git add application/web/index.html
git commit -m "Change web page title"
git push origin main
```

3. Wait for GitHub Actions `Build Web` to push the image. Copy the commit SHA:

```bash
WEB_SHA=$(git rev-parse HEAD)
echo "$WEB_SHA"
```

Confirm the image exists (optional):

```bash
aws ecr describe-images \
  --repository-name shoplite-web \
  --image-ids imageTag="$WEB_SHA" \
  --region ap-southeast-1
```

4. Point development at that image in this GitOps repo and push. That Git commit is the desired state; ArgoCD applies it.

```bash
cd /path/to/shoplite-gitops
```

Set `web.image.tag` in `envs/dev/values.yaml` to `"$WEB_SHA"`, then:

```bash
git add envs/dev/values.yaml
git commit -m "Deploy web title change to development"
git push origin main
```

5. Wait for ArgoCD (no deploy command):

```bash
kubectl -n argocd get application shoplite-dev -w
kubectl -n shoplite-dev get deploy web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

When `shoplite-dev` is `Synced` / `Healthy` and the web image tag matches `$WEB_SHA`, open the app:

```text
http://dev.shoplite.local
```

The browser tab title should be `ShopLite GitOps`. Hard-refresh if the old title is cached.

Production does not pick this up until someone updates `envs/prod/values.yaml` and manually syncs `shoplite-prod`.

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

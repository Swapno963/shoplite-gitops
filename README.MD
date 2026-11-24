# shoplite-gitops

Desired state for ShopLite on the cluster. The app and Helm chart live in [shoplite-app](https://github.com/Swapno963/shoplite-app). This repo is what Argo CD uses for environments.

Two namespaces:

- **shoplite-dev** — auto-sync, prune, self-heal
- **shoplite-prod** — auto-sync, self-heal, no prune

When GitHub Actions builds an image on `main` in shoplite-app, it writes the SHA into `envs/dev/values.yaml` here. Prod stays on `stable` until I change `envs/prod/values.yaml`.

## Layout

```text
apps/argocd-applications/   AppProject + shoplite-dev / shoplite-prod
envs/dev/values.yaml        replicas, ingress host, image tags
envs/prod/values.yaml       same for production
```

Argo CD renders `helm/shoplite` from shoplite-app `main`, then overlays the matching file from `envs/`.

```text
shoplite-app (chart + images) ──┐
                                ├── Argo CD → shoplite-dev / shoplite-prod
shoplite-gitops (this repo)  ───┘
```

Users still hit the Terraform ALB → ingress-nginx NodePort `30080` → ClusterIP services in the namespace.

## Sync

| App | Namespace | Auto-sync | Self-heal | Prune |
|-----|-----------|-----------|-----------|-------|
| `shoplite-dev` | `shoplite-dev` | yes | yes | yes |
| `shoplite-prod` | `shoplite-prod` | yes | yes | no |

Secrets (`shoplite-secret`) are created in the namespace, not in Git. Cluster install and Argo CD bootstrap notes are in shoplite-app `evidence/`.

# GitOps and Kargo

This directory is the GitOps source of truth for the `kargo-showcase`
promotion pipeline.

## Layout

```text
gitops/
├── argocd/
│   ├── appproject.yaml          # AppProject for the environment Applications
│   └── applicationset.yaml      # One Application per directory under envs/
├── apps/kargo-showcase/
│   ├── base/values.yaml         # shared Helm values
│   ├── release.yaml             # Git Freight source (release marker)
│   ├── envs/<env>/values.yaml   # environment configuration
│   ├── envs/<env>/release.yaml  # promoted release marker (written by Kargo)
│   └── kargo/                   # Project, Warehouse, PromotionTask, Stages
└── bootstrap/root-app.yaml      # optional app-of-apps bootstrap
```

The chart itself lives in `charts/kargo-showcase`; the environment directories
are Helm value overlays. Nothing is duplicated per environment.

## Bootstrap

Kargo and Argo CD must already be installed (`task cluster:up`).

```sh
task gitops:apply
```

This creates the Argo CD app-of-apps, which applies:

- the `kargo-showcase` AppProject and the ApplicationSet;
- the Kargo Project, ProjectConfig, Warehouse, PromotionTask and Stages.

The ApplicationSet's Git directory generator discovers
`gitops/apps/kargo-showcase/envs/*` and creates:

- `kargo-showcase-dev` in `kargo-showcase-dev`;
- `kargo-showcase-staging` in `kargo-showcase-staging`;
- `kargo-showcase-prod` in `kargo-showcase-prod`.

## Git credentials for Kargo

Argo CD only needs read access; this repository is public. Kargo needs write
access to commit the promoted release marker:

```sh
cp gitops/apps/kargo-showcase/credentials.example.yaml \
   gitops/apps/kargo-showcase/credentials.yaml
# edit username and PAT
kubectl apply -f gitops/apps/kargo-showcase/credentials.yaml
```

The real `credentials.yaml` is ignored by Git.

## Promotion flow

The image `kargo-showcase:dev` is already loaded into kind by `task cluster:up`,
so this lab does not poll an external image registry. Instead the Warehouse
watches `gitops/apps/kargo-showcase/release.yaml`:

```yaml
image:
  tag: dev
podLabels:
  kargo.akuity.io/release: v1
```

The promoted value is the `kargo.akuity.io/release` pod label. This keeps the
lab fully local while still exercising real Kargo Freight and promotions.

1. `dev` subscribes directly to the Warehouse and is auto-promoted.
2. `staging` subscribes to `dev` and is auto-promoted once `dev` is healthy.
3. `prod` subscribes to `staging` and requires a manual promotion in Kargo.

The shared PromotionTask copies the Freight's `release.yaml` into
`envs/<stage>/release.yaml`, commits it to `main`, pushes, and asks Argo CD to
sync the matching Application.

### Simulate a release

Edit the release marker and push:

```sh
# change v1 to v2 in gitops/apps/kargo-showcase/release.yaml
git add gitops/apps/kargo-showcase/release.yaml
git commit -m "release v2"
git push
```

The Warehouse creates new Freight. Dev and staging follow automatically; prod
waits for a manual promotion from the Kargo UI.

## Environment URLs

| Environment | URL |
| --- | --- |
| dev | <http://dev.kargo-showcase.localhost/api/random> |
| staging | <http://staging.kargo-showcase.localhost/api/random> |
| prod | <http://prod.kargo-showcase.localhost/api/random> |

Kargo is exposed at <http://kargo.localhost> (`admin` / `admin`, local only).

## Moving to image promotions

When an image registry is available, add an image subscription to the
Warehouse and change the PromotionTask to update `image.tag` in the target
environment (for example with `yaml-update`). The ApplicationSet and Stage
structure do not change.

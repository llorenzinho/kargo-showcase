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
│   └── kargo/                   # Namespace, Project, Warehouse, PromotionTask, Stages
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

`task gitops:apply` also applies `gitops/apps/kargo-showcase/credentials.yaml`
if it exists. The ApplicationSet's Git directory generator discovers
`gitops/apps/kargo-showcase/envs/*` and creates:

- `kargo-showcase-dev` in `kargo-showcase-dev`;
- `kargo-showcase-staging` in `kargo-showcase-staging`;
- `kargo-showcase-prod` in `kargo-showcase-prod`.

## Git credentials for Kargo

Argo CD only needs read access; this repository is public. Kargo needs write
access to commit the promoted release marker. Create the credentials file
before running `task gitops:apply` so the task can apply it:

```sh
cp gitops/apps/kargo-showcase/credentials.example.yaml \
   gitops/apps/kargo-showcase/credentials.yaml
# edit username and PAT
task gitops:apply
```

If you apply the credentials later, run `kubectl apply -f
gitops/apps/kargo-showcase/credentials.yaml` and retry any failed promotion
from the Kargo UI. The real `credentials.yaml` is ignored by Git.

### PAT permissions

Kargo uses the token for both `git-clone` and `git-push`. The Warehouse can
read this public repository anonymously, but a promotion must push a commit to
`main`.

For a **fine-grained PAT**:

- Resource owner: the account that owns the repository.
- Repository access: **Only select repositories** → `llorenzinho/kargo-showcase`.
- Repository permissions:
  - **Contents: Read and write** — required for clone and push.
  - **Metadata: Read-only** — automatically included.
- If the repository belongs to an organization with SAML/SSO enabled,
  authorize the token for that organization.

For a **classic PAT**:

- `public_repo` is sufficient for this public repository.
- `repo` also works, but is broader than necessary.

The GitHub username goes in `username`; the PAT goes in `password`. Do not grant
`Administration`, `Workflows`, or other unrelated scopes for this demo. If the
token expires or is revoked, promotions fail at `git-push`; update the Secret
and retry the Promotion.

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

The promoted value is the whole release marker: `image.tag` and the
`kargo.akuity.io/release` pod label. This keeps the lab fully local while still
exercising real Kargo Freight and promotions.

1. `dev` subscribes directly to the Warehouse and is auto-promoted.
2. `staging` subscribes to `dev` and is auto-promoted once `dev` is healthy.
3. `prod` subscribes to `staging` and requires a manual promotion in Kargo.

The shared PromotionTask copies the Freight's `release.yaml` into
`envs/<stage>/release.yaml`, commits it to `main`, pushes, and asks Argo CD to
sync the matching Application. Do not edit the `envs/*/release.yaml` files by
hand: Kargo owns them.

### Simulate a release

For an **image release**, first build and load the new image into kind:

```sh
task image:build VERSION=v2
```

This simulates a CI pipeline publishing `kargo-showcase:v2`; locally,
`kind load docker-image` replaces the registry push.

Then update the source release marker:

```yaml
# gitops/apps/kargo-showcase/release.yaml
image:
  tag: v2
podLabels:
  kargo.akuity.io/release: v2
```

Commit and push:

```sh
git add gitops/apps/kargo-showcase/release.yaml
git commit -m "release v2"
git push
```

The Warehouse creates new Freight. Kargo copies the whole release marker into
each environment overlay, so the `image.tag` and the release label are promoted
together. Dev and staging follow automatically; prod waits for a manual
promotion from the Kargo UI.

For a **config-only release**, leave `image.tag: dev` and bump only
`podLabels["kargo.akuity.io/release"]`.

## Environment URLs

| Environment | URL |
| --- | --- |
| dev | <http://dev.kargo-showcase.localhost/api/random> |
| staging | <http://staging.kargo-showcase.localhost/api/random> |
| prod | <http://prod.kargo-showcase.localhost/api/random> |

Kargo is exposed at <http://kargo.localhost> (`admin` / `admin`, local only).

## Moving to image promotions

This lab already simulates image promotion through Git: build/load a version
with `task image:build VERSION=v2`, set `image.tag: v2` in the source release
marker, and push. Kargo promotes the marker through dev, staging and prod, so
each environment eventually references `kargo-showcase:v2`.

For a production setup, replace the Git Warehouse with an image subscription
and change the PromotionTask to update `image.tag` in the target environment
(for example with `yaml-update`). The ApplicationSet and Stage structure do not
change.

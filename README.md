# kargo-showcase

Minimal Go HTTP server, managed with [mise](https://mise.jdx.dev/).

## Endpoints

| Method | Path          | Description                  |
| ------ | ------------- | ---------------------------- |
| GET    | `/healthz`    | Liveness probe               |
| GET    | `/readyz`     | Readiness probe              |
| GET    | `/api/random` | Returns a random number 0-99 |

## Local development

```sh
mise install
```

The server listens on `:8080` by default; override it with `PORT`.

```sh
mise run run                  # or: task run
curl localhost:8080/healthz
curl localhost:8080/readyz
curl localhost:8080/api/random
```

The `Taskfile.yml` also provides `task build`, `task test`, `task vet` and
`task docker:build`. If mise is not activated in your shell, prefix commands
with `mise exec --`, e.g. `mise exec -- task run`. The equivalent mise tasks
(`mise run build`, `mise run test`, ...) are available too.

## Container image

```sh
task docker:build             # or: mise run docker-build
docker run --rm -p 8080:8080 kargo-showcase:dev
```

The Dockerfile builds a static binary and ships it on a distroless, non-root
image.

## Kubernetes with kind

```sh
task cluster:up     # cluster + ingress-nginx + image + Argo CD + Kargo
task cluster:down   # delete the cluster
```

`kind` is installed by mise, and the cluster is defined in `cluster.yaml`
(one control-plane node with ports 80/443 published to the host).
`task cluster:up` also installs ingress-nginx, Argo CD and Kargo, exposes them
through Ingress resources and prints their access credentials.

### Argo CD

`task cluster:up` prints the URL, username and password at the end. Argo CD is
available at <http://argocd.localhost> and the Ingress manifest lives in
`utils/argocd-ingress.yaml`.

To read the password again:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

The Argo CD chart version is pinned by `ARGOCD_CHART_VERSION` in the
`Taskfile.yml`; reinstall it standalone with `task argocd:install`.

### Kargo

Kargo is installed **by Argo CD**: `utils/cert-manager-app.yaml` and
`utils/kargo-app.yaml` are Argo CD Applications that sync the cert-manager and
Kargo Helm charts. `task kargo:install` applies them, waits for Argo CD to
report them healthy, applies the Kargo Ingress and prints the credentials.

- URL: <http://kargo.localhost>
- Username: `admin`
- Password: `admin` (local demo only)

The Kargo admin password hash and token signing key in `utils/kargo-app.yaml`
are intentionally local-only values.

## GitOps and promotions

The `gitops/` directory contains the ApplicationSet, the per-environment Helm
value overlays and the Kargo pipeline. See [`gitops/README.md`](gitops/README.md)
for the full flow.

```sh
task gitops:apply
```

This bootstraps the Argo CD ApplicationSet and the Kargo Project, Warehouse,
PromotionTask and Stages. The ApplicationSet creates one Argo CD Application
per environment under `gitops/apps/kargo-showcase/envs/`:

| Environment | URL |
| --- | --- |
| dev | <http://dev.kargo-showcase.localhost/api/random> |
| staging | <http://staging.kargo-showcase.localhost/api/random> |
| prod | <http://prod.kargo-showcase.localhost/api/random> |

The image `kargo-showcase:dev` is already loaded into kind, so this lab does not
push to an external registry. Kargo promotes a Git release marker
(`gitops/apps/kargo-showcase/release.yaml`): dev and staging are auto-promoted,
prod requires a manual promotion in Kargo.

Kargo needs write access to the repository to commit promotions. Copy
`gitops/apps/kargo-showcase/kargo/credentials.example.yaml` to
`credentials.yaml`, fill in a GitHub username and PAT, and apply it after the
Kargo Project exists or after `task gitops:apply`.

## Helm chart

The chart in `charts/kargo-showcase` deploys a Deployment with liveness and
readiness probes, a ClusterIP Service and an Ingress.

`task cluster:up` installs ingress-nginx (kind manifest, pinned by
`INGRESS_NGINX_VERSION`) before loading the image and setting up Argo CD.
Once the cluster is up, deploy the chart:

```sh
helm upgrade --install kargo-showcase charts/kargo-showcase \
  --namespace kargo-showcase --create-namespace

curl http://localhost/api/random
```

The default values expect the image `kargo-showcase:dev` (already loaded into
kind by `task cluster:up`) and the `nginx` ingress class. Override them with
`--set image.repository=...`, `--set image.tag=...` or
`--set ingress.className=...`.

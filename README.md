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
task cluster:up     # cluster + ingress-nginx + image + Argo CD
task cluster:down   # delete the cluster
```

`kind` is installed by mise, and the cluster is defined in `cluster.yaml`
(one control-plane node with ports 80/443 published to the host).
`task cluster:up` also installs ingress-nginx, exposes Argo CD through it and
prints its access credentials.

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

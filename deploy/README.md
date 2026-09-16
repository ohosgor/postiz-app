# Postiz on the cluster

Postiz runs the social media side of getkiyas: one place to write a post and schedule
it to every network, instead of a phone and five apps.

This directory is the only thing in this fork that is ours. Everything else is
upstream (`gitroomhq/postiz-app`) and is meant to stay mergeable — `git pull upstream
main` should never touch a file in here.

```
deploy/
  chart/            the Helm chart Argo CD syncs
  argocd/postiz.yaml the Application, applied once by hand
```

## What runs

Four workloads in the `postiz` namespace, and nothing outside it:

| Workload | What it is |
|---|---|
| `postiz` | the application — nginx on 5000, Next.js frontend, NestJS backend and the Temporal worker, all under pm2 in one container (upstream's shape) |
| `postiz-postgres` | PostgreSQL 17, holding both the `postiz` database and Temporal's two |
| `postiz-redis` | Redis 7.2, the in-process queue store |
| `postiz-temporal` | Temporal `auto-setup`, which every scheduled post goes through |

It is a separate namespace on purpose. Nothing here talks to the catalogue, and the
whole thing should be deletable without a moment's thought about the price data.

## Bootstrap

Three things exist outside git because they are secrets or one-time acts.

**1. The image pull secret.** The GHCR package is private, so copy the credentials
the other namespaces already use:

```bash
kubectl create namespace postiz
kubectl get secret ghcr-credentials -n kiyas -o yaml \
  | sed 's/namespace: kiyas/namespace: postiz/' \
  | kubectl apply -n postiz -f -
```

**2. The application secret.** Every key in it becomes an environment variable in the
container, which is what makes adding a social network a secret edit rather than a
chart change.

```bash
kubectl create secret generic postiz-secrets -n postiz \
  --from-literal=POSTGRES_USER=postiz \
  --from-literal=POSTGRES_PASSWORD="$(openssl rand -hex 24)" \
  --from-literal=JWT_SECRET="$(openssl rand -hex 32)"
```

`JWT_SECRET` signs the session cookies: changing it later logs everyone out, and
`POSTGRES_PASSWORD` cannot be changed by editing the secret alone once the database
has initialised with it.

Social network credentials go in the same secret, one key per value, named exactly as
upstream's `docker-compose.yaml` names them:

```bash
kubectl patch secret postiz-secrets -n postiz --type merge \
  -p "{\"stringData\":{\"X_API_KEY\":\"...\",\"X_API_SECRET\":\"...\"}}"
kubectl rollout restart deploy/postiz -n postiz
```

**3. The Argo CD Application.**

```bash
kubectl apply -f deploy/argocd/postiz.yaml
```

From then on Argo CD watches `deploy/chart` on `main`. Nothing else here is applied
by hand.

## The first account

`disableRegistration` ships as `"false"` so the first account can be created. Once
you have logged in, set it to `"true"` in `values.yaml` and let Argo CD sync —
otherwise anyone who finds the hostname can register on it.

## Images

`.github/workflows/build-image.yaml` builds `linux/arm64` only, from `Dockerfile.dev`
(upstream's one and only Dockerfile; the name is historical), pushes
`ghcr.io/ohosgor/postiz-app:<sha>` and then commits the new tag into
`chart/values.yaml`. Argo CD picks that commit up and rolls the deployment.

So updating Postiz is: merge upstream into `main`, wait.

```bash
git fetch upstream && git merge upstream/main && git push
```

The workflow ignores pushes that only touch `deploy/`, `.github/` or Markdown —
without that, its own write-back commit would trigger it again.

Upstream's own workflows (CodeQL, Jenkins-era builds, the extension publisher) are
left in the tree but disabled in this fork's Actions settings; deleting them would
conflict on every merge.

## Public access

`postiz.getkiyas.com`, through the same cloudflared tunnel as everything else — the
route lives in `kiyas-market-sync/deploy/cloudflared/values.yaml`, pointing at
`http://postiz.postiz.svc.cluster.local:5000`. There is no ingress controller in this
cluster; the tunnel is the only way in.

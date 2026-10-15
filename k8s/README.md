# Kubernetes deployment

Manifests to run Weft on the backbone cluster: one container (Kestrel serves
the SPA + API on :8080) backed by Postgres via
[CloudNativePG](https://cloudnative-pg.io). On boot the app applies EF
migrations and seeds itself — there is no separate migrate job.

| File | What |
| --- | --- |
| `namespace.yaml` | `weft` namespace |
| `cnpg.yaml` | CNPG `Cluster` `weft-db` (5Gi, single instance) |
| `app.yaml` | ConfigMap + Deployment + Service + Ingress for `weft.voltrum.co` |

## Prerequisites

The cluster needs: the CNPG operator (namespace `cnpg-system`), ingress-nginx,
ceph-block storage, and an image registry to pull from. The manifests assume
Harbor at `10.0.0.56:5000` and a wildcard `*.voltrum.co` DNS to cloudflared —
adjust the `image:` field and the Ingress host if your cluster differs.

## Build and push the image

```sh
docker build -t 10.0.0.56:5000/library/weft:latest .
docker push 10.0.0.56:5000/library/weft:latest
```

Pin to a content tag (git sha) instead of `latest` when you want rollbacks.

## Deploy

```sh
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/cnpg.yaml
kubectl -n weft wait cluster/weft-db --for=condition=Ready --timeout=300s
kubectl apply -f k8s/app.yaml
```

Smoke test:

```sh
curl http://weft.voltrum.co/health          # {"status":"ok"}
```

Log in with the seeded account (`admin@example.com` / `admin`) — **change or
delete it** once real users exist.

## How the database wiring works

`cnpg.yaml` bootstraps a `weft` database owned by role `app` and has CNPG
generate the `weft-db-credentials` secret. The Deployment injects that
secret's `uri` key (e.g. `postgresql://app:…@weft-db-rw.weft.svc:5432/weft`)
as `Database__ConnectionString` — Npgsql accepts URI connection strings, so no
secret templating is needed. `Database__Provider=postgres` keeps the app on
the migrate-on-boot path (vs the SQLite test path).

## Notes and follow-ups

- **Replicas stay at 1.** Migrations + seeding run at container start; scale
  up only after boot completes if you want HA (and add real migration
  locking first).
- **No backups yet.** When the data becomes load-bearing, add a
  `barmanObjectStore` with a `retentionPolicy` plus a `ScheduledBackup` to
  `cnpg.yaml` — without the retention policy, base backups + WAL archives
  will eventually fill the storage cluster.
- **Registry quirk.** If Harbor's registry is unavailable (it has a history of
  CrashLooping on the backbone), import the image into the node's containerd
  store directly and set `imagePullPolicy: Never` on the Deployment.

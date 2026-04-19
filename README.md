# infra-image-mirror

Public mirror of upstream infrastructure images to `ghcr.io`, for Kubernetes clusters that need to bootstrap independently of their own in-cluster image registry.

## Background

A Kubernetes cluster that hosts its own container registry can hit a bootstrap deadlock when any of the registry's own runtime dependencies (its cache redis, its database) are configured to pull their images through that same registry. If the registry is down — for whatever reason — those dependencies can't start, which keeps the registry down.

This repo breaks that loop by publishing the affected images to `ghcr.io`, which is always reachable independently of the cluster's own registry. Cluster values/manifests reference these `ghcr.io` tags for the bootstrap-critical components.

## What's mirrored

| Tag | Source type | Upstream |
|---|---|---|
| `redis:8.6.1` | copy | `docker.io/bitnami/redis:8.6.1` |
| `redis-exporter:1.80.1` | copy | `docker.io/bitnami/redis-exporter:1.80.1` |
| `postgres-exporter:0.18.1` | copy | `docker.io/bitnami/postgres-exporter:0.18.1` |
| `cnpg-postgresql:18.3-postgis-vector[-barman]` | in-house build | Base: `ghcr.io/cloudnative-pg/postgresql:18.3-standard-bookworm`. Added layers: PostGIS 3.6.x (apt), pgvector 0.8.2 (compiled from source), barman-cli-cloud + Azure SDK. Dockerfile lives in a separate private cluster-config repo. |

## How it runs

`.github/workflows/sync.yml`:
- **Cron:** weekly, Sundays 03:00 UTC.
- **Manual:** `workflow_dispatch`.
- **`sync-copy` job:** skopeo-copies the three upstream images, preserving multi-arch manifests.
- **`build-cnpg` job:** stub — needs a decision on how this workflow accesses the private Dockerfile (public repo / deploy key / fine-grained PAT). See the commented block in `sync.yml`.

## Package visibility

New GitHub Packages default to **private** on first push. After the first successful workflow run, flip each package to public via the org's Packages tab → click the package → Settings → Danger Zone → Change visibility → Public. This step is not exposed via the API.

Once public, pulls are anonymous — no imagePullSecrets needed on the consuming cluster.

## License

Mirrored images retain their upstream licenses. The CNPG custom build adds layers under the same licenses as the components involved (PostgreSQL, PostGIS, pgvector, barman, Azure SDKs — all permissive OSS).

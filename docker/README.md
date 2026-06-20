# GoBGP container image (Alpine build)

This directory builds a small **Alpine**-based image containing both GoBGP
binaries:

- `gobgpd` — the BGP daemon (image `ENTRYPOINT`)
- `gobgp` — the CLI (on `PATH`, for `docker exec` / debugging)

Published to `ghcr.io/fivetime/gobgp`.

| Tag | Meaning |
| --- | --- |
| `latest` | Newest release tag built |
| `master` | Newest commit on `master` |
| `<git-sha>` | Specific commit on `master` |
| `v<x.y.z>` | Release tag (synced from upstream via `sync-upstream`) |

Images are multi-arch: `linux/amd64`, `linux/arm64`, `linux/arm/v7`,
`linux/386` (built on a single ubuntu-latest runner via QEMU).

## Why this works on Alpine

GoBGP compiles to a fully static binary (`CGO_ENABLED=0`, `-tags netgo`,
matching `.goreleaser.yml`), so the runtime needs no glibc. The image adds only
`ca-certificates` (for gRPC TLS / outbound TLS). No `-dirty` version hacks: the
build version is injected via ldflags into `internal/pkg/version`
(`COMMIT`, `METADATA`), exactly as documented in `CONTRIBUTING.md`.

## Building locally

The build context must be the repo root (the Dockerfile does `COPY . .`):

```bash
# from repo root
docker build -f docker/Dockerfile \
  --build-arg VERSION=dev \
  --build-arg COMMIT="$(git rev-parse --short HEAD)" \
  --build-arg DATE="$(date -u +%Y%m%d)" \
  -t gobgp:dev .
```

Multi-arch (matches CI):

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7,linux/386 \
  -f docker/Dockerfile \
  --build-arg VERSION=dev \
  -t ghcr.io/fivetime/gobgp:dev \
  --push .
```

CI workflows (`build-from-master.yaml` / `build-from-tags.yaml`) set the
build-args and the per-arch matrix automatically.

## Running

`gobgpd` is the entrypoint. With no arguments it starts and waits for
configuration over the gRPC API (`:50051`). To run with a config file, mount it
and pass `-f`:

```bash
docker run --rm --net=host \
  -v "$PWD/gobgpd.toml:/etc/gobgp/gobgpd.toml:ro" \
  ghcr.io/fivetime/gobgp:latest \
  -f /etc/gobgp/gobgpd.toml
```

Then drive it with the bundled CLI:

```bash
docker exec <container> gobgp global rib
docker exec <container> gobgp neighbor
```

Ports: `179/tcp` (BGP), `50051/tcp` (gRPC API). BGP needs the host network (or
`CAP_NET_BIND_SERVICE` / privileged binding to port 179), hence `--net=host`
above. See `example.yml` for a sample config and a Kubernetes Deployment.

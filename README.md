# docker-images

Prebuilt public Docker images on **[ghcr.io/cedricfarinazzo](https://github.com/cedricfarinazzo?tab=packages)**. Built once, reused everywhere — so downstream projects don't waste minutes recompiling heavy native dependencies.

[![release](https://github.com/cedricfarinazzo/docker-images/actions/workflows/release.yml/badge.svg)](https://github.com/cedricfarinazzo/docker-images/actions/workflows/release.yml)
[![nightly py-ta-lib](https://github.com/cedricfarinazzo/docker-images/actions/workflows/nightly-py-ta-lib.yml/badge.svg)](https://github.com/cedricfarinazzo/docker-images/actions/workflows/nightly-py-ta-lib.yml)
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)

## Features

- **Multi-arch**: every image ships `linux/amd64` and `linux/arm64` in the same manifest. Pull works the same on Intel laptops, Apple Silicon, and ARM cloud runners.
- **Matrix builds**: one tag per `(version × distro × interpreter)` combo, plus rolling aliases (`latest`, `latest-<distro>`, `latest-py<py>`, …) so callers can pin as tight or as loose as they want.
- **Nightly rebuilds**: every image is rebuilt at 04:00 UTC with `--no-cache --pull` so upstream base-image CVEs land in a `nightly-*` tag without waiting for a release.
- **Per-image semver**: each image has its own version and changelog. Tags are immutable; rolling aliases move forward.
- **Narrow-COPY friendly**: heavy bits live in known paths (e.g. `/usr/lib/libta_lib.*`) so downstream `Dockerfile`s can pull just what they need with `COPY --from=...`.

## Images

| Image | What | Quick pull |
|-------|------|------------|
| [`py-ta-lib`](./py-ta-lib) | Python `3.12` / `3.13` / `3.14` + [TA-Lib](https://ta-lib.org/) C library `0.6.4` + build toolchain. The `ta-lib` Python binding is **not** preinstalled (release cadence is independent of the C lib — pin it yourself). Debian-slim and Alpine variants. | `docker pull ghcr.io/cedricfarinazzo/py-ta-lib:latest` |
| [`tftp-hpa`](./tftp-hpa) | TFTP server (`tftp-hpa`) on Alpine `3.23`. Multi-arch. Volume at `/data`, exposes UDP 69. | `docker pull ghcr.io/cedricfarinazzo/tftp-hpa:latest` |
| [`git-server`](./git-server) | Self-hosted git server: gitolite ACLs over SSH **and** nginx smart-HTTP. No web UI, transport only. Alpine, supervised by s6-overlay. Multi-arch. Exposes 22 + 80. | `docker pull ghcr.io/cedricfarinazzo/git-server:latest` |
| [`rover`](./rover) | [Apollo Rover CLI](https://www.apollographql.com/docs/rover/) + Supergraph composition plugin baked in. Debian-slim. Multi-arch. Drop-in for federated GraphQL stack bring-up — no per-deploy CLI download. | `docker pull ghcr.io/cedricfarinazzo/rover:latest` |

Each image directory has its own `README.md` with the full tag list, supported build args, and downstream recipes.

## Usage

### As a base image

```dockerfile
FROM ghcr.io/cedricfarinazzo/py-ta-lib:latest
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
```

### Pin to an immutable combo

```bash
docker pull ghcr.io/cedricfarinazzo/py-ta-lib:1.0.0-py3.13-ta0.6.4-alpine
```

### Narrow COPY (don't pull the full image)

```dockerfile
FROM python:3.14-slim
RUN apt-get update && apt-get install -y --no-install-recommends build-essential && rm -rf /var/lib/apt/lists/*
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/lib/libta_lib.*    /usr/lib/
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/include/ta-lib     /usr/include/ta-lib
RUN pip install --no-cache-dir ta-lib==0.6.7
```

## Tag conventions

Every image follows the same scheme:

- **Immutable per combo**: `<semver>-<axis1>-<axis2>-...` (e.g. `1.0.0-py3.14-ta0.6.4-debian`).
- **Rolling aliases**: `latest`, `latest-<distro>`, `latest-py<py>`, semver truncations (`1.0`, `1`), and per-combo rollers.
- **Nightly**: `nightly`, `nightly-<distro>`, `nightly-py<py>`, plus dated immutable snapshots `nightly-YYYYMMDD-...`. Nightly tags never overwrite semver tags.

The image's own README enumerates every alias.

## Reporting issues

Open an issue on this repo with the failing pull command, the image tag, and `docker buildx imagetools inspect` output for the affected tag.

## License

MIT — see [LICENSE](./LICENSE).

## Internals

- [`CLAUDE.md`](./CLAUDE.md) — repo layout, conventions, full release pipeline.
- [`.github/workflows/README.md`](./.github/workflows/README.md) — CI workflow reference.
- [`<image>/`](./py-ta-lib) — per-image source (Dockerfiles, `versions.json`, README).


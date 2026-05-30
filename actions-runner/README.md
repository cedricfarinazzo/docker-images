# actions-runner

GitHub Actions self-hosted runner image extending [`ghcr.io/actions/actions-runner`](https://github.com/actions/runner/pkgs/container/actions-runner) with a full build toolchain baked in.

```bash
docker pull ghcr.io/cedricfarinazzo/actions-runner:latest
```

## What's included

On top of the official runner base (Ubuntu):

| Tool | Details |
|------|---------|
| Core build tools | `build-essential`, `cmake`, `gcc`, `g++`, `make`, `pkg-config` |
| Python | `python3`, `python3-pip`, `python3-venv`, `python-is-python3` |
| Docker | `docker-ce-cli`, `docker-compose-plugin`, `docker-buildx-plugin` |
| Node.js | `22.x` via NodeSource + `yarn` |
| yq | Latest from `mikefarah/yq` (multi-arch) |
| Dev utilities | `jq`, `shellcheck`, `rsync`, `parallel`, `sshpass`, `sqlite3`, `7zip`, and more |

## Tags

| Tag | Description |
|-----|-------------|
| `latest` | Latest release, multi-arch |
| `<semver>` | Immutable release (e.g. `1.0.0`) |
| `<major>.<minor>` | Rolling minor (e.g. `1.0`) |
| `<major>` | Rolling major (e.g. `1`) |
| `nightly` | Daily rebuild from `ghcr.io/actions/actions-runner:latest` |
| `nightly-YYYYMMDD` | Dated nightly snapshot |

## Usage

### In a workflow (runs-on: self-hosted)

```yaml
jobs:
  build:
    runs-on: self-hosted
    container:
      image: ghcr.io/cedricfarinazzo/actions-runner:latest
```

### As a base image

```dockerfile
FROM ghcr.io/cedricfarinazzo/actions-runner:latest
USER root
RUN apt-get update && apt-get install -y --no-install-recommends <extra> && rm -rf /var/lib/apt/lists/*
USER runner
```

## Multi-arch

Every tag ships `linux/amd64` and `linux/arm64` in the same manifest.

## License

MIT

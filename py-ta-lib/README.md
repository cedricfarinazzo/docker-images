# py-ta-lib

Python image with the [TA-Lib](https://ta-lib.org/) C library pre-built and the `ta-lib` Python binding installed. Built as a matrix of Python versions × TA-Lib versions × Linux distros, multi-arch (amd64 + arm64).

**Registry**: `ghcr.io/cedricfarinazzo/py-ta-lib`

## Quick pull

```bash
docker pull ghcr.io/cedricfarinazzo/py-ta-lib:latest
docker run --rm ghcr.io/cedricfarinazzo/py-ta-lib:latest \
  python -c "import talib; print(talib.__version__)"
```

## Matrix

Driven by [`versions.json`](./versions.json):

| Axis    | Values                          | Default |
|---------|---------------------------------|---------|
| Python  | 3.12, 3.13, 3.14                | 3.14    |
| TA-Lib  | 0.6.4                           | 0.6.4   |
| Distro  | debian (slim), alpine           | debian  |
| Arch    | linux/amd64, linux/arm64        | —       |

To add a Python version or distro: edit `versions.json` and commit `feat(py-ta-lib): ...`.

## Tag scheme

### Immutable per-combo tags (every matrix cell)

`<semver>-py<pyver>-ta<taver>-<distro>` and its semver-truncated variants:

| Pattern | Example |
|---------|---------|
| `<semver>-py<py>-ta<ta>-<distro>`        | `1.0.0-py3.14-ta0.6.4-debian` |
| `<major>.<minor>-py<py>-ta<ta>-<distro>` | `1.0-py3.14-ta0.6.4-debian`   |
| `<major>-py<py>-ta<ta>-<distro>`         | `1-py3.14-ta0.6.4-debian`     |
| `py<py>-ta<ta>-<distro>` (rolling)       | `py3.14-ta0.6.4-debian`       |

### Rolling aliases

| Tag | Points to |
|-----|-----------|
| `latest` | latest semver, default python, default distro (currently `1.x-py3.14-ta0.6.4-debian`) |
| `latest-debian`, `latest-alpine` | latest semver, default python, that distro |
| `latest-py3.12`, `latest-py3.13`, `latest-py3.14` | latest semver, that python, default distro |
| `latest-py<py>-<distro>` | latest semver, that combo |
| `<semver>`, `<major>.<minor>`, `<major>` | default combo at that semver |
| `<semver>-<distro>`, `<major>.<minor>-<distro>`, `<major>-<distro>` | default python at that semver + distro |

### Nightly tags

The nightly workflow rebuilds the full matrix daily with `--no-cache --pull` to pick up upstream base-image security fixes. It does **not** touch semver tags.

| Tag | Meaning |
|-----|---------|
| `nightly` | last night's default combo |
| `nightly-debian`, `nightly-alpine` | last night, default python, that distro |
| `nightly-py3.12`, `nightly-py3.13`, `nightly-py3.14` | last night, that python, default distro |
| `nightly-py<py>-ta<ta>-<distro>` | rolling per-combo nightly |
| `nightly-YYYYMMDD-py<py>-ta<ta>-<distro>` | dated immutable nightly snapshot |

## Downstream usage

### As a base image (simplest)

```dockerfile
FROM ghcr.io/cedricfarinazzo/py-ta-lib:latest
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
```

### Narrow-path copy (don't pull whole /usr)

Use only TA-Lib's artefacts in your own image:

```dockerfile
FROM python:3.14-slim

# Pull only TA-Lib bits, not the rest of /usr
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/lib/libta_lib.so* /usr/lib/
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/lib/libta_lib.a   /usr/lib/
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/include/ta-lib    /usr/include/ta-lib

RUN pip install --no-cache-dir ta-lib==0.6.4
```

This avoids inheriting the full base layer when you only need TA-Lib.

## Build args

| Arg              | Default  | Description                                          |
|------------------|----------|------------------------------------------------------|
| `PYTHON_VERSION` | `3.14`   | Python minor (used as `python:${PYTHON_VERSION}-{slim,alpine}`) |
| `TALIB_VERSION`  | `0.6.4`  | TA-Lib C lib version; URL derived from it            |

## Build locally

```bash
# Debian
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f py-ta-lib/Dockerfile.debian \
  --build-arg PYTHON_VERSION=3.14 \
  --build-arg TALIB_VERSION=0.6.4 \
  -t py-ta-lib:test-debian \
  py-ta-lib/

# Alpine
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f py-ta-lib/Dockerfile.alpine \
  --build-arg PYTHON_VERSION=3.14 \
  --build-arg TALIB_VERSION=0.6.4 \
  -t py-ta-lib:test-alpine \
  py-ta-lib/
```

## Upstream sources

- TA-Lib C: <https://github.com/ta-lib/ta-lib>
- ta-lib Python: <https://github.com/TA-Lib/ta-lib-python>
- Python base: <https://hub.docker.com/_/python>

## License

MIT. TA-Lib itself is BSD-licensed; see upstream.

# py-ta-lib

Python image with the [TA-Lib](https://ta-lib.org/) C library pre-built and a working C toolchain. The Python binding (`ta-lib` on PyPI) is **not** pre-installed — its release cadence is independent of the C library, so callers pin their own version.

Built as a matrix of Python versions × TA-Lib C versions × Linux distros, multi-arch (amd64 + arm64).

**Registry**: `ghcr.io/cedricfarinazzo/py-ta-lib`

## Quick pull

```bash
docker pull ghcr.io/cedricfarinazzo/py-ta-lib:latest
docker run --rm ghcr.io/cedricfarinazzo/py-ta-lib:latest \
  sh -c 'ta-lib-config --version && ls /usr/lib/libta_lib*'
```

Verify the Python binding works on top:

```bash
docker run --rm ghcr.io/cedricfarinazzo/py-ta-lib:latest \
  sh -c 'pip install ta-lib==0.6.7 && python -c "import talib; print(talib.__version__)"'
```

## What's in the image

- Python (3.12 / 3.13 / 3.14) from the official `python:<ver>-slim` / `python:<ver>-alpine` base.
- TA-Lib C library `0.6.4` (sha256-verified during the multi-stage build) copied into `/usr`: `libta_lib.so*`, `libta_lib.a`, headers in `/usr/include/ta-lib/`, `ta-lib-config` in `/usr/bin`, `ta-lib.pc` in `/usr/lib/pkgconfig`.
- **No build toolchain.** The multi-stage Dockerfile leaves `build-essential` / `build-base` and `wget` in the builder stage; the runtime image carries the C library only. If you `pip install ta-lib` on top, add the build toolchain in your own layer (see below).
- Sensible Python container defaults (`PYTHONUNBUFFERED=1`, `PYTHONDONTWRITEBYTECODE=1`, `PIP_NO_CACHE_DIR=1`, `PIP_DISABLE_PIP_VERSION_CHECK=1`).
- `TA_LIBRARY_PATH=/usr/lib`, `TA_INCLUDE_PATH=/usr/include`, `LD_LIBRARY_PATH=/usr/lib` so the `ta-lib` Python binding's setup.py finds the headers + lib without extra config.

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

The `ta<ver>` in the tag is the **C library** version. The Python binding version is decided by the downstream `pip install`.

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

### As a base image (debian variant)

```dockerfile
FROM ghcr.io/cedricfarinazzo/py-ta-lib:latest

# Build toolchain is NOT in the base image. Add it for `pip install ta-lib`.
RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/* \
 && pip install --no-cache-dir ta-lib==0.6.7 \
 && apt-get purge -y build-essential && apt-get autoremove -y

COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
```

### As a base image (alpine variant)

```dockerfile
FROM ghcr.io/cedricfarinazzo/py-ta-lib:latest-alpine

RUN apk add --no-cache --virtual .build build-base \
 && pip install --no-cache-dir ta-lib==0.6.7 \
 && apk del .build
```

### Narrow-path copy (don't pull whole /usr)

Use only TA-Lib's artefacts in your own image:

```dockerfile
FROM python:3.14-slim

RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential \
 && rm -rf /var/lib/apt/lists/*

# Pull only TA-Lib bits, not the rest of /usr
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/lib/libta_lib.so* /usr/lib/
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/lib/libta_lib.a   /usr/lib/
COPY --from=ghcr.io/cedricfarinazzo/py-ta-lib:py3.14-ta0.6.4-debian /usr/include/ta-lib    /usr/include/ta-lib

RUN pip install --no-cache-dir ta-lib==0.6.7
```

This avoids inheriting the full base layer when you only need the C library.

## Build args

| Arg              | Default  | Description                                          |
|------------------|----------|------------------------------------------------------|
| `PYTHON_VERSION` | `3.14`   | Python minor (used as `python:${PYTHON_VERSION}-{slim,alpine}`) |
| `TALIB_VERSION`  | `0.6.4`  | TA-Lib C lib version; URL derived from it            |
| `TALIB_SHA256`   | sha256 of v0.6.4 src tarball | sha256 of the TA-Lib tarball; verified after download. Bump together with `TALIB_VERSION`. |

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
- ta-lib Python (install yourself): <https://github.com/TA-Lib/ta-lib-python>
- Python base: <https://hub.docker.com/_/python>

## License

MIT. TA-Lib itself is BSD-licensed; see upstream.

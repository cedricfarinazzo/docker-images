# tftp-hpa

Minimal TFTP server image based on [tftp-hpa](https://git.kernel.org/pub/scm/network/tftp/tftp-hpa.git/) on Alpine. Multi-arch (`linux/amd64` + `linux/arm64`).

**Registry**: `ghcr.io/cedricfarinazzo/tftp-hpa`

## Quick pull

```bash
docker pull ghcr.io/cedricfarinazzo/tftp-hpa:latest
```

## Run

```bash
docker run -d \
  --name tftp \
  -p 69:69/udp \
  -v $(pwd)/tftp-root:/data \
  ghcr.io/cedricfarinazzo/tftp-hpa:latest
```

The TFTP root is `/data`. Mount your own directory there. The container runs `in.tftpd` in foreground (`-L`) with verbose logging (`-vvv`), `--secure` chroot to `/data`, and `--create` so PUTs can create new files. Adjust by overriding `CMD`:

```bash
docker run --rm -it -p 69:69/udp -v $(pwd)/tftp-root:/data \
  ghcr.io/cedricfarinazzo/tftp-hpa:latest \
  in.tftpd -L -v -u root --secure /data           # read-only, no --create
```

## What's in the image

- Alpine `3.23` (axis, see `versions.json`).
- `tftp-hpa` from the Alpine community repo.
- Volume mount point at `/data`.
- `EXPOSE 69/udp`.

## Matrix

| Axis    | Values   | Default |
|---------|----------|---------|
| Alpine  | 3.23     | 3.23    |
| Arch    | amd64, arm64 | — |

To add a newer Alpine version: edit `versions.json` and commit `feat(tftp-hpa): ...`.

## Tag scheme

### Immutable per-combo tags

| Pattern | Example |
|---------|---------|
| `<semver>-alpine<ver>`        | `1.0.0-alpine3.23` |
| `<major>.<minor>-alpine<ver>` | `1.0-alpine3.23`   |
| `<major>-alpine<ver>`         | `1-alpine3.23`     |
| `alpine<ver>` (rolling)       | `alpine3.23`       |

### Rolling aliases

| Tag | Points to |
|-----|-----------|
| `latest` | latest semver, default alpine version |
| `<semver>`, `<major>.<minor>`, `<major>` | default combo at that semver |
| `latest-alpine<ver>` | latest semver, that alpine version |

### Nightly tags

| Tag | Meaning |
|-----|---------|
| `nightly` | last night's default combo |
| `nightly-alpine<ver>` | rolling per alpine version |
| `nightly-YYYYMMDD-alpine<ver>` | dated immutable nightly snapshot |

## Build args

| Arg              | Default  | Description                                  |
|------------------|----------|----------------------------------------------|
| `ALPINE_VERSION` | `3.23`   | Alpine base tag (`alpine:${ALPINE_VERSION}`) |

## Build locally

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f tftp-hpa/Dockerfile \
  --build-arg ALPINE_VERSION=3.23 \
  -t tftp-hpa:test \
  tftp-hpa/
```

## Security note

The default `CMD` runs `in.tftpd` as `root` inside the container with `--create` enabled — anyone able to reach UDP 69 can read **and write** files under `/data`. TFTP has no authentication. Put this behind a network ACL or run with a stricter `CMD` (drop `--create`, use `-u nobody`, restrict the bind interface) if exposed beyond a trusted segment.

## Upstream sources

- tftp-hpa: <https://git.kernel.org/pub/scm/network/tftp/tftp-hpa.git/>
- Alpine base: <https://hub.docker.com/_/alpine>

## License

MIT. tftp-hpa is BSD-licensed; see upstream.

# docker-images

Prebuilt public Docker images on `ghcr.io/cedricfarinazzo`. Multi-arch (`amd64` + `arm64`). Rebuilt nightly for upstream security fixes.

## Images

| Image | What | Quick pull |
|-------|------|------------|
| [`py-ta-lib`](./py-ta-lib) | Python (3.12 / 3.13 / 3.14) + [TA-Lib](https://ta-lib.org/) C library + `ta-lib` Python binding. Debian + Alpine variants. | `docker pull ghcr.io/cedricfarinazzo/py-ta-lib:latest` |

## Usage

As a base image:

```dockerfile
FROM ghcr.io/cedricfarinazzo/py-ta-lib:latest
```

Pick a specific combo (immutable):

```bash
docker pull ghcr.io/cedricfarinazzo/py-ta-lib:1.0.0-py3.13-ta0.6.4-alpine
```

See each image's `README.md` for full tag list, build args, and narrow-COPY recipes.

## License

MIT — see [LICENSE](./LICENSE).

## Contributing / internals

- [`CLAUDE.md`](./CLAUDE.md) — repo layout, conventions, release pipeline.
- [`.github/workflows/README.md`](./.github/workflows/README.md) — CI workflows.

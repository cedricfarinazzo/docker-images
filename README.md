# docker-images

Public monorepo of reusable Docker images published to **GitHub Container Registry** under [`ghcr.io/cedricfarinazzo`](https://github.com/cedricfarinazzo?tab=packages).

Each image lives in its own directory at the repo root and ships its own `Dockerfile(s)`, `README.md`, `versions.json`, and `semantic-release` config. Releases are per-image, driven by Conventional Commits via [`multi-semantic-release`](https://github.com/dhoulb/multi-semantic-release) + [`semantic-release-monorepo`](https://github.com/pmowrer/semantic-release-monorepo).

## Images

| Image | Description | Pull |
|-------|-------------|------|
| [`py-ta-lib`](./py-ta-lib) | Python + [TA-Lib](https://ta-lib.org/) C library, matrix of python versions × TA-Lib versions × distros (debian/alpine), multi-arch (amd64/arm64) | `docker pull ghcr.io/cedricfarinazzo/py-ta-lib:latest` |

## Layout

```
docker-images/
├── .github/workflows/   # CI: release + per-image build + nightly rebuilds
├── py-ta-lib/           # image dir
│   ├── Dockerfile.debian
│   ├── Dockerfile.alpine
│   ├── versions.json
│   ├── package.json
│   ├── .releaserc.json
│   └── README.md
├── .releaserc.base.json # shared semantic-release config
├── package.json         # workspace root
└── LICENSE              # MIT
```

## Release flow

1. **Commit** with conventional commits, **scoped by image dir**:
   - `feat(py-ta-lib): add python 3.15`
   - `fix(py-ta-lib): pin TA-Lib sha256`
2. **Push to `main`** → `release.yml` runs `multi-semantic-release`:
   - Detects which image dirs changed.
   - Computes next semver per image (commits scoped by path).
   - Creates a git tag `py-ta-lib-v1.2.3` + GitHub Release per image.
3. Tag push (`py-ta-lib-v*`) triggers `build-py-ta-lib.yml`:
   - Reads `py-ta-lib/versions.json` to expand build matrix (python × talib × distro).
   - Builds + pushes multi-arch (amd64 + arm64) images to `ghcr.io`.
   - Creates rolling aliases (`latest`, `latest-<distro>`, `latest-py<py>`, etc.).
4. Daily at 04:00 UTC, `nightly-py-ta-lib.yml` rebuilds the whole matrix with `--no-cache --pull` to pick up upstream base-image security fixes, publishing `nightly-*` tags.

## Tag scheme (py-ta-lib)

Full immutable tag: `<semver>-py<pyver>-ta<taver>-<distro>` — e.g. `1.0.0-py3.14-ta0.6.4-debian`.

Rolling aliases:
- `latest` — default combo (latest semver + default python + default distro)
- `latest-debian`, `latest-alpine`
- `latest-py3.14`, `latest-py3.13`, `latest-py3.12`
- `latest-py<py>-<distro>` — every (python, distro) combo
- Semver aliases: `<semver>`, `<major>.<minor>`, `<major>` and their `-<distro>` variants
- `nightly`, `nightly-<distro>`, `nightly-py<py>`, `nightly-py<py>-ta<ta>-<distro>`, `nightly-YYYYMMDD-...`

See [`py-ta-lib/README.md`](./py-ta-lib/README.md) for full per-image details.

## Adding a new image

1. Create `<image-name>/` at repo root.
2. Drop `Dockerfile` (or `Dockerfile.<distro>`), `README.md`, `versions.json`, `package.json`, `.releaserc.json` in it.
3. Add it to the `workspaces` array in root `package.json`.
4. Add a `.github/workflows/build-<image-name>.yml` (copy from `build-py-ta-lib.yml`).
5. Optional: add `nightly-<image-name>.yml` for daily rebuilds.
6. Document it in this README's table and in [`CLAUDE.md`](./CLAUDE.md).
7. Commit with `feat(<image-name>): scaffold image`.

## Conventional Commits

Use the **image directory name as the scope**: `feat(py-ta-lib): ...`, `fix(py-ta-lib): ...`. Commits without a scope (or with an unknown scope) are ignored by per-image release detection.

Types that bump versions:
- `feat:` → minor
- `fix:` / `perf:` → patch
- `BREAKING CHANGE:` footer → major

Other types (`chore:`, `docs:`, `ci:`, `refactor:`, `test:`, `build:`) do not bump.

## License

MIT — see [LICENSE](./LICENSE).

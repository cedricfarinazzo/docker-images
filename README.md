# docker-images

Public monorepo of reusable Docker images published to **GitHub Container Registry** under [`ghcr.io/cedricfarinazzo`](https://github.com/cedricfarinazzo?tab=packages).

Each image lives in its own directory at the repo root and ships its own `Dockerfile(s)`, `README.md`, `versions.json`, `package.json`, and `.releaserc.json`. Releases are per-image, driven by [Conventional Commits](https://www.conventionalcommits.org/) via [`semantic-release`](https://semantic-release.gitbook.io/) + [`semantic-release-monorepo`](https://github.com/pmowrer/semantic-release-monorepo), invoked from each package directory in a loop by the release workflow.

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
│   ├── CHANGELOG.md
│   └── README.md
├── package.json         # workspace root (declares image dirs as workspaces)
├── CLAUDE.md            # Claude Code orientation
└── LICENSE              # MIT
```

## Release flow

1. **Commit** with Conventional Commits, **scoped by image dir**:
   - `feat(py-ta-lib): add python 3.15`
   - `fix(py-ta-lib): pin TA-Lib sha256`
2. **Push to `main`** → `release.yml` (runs on `bun` + `bunx semantic-release`):
   - Reads the `workspaces` array in root `package.json`.
   - For each workspace `cd`s into the directory and runs `bunx semantic-release`.
   - `semantic-release-monorepo` filters the git commit set to commits that touch files inside the package directory.
   - Conventional Commit types in the filtered set determine the next semver bump per package.
   - For packages that get a bump: writes `CHANGELOG.md`, bumps `package.json`, creates a git tag `<pkg>-v<semver>` (e.g. `py-ta-lib-v1.2.3`), and publishes a GitHub Release.
   - After the loop the workflow inspects new tags at HEAD and dispatches the matching `build-<pkg>.yml` (workaround: tags pushed by `GITHUB_TOKEN` don't auto-trigger other workflows).
3. `build-py-ta-lib.yml` (triggered by `workflow_dispatch` from the release workflow, or by a manual `py-ta-lib-v*` tag push):
   - Reads `py-ta-lib/versions.json` to expand the build matrix (python × talib × distro).
   - Builds + pushes multi-arch (amd64 + arm64) images to `ghcr.io`.
   - Creates rolling aliases (`latest`, `latest-<distro>`, `latest-py<py>`, etc.) via `buildx imagetools create`.
4. Daily at 04:00 UTC, `nightly-py-ta-lib.yml` rebuilds the whole matrix with `--no-cache --pull` to pick up upstream base-image security fixes, publishing `nightly-*` tags only (semver tags untouched).

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
2. Drop `Dockerfile` (or `Dockerfile.<distro>`), `README.md`, `versions.json`, `package.json`, `.releaserc.json` in it. Copy from `py-ta-lib/` as a template.
3. Add `<image-name>` to the `workspaces` array in root `package.json`.
4. Add `.github/workflows/build-<image-name>.yml` (copy from `build-py-ta-lib.yml`, adjust tag prefix + dockerfile paths).
5. Optional: add `nightly-<image-name>.yml` for daily rebuilds.
6. Document it in this README's table and in [`CLAUDE.md`](./CLAUDE.md).
7. Commit with `feat(<image-name>): scaffold image`. The release workflow will detect it on next push to `main`.

## Conventional Commits

Use the **image directory name as the scope**: `feat(py-ta-lib): ...`, `fix(py-ta-lib): ...`. `semantic-release-monorepo` filters commits **by file path** — only commits that touch files inside the image's directory count toward that image's release. Scoping the commit message is a convention for readability and changelogs; the path filter is what actually drives per-image bumps.

Types that bump versions:
- `feat:` → minor
- `fix:` / `perf:` → patch
- `BREAKING CHANGE:` footer → major

Other types (`chore:`, `docs:`, `ci:`, `refactor:`, `test:`, `build:`) do not bump.

## License

MIT — see [LICENSE](./LICENSE).

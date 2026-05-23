# CLAUDE.md — docker-images

Orientation for Claude Code sessions in this repository.

## Purpose

Monorepo of public Docker images published to `ghcr.io/cedricfarinazzo/<image>`. Centralises build-heavy bases (e.g. TA-Lib C library) so downstream projects pull prebuilt images instead of recompiling each time.

## Layout

- `py-ta-lib/` — first image. Python + TA-Lib C library, matrix of python × talib × distro.
- `.github/workflows/` — CI: per-image build (on tag), nightly rebuild, semantic-release.
- `.releaserc.base.json` — shared semantic-release config; per-image `.releaserc.json` extends it.
- `package.json` — workspace root, holds `semantic-release` / `multi-semantic-release` deps.

Image directories sit **at the repo root** — there is no `images/` wrapper.

## Conventions

- **Conventional Commits with image-dir scope**: `feat(py-ta-lib): ...`. Scope = directory name. `multi-semantic-release` + `semantic-release-monorepo` filter commits by path; unscoped commits don't bump any image.
- **Per-image versioning**: each image has its own semver. Tags: `<image>-v<semver>` (e.g. `py-ta-lib-v1.2.3`).
- **Matrix metadata** lives in `<image>/versions.json` — single source of truth. Workflows read it to expand build matrices. To add a python or distro variant, edit `versions.json`, do not edit the workflow.
- **Tag scheme** for built images: `<semver>-py<pyver>-ta<taver>-<distro>` plus rolling aliases (see image README).
- **License**: MIT (root `LICENSE`).

## How releases work

1. Push commits to `main`.
2. `.github/workflows/release.yml` runs `multi-semantic-release`. For each image dir whose commits warrant a bump, it creates a tag `<image>-v<X.Y.Z>` and a GitHub Release.
3. Tag push matches `<image>-v*` → `build-<image>.yml` runs: reads `versions.json`, builds the full matrix (python × talib × distro) on amd64 + arm64 via buildx, pushes to `ghcr.io`, then creates rolling aliases (`latest`, `latest-<distro>`, etc.) via `buildx imagetools create`.
4. `nightly-<image>.yml` re-runs the same matrix daily at 04:00 UTC with `--no-cache --pull` to pick up upstream base-image security fixes. Publishes `nightly-*` tags only — never touches semver tags.

## Adding a new image — checklist

1. `mkdir <name>/` at repo root.
2. Files inside:
   - `Dockerfile` (or `Dockerfile.<distro>` if multi-distro)
   - `README.md` — pull command, tag list, downstream usage, build args
   - `versions.json` — matrix axes + defaults
   - `package.json` — `{ "name": "<name>", "version": "0.0.0", "private": true }`
   - `.releaserc.json` — `{ "extends": ["semantic-release-monorepo", "../.releaserc.base.json"] }`
3. Append `<name>` to root `package.json` `workspaces`.
4. Copy `.github/workflows/build-py-ta-lib.yml` → `build-<name>.yml`, adjust tag prefix + dockerfile paths.
5. Copy `nightly-py-ta-lib.yml` → `nightly-<name>.yml` if a daily rebuild is wanted.
6. Update root `README.md` images table.
7. Initial commit: `feat(<name>): scaffold image`. Once merged → tag → first build.

## Commands

- Local build (debian, py3.14): `docker buildx build --platform linux/amd64,linux/arm64 -f py-ta-lib/Dockerfile.debian --build-arg PYTHON_VERSION=3.14 --build-arg TALIB_VERSION=0.6.4 -t py-ta-lib:test py-ta-lib/`
- Quick smoke: `docker run --rm py-ta-lib:test python -c "import talib; print(talib.__version__)"`
- Dry-run a release (does not push tags): `npx multi-semantic-release --dry-run`
- Inspect multi-arch manifest: `docker buildx imagetools inspect ghcr.io/cedricfarinazzo/py-ta-lib:latest`

## Per-dir READMEs

- [`README.md`](./README.md) — public repo intro
- [`.github/README.md`](./.github/README.md) — what's under `.github/`
- [`.github/workflows/README.md`](./.github/workflows/README.md) — workflows reference
- [`py-ta-lib/README.md`](./py-ta-lib/README.md) — py-ta-lib image details

## Jira

[VC-105](https://sedinfra.atlassian.net/browse/VC-105) — origin story for this repo.

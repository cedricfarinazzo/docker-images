# CLAUDE.md — docker-images

Orientation for Claude Code sessions in this repository.

## Purpose

Monorepo of public Docker images published to `ghcr.io/cedricfarinazzo/<image>`. Centralises build-heavy bases (e.g. TA-Lib C library) so downstream projects pull prebuilt images instead of recompiling each time.

## Layout

- `py-ta-lib/` — first image. Python + TA-Lib C library, matrix of python × talib × distro.
- `.github/workflows/` — CI: release (root), per-image build (on tag dispatch), per-image nightly rebuild.
- `package.json` — workspace root, declares image dirs in `workspaces` and holds `semantic-release` + `semantic-release-monorepo` devDependencies. Release CI uses `bun`.

Image directories sit **at the repo root** — there is no `images/` wrapper.

## Conventions

- **Conventional Commits with image-dir scope**: `feat(py-ta-lib): ...`. Scope = directory name. `semantic-release-monorepo` filters commits **by file path** (touches `<pkg>/...`). The scope string is for readability/changelogs; the path filter is what gates per-image bumps.
- **Per-image versioning**: each image has its own semver and `CHANGELOG.md`. Tags: `<image>-v<semver>` (e.g. `py-ta-lib-v1.2.3`).
- **Matrix metadata** lives in `<image>/versions.json` — single source of truth. Workflows read it to expand build matrices. To add a python or distro variant, edit `versions.json`, do not edit the workflow.
- **Tag scheme** for built images: `<semver>-py<pyver>-ta<taver>-<distro>` plus rolling aliases (see image README).
- **License**: MIT (root `LICENSE`).

## How releases work

1. Push commits to `main`.
2. `.github/workflows/release.yml` loops over `workspaces` in root `package.json`, cds into each pkg dir, and runs `bunx semantic-release`. `semantic-release-monorepo` is extended in each pkg's `.releaserc.json` and applies a path filter so only commits touching that pkg dir count toward its bump.
3. For pkgs that get a bump: `CHANGELOG.md` and `package.json` are written, a tag `<pkg>-v<X.Y.Z>` is created, a GitHub Release is published.
4. After the loop, the workflow inspects tags at HEAD and dispatches the matching `build-<pkg>.yml` via `gh workflow run` (workaround: `GITHUB_TOKEN`-pushed tags don't auto-fire other workflows).
5. `build-<pkg>.yml`: reads `versions.json`, expands matrix (python × talib × distro), builds amd64 + arm64 via buildx, pushes to `ghcr.io`, then creates rolling aliases (`latest`, `latest-<distro>`, etc.) via `buildx imagetools create`.
6. `nightly-<pkg>.yml`: daily at 04:00 UTC rebuilds the same matrix with `--no-cache --pull`. Publishes `nightly-*` tags only — never touches semver tags.

## Adding a new image — checklist

1. `mkdir <name>/` at repo root.
2. Files inside:
   - `Dockerfile` (or `Dockerfile.<distro>` if multi-distro)
   - `README.md` — pull command, tag list, downstream usage, build args
   - `versions.json` — matrix axes + defaults
   - `package.json` — `{ "name": "<name>", "version": "0.0.0", "private": true }`
   - `.releaserc.json` — `{ "extends": "semantic-release-monorepo", "branches": ["main"], "plugins": [ "@semantic-release/commit-analyzer", "@semantic-release/release-notes-generator", "@semantic-release/changelog", "@semantic-release/git", "@semantic-release/github" ] }` (must include `commit-analyzer` + `release-notes-generator` so `semantic-release-monorepo` wraps them with the path filter)
3. Append `<name>` to root `package.json` `workspaces`.
4. Copy `.github/workflows/build-py-ta-lib.yml` → `build-<name>.yml`, adjust tag prefix + dockerfile paths.
5. Copy `nightly-py-ta-lib.yml` → `nightly-<name>.yml` if a daily rebuild is wanted.
6. Update root `README.md` images table.
7. Initial commit: `feat(<name>): scaffold image`. Once pushed to `main` → release workflow detects + cuts tag → dispatch fires → first build.

## Commands

- Install dev deps locally: `bun install`
- Dry-run a release for a pkg: `cd py-ta-lib && bunx semantic-release --dry-run`
- Local build (debian, py3.14): `docker buildx build --platform linux/amd64,linux/arm64 -f py-ta-lib/Dockerfile.debian --build-arg PYTHON_VERSION=3.14 --build-arg TALIB_VERSION=0.6.4 -t py-ta-lib:test py-ta-lib/`
- Quick smoke: `docker run --rm py-ta-lib:test python -c "import talib; print(talib.__version__)"`
- Inspect multi-arch manifest: `docker buildx imagetools inspect ghcr.io/cedricfarinazzo/py-ta-lib:latest`

## Per-dir READMEs

- [`README.md`](./README.md) — public repo intro
- [`.github/README.md`](./.github/README.md) — what's under `.github/`
- [`.github/workflows/README.md`](./.github/workflows/README.md) — workflows reference
- [`py-ta-lib/README.md`](./py-ta-lib/README.md) — py-ta-lib image details

## Jira

[VC-105](https://sedinfra.atlassian.net/browse/VC-105) — origin story for this repo.

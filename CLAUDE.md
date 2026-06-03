# CLAUDE.md — docker-images

Orientation for Claude Code sessions in this repository.

## Purpose

Monorepo of public Docker images published to `ghcr.io/cedricfarinazzo/<image>`. Centralises build-heavy bases (e.g. TA-Lib C library) so downstream projects pull prebuilt images instead of recompiling each time.

## Layout

- `py-ta-lib/` — first image. Python + TA-Lib **C library only** (no Python binding, no build toolchain in the runtime image). Multi-stage Dockerfile: builder compiles TA-Lib from a sha256-verified upstream tarball, runtime copies the resulting `libta_lib.*`, headers, `ta-lib.pc`, and `ta-lib-config` into `/usr`. Matrix of python × talib × distro. Downstream callers that `pip install ta-lib` must layer their own build toolchain on top.
- `tftp-hpa/` — TFTP server (`tftp-hpa`) on Alpine. Single Dockerfile, alpine version is the only image axis (plus arch). Runs `in.tftpd` foreground; serves `/data`.
- `git-server/` — Self-hosted git server (gitolite over SSH + nginx smart-HTTP, no web UI). Single Dockerfile on Alpine with s6-overlay supervising sshd / nginx / fcgiwrap. Image carries a `rootfs/` tree merged into the container (`/etc/ssh/sshd_config.d/`, `/etc/nginx/http.d/`, `/etc/s6-overlay/`, `/etc/cont-init.d/`, CGI shim under `/usr/local/bin/`). SSH and HTTP share the **same** gitolite ACL via `REMOTE_USER`.
- `rover/` — Apollo Rover CLI + Supergraph composition plugin on debian-slim. Single Dockerfile parameterised by `ROVER_VERSION` × `SUPERGRAPH_VERSION` × `DEBIAN_VERSION` axes. Multi-arch via `TARGETARCH` (rover supergraph plugin ships per-arch tarballs from `rover.apollo.dev`). `ENTRYPOINT ["rover"]`; pre-sets `APOLLO_ELV2_LICENSE=accept` + `APOLLO_ROVER_SKIP_UPDATE_CHECK=1` so downstream compose files only override `command:`.
- `actions-runner/` — GitHub Actions self-hosted runner. Extends `ghcr.io/actions/actions-runner:latest` (Ubuntu base) with Docker CLI + Compose v2 + Buildx, Node.js 22 + yarn, yq (multi-arch via `TARGETARCH`), and a full build toolchain (`build-essential`, `cmake`, `python3`, `shellcheck`, etc.). No version matrix axes — single variant, two arches (amd64 + arm64). `versions.json` is `{}`. Tags: `latest`, `<semver>`, `<major>.<minor>`, `<major>`, `nightly`, `nightly-YYYYMMDD`.
- `.github/workflows/` — CI: release (root), per-image build (on tag dispatch), per-image nightly rebuild, and `pr-build.yml` (build-only validation of changed images on pull requests — no push, `contents:read` only).
- `package.json` — workspace root, declares image dirs in `workspaces` and holds `semantic-release` + `semantic-release-monorepo` devDependencies. Release CI uses `bun`.

Image directories sit **at the repo root** — there is no `images/` wrapper.

## Conventions

- **Conventional Commits — scope MUST be an image folder name**: `feat(py-ta-lib): ...`, `fix(git-server): ...`, `feat(tftp-hpa)!: ...`. The allowed scopes are exactly the entries in the `workspaces` array of root `package.json`. Generic scopes (`security`, `ci`, `docs`, `release`, `repo`, etc.) are wrong and will leave commits out of any image's changelog.
- **Combining changes across modules is fine.** A single commit may touch files in multiple image directories — `semantic-release-monorepo` filters commits by file path, so each affected image picks the commit up regardless of which one is named in the scope. If the work logically belongs to one image, scope it to that one. If it spans two and you want clean per-image release notes, split into two commits, one per scope.
- **Cross-cutting repo changes** (root `README.md`, `CLAUDE.md`, `.github/workflows/release.yml`, `.github/workflows/pr-build.yml`, root `package.json`) can ride along inside an image-scoped commit using the most-affected image as scope, or stand alone as `chore(<image>): ...` / `docs(<image>): ...` if no real bump is wanted (`chore` / `docs` don't bump).
- **Per-image versioning**: each image has its own semver and `CHANGELOG.md`. Tags: `<image>-v<semver>` (e.g. `py-ta-lib-v1.2.3`).
- **Matrix metadata** lives in `<image>/versions.json` — single source of truth. Workflows read it to expand build matrices. To add a python or distro variant, edit `versions.json`, do not edit the workflow.
- **Tag scheme** for built images: `<semver>-py<pyver>-ta<taver>-<distro>` plus rolling aliases (see image README). `ta<ver>` is the **C library** version shipped in the image; the Python binding version is whatever the downstream caller pins.
- **License**: MIT (root `LICENSE`).
- **GitHub Actions pinned by SHA**: every `uses:` in `.github/workflows/*.yml` is pinned to a full 40-char commit SHA, with the human-readable tag in a trailing comment, e.g. `actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5 # v4`. Mutable tags like `@v4` are forbidden — they let action authors silently re-point to new code. Renovate's `github-actions` manager recognises the `<owner>/<repo>@<sha> # <tag>` pattern and bumps both the SHA and the comment when new tags ship.

## How releases work

1. Push commits to `main`.
2. `.github/workflows/release.yml` loops over `workspaces` in root `package.json`, cds into each pkg dir, and runs `bunx semantic-release`. `semantic-release-monorepo` is extended in each pkg's `.releaserc.json` and applies a path filter so only commits touching that pkg dir count toward its bump.
3. For pkgs that get a bump: `CHANGELOG.md` and `package.json` are written, a tag `<pkg>-v<X.Y.Z>` is created, a GitHub Release is published.
4. After the loop, the workflow inspects tags at HEAD and dispatches the matching `build-<pkg>.yml` via `gh workflow run` (workaround: `GITHUB_TOKEN`-pushed tags don't auto-fire other workflows).
5. `build-<pkg>.yml`: reads `versions.json`, expands matrix (python × talib × distro) and fans each combo out to **native amd64** (`ubuntu-latest`) and **native arm64** (`ubuntu-24.04-arm`) runners (no QEMU). Each leg pushes by digest only; a `merge` matrix job (one per combo) downloads both digests and runs `buildx imagetools create` to publish the full per-combo multi-arch manifest. `aliases` runs after `merge` and creates the rolling aliases (`latest`, `latest-<distro>`, `latest-py<py>`, `latest-py<py>-<distro>`, semver truncations + their `-<distro>` variants).
6. `nightly-<pkg>.yml`: same per-arch / merge / aliases shape as the build workflow, but daily at 04:00 UTC with `--no-cache --pull`. Publishes `nightly-*` tags only — never touches semver tags.

## Adding a new image — checklist

> The workflow filename must be `build-<workspace-name>.yml` (and optionally `nightly-<workspace-name>.yml`). `release.yml`'s dispatch step constructs the filename from the package name, so this is load-bearing.

### 1. Create the image directory

`mkdir <name>/` at repo root. `<name>` is the workspace name (used as scope in commits, in tag prefix `<name>-v...`, in workflow filenames).

### 2. Per-image files

#### `<name>/Dockerfile` (or `Dockerfile.<distro>` per distro)

Single-stage or multi-stage as appropriate. Parameterise with `ARG`s for the axes declared in `versions.json` (e.g. `PYTHON_VERSION`, `TALIB_VERSION`). Add OCI labels at the end:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/cedricfarinazzo/docker-images" \
      org.opencontainers.image.licenses="MIT" \
      org.opencontainers.image.title="<name>"
```

#### `<name>/versions.json`

```json
{
  "<axis1>": ["val1", "val2"],
  "<axis2>": ["..."],
  "distros": ["debian", "alpine"],
  "default_<axis1>": "val1",
  "default_<axis2>": "...",
  "default_distro": "debian"
}
```

Workflows read this file to expand the build matrix and to decide which combo gets the top-level rolling aliases.

#### `<name>/package.json`

```json
{
  "name": "<name>",
  "version": "0.0.0",
  "private": true,
  "description": "<one-liner>",
  "repository": { "type": "git", "url": "https://github.com/cedricfarinazzo/docker-images.git", "directory": "<name>" },
  "license": "MIT"
}
```

#### `<name>/.releaserc.json`

```json
{
  "extends": "semantic-release-monorepo",
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/changelog", { "changelogFile": "CHANGELOG.md" }],
    ["@semantic-release/git", {
      "assets": ["CHANGELOG.md", "package.json"],
      "message": "chore(release): ${nextRelease.gitTag} [skip ci]\n\n${nextRelease.notes}"
    }],
    "@semantic-release/github"
  ]
}
```

Required: `commit-analyzer` and `release-notes-generator` MUST be in the plugins array. `semantic-release-monorepo` wraps **every plugin in the array** with its path filter — if `commit-analyzer` is absent the filter has nothing to wrap and no commits get classified.

Do **not** pre-create `CHANGELOG.md`. semantic-release writes it on first release.

#### `<name>/README.md`

Pull command, full tag list, downstream usage (incl. narrow-COPY recipe), supported build args, matrix table, upstream sources, license. Copy `py-ta-lib/README.md` as a template.

### 3. Register the workspace

Append `"<name>"` to `workspaces` in root `package.json`.

### 4. Workflow files

#### `.github/workflows/build-<name>.yml`

Copy `build-py-ta-lib.yml`. **Keep the SHA pins on every `uses:` line** — don't replace them with `@v<N>` tags. If you need a new action, resolve its SHA first:

```bash
gh api repos/<owner>/<repo>/git/refs/tags/<tag> --jq '.object.sha'
```

then write it as `uses: <owner>/<repo>@<sha> # <tag>`. Renovate will keep both in sync going forward.

Adjust:

- `name:` → `build-<name>`
- `on.push.tags` → `<name>-v*.*.*`
- `env.IMAGE` → `ghcr.io/cedricfarinazzo/<name>`
- `setup` step: `F=<name>/versions.json` (both places)
- `setup` matrix jq: rewrite the axis loops if axes differ from py-ta-lib's `python × talib × distros`. Output objects must still include `arch` + `runs_on` keys.
- `setup` per-axis output lines: emit the lists needed by the aliases job
- `build` step `file:` → `<name>/Dockerfile.<distro>` (or `<name>/Dockerfile` if not multi-distro)
- `build` step `build-args:` → the axes in your `versions.json`
- `cache-from` / `cache-to` scope → `<name>-...`
- `merge` job tag computation: rewrite `SUFFIX` and `TAG_ARGS` to match your axes
- `aliases` job: rewrite the alias generation loops to match the rolling-alias shape you want

#### `.github/workflows/nightly-<name>.yml` (optional)

Copy `nightly-py-ta-lib.yml`. Same adjustments as the build workflow. Drop if a daily rebuild isn't needed.

#### `.github/workflows/pr-build.yml` (shared — register the new image)

`pr-build.yml` is **one shared workflow**, not per-image. It build-validates changed images on PRs (`push: false`, no registry login). To wire in a new image, edit two hardcoded spots in it:

- add the image dir to `on.pull_request.paths`, and
- append an entry to the `DEFS` array in the `detect` job (one per Dockerfile — `py-ta-lib` has two: `Dockerfile.debian` + `Dockerfile.alpine`), with `name` / `image` / `context` / `file`.

`detect` diffs the PR against its base and builds only the images whose dir changed (a change to `pr-build.yml` itself rebuilds everything), each on **native amd64 + arm64**. Build args use the Dockerfile `ARG` defaults — no `versions.json` matrix here; the full python × talib × distro matrix stays in `build-<name>.yml`.

### 5. Document it

- Add a row to the images table in root `README.md`.
- Mention the image in this CLAUDE.md `Layout` section.

### 6. First commit

```
feat(<name>): scaffold image
```

On push to `main`:
- `release.yml` detects the new workspace, finds the `feat:` commit touching `<name>/`, cuts tag `<name>-v1.0.0`, publishes the GitHub Release.
- The dispatch step in `release.yml` calls `gh workflow run build-<name>.yml --ref <name>-v1.0.0 -f semver=1.0.0`.
- `build-<name>.yml` runs per-arch on native runners, merges into multi-arch manifests, creates aliases.
- If `nightly-<name>.yml` exists, it kicks in at the next 04:00 UTC cron tick.

## Commands

- Install dev deps locally: `bun install`
- Dry-run a release for a pkg: `cd py-ta-lib && bunx semantic-release --dry-run`
- Local build (debian, py3.14): `docker buildx build --platform linux/amd64,linux/arm64 -f py-ta-lib/Dockerfile.debian --build-arg PYTHON_VERSION=3.14 --build-arg TALIB_VERSION=0.6.4 -t py-ta-lib:test py-ta-lib/`
- Quick smoke (C lib only): `docker run --rm py-ta-lib:test sh -c 'ta-lib-config --version && ls /usr/lib/libta_lib*'`
- Quick smoke (with downstream binding): `docker run --rm py-ta-lib:test sh -c 'pip install ta-lib==0.6.7 && python -c "import talib; print(talib.__version__)"'`
- Inspect multi-arch manifest: `docker buildx imagetools inspect ghcr.io/cedricfarinazzo/py-ta-lib:latest`

## Per-dir READMEs

- [`README.md`](./README.md) — public repo intro
- [`.github/workflows/README.md`](./.github/workflows/README.md) — workflows reference
- [`py-ta-lib/README.md`](./py-ta-lib/README.md) — py-ta-lib image details


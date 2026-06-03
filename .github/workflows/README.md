# Workflows

GitHub Actions for the monorepo. One release workflow at the root; two workflows per image (build on tag + nightly rebuild); plus one shared PR workflow that build-validates changed images without pushing. The build/nightly jobs are structured as `setup → build → merge → aliases` so per-arch legs run on native runners and are stitched into multi-arch manifests by digest.

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| [`release.yml`](./release.yml) | `push` to `main` + `workflow_dispatch` | Loops over `workspaces` in root `package.json`, cds into each pkg dir, runs `bunx semantic-release` (with `semantic-release-monorepo` providing the path filter). For each pkg that bumps, creates a tag `<pkg>-v<semver>` + GitHub Release. After the loop, dispatches `build-<pkg>.yml` via `gh workflow run` for tags created at HEAD. |
| [`build-py-ta-lib.yml`](./build-py-ta-lib.yml) | `push` tag `py-ta-lib-v*.*.*` + `workflow_dispatch` (with `semver` input) | Reads `py-ta-lib/versions.json`. Fans the python × talib × distro matrix out across **native amd64** (`ubuntu-latest`) and **native arm64** (`ubuntu-24.04-arm`) runners — no QEMU. Each leg pushes by digest (`push-by-digest=true`, `name-canonical=true`). A `merge` matrix job (one per combo) downloads both digests and runs `buildx imagetools create` to publish the combo's full tag list (semver + truncations + rolling `py<py>-ta<ta>-<distro>`). An `aliases` job then creates the rolling aliases (`latest`, `latest-<distro>`, `latest-py<py>`, `latest-py<py>-<distro>`, semver truncations + `-<distro>` variants). |
| [`nightly-py-ta-lib.yml`](./nightly-py-ta-lib.yml) | `schedule: '0 4 * * *'` + `workflow_dispatch` | Same native per-arch fan-out / merge / aliases shape as `build-py-ta-lib.yml`, but daily with `pull: true` + `no-cache: true` to pick up upstream base-image security fixes. Publishes `nightly-*` tags only (never overwrites semver tags). |
| [`pr-build.yml`](./pr-build.yml) | `pull_request` (paths-filtered to image dirs) | **Shared, build-only.** A `detect` job diffs the PR against its base and emits a matrix of just the changed images (a change to `pr-build.yml` itself rebuilds everything); each builds on **native amd64 + arm64** with `push: false` and no registry login. Build args fall back to the Dockerfile `ARG` defaults — no `versions.json` matrix. Proves Dockerfiles still compile on both arches before merge. |

> Note on dispatch: tags pushed by the auto-issued `GITHUB_TOKEN` (which is what semantic-release uses) don't trigger downstream workflows. That's why `release.yml` calls `gh workflow run` at the end — it's the workaround instead of requiring a personal access token.

## Native per-arch matrix details

- `setup` reads `versions.json` and emits two matrices in its `outputs`:
  - `build_matrix` — one entry per `(combo × arch)`, with `runs_on: ubuntu-latest` for amd64 and `runs_on: ubuntu-24.04-arm` for arm64.
  - `combo_matrix` — one entry per `combo` (no arch); consumed by the `merge` job.
- `build` runs in parallel across all `(combo × arch)` entries. Each pushes a single-platform image by digest only, no tag. Digests are uploaded as `actions/upload-artifact` named `digests-py<py>-ta<ta>-<distro>-<arch>` so the `merge` job can find them by combo prefix.
- `merge` runs once per combo. It downloads every artifact matching `digests-py<py>-ta<ta>-<distro>-*` (i.e. both archs of that combo), then runs `docker buildx imagetools create -t <tag1> -t <tag2> ... <image>@sha256:<digest1> <image>@sha256:<digest2>` to publish the multi-arch manifest list with the full per-combo tag set.
- `aliases` runs after `merge` succeeds and uses `buildx imagetools create --tag <alias> <image>:<canonical>` to point each rolling alias at the freshly-published canonical tag. The default combo's manifest is used for top-level aliases (`latest`, `<semver>`, etc.).

Cache scopes are per-arch (`type=gha,scope=<image>-<py>-<ta>-<distro>-<arch>`) so amd64 and arm64 builds don't fight over the same layer cache.

## Secrets / permissions

All workflows use the auto-issued `GITHUB_TOKEN`. No external secrets needed.

| Workflow | Permissions |
|----------|-------------|
| `release.yml` | `contents: write`, `issues: write`, `pull-requests: write`, `actions: write` |
| `build-*.yml` | `contents: read`, `packages: write` |
| `nightly-*.yml` | `contents: read`, `packages: write` |
| `pr-build.yml` | `contents: read` (build-only, no push) |

## Adding a new image's workflows

1. Copy `build-py-ta-lib.yml` → `build-<image>.yml`:
   - Adjust `on.push.tags` to `<image>-v*.*.*`.
   - Adjust `IMAGE` env, matrix-source `versions.json` path, and the `file:` argument in the build step (`<image>/Dockerfile.<distro>`).
   - Adjust the per-combo suffix used in tag computation if the image's axes differ from `py × ta × distro`.
2. Copy `nightly-py-ta-lib.yml` → `nightly-<image>.yml` (same adjustments).
3. No changes needed to `release.yml` — it iterates the `workspaces` array in root `package.json`, so adding the new image directory to that array is enough. The post-release dispatch step expects the build workflow file to be named `build-<workspace-name>.yml`.
4. Register the image in the shared `pr-build.yml`: add its dir to `on.pull_request.paths` and append an entry (one per Dockerfile) to the `DEFS` array in the `detect` job.

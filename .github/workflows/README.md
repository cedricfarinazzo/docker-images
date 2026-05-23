# Workflows

GitHub Actions for the monorepo. One release workflow at the root; two workflows per image (build on tag + nightly rebuild).

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| [`release.yml`](./release.yml) | `push` to `main` + `workflow_dispatch` | Loops over `workspaces` in root `package.json`, cds into each pkg dir, runs `bunx semantic-release` (with `semantic-release-monorepo` providing the path filter). For each pkg that bumps, creates a tag `<pkg>-v<semver>` + GitHub Release. After the loop, dispatches `build-<pkg>.yml` via `gh workflow run` for tags created at HEAD. |
| [`build-py-ta-lib.yml`](./build-py-ta-lib.yml) | `push` tag `py-ta-lib-v*` + `workflow_dispatch` (with `semver` input) | Reads `py-ta-lib/versions.json`, expands python × talib × distro matrix, buildx-builds amd64 + arm64, pushes to `ghcr.io`, then creates rolling aliases (`latest`, `latest-<distro>`, `latest-py<py>`, `latest-py<py>-<distro>`, semver aliases). |
| [`nightly-py-ta-lib.yml`](./nightly-py-ta-lib.yml) | `schedule: '0 4 * * *'` + `workflow_dispatch` | Same matrix as the build workflow but with `--no-cache --pull` to pick up upstream base-image security fixes. Publishes `nightly-*` tags only (never overwrites semver tags). |

> Note on dispatch: tags pushed by the auto-issued `GITHUB_TOKEN` (which is what semantic-release uses) don't trigger downstream workflows. That's why `release.yml` calls `gh workflow run` at the end — it's the workaround instead of requiring a personal access token.

## Secrets / permissions

All workflows use the auto-issued `GITHUB_TOKEN`. No external secrets needed.

| Workflow | Permissions |
|----------|-------------|
| `release.yml` | `contents: write`, `issues: write`, `pull-requests: write`, `actions: write` |
| `build-*.yml` | `contents: read`, `packages: write` |
| `nightly-*.yml` | `contents: read`, `packages: write` |

## Adding a new image's workflows

1. Copy `build-py-ta-lib.yml` → `build-<image>.yml`:
   - Adjust `on.push.tags` to `<image>-v*.*.*`.
   - Adjust `IMAGE` env, matrix-source `versions.json` path, and dockerfile path in the build step.
2. Copy `nightly-py-ta-lib.yml` → `nightly-<image>.yml` (same adjustments).
3. No changes needed to `release.yml` — it iterates the `workspaces` array in root `package.json`, so adding the new image directory to that array is enough. The post-release dispatch step expects the build workflow file to be named `build-<workspace-name>.yml`.

# Workflows

GitHub Actions for the monorepo. One release workflow at the root; two workflows per image (build on tag + nightly rebuild).

## Workflows

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| [`release.yml`](./release.yml) | `push` to `main` | Runs `multi-semantic-release`. Detects which image dirs changed (via path-scoped conventional commits) and cuts a per-image tag `<image>-v<semver>` + GitHub Release. |
| [`build-py-ta-lib.yml`](./build-py-ta-lib.yml) | `push` tag `py-ta-lib-v*` | Reads `py-ta-lib/versions.json`, expands python × talib × distro matrix, buildx-builds amd64 + arm64, pushes to `ghcr.io`, then creates rolling aliases (`latest`, `latest-<distro>`, `latest-py<py>`, `latest-py<py>-<distro>`, semver aliases). |
| [`nightly-py-ta-lib.yml`](./nightly-py-ta-lib.yml) | `schedule: '0 4 * * *'` + `workflow_dispatch` | Same matrix as the build workflow but with `--no-cache --pull` to pick up upstream base-image security fixes. Publishes `nightly-*` tags only (never overwrites semver tags). |

## Secrets / permissions

All workflows use the auto-issued `GITHUB_TOKEN`. No external secrets needed.

| Workflow | Permissions |
|----------|-------------|
| `release.yml` | `contents: write`, `issues: write`, `pull-requests: write` |
| `build-*.yml` | `contents: read`, `packages: write` |
| `nightly-*.yml` | `contents: read`, `packages: write` |

## Adding a new image's workflows

1. Copy `build-py-ta-lib.yml` → `build-<image>.yml`:
   - Adjust `on.push.tags` to `<image>-v*.*.*`.
   - Adjust the `image_name` / matrix-source path / dockerfile path env.
2. Copy `nightly-py-ta-lib.yml` → `nightly-<image>.yml` (same adjustments).
3. No changes needed to `release.yml` — `multi-semantic-release` auto-detects new image dirs via the root `workspaces` list.

# project-standards

Shared maintenance standards for spdolan's repositories, kept in one place so a fix lands everywhere on the next run instead of in a PR per repository:

- **Reusable workflows**: a conventional PR-title check, release-please with line tags, and language-agnostic CI through mise.
- **Templates**: a PR template and issue forms, copied into each repository.
- **A Renovate preset**: weekly grouped updates with `chore(deps)` titles.
- **Starting configs**: mise, Biome, ruff and `.editorconfig` files to copy.

The `repo-standards` skill in [spdolan/skills](https://github.com/spdolan/skills) (plugin `maintenance`) audits a repository and wires these in. This README is its contract: the inputs and versions below are what it uses.

## Versions

Releases are cut by release-please from conventional PR titles (this repository runs its own `release-please.yml`). Callers pin the **line tag**, `@v0.1`, which moves to every non-breaking release; a breaking change (`feat!:`) starts `v0.2`, and callers move by editing the ref. Pin a full commit sha instead if a repository must never change under you; Renovate then proposes updates (the preset groups them into one PR).

Tags here are `vX.Y.Z` without a component prefix, unlike other spdolan repositories, because they are refs people type in `uses:` lines, where the GitHub Actions convention is `@v1`.

## Reusable workflows

Copy a caller from `templates/workflows/` (`ci.yml`, `pr-title.yml`, `release.yml`) into the repository's `.github/workflows/`.

### `pr-title.yml`

Fails unless the PR title is a conventional commit: `<type>(<scope>)<!>: <summary>`, or a GitHub `Revert "…"` title. No token, no third-party action.

```yaml
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
permissions: {}
jobs:
  title:
    uses: spdolan/project-standards/.github/workflows/pr-title.yml@v0.1
```

| Input | Default |
|---|---|
| `types` | `feat\|fix\|chore\|docs\|ci\|refactor\|test\|perf\|build\|style\|revert` |

### `ci-mise.yml`

Checks out, installs the caller's `mise.toml` toolchain with `jdx/mise-action`, runs mise tasks in order, then summarises JUnit files in the job summary (pass or fail). The caller defines the tasks; the convention is `lint` (no writes), `fmt`, `test` (JUnit to `reports/`) and `check`.

```yaml
on:
  push:
    branches: [main]
  pull_request:
permissions:
  contents: read
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  check:
    uses: spdolan/project-standards/.github/workflows/ci-mise.yml@v0.1
```

| Input | Default |
|---|---|
| `tasks` | `lint test` |
| `working-directory` | `.` |
| `report-paths` | `reports/junit*.xml` (relative to `working-directory`) |
| `runs-on` | `ubuntu-24.04` |
| `timeout-minutes` | `15` |

A reusable workflow can't take service containers, system packages or extra permissions from the caller. Repositories that need them (Postgres, GDAL, GitHub Packages) keep their own CI and use `jdx/mise-action` and the same tasks directly.

### `release-please.yml`

Runs release-please in manifest mode (`release-please-config.json`, `.release-please-manifest.json` in the caller) and, with `line-tag: true`, moves a line tag for each released package: `v0.N` before 1.0 and `vN` after, prefixed with the component when tags carry one (`ts-v0.2`).

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
concurrency:
  group: release
  cancel-in-progress: false
permissions:
  contents: write
  pull-requests: write
jobs:
  release:
    uses: spdolan/project-standards/.github/workflows/release-please.yml@v0.1
    with:
      line-tag: true
    secrets:
      RELEASE_APP_CLIENT_ID: ${{ secrets.FACTORY_RELEASE_APP_CLIENT_ID }}
      RELEASE_APP_PRIVATE_KEY: ${{ secrets.FACTORY_RELEASE_APP_PRIVATE_KEY }}
  publish:
    needs: release
    if: needs.release.outputs.release_created == 'true'
    # ... build and publish from needs.release.outputs.tag_name
```

| Input | Default |
|---|---|
| `config-file` | `release-please-config.json` |
| `manifest-file` | `.release-please-manifest.json` |
| `line-tag` | `false` |

| Secret (optional) | |
|---|---|
| `RELEASE_APP_CLIENT_ID`, `RELEASE_APP_PRIVATE_KEY` | A GitHub App installed on the caller. Its token opens the release PR, so CI runs on it. Without them, `GITHUB_TOKEN` is used: it works, but GitHub starts no workflows on PRs or tags it creates. spdolan's repositories pass the factory release app's `FACTORY_RELEASE_APP_*` secrets. |

| Output | |
|---|---|
| `release_created`, `tag_name` | the root package (path `.`) |
| `releases_created`, `paths_released` | any package; JSON array of paths |
| `outputs` | every release-please output as JSON: `fromJSON(needs.release.outputs.outputs)['ts--release_created']` |

Publish from a job in the caller gated on these outputs, not from a tag-triggered workflow. The caller must grant `contents: write` and `pull-requests: write`, because a called workflow can't have more than its caller. The repository also needs squash merges with the PR title as the commit message, and, on the `GITHUB_TOKEN` fallback, Settings → Actions → General → "Allow GitHub Actions to create and approve pull requests".

## PR template and issue forms

`.github/pull_request_template.md` and `.github/ISSUE_TEMPLATE/` (bug, feature) are this repository's own and the copies other repositories take: copy them into the repository's `.github/` and adapt the testing checklist to its commands. GitHub would apply them account-wide only from a repository named exactly `.github`; this one is named for what it holds instead, so each repository carries its own copy (the `repo-standards` skill does the copying).

## Renovate preset

`default.json`. Use it with:

```json
{ "$schema": "https://docs.renovatebot.com/renovate-schema.json", "extends": ["github>spdolan/project-standards"] }
```

It groups minor, patch, pin and digest updates into one weekly PR, pins action digests, waits 3 days after a release, and titles PRs `chore(deps): …` so they pass the PR-title check and don't release on their own. Workflows from this repository are left on their line tag (not converted to digests) and get their own PR, without the wait, when a new line appears. Packages that spdolan/factory's `dependency-bump` stage updates go in the repository's own `ignoreDeps`. Renovate runs only where the Renovate GitHub App is installed.

## Starting configs

`templates/`: `mise/` (`python-uv.toml`, `ts-bun.toml`, `ts-node.toml`, `go.toml`), `biome.json`, `ruff.toml`, `editorconfig` (save as `.editorconfig`). Copy and adapt them, keeping the versions a repository already pins.

## Develop

```sh
mise install
mise run lint      # actionlint (with shellcheck on run blocks) and zizmor
```

CI runs the same task through `ci-mise.yml`, so a change to it is exercised before release. Use a conventional PR title; squash-merge.

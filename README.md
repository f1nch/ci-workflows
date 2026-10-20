# ci-workflows

Reusable GitHub Actions workflows for my Node/TypeScript apps.

## node-ci

Pull-request CI for any npm or pnpm app. It runs the app's own `package.json` scripts, so each app decides *how* it typechecks, lints, tests and builds; the workflow only makes sure it happens.

| Job | What it does |
| --- | --- |
| `checks` | Install with the lockfile frozen (fails if it's out of date), then run `typecheck`, `lint`, `test`, `build` in order. A missing script is skipped with a warning instead of failing. |
| `audit` | `npm audit` / `pnpm audit`, failing at `high` or above by default. |
| `e2e` | Optional. Runs an end-to-end script (e.g. Playwright) after an optional setup command. |

The package manager is picked from the lockfile (`pnpm-lock.yaml` or `package-lock.json`). For pnpm, the version comes from the `packageManager` field in `package.json`. The Node version comes from the `node-version` input, then `.nvmrc` / `.node-version`, then the latest LTS.

### Use it

Add `.github/workflows/ci.yml` to the app:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1' # weekly: audit only, so new advisories show up without a code change
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  ci:
    uses: f1nch/ci-workflows/.github/workflows/node-ci.yml@v1
```

That's enough for most apps. Pass inputs under `with:` to change the defaults. [examples/pnpm-app-ci.yml](examples/pnpm-app-ci.yml) is a pnpm monorepo with Playwright e2e.

### Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `node-version` | `''` | Node version. Empty = `.nvmrc` / `.node-version`, else latest LTS. |
| `working-directory` | `.` | Folder with `package.json` and the lockfile. |
| `scripts` | `typecheck lint test build` | Scripts the `checks` job runs, in order. |
| `audit-level` | `high` | `low`, `moderate`, `high` or `critical`. Empty skips the audit job. |
| `audit-prod-only` | `false` | Only audit production dependencies. |
| `e2e-script` | `''` | Script for end-to-end tests. Empty = no e2e job. |
| `e2e-setup` | `''` | Command run before e2e, e.g. `npx playwright install --with-deps chromium`. |

### Blocking merges

To stop a PR merging while CI is red, add `ci / checks`, `ci / audit` (and `ci / e2e` if used) as required status checks in the app repo's branch protection for `main`. The names show up there after the first run. On a free plan this only works for public repos.

### Scheduled runs

On `schedule` events only the `audit` job runs; `checks` and `e2e` are skipped.

## Versions

Callers pin a major tag (`@v1`). Non-breaking fixes move the `v1` tag forward; breaking changes (renamed inputs, different defaults) get `v2`.

To release a fix:

```sh
git tag -f v1 && git push -f origin v1
```

[`self-test.yml`](.github/workflows/self-test.yml) runs `node-ci` against the small npm app in [`test/npm-app`](test/npm-app) on every PR and push to `main`. It has no `lint` script on purpose, to test the skip path.

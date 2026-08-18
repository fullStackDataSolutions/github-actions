# Centralized CI/CD Workflows

This repository contains centralized GitHub Actions workflows for CI/CD across all FullStackDataSolutions projects. It provides a reusable typecheck/lint/test/build pipeline plus optional e2e support, so backend and frontend projects on the same stack (Node/pnpm, NestJS or Next.js) don't each reinvent CI from scratch.

## Purpose

- Centralized management of CI/CD pipelines
- Consistent quality gates across all projects
- Reduced duplication and maintenance overhead
- Standardized tooling and processes

## Architecture

Only two workflows here are actually reusable (`workflow_call`): `ci.yml` and `quality.yml`. There is no `backend-ci.yml`/`frontend-ci.yml` in this repo to call directly - those live **in each consuming project's own `.github/workflows/`**, as thin project-specific callers into `ci.yml`. `templates/backend-ci.yml` and `templates/frontend-ci.yml` are the canonical starting point for a new project's copy of those files.

```
consuming project's .github/workflows/backend-ci.yml (from templates/backend-ci.yml)
  -> calls this repo's ci.yml (workflow_call)
       -> calls this repo's quality.yml (typecheck/lint/test/build)
       -> optionally runs the e2e job, using .github/actions/e2e
```

### Why the project-level file isn't just `paths:` filtered

The obvious way to skip backend CI on a frontend-only PR (and vice versa) is a trigger-level `paths:` filter on `on.pull_request`. **Don't do that** if this check is going to be a branch-protection required status check: a workflow that never triggers produces no check run at all, and GitHub's required-status-checks feature can't distinguish "correctly doesn't apply here" from "still pending" - it just waits forever, permanently blocking the PR's merge button.

The templates instead detect the relevant path change in a `changes` job that always runs (no trigger-level filter), and gate the real work with a job-level `if:`. A job skipped via `if:` still reports a "skipped" check under its own name, which GitHub does treat as satisfying a required check - but that only works for a job with its own steps. It does **not** work for a job that only `uses:` a reusable workflow (like the `backend-ci:`/`frontend-ci:` jobs that call `ci.yml`): skipping the *caller* means the reusable workflow never runs, so its nested checks (e.g. `backend-ci / quality / quality`) never get created either - same problem, one level deeper.

That's why each template ends with a small always-running aggregator job (`backend-ci-required` / `frontend-ci-required`): it has its own steps, so it always reports something - a trivial pass when the relevant path wasn't touched, or the real result when it was. **Point this project's branch-protection required status check at that aggregator job's name, never at a nested `<job> / <inner-job>` name.**

## Available Workflows

### `ci.yml`

Orchestrates `quality.yml` plus an optional Playwright e2e job. Called from a project's `backend-ci.yml`/`frontend-ci.yml` (see `templates/`), not called directly by application code.

### `quality.yml`

Runs typecheck, lint, test, and build for one `working-directory`. If the caller passes `nextjs-cache: true`, it also caches and uploads the `.next` build output - but **only when `e2e-enabled: true`** is also passed. Nothing downloads that artifact unless the e2e job actually runs, so a caller that never needs e2e (a NestJS backend with its own dedicated e2e job, or a frontend PR with no e2e-relevant changes) should pass `e2e-enabled: false` to skip the upload. Uploading an unused ~100MB artifact on every single run is exactly what exhausts a repo's Actions storage quota and starts failing otherwise-green quality jobs at the upload step - if you see `Failed to CreateArtifact: Artifact storage quota has been hit`, this is almost always why.

### `.github/actions/e2e`

Composite action used by `ci.yml`'s `e2e` job: downloads the `quality.yml`-uploaded Next.js build, boots a Postgres service, optionally runs a project-supplied setup command (for a sibling backend service, migrations, etc.), then runs Playwright.

## Using These Workflows

1. Copy `templates/backend-ci.yml` and/or `templates/frontend-ci.yml` into the consuming project's `.github/workflows/`, adjusting the customization points marked in each file's comments (e2e image/setup, e2e-relevant path list, Node/pnpm versions).
2. Set that project's branch-protection required status checks to `backend-ci-required` / `frontend-ci-required` - the aggregator jobs, not any nested check name.
3. Confirm the project has `typecheck`, `lint`, `test`, and `build` scripts in `package.json` (see below).

### Required Scripts

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "test": "jest",
    "build": "next build"
  }
}
```

(Backend projects typically use `nest build`/plain `tsc` instead of `next build` for `build`, and don't set `nextjs-cache`/e2e artifact upload at all.)

## Prerequisites

### Self-Hosted Runners

These workflows require **self-hosted runners** (`runs-on: self-hosted`) in the consuming repository.

## Customization

### `ci.yml` inputs

| Input | Description | Default | Type |
|-------|-------------|---------|------|
| `working-directory` | Directory containing the app (`backend`, `frontend`, ...) | *(required)* | string |
| `node-version` | Node.js version | *(required)* | string |
| `pnpm-version` | pnpm version | *(required)* | string |
| `nextjs-cache` | Cache and upload the `.next` build (frontend only) | `false` | boolean |
| `e2e-enabled` | Whether the e2e job may run, and whether `quality.yml` uploads its build artifact for it | `true` | boolean |
| `e2e-setup-command` | Shell command/script run before e2e tests (e.g. to boot a sibling backend) | `''` | string |
| `e2e-postgres-user`/`e2e-postgres-password`/`e2e-postgres-db`/`e2e-postgres-image` | e2e job's Postgres service config | `postgres`/`postgres`/`postgres`/`postgres:18` | string |
| `e2e-infisical-environment` | Infisical environment slug for e2e secrets, empty to skip | `''` | string |
| `e2e-setup-cache-path` | Sibling directory whose `node_modules` should be cached for the setup command | `''` | string |
| `e2e-setup-build-command` | Command to prebuild a sibling directory before the setup command runs | `''` | string |

Set `e2e-enabled: false` for any caller (or any specific run, via a `needs`-derived expression) that never needs the e2e job - this also skips the otherwise-unconditional Next.js build upload.

## Support

For issues or questions, please contact the DevOps team.

Built for [Andrew O](https://fullstackdatasolution.slack.com/archives/D0ADK99J8M9/p1777329521750599) by [Kilo for Slack](https://kilo.ai/features/slack-integration)

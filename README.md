# Centralized CI/CD Workflows

This repository contains centralized GitHub Actions workflows for CI/CD across all FullStackDataSolutions projects. It provides reusable workflows for both backend and frontend projects, ensuring consistent quality checks and build processes.

## Purpose

- Centralized management of CI/CD pipelines
- Consistent quality gates across all projects
- Reduced duplication and maintenance overhead
- Standardized tooling and processes

## Available Workflows

### Backend CI (`backend-ci.yml`)

Runs the following checks on backend services:

1. **Typecheck** - Validates TypeScript types
2. **Lint** - Code quality and style checks
3. **Build** - Compiles the application
4. **Test** - Runs test suite

### Frontend CI (`frontend-ci.yml`)

Runs the following checks on frontend applications:

1. **Typecheck** - Validates TypeScript/JavaScript types
2. **Lint** - Code quality and style checks
3. **Build** - Bundles and builds the application
4. **Test** - Runs test suite

## Using These Workflows

Both workflows are designed to be called from other repositories using `workflow_call`.

### Backend Workflow

Add this to your backend project's `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  backend-ci:
    uses: fullStackDataSolutions/github-actions/.github/workflows/backend-ci.yml@main
    with:
      node-version: '20'
      pnpm-version: '9'
    secrets: inherit
```

### Frontend Workflow

Add this to your frontend project's `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  frontend-ci:
    uses: fullStackDataSolutions/github-actions/.github/workflows/frontend-ci.yml@main
    with:
      node-version: '20'
      pnpm-version: '9'
    secrets: inherit
```

## Prerequisites

### Self-Hosted Runners

Both workflows require **self-hosted runners** to be configured in the consuming repository. This is because certain build steps may require specific environment dependencies.

To set up a self-hosted runner:

1. Go to repository Settings → Actions → Runners
2. Click "Add new runner"
3. Follow the installation instructions for your platform
4. Ensure the runner is labeled appropriately (e.g., `self-hosted`, `linux`, `ubuntu-latest`)

### Required Scripts

Your project must have the following scripts defined in `package.json`:

#### Backend Projects

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "build": "tsc && vite build",
    "test": "vitest run"
  }
}
```

#### Frontend Projects

```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint .",
    "build": "vite build",
    "test": "vitest run"
  }
}
```

## Setup Instructions

### For Repository Admins

1. Clone this repository locally:

   ```bash
   git clone https://github.com/fullStackDataSolutions/github-actions.git
   cd github-actions
   ```

2. Modify workflows as needed in `.github/workflows/`

3. Commit and push changes:

   ```bash
   git add .
   git commit -m "Update CI workflows"
   git push origin main
   ```

### For Project Teams

1. Ensure your project has the required scripts in `package.json` (see above)

2. Create workflow files in your project:

   ```bash
   mkdir -p .github/workflows
   ```

   Add `ci.yml` with the appropriate workflow reference (see "Using These Workflows" section)

3. Set up self-hosted runners in your repository (see Prerequisites)

4. Test the workflow by opening a PR or pushing to a branch

## Customization

### Input Parameters

Both workflows accept the following optional inputs:

| Input | Description | Default | Type |
|-------|-------------|---------|------|
| `node-version` | Node.js version to use | `20` | string |
| `pnpm-version` | pnpm version to use | `9` | string |

Example with custom versions:

```yaml
jobs:
  frontend-ci:
    uses: fullStackDataSolutions/github-actions/.github/workflows/frontend-ci.yml@main
    with:
      node-version: '22'
      pnpm-version: '10'
```

## Support

For issues or questions, please contact the DevOps team.

Built for [Andrew O](https://fullstackdatasolution.slack.com/archives/D0ADK99J8M9/p1777329521750599) by [Kilo for Slack](https://kilo.ai/features/slack-integration)

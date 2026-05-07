# TypeScript / JavaScript CI/CD

[<- Back to README](../README.md) | [View workflow](../workflows/ci-typescript.yml)

---

## Overview

This workflow runs on every push and pull request to your main branches. It sets up a **matrix** across Node.js 20, 22, and 24, caches npm dependencies, and runs your lint, type check, test, and build scripts.

## Pipeline Stages

```
push / PR
  |
  |- ci (Node 20) --> lint -> typecheck -> test -> build
  |- ci (Node 22) --> lint -> typecheck -> test -> build
  |- ci (Node 24) --> lint -> typecheck -> test -> build
        |
        |- docker (optional, main only) --> build & push to GHCR
        |- deploy (optional, main only) --> your deploy commands
        |- release (optional, on v* tag) --> GitHub Release
```

## Prerequisites

The workflow expects these npm scripts in your `package.json` (all optional - steps are skipped if the script doesn't exist):

| Script       | Purpose                        | Example tool            |
|-------------|--------------------------------|-------------------------|
| `lint`       | Linting                        | ESLint, Biome           |
| `typecheck`  | Static type analysis           | `tsc --noEmit`          |
| `test`       | Run tests                      | Vitest, Jest            |
| `build`      | Production build               | tsc, Vite, Next.js      |

## Using yarn or pnpm

Replace two things in the workflow:

**yarn:**
```yaml
cache: "yarn"
# ...
run: yarn install --frozen-lockfile
```

**pnpm:**
```yaml
# Add this step before setup-node:
- uses: pnpm/action-setup@v4
  with:
    version: 9

# In setup-node, change cache:
- uses: actions/setup-node@v6
  with:
    node-version: ${{ matrix.node-version }}
    cache: "pnpm"

# Replace npm ci with:
- name: Install dependencies
  run: pnpm install --frozen-lockfile
```

## Adding Coverage

Uncomment the Codecov step in the workflow, then:

1. Sign up at [codecov.io](https://codecov.io) and add your repo
2. Copy the `CODECOV_TOKEN`
3. Add it as a secret in your repo: **Settings -> Secrets -> Actions -> New repository secret**
4. Make sure your test script outputs coverage (e.g., `vitest --coverage`)

## Enabling Docker

Uncomment the `docker` job. It uses **GitHub Container Registry** (ghcr.io) by default - no extra secrets needed since it uses `GITHUB_TOKEN`.

Your image will be available at:
```
ghcr.io/your-username/your-repo:latest
ghcr.io/your-username/your-repo:<commit-sha>
```

## Creating Releases

1. Uncomment the `release` job
2. Tag your commit: `git tag v1.0.0 && git push --tags`
3. The workflow creates a GitHub Release with auto-generated release notes

## Secrets Reference

| Secret          | Required | Used by        | Where to get it                    |
|----------------|----------|----------------|------------------------------------|
| `GITHUB_TOKEN`  | No*      | Docker push    | Automatic - provided by Actions    |
| `CODECOV_TOKEN` | Optional | Coverage upload | [codecov.io](https://codecov.io)   |
| `VERCEL_TOKEN`  | Optional | Deploy         | Vercel dashboard -> Settings -> Tokens |
| `DEPLOY_KEY`    | Optional | Deploy         | Your hosting provider              |

*`GITHUB_TOKEN` is automatically available in every workflow run.

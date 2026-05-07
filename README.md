<div align="center">

# 🚀 CI/CD Starter Templates

**Production-ready, heavily commented GitHub Actions workflows you can drop into any project.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-v6-2088FF?logo=githubactions&logoColor=white)](https://github.com/features/actions)

[Quick Start](#-quick-start) | [Templates](#-templates) | [Docs](docs/) | [Contributing](CONTRIBUTING.md)

</div>

---

## Why?

Every new project needs CI. Every time, you end up copying a workflow from another repo, ripping out the project-specific parts, and hoping you didn't break anything. These templates fix that - they're generic, self-documenting, and ready to go in under a minute.

Every line is commented. You don't need to read separate docs to understand what's happening - the workflow file itself explains every trigger, step, flag, and option. Great for learning GitHub Actions or onboarding teammates.

## ✨ What's Included

| | Template | Lint | Type Check | Test | Build | Security | Docker | Deploy | Release |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 🟨 | [TypeScript / JavaScript](workflows/ci-typescript.yml) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 🐹 | [Go](workflows/ci-go.yml) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 🐍 | [Python](workflows/ci-python.yml) | ✅ | ✅ | ✅ | - | ✅ | ✅ | ✅ | ✅ |

**Active by default:** lint, type check, test, build, concurrency control, dependency caching, matrix testing.
**Ready to uncomment:** security scanning, coverage uploads, Docker, deploy, release.

## 🏁 Quick Start

**1. Pick a template and copy it:**

```bash
mkdir -p .github/workflows

# TypeScript / JavaScript
curl -o .github/workflows/ci.yml \
  https://raw.githubusercontent.com/YOUR_USER/ci-templates/main/workflows/ci-typescript.yml

# Go
curl -o .github/workflows/ci.yml \
  https://raw.githubusercontent.com/YOUR_USER/ci-templates/main/workflows/ci-go.yml

# Python
curl -o .github/workflows/ci.yml \
  https://raw.githubusercontent.com/YOUR_USER/ci-templates/main/workflows/ci-python.yml
```

**2. Commit and push:**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add CI/CD workflow"
git push
```

**3. That's it.** CI runs on every push and PR. Read the comments in the file to understand each piece and customize it.

## 📖 Templates

### TypeScript / JavaScript

Matrix testing across **Node 20, 22, 24** with npm caching. Runs lint -> type check -> test -> build out of the box. Security audit, Docker, deploy, and release are commented out and ready to enable. Every step explains *why* it's there - from why `npm ci` beats `npm install` to why `npm run test` is used instead of `npm test`.

**Tools:** ESLint/Biome (your choice), TypeScript, Vitest/Jest, npm audit (optional)
**Package managers:** npm by default, comments explain how to switch to yarn or pnpm

📄 [Workflow](workflows/ci-typescript.yml) | 📘 [Full docs](docs/typescript.md)

### Go

Uses **golangci-lint** for linting, **go vet** for static analysis, tests with **race detection** and coverage reporting. Comments explain every flag (`-race`, `-covermode=atomic`, `-ldflags="-s -w"`). Optional **govulncheck** step for vulnerability scanning with reachability analysis. Includes an optional cross-compile job that builds binaries for 5 OS/arch targets and attaches them to GitHub Releases.

**Tools:** golangci-lint, go vet, go test, govulncheck (optional)
**Targets:** linux/amd64, linux/arm64, darwin/amd64, darwin/arm64, windows/amd64

📄 [Workflow](workflows/ci-go.yml) | 📘 [Full docs](docs/go.md)

### Python

Matrix testing across **Python 3.11, 3.12, 3.13**. Lints with **ruff** (check + format), type checks with **mypy**, tests with **pytest + coverage**. Optional **pip-audit** step for dependency vulnerability scanning. Comments explain why versions need quoting in YAML, what `|| true` does for mypy, and how trusted publishing works. Includes a **PyPI publish** job using OIDC - no API tokens needed.

**Tools:** ruff, mypy, pytest, pytest-cov, pip-audit (optional)
**Package managers:** pip by default, docs cover poetry and uv

📄 [Workflow](workflows/ci-python.yml) | 📘 [Full docs](docs/python.md)

### Deploying to Your Server

New to deployment? The **[deployment guide](docs/deploying.md)** walks you through setting up SSH keys, GitHub Secrets, and server access step by step. Covers SSH/rsync deploys, Docker deploys, and cloud platforms (Vercel, Fly.io, AWS).

### Deploying to Kubernetes

The **[Kubernetes guide](docs/kubernetes.md)** covers deploying to K8s clusters from GitHub Actions. Includes self-hosted clusters with kubeconfig, managed clusters (AWS EKS, Google GKE, Azure AKS) with OIDC auth, Helm deployments with atomic rollback, RBAC setup for least-privilege service accounts, and automatic rollback on failure.

## 🔧 Customisation Cheatsheet

| I want to...                        | Do this                                                                 |
|-----------------------------------|-------------------------------------------------------------------------|
| Change language versions          | Edit the `matrix` values in the `strategy` block                        |
| Use yarn instead of npm           | Change `cache: "npm"` -> `cache: "yarn"` and `npm ci` -> `yarn install`  |
| Add environment variables         | Add them under the top-level `env:` block                               |
| Push Docker images                | Uncomment the `docker` job - uses GHCR by default                       |
| Deploy on push to main            | Uncomment the `deploy` job and add your deploy commands                  |
| Create releases on tags           | Uncomment the `release` job, then push a tag like `v1.0.0`              |
| Add coverage reporting            | Uncomment the Codecov step and add `CODECOV_TOKEN` to your repo secrets |
| Enable security scanning          | Uncomment the security audit step and/or the `dependency-review` job    |
| Run on more branches              | Edit the `branches` list in the `on:` trigger                           |
| Require approval before deploy    | Set up an environment in Settings -> Environments                        |
| Deploy to Kubernetes              | See the [Kubernetes guide](docs/kubernetes.md) for kubectl, Helm, and OIDC |
| Publish a Python package          | Uncomment `publish` job + set up trusted publishing on pypi.org          |

## 🔩 Action Versions

All templates use the latest stable versions as of May 2026:

| Action | Version | Notes |
|---|---|---|
| `actions/checkout` | v6 | Improved credential storage |
| `actions/setup-node` | v6 | Auto-detects npm caching |
| `actions/setup-python` | v6 | Node 24 runtime |
| `actions/setup-go` | v6 | Reads `toolchain` from go.mod |
| `docker/build-push-action` | v7 | **Required** - v5 broke with GitHub Cache API v2 |
| `docker/setup-buildx-action` | v4 | |
| `docker/login-action` | v4 | |
| `golangci/golangci-lint-action` | v6 | |
| `softprops/action-gh-release` | v2 | |
| `actions/dependency-review-action` | v4 | PR-only, flags new vulnerable deps |
| `codecov/codecov-action` | v4 | |

## 📁 Repo Structure

```
ci-templates/
|-- .github/
|   +-- dependabot.yml       keeps action versions up to date automatically
|-- README.md
|-- LICENSE
|-- CONTRIBUTING.md
|-- workflows/
|   |-- ci-typescript.yml    ~78% comments, reads like a tutorial
|   |-- ci-go.yml            ~81% comments
|   +-- ci-python.yml        ~75% comments
+-- docs/
    |-- typescript.md        yarn/pnpm setup, Codecov, Docker, secrets
    |-- go.md                cross-compile, Dockerfile, build flags
    |-- python.md            poetry/uv, PyPI trusted publishing, ruff/mypy config
    |-- deploying.md         SSH keys, server access, cloud platforms
    +-- kubernetes.md        EKS, GKE, AKS, Helm, RBAC, rollback
```

## 🤝 Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on adding new templates or improving existing ones.

The short version: keep it generic, comment everything, and make sure commented-out sections are valid YAML when uncommented.

## 📄 License

[MIT](LICENSE) - use these templates however you like.

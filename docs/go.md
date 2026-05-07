# Go CI/CD

[<- Back to README](../README.md) | [View workflow](../workflows/ci-go.yml)

---

## Overview

This workflow lints with golangci-lint, runs `go vet`, then tests with **race detection** and coverage. Optionally cross-compiles binaries for multiple platforms and attaches them to GitHub Releases.

## Pipeline Stages

```
push / PR
  |
  |- ci --> lint (golangci-lint) -> vet -> test (-race) -> build
              |
              |- docker (optional, main only) --> build & push to GHCR
              |- deploy (optional, main only) --> your deploy commands
              |- on v* tag:
                   |- build-binaries --> cross-compile for 5 targets
                   |- release --> GitHub Release with binaries attached
```

## Cross-Compilation Targets

The optional `build-binaries` job compiles for:

| OS      | Architecture | Output                     |
|---------|-------------|----------------------------|
| Linux   | amd64       | `myapp-linux-amd64`        |
| Linux   | arm64       | `myapp-linux-arm64`        |
| macOS   | amd64       | `myapp-darwin-amd64`       |
| macOS   | arm64       | `myapp-darwin-arm64`       |
| Windows | amd64       | `myapp-windows-amd64.exe`  |

To add more targets, extend the `matrix.include` list.

## Customising the Linter

The workflow uses [golangci-lint](https://golangci-lint.run/) with default settings. To customise, add a `.golangci.yml` at the root of your repo:

```yaml
# .golangci.yml
linters:
  enable:
    - govet
    - errcheck
    - staticcheck
    - gosimple
    - unused
    - gocritic
    - gofumpt

linters-settings:
  gocritic:
    enabled-tags:
      - diagnostic
      - style
      - performance

issues:
  exclude-use-default: false
```

## Adding Coverage

Uncomment the Codecov step in the workflow, then:

1. Sign up at [codecov.io](https://codecov.io) and add your repo
2. Add `CODECOV_TOKEN` as a repo secret
3. Coverage is already output to `coverage.out` by the test step

## Enabling Docker

Uncomment the `docker` job. A typical Go Dockerfile with multi-stage build:

```dockerfile
FROM golang:1.24-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app/server .

FROM alpine:3.20
RUN apk add --no-cache ca-certificates
COPY --from=build /app/server /server
ENTRYPOINT ["/server"]
```

## Creating Releases with Binaries

1. Uncomment both `build-binaries` and `release` jobs
2. Tag your commit: `git tag v1.0.0 && git push --tags`
3. The workflow compiles binaries for all platforms and creates a GitHub Release with them attached

## Build Flags

The cross-compile job uses `-ldflags="-s -w"` to strip debug info and reduce binary size. To inject version info at build time, add:

```yaml
run: |
  go build -ldflags="-s -w -X main.version=${{ github.ref_name }}" -o ...
```

Then in your Go code:

```go
var version = "dev"
```

## Secrets Reference

| Secret          | Required | Used by        | Where to get it                  |
|----------------|----------|----------------|----------------------------------|
| `GITHUB_TOKEN`  | No*      | Docker, Release | Automatic - provided by Actions  |
| `CODECOV_TOKEN` | Optional | Coverage upload | [codecov.io](https://codecov.io) |
| `DEPLOY_KEY`    | Optional | Deploy         | Your hosting provider            |

*`GITHUB_TOKEN` is automatically available in every workflow run.

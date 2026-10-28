# Shared GA - Reusable Workflows

> Centralized reusable workflows for consistent CI/CD across all repositories

## Quick Start

Reference any workflow with the `uses` keyword:

```yaml
name: Check

on:
  pull_request:
    branches: [main, dev]

jobs:
  go-check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
      lint: true
    secrets: inherit

  go-security:
    uses: duynhlab/gha-workflows/.github/workflows/go-security.yml@main
    with:
      go-version-file: go.mod
    permissions:
      actions: read
      contents: read
      pull-requests: read
      security-events: write
```

Complete caller workflows for a Go service: **[examples/](examples/)**.

## Available Workflows

| Workflow | Purpose | Features |
|----------|---------|----------|
| **[pr-checks.yml](.github/workflows/pr-checks.yml)** | PR handling | Branch validation, CODEOWNERS, Slack notifications |
| **[go-check.yml](.github/workflows/go-check.yml)** | Go code quality | Unit + integration tests, linting, coverage summaries |
| **[gitleaks.yml](.github/workflows/gitleaks.yml)** | Secret scanning (source) | Gitleaks CLI, PR diff / full scan, SARIF, job summary |
| **[go-security.yml](.github/workflows/go-security.yml)** | **Security entrypoint for Go services** | CodeQL + Govulncheck + Dependency Review behind one stable `Security Gate` check |
| **[codeql.yml](.github/workflows/codeql.yml)** | CodeQL analysis (one language per call) | `go` (autobuild/manual) or `actions` (none), `security-extended`, per-language SARIF category |
| **[govulncheck.yml](.github/workflows/govulncheck.yml)** | Go dependency vulnerabilities | Reachability analysis, pinned tool version, real blocking gate, SARIF |
| **[dependency-review.yml](.github/workflows/dependency-review.yml)** | PR dependency + license policy | Blocks new vulnerable dependencies, license audit split from the vuln gate |
| **[docker-build-go.yml](.github/workflows/docker-build-go.yml)** | Docker build (Go) | **Scan-before-push**, immutable tags, multi-platform, caching, provenance |
| **[docker-build-node.yml](.github/workflows/docker-build-node.yml)** | Docker build (Node) | **Scan-before-push**, same contract as the Go builder, also publishes `:latest` |
| **[trivy-scan.yml](.github/workflows/trivy-scan.yml)** | Image vulnerability report | Trivy post-push scan, SARIF, Google Sheets reporting |
| **[docker-sign.yml](.github/workflows/docker-sign.yml)** | Cosign image signing | Keyless OIDC signing |
| **[goreleaser.yml](.github/workflows/goreleaser.yml)** | Go binary release (on `v*` tags) | Archives, `checksums.txt`, cosign signature, syft SBOMs |
| **[sonarqube.yml](.github/workflows/sonarqube.yml)** | SonarCloud analysis | Go coverage (unit + integration), Quality Gate |
| **[tf-lint.yml](.github/workflows/tf-lint.yml)** | Terraform validation | Format check, TFLint analysis |
| **[status.yml](.github/workflows/status.yml)** | Build notifications | Slack, Google Sheets, job summaries |
| **[delete-workflow-runs.yml](.github/workflows/delete-workflow-runs.yml)** | Delete old Actions runs | Manual cleanup, retain window, dry run by default |

> **Pipeline pattern:** the builder workflows scan with Trivy **before** pushing, so an image only
> reaches GHCR if it passes. `trivy-scan.yml` remains available for optional post-push SARIF
> reporting.

## Documentation

| Document | What's in it |
|----------|--------------|
| **[docs/workflows.md](docs/workflows.md)** | Reference — inputs, outputs, secrets, usage for every workflow |
| **[docs/actions.md](docs/actions.md)** | Composite actions used internally |
| **[docs/architecture.md](docs/architecture.md)** | Pipeline diagrams: PR, push to main, scheduled scan |
| **[docs/go-security.md](docs/go-security.md)** | Security contract: policy matrix, rulesets, rollout, exceptions |
| **[docs/configuration.md](docs/configuration.md)** | Required secrets, prerequisites, CODEOWNERS |
| **[docs/troubleshooting.md](docs/troubleshooting.md)** | Common failures and fixes |
| **[examples/](examples/)** | Copy-paste caller workflows for a Go service |

Start at **[docs/](docs/)** for the full index.

## Required Secrets

`SLACK_BOT_TOKEN` and `SONAR_TOKEN` are required by the workflows that use them;
`GSHEET_CLIENT_EMAIL` / `GSHEET_PRIVATE_KEY` are optional for Google Sheets reporting. The
security workflows need **no secrets at all**.

Full table and setup steps: [docs/configuration.md](docs/configuration.md#required-secrets).

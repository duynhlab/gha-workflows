# Workflow Usage Guide

Detailed documentation for all available workflows with inputs, outputs, and examples.

---

## pr-checks.yml

**Purpose:** Pull request validation and notifications
**Trigger:** PR events only
**Features:** Branch validation (gitflow prefixes), CODEOWNERS tagging, Slack event notifications

> Requires a valid CODEOWNERS file in the caller repo for owner mentions in Slack. See [CODEOWNERS](#codeowners) for format and placement.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `slack_channel_id` | string | - | **Yes** | Slack channel ID to post PR notifications (e.g. `C0AD82A9A74`) |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Outputs

| Output | Description |
|--------|-------------|
| `slack_thread_ts` | Slack thread timestamp for reply threading (pass to `status.yml`) |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `SLACK_BOT_TOKEN` | **Yes** | Slack bot token for authentication |

### Usage

```yaml
jobs:
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: duynhlab/gha-workflows/.github/workflows/pr-checks.yml@main
    with:
      slack_channel_id: "C0AD82A9A74"
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## go-check.yml

**Purpose:** Go code quality assurance
**Trigger:** PR events
**Features:** Unit testing (with optional stable Go matrix), linting, coverage artifacts, coverage job summaries

When tests produce a coverage profile (`coverage.out` / `coverage-integration.out`), the workflow writes a **job summary** (same pattern as Gitleaks) with total coverage and a collapsible per-package breakdown. Artifacts are still uploaded for SonarCloud.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `command-test` | string | - | **Yes** | Test command to execute |
| `setup-go` | boolean | `true` | No | Install Go automatically |
| `gomod-path` | string | `"go.mod"` | No | Path to go.mod file |
| `lint` | boolean | `false` | No | Enable linting |
| `lint-path` | string | `".golangci.yml"` | No | Lint config file path |
| `lint-timeout` | string | `"10m"` | No | Lint timeout duration |
| `lint-version` | string | `"v2.6.0"` | No | golangci-lint version |
| `test-stable` | boolean | `false` | No | Also test against stable Go version (non-blocking compatibility check) |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Usage

**Basic:**
```yaml
jobs:
  go-check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test ./...'
    secrets: inherit
```

**With linting and stable Go matrix:**
```yaml
jobs:
  go-check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test -race -coverprofile=coverage.out -covermode=atomic ./...'
      lint: true
      lint-version: 'v2.6.0'
      test-stable: true
    secrets: inherit
```

---

## gitleaks.yml

**Purpose:** Secret scanning for source code and git history
**Trigger:** Caller-defined (recommended on PR + push, excluding tags)
**Features:** Gitleaks CLI binary (MIT, open-source), auto PR diff / full scan, SARIF upload to GitHub Security tab, job summary

> Uses a composite action (`.github/actions/gitleaks/`) that downloads the gitleaks binary directly. No license key required -- works for both personal and organization repositories.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `config-path` | string | `""` | No | Path to custom `.gitleaks.toml` |
| `gitleaks-version` | string | `"8.21.2"` | No | Gitleaks CLI version to download |
| `scan-mode` | string | `"auto"` | No | `auto` (PR=diff, push=full), `full`, or `diff` |

### Outputs

| Output | Description |
|--------|-------------|
| `exit-code` | Gitleaks exit code (`0`=clean, `1`=leaks found) |
| `sarif` | Path to SARIF report file |
| `summary` | Number of findings |

### Usage

**PR block, push warn (recommended):**
```yaml
jobs:
  gitleaks:
    if: "!startsWith(github.ref, 'refs/tags/')"
    uses: duynhlab/gha-workflows/.github/workflows/gitleaks.yml@main
    continue-on-error: ${{ github.event_name != 'pull_request' }}
    with:
      config-path: '.gitleaks.toml' # optional
    permissions:
      contents: read
      security-events: write
```

---

## docker-build-go.yml

**Purpose:** Build a Go service Docker image, scan for vulnerabilities, and push to GHCR only if clean
**Features:** **Scan-before-push** (Trivy), multi-platform builds, registry caching, provenance, SBOM, default tagging via `docker/metadata-action`

> This is the **Go-specific builder**. It outputs `tags`, `digest`, and `scan-status` that can be consumed by `docker-sign.yml` and optionally `trivy-scan.yml` (for reporting). For other stacks, see `docker-build-node.yml`, `docker-build-python.yml`, etc. with the same output interface.

> **Security**: When `scan-before-push` is enabled (default), the image is built locally (`--load`), scanned with Trivy, and only pushed to GHCR if no vulnerabilities matching the severity filter are found. This prevents FluxCD from auto-deploying vulnerable images.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `image-name` | string | - | **Yes** | Docker image name (without registry prefix) |
| `dockerfile` | string | `"Dockerfile"` | No | Path to Dockerfile |
| `context` | string | `"."` | No | Docker build context |
| `push` | boolean | `false` | No | Push image to registry |
| `platforms` | string | `"linux/amd64"` | No | Target platforms |
| `build-args` | string | `""` | No | Build-time variables |
| `tags` | string | `""` | No | Custom tags (comma-separated); empty = default tagging |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `sbom` | boolean | `false` | No | Generate SBOM attestation |
| `scan-before-push` | boolean | `true` | No | Scan image with Trivy before pushing to registry |
| `scan-severity` | string | `"CRITICAL,HIGH"` | No | Trivy severity filter for pre-push scan |
| `scan-exit-code` | string | `"1"` | No | Trivy exit code on vulnerability found (`0`=warn, `1`=block push) |
| `scan-ignore-unfixed` | boolean | `true` | No | Ignore vulnerabilities without a fix available |

### Outputs

| Output | Description |
|--------|-------------|
| `tags` | Generated image tags (newline-separated) |
| `digest` | Image digest (`sha256:...`) |
| `scan-status` | Pre-push scan result: `pass`, `fail`, or `skipped` |

### Usage

**With scan-before-push (default, recommended):**
```yaml
jobs:
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-go.yml@main
    with:
      image-name: my-service
      push: true
      # scan-before-push: true      # default
      # scan-severity: 'CRITICAL,HIGH'  # default
      # scan-exit-code: '1'          # default — blocks push on CVEs
    secrets: inherit
    permissions:
      contents: read
      packages: write
      actions: read
```

**Without scan (opt-out):**
```yaml
jobs:
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-go.yml@main
    with:
      image-name: my-service
      push: true
      scan-before-push: false
    secrets: inherit
    permissions:
      contents: read
      packages: write
      actions: read
```

> **Note**: `--load` (used for local scanning) only supports single-platform builds. If `platforms` is `linux/amd64` (default), this works. For multi-arch builds, the amd64 image is scanned locally; the multi-arch push only proceeds if the scan passes.

---

## docker-sign.yml

**Purpose:** Sign Docker images using Cosign (keyless / OIDC)
**Features:** Keyless signing via Sigstore OIDC, signs all tags for a given digest

> Chain after any builder workflow that outputs `tags` and `digest`. Typically placed after `trivy-scan.yml` so images are only signed after passing vulnerability checks.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `tags` | string | - | **Yes** | Image tags to sign (newline-separated, from `docker/metadata-action`) |
| `digest` | string | - | **Yes** | Image digest (`sha256:...`) |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

### Usage

```yaml
jobs:
  sign:
    needs: [build]
    uses: duynhlab/gha-workflows/.github/workflows/docker-sign.yml@main
    with:
      tags: ${{ needs.build.outputs.tags }}
      digest: ${{ needs.build.outputs.digest }}
    secrets: inherit
    permissions:
      contents: read
      packages: write
      id-token: write
```

---

## trivy-scan.yml

**Purpose:** Docker image vulnerability reporting (post-push)
**Features:** Trivy scanner, SARIF upload to GitHub Security tab, step summary, optional Google Sheets reporting, structured outputs

> **Note**: This workflow is for **reporting only** — it is no longer the security gate. The security gate is now built into `docker-build-go.yml` and `docker-build-node.yml` via `scan-before-push`. Use `trivy-scan.yml` for detailed SARIF reporting and Google Sheets tracking after the image has been pushed.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `image-ref` | string | - | **Yes** | Full image reference (e.g. `ghcr.io/org/app@sha256:...`) |
| `severity` | string | `"CRITICAL,HIGH"` | No | Comma-separated severity levels to scan |
| `exit-code` | string | `"1"` | No | `0` = warn only, `1` = fail on vulnerabilities |
| `ignore-unfixed` | boolean | `true` | No | Skip vulnerabilities without available fix |
| `scanners` | string | `"vuln"` | No | What to scan (`vuln`, `secret`, `misconfig`) |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `gsheet_spreadsheet_id` | string | `""` | No | Google Sheet ID for security scan reporting (tab: `security-scan`) |

### Outputs

| Output | Description |
|--------|-------------|
| `status` | `pass` or `fail` |
| `critical` | Count of CRITICAL vulnerabilities |
| `high` | Count of HIGH vulnerabilities |
| `medium` | Count of MEDIUM vulnerabilities |
| `low` | Count of LOW vulnerabilities |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `GSHEET_CLIENT_EMAIL` | No | Google service account email (for Sheets reporting) |
| `GSHEET_PRIVATE_KEY` | No | Google service account private key (for Sheets reporting) |

### Usage

**Standalone (after a custom build):**
```yaml
jobs:
  build:
    # ... your build job that outputs tags and digest ...

  scan:
    needs: [build]
    uses: duynhlab/gha-workflows/.github/workflows/trivy-scan.yml@main
    with:
      image-ref: ghcr.io/${{ github.repository }}/my-app@${{ needs.build.outputs.digest }}
      severity: 'CRITICAL,HIGH'
      exit-code: '1'
    secrets: inherit
    permissions:
      contents: read
      packages: read
      security-events: write
```

**With Google Sheets reporting:**
```yaml
jobs:
  scan:
    uses: duynhlab/gha-workflows/.github/workflows/trivy-scan.yml@main
    with:
      image-ref: ghcr.io/${{ github.repository }}/my-app@sha256:abc123...
      gsheet_spreadsheet_id: "1AbC...xYz"
    secrets:
      GSHEET_CLIENT_EMAIL: ${{ secrets.GSHEET_CLIENT_EMAIL }}
      GSHEET_PRIVATE_KEY: ${{ secrets.GSHEET_PRIVATE_KEY }}
    permissions:
      contents: read
      packages: read
      security-events: write
```

> When using Google Sheets, create a tab named `security-scan` in your spreadsheet. Each scan appends a row with: Timestamp, Workflow URL, Repository, Image, Critical, High, Medium, Low, Status, Branch, Author.

---

## sonarqube.yml

**Purpose:** SonarCloud code analysis
**Features:** Go coverage integration, Quality Gate check, configurable exclusions

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `project-key` | string | - | **Yes** | SonarCloud project key (e.g. `org_project-name`) |
| `organization` | string | - | **Yes** | SonarCloud organization |
| `sources` | string | `"."` | No | Comma-separated source directories |
| `exclusions` | string | `"**/vendor/**,**/node_modules/**,**/*_test.go,**/testdata/**"` | No | Comma-separated exclude patterns |
| `go-version` | string | `"1.25"` | No | Go version to use |
| `coverage-path` | string | `"coverage.out"` | No | Path to Go coverage file |
| `artifact-name` | string | `"coverage-report"` | No | Artifact name containing coverage report |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `quality-gate-wait` | boolean | `true` | No | Wait for Quality Gate result |
| `quality-gate-timeout` | number | `300` | No | Quality Gate polling timeout (seconds) |
| `fail-on-quality-gate` | boolean | `true` | No | Fail the job if Quality Gate fails |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `SONAR_TOKEN` | **Yes** | SonarCloud authentication token |

### Usage

```yaml
jobs:
  go-check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test -race -coverprofile=coverage.out -covermode=atomic ./...'
    secrets: inherit

  sonar:
    needs: [go-check]
    uses: duynhlab/gha-workflows/.github/workflows/sonarqube.yml@main
    with:
      project-key: my-org_my-project
      organization: my-org
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

## tf-lint.yml

**Purpose:** Terraform validation and linting
**Features:** `terraform fmt` check, TFLint analysis with plugin caching

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `tflint_config_path` | string | - | No | Custom path to `.tflint.hcl` config |
| `tflint_minimum_failure_severity` | string | `"warning"` | No | Minimum severity to cause failure |
| `tflint_force` | boolean | `false` | No | Force TFLint to return exit code 0 |

### Usage

```yaml
jobs:
  terraform-check:
    uses: duynhlab/gha-workflows/.github/workflows/tf-lint.yml@main
    with:
      tflint_minimum_failure_severity: 'error'
```

---

## status.yml

**Purpose:** CI status reporting and notifications
**Features:** Workflow/job status aggregation, Slack notifications (with thread support), Google Sheets reporting, job summary table

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `slack_msg` | string | *(auto-generated)* | No | Custom Slack message |
| `slack_channel_id` | string | `""` | No | Slack channel ID for notifications |
| `slack_thread_ts` | string | `""` | No | Slack thread timestamp (from `pr-checks.yml` output) for reply threading |
| `gsheet_spreadsheet_id` | string | `""` | No | Google Sheet spreadsheet ID for CI reporting |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `SLACK_BOT_TOKEN` | **Yes** | Slack bot token for notifications |
| `GSHEET_CLIENT_EMAIL` | No | Google service account email for Sheets API |
| `GSHEET_PRIVATE_KEY` | No | Google service account private key for Sheets API |

### Usage

**Basic (Slack only):**
```yaml
jobs:
  notify:
    needs: [build, test]
    if: always()
    uses: duynhlab/gha-workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "C0AD82A9A74"
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

**With PR thread and Google Sheets:**
```yaml
jobs:
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: duynhlab/gha-workflows/.github/workflows/pr-checks.yml@main
    with:
      slack_channel_id: "C0AD82A9A74"
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

  # ... other jobs ...

  notify:
    needs: [pr-checks, build, test]
    if: always()
    uses: duynhlab/gha-workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "C0AD82A9A74"
      slack_thread_ts: ${{ needs.pr-checks.outputs.slack_thread_ts }}
      gsheet_spreadsheet_id: "1AbC...xYz"
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
      GSHEET_CLIENT_EMAIL: ${{ secrets.GSHEET_CLIENT_EMAIL }}
      GSHEET_PRIVATE_KEY: ${{ secrets.GSHEET_PRIVATE_KEY }}
```

---

## Composite Action: slack-notification

**Location:** `.github/actions/slack-notification/action.yml`
**Purpose:** Send CI/CD status notification to Slack with thread support
**Used by:** `status.yml` workflow internally

- If `thread_ts` is empty: sends a standalone CI status message (push-to-main flow).
- If `thread_ts` is provided: replies in that Slack thread (PR flow).
- If `update_ts` is provided: updates an existing message instead of sending a new one.

### Inputs

| Parameter | Required | Description |
|-----------|----------|-------------|
| `channel_id` | **Yes** | Slack channel ID |
| `status` | **Yes** | CI status: `success`, `failed`, `cancelled` |
| `thread_ts` | No | Slack thread timestamp to reply to |
| `update_ts` | No | Slack message timestamp to update (sends new message if empty) |
| `reply_broadcast` | No | Whether reply should be broadcast to channel (default: `false`) |
| `slack_bot_token` | **Yes** | Slack bot token |

### Outputs

| Output | Description |
|--------|-------------|
| `ts` | Timestamp of sent message |
| `thread_ts` | Thread timestamp for chaining replies |

> This action is used internally by `status.yml`. You do not need to call it directly unless building custom notification workflows.

---

## Architecture

The `docker-build-go.yml` is the **Go-specific builder** with integrated Trivy scanning. Future stacks (`docker-build-node.yml`, `docker-build-python.yml`, etc.) follow the same output interface (`tags` + `digest` + `scan-status`), sharing the downstream sign workflow:

```mermaid
flowchart TD
    subgraph builders ["Stack-specific Builders (scan-before-push)"]
        GO["docker-build-go.yml<br/>(--load → Trivy → push)"]
        NODE["docker-build-node.yml<br/>(--load → Trivy → push)"]
        PYTHON["docker-build-python.yml (future)"]
    end
    subgraph shared ["Shared Downstream"]
        SIGN["docker-sign.yml"]
        TRIVY_RPT["trivy-scan.yml<br/>(SARIF reporting, optional)"]
    end
    GO -->|"outputs: tags, digest, scan-status"| SIGN
    NODE -->|"outputs: tags, digest, scan-status"| SIGN
    PYTHON -->|"outputs: tags, digest, scan-status"| SIGN
    GO -.->|"optional"| TRIVY_RPT
    NODE -.->|"optional"| TRIVY_RPT
```

### Overall CI Flow (Primary Diagram)

Canonical high-level CI flow with scan-before-push.

```mermaid
flowchart TD
    subgraph prPush ["PR / Push (excluding tags)"]
        PRCHECKS["pr-checks.yml"]
        GOCHECK["go-check.yml"]
        GITLEAKS["gitleaks.yml"]
        SONAR["sonarqube.yml"]
    end

    subgraph buildOnly ["Push to main/dev only"]
        BUILD["docker-build-go.yml<br/>(--load → Trivy → push)"]
        SIGN["docker-sign.yml"]
        TRIVY_RPT["trivy-scan.yml<br/>(optional report)"]
    end

    NOTIFY["status.yml"]

    GOCHECK --> SONAR
    GITLEAKS --> SONAR
    SONAR --> BUILD
    BUILD --> SIGN
    BUILD -.-> TRIVY_RPT

    PRCHECKS --> NOTIFY
    GOCHECK --> NOTIFY
    GITLEAKS --> NOTIFY
    SONAR --> NOTIFY
    BUILD --> NOTIFY
    SIGN --> NOTIFY
    TRIVY_RPT --> NOTIFY
```

### Detailed CI Flow: Pull Request

On PR branches: run code quality + secret scanning + notifications. Docker jobs are **skipped**.

```mermaid
flowchart TD
    subgraph pr_flow ["PR Flow"]
        PR["pr-checks.yml"]
        GOCHECK["go-check.yml"]
        GITLEAKS["gitleaks.yml"]
        SONAR["sonarqube.yml"]
        NOTIFY["status.yml"]

        GOCHECK --> SONAR
        GITLEAKS --> SONAR
        PR --> NOTIFY
        GOCHECK --> NOTIFY
        GITLEAKS --> NOTIFY
        SONAR --> NOTIFY
    end
```

### Detailed CI Flow: Push to main (merged)

Full pipeline on main: build (with integrated scan) -> sign. If pre-push scan fails, the image is never pushed and signing is automatically skipped.

```mermaid
flowchart TD
    subgraph main_flow ["Main Flow"]
        GOCHECK2["go-check.yml"]
        GITLEAKS2["gitleaks.yml"]
        SONAR2["sonarqube.yml"]

        BUILD["docker-build-go.yml<br/>(--load → Trivy → push)"]
        SIGN["docker-sign.yml"]
        TRIVY_RPT["trivy-scan.yml<br/>(SARIF report, optional)"]

        DBINIT["docker-build-go.yml (migration)<br/>(--load → Trivy → push)"]

        NOTIFY2["status.yml"]

        GOCHECK2 --> SONAR2
        GITLEAKS2 --> SONAR2
        SONAR2 --> BUILD
        SONAR2 --> DBINIT
        BUILD -->|"scan pass → pushed"| SIGN
        BUILD -.->|"scan fail"| BUILD_FAIL["Image NOT pushed"]
        BUILD -.->|"optional"| TRIVY_RPT

        BUILD --> NOTIFY2
        SIGN --> NOTIFY2
        TRIVY_RPT --> NOTIFY2
        GITLEAKS2 --> NOTIFY2
        DBINIT --> NOTIFY2
    end

    style BUILD fill:#3b82f6,color:#fff
    style BUILD_FAIL fill:#ef4444,color:#fff
```

---

## Complete Examples

### Go Project Pipeline

```yaml
name: CI Pipeline

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main, dev]

jobs:
  # PR Validation
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: duynhlab/gha-workflows/.github/workflows/pr-checks.yml@main
    with:
      slack_channel_id: "C0AD82A9A74"
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}

  # Go quality checks
  go-check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test -race -coverprofile=coverage.out -covermode=atomic ./...'
      lint: true
    secrets: inherit

  # Secret scanning
  gitleaks:
    if: "!startsWith(github.ref, 'refs/tags/')"
    uses: duynhlab/gha-workflows/.github/workflows/gitleaks.yml@main
    continue-on-error: ${{ github.event_name != 'pull_request' }}
    permissions:
      contents: read
      security-events: write

  # SonarCloud analysis
  sonar:
    needs: [go-check, gitleaks]
    uses: duynhlab/gha-workflows/.github/workflows/sonarqube.yml@main
    with:
      project-key: my-org_my-project
      organization: my-org
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # Notify status
  notify:
    needs: [pr-checks, go-check, gitleaks, sonar]
    if: always()
    uses: duynhlab/gha-workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "C0AD82A9A74"
      slack_thread_ts: ${{ needs.pr-checks.outputs.slack_thread_ts }}
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

### Docker Build Pipeline (build with scan -> sign)

Service repos call the builder workflow which handles scan-before-push internally. Each job passes outputs to the next via `needs`:

```yaml
name: Docker Build

on:
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: read
  packages: write
  id-token: write
  actions: read
  security-events: write

jobs:
  # Step 1: Build, scan, and push (scan is integrated)
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-go.yml@main
    with:
      image-name: my-service
      push: true
      # scan-before-push: true  # default — scans before pushing
    secrets: inherit
    permissions:
      contents: read
      packages: write
      actions: read

  # Step 2 (optional): Detailed vulnerability report (SARIF + Google Sheets)
  trivy-report:
    needs: [build]
    if: needs.build.outputs.scan-status == 'pass'
    uses: duynhlab/gha-workflows/.github/workflows/trivy-scan.yml@main
    with:
      image-ref: ghcr.io/${{ github.repository }}/my-service@${{ needs.build.outputs.digest }}
      severity: 'CRITICAL,HIGH,MEDIUM'
      exit-code: '0'         # non-blocking (reporting only)
      ignore-unfixed: true
    secrets: inherit
    permissions:
      contents: read
      packages: read
      security-events: write

  # Step 3: Sign (auto-skipped if build fails due to scan)
  sign:
    needs: [build]
    uses: duynhlab/gha-workflows/.github/workflows/docker-sign.yml@main
    with:
      tags: ${{ needs.build.outputs.tags }}
      digest: ${{ needs.build.outputs.digest }}
    secrets: inherit
    permissions:
      contents: read
      packages: write
      id-token: write

  notify:
    needs: [build, trivy-report, sign]
    if: always()
    uses: duynhlab/gha-workflows/.github/workflows/status.yml@main
    with:
      slack_channel_id: "C0AD82A9A74"
    secrets:
      SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

> **How it works:** The builder workflow handles `--load` → Trivy scan → push internally. If the scan fails, the image is never pushed and the job fails. `sign` depends on `build` — if build fails, sign is auto-skipped. `trivy-report` is optional and non-blocking (for SARIF + Google Sheets reporting).

---

## Configuration

### Required Secrets

Set these in your repository settings (Settings → Secrets and variables → Actions):

| Secret | Used By | Required | Description |
|--------|---------|----------|-------------|
| `SLACK_BOT_TOKEN` | pr-checks.yml, status.yml | **Yes** | Slack bot token for notifications |
| `SONAR_TOKEN` | sonarqube.yml | **Yes** | SonarCloud authentication token |
| `GSHEET_CLIENT_EMAIL` | status.yml, trivy-scan.yml | No | Google service account email |
| `GSHEET_PRIVATE_KEY` | status.yml, trivy-scan.yml | No | Google service account private key |

### Repository Setup

1. **Create a CODEOWNERS file** (see [CODEOWNERS](#codeowners) section below)
2. **Configure Slack channels** (optional):
   - `#ci-alert` - General CI notifications
   - `#pull-request-main` - PR notifications for main branch
   - `#pull-request-dev` - PR notifications for dev branch
   - `#dev-notifications` - Development updates

---

## CODEOWNERS

### How `pr-checks.yml` uses CODEOWNERS

The `pr-checks.yml` workflow automatically extracts global code owners from the caller repo's CODEOWNERS file and tags them in the Slack PR notification (e.g. `cc @duynhne @duyhenryer @duynebot`).

**Discovery order** (matches [GitHub's precedence](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners#codeowners-file-location)):

1. `.github/CODEOWNERS`
2. `CODEOWNERS` (repo root)
3. `docs/CODEOWNERS`

The workflow sparse-checks out all three paths and uses the first file found. It then reads the **global owners line** (the line starting with `*`) and extracts all `@user` / `@org/team` mentions.

If no CODEOWNERS file exists or no `*` line is found, the Slack message is still sent -- just without owner mentions.

### File format (GitHub spec)

Each line is a **file pattern** followed by one or more **owners** (`@username` or `@org/team-name`). Owners must have write access to the repo.

```gitignore
# Global owners -- requested for review on every PR
* @duynhne @duyhenryer @duynebot

# Team-based ownership
*.go        @duynhne/backend-team
*.ts *.tsx  @duynhne/frontend-team
*.tf        @duynhne/infra-team

# Directory-specific
/cmd/       @duynhne/backend-team
/terraform/ @duynhne/infra-team
/docs/      @duynhne

# Protect the CODEOWNERS file itself
/.github/CODEOWNERS @duynhne
```

### Key rules

- The `*` (wildcard) line defines **default owners** for all files -- this is the line `pr-checks.yml` reads for Slack mentions
- Order matters: the **last matching pattern** wins for GitHub's review request
- Teams use the format `@org/team-name` and must have explicit write access
- Inline comments start with `#`
- One pattern per line; multiple owners on the same line

### Where to place it

Recommended: `.github/CODEOWNERS` -- keeps repo root clean and follows GitHub's highest-priority lookup path.

> For full syntax details, see [GitHub docs: About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners).

---

## Troubleshooting

### "SLACK_BOT_TOKEN not found"

**Solution:** Add the secret in repository settings:
1. Go to Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Name: `SLACK_BOT_TOKEN`
4. Value: Your Slack bot token

### "Branch validation failed"

**Solution:** Use correct branch naming:
- `hotfix/critical-bug`
- `release/v1.2.3`
- `fix-something` (not allowed -- use `fix/something` instead)

### "CODEOWNERS not found" or Slack shows no owners

**Solution:** Create a CODEOWNERS file with a global owners line (`* @user1 @user2`). The workflow checks `.github/CODEOWNERS`, then `CODEOWNERS` (root), then `docs/CODEOWNERS`. See [CODEOWNERS](#codeowners) for details.

### "Lint timeout exceeded"

**Solution:** Increase timeout in workflow:
```yaml
with:
  lint-timeout: '20m'  # Increase from default 10m
```

### "TFLint config not found"

**Solution:** Either:
1. Create a `.tflint.hcl` file, or
2. Specify custom path:
```yaml
with:
  tflint_config_path: 'terraform/.tflint.hcl'
```

### "SonarCloud Quality Gate failed"

**Solution:** Either fix the issues reported by SonarCloud, or temporarily disable failure:
```yaml
with:
  fail-on-quality-gate: false
```

### "Trivy scan blocking image push"

**Solution:** If you want to scan without blocking the push, set `scan-exit-code` to `'0'` in your `docker-build-go.yml` call:
```yaml
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-go.yml@main
    with:
      image-name: my-service
      push: true
      scan-exit-code: '0'  # Warn only, don't block push
```

Or to skip scanning entirely, set `scan-before-push: false`.

---

## Additional Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Go Testing](https://golang.org/doc/code.html#Testing)
- [TFLint Documentation](https://github.com/terraform-linters/tflint)
- [SonarCloud Documentation](https://docs.sonarsource.com/sonarcloud/)
- [Cosign Documentation](https://docs.sigstore.dev/cosign/overview/)
- [Trivy Documentation](https://trivy.dev/latest/docs/)
- [Slack API](https://api.slack.com/)

---

<div align="center">

**Need help?** [Open an issue](https://github.com/duynhlab/gha-workflows/issues)

[Back to Top](#workflow-usage-guide)

</div>

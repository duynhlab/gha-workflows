# Workflow Reference

Inputs, outputs, secrets and usage for every reusable workflow in this repository.

- Pipeline diagrams — [architecture.md](architecture.md)
- Composite actions — [actions.md](actions.md)
- Security policy and rollout — [go-security.md](go-security.md)
- Copy-paste caller workflows — [../examples/](../examples/)

Every example below references `@main` for readability. **Callers pin a commit
SHA** — see [Referencing these workflows](#referencing-these-workflows).

## Referencing these workflows

Pin the SHA, with the branch as a trailing comment:

```yaml
uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@e321897… # main
```

**This repository is deliberately not tagged.** The `v1.0.x` tags are history
and no longer move. That is not a gap in the update path: for a reference pinned
to a SHA that carries no tag, Dependabot resolves the update to the latest commit
on the default branch, and the commit it proposes is itself untagged, so the
arrangement sustains itself. Callers therefore track `main` while keeping the
audit trail and immutability of a SHA pin.

Cutting a tag would flip that back to tag-tracking — so do not add one unless
the intent really is to freeze consumers on a release line.

## Contents

| Group | Workflows |
|-------|-----------|
| [PR & notification](#pr--notification) | `pr-checks`, `status` |
| [Go](#go) | `go-check` |
| [Security](#security) | `gitleaks`, `go-security`, `codeql`, `govulncheck`, `dependency-review` |
| [Container images](#container-images) | `docker-build-go`, `docker-build-node`, `docker-sign`, `trivy-scan` |
| [Release](#release) | `goreleaser` |
| [Code quality](#code-quality) | `sonarqube` |
| [Infrastructure](#infrastructure) | `tf-lint` |

---

# PR & notification

## pr-checks.yml

**Purpose:** Pull request validation and notifications
**Trigger:** PR events only
**Features:** Branch validation (gitflow prefixes), CODEOWNERS tagging, Slack event notifications

> Requires a valid CODEOWNERS file in the caller repo for owner mentions in Slack. See [configuration.md#codeowners](configuration.md#codeowners) for format and placement.

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

## status.yml

**Purpose:** CI status reporting and notifications
**Features:** Workflow/job status aggregation, Slack notifications (with thread support), Google Sheets reporting, job summary table

> Report-only. Never make this a required status check — a Slack or Sheets outage must not block a merge.

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

> The Google Sheet must have two tabs: `workflow-status` and `jobs-status`.

---

# Go

## go-check.yml

**Purpose:** Go code quality assurance
**Trigger:** PR events
**Features:** Unit testing (with optional stable Go matrix), optional integration tests, linting, coverage artifacts, coverage job summaries

When tests produce a coverage profile (`coverage.out` / `coverage-integration.out`), the workflow writes a **job summary** with total coverage and a collapsible per-package breakdown. Artifacts are still uploaded for SonarCloud.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `command-test` | string | - | **Yes** | Test command to execute |
| `setup-go` | boolean | `true` | No | Install Go automatically |
| `gomod-path` | string | `"go.mod"` | No | Path to go.mod file |
| `lint` | boolean | `false` | No | Enable linting (PR events only) |
| `lint-path` | string | `".golangci.yml"` | No | Lint config file path |
| `lint-timeout` | string | `"10m"` | No | Lint timeout duration |
| `lint-version` | string | `"v2.12.2"` | No | **Fallback** golangci-lint version — used only when the caller has no `tools/go.mod`. See [Pinning the linter](#pinning-the-linter) |
| `test-stable` | boolean | `false` | No | Also test against stable Go version (non-blocking compatibility check) |
| `integration` | boolean | `false` | No | Run integration tests (`go test -tags=integration`). Needs a Docker daemon for testcontainers; `ubuntu-latest` has one |
| `integration-command` | string | *(see below)* | No | Command for the integration job. Must write `coverage-integration.out` |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |

Default `integration-command`:

```
go test -tags=integration -covermode=atomic -coverprofile=coverage-integration.out ./...
```

### Jobs

| Job | Runs when | Notes |
|-----|-----------|-------|
| `Test` | Always | The check to require in the ruleset |
| `Integration Test` | `integration: true` | Sets `TESTCONTAINERS_RYUK_DISABLED=true` — the Ryuk reaper intermittently hangs on GitHub runners, and the runner is ephemeral anyway |
| `Test (stable)` | `test-stable: true` | `continue-on-error` — compatibility signal only |
| `Lint` | `lint: true` **and** event is `pull_request` | Does **not** run on `push`, so `main` is never linted — a PR merged with Lint red leaves those findings unreported |

### Pinning the linter

Pin golangci-lint in a **`tools/go.mod`** rather than through `lint-version`:

```
tools/go.mod   →   tool github.com/golangci/golangci-lint/v2/cmd/golangci-lint
```

```bash
# create it once
mkdir tools && cd tools && go mod init <module>/tools
go get -tool github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.12.2

# run it — same command locally and in CI
go -C tools build -o .bin/golangci-lint github.com/golangci/golangci-lint/v2/cmd/golangci-lint
./.bin/golangci-lint run
```

`lint-version` is a plain string input, so **nothing updates it** — no Dependabot
ecosystem reads a workflow input value. That is how the default sat on `v2.6.0`
while every local install had moved on, and 63 findings never reached CI. A
`tools/go.mod` is a real module: Dependabot's `gomod` ecosystem bumps it, per
repository, in that repository's own PR.

Keep the tool in its **own module**, not the service `go.mod` — golangci-lint
pulls roughly 200 dependencies, and they have no business in the service's
`go.sum` or its `govulncheck` results. Add `.bin/` to `.gitignore`.

The Lint job prefers `tools/go.mod` when present and falls back to
`lint-version` otherwise, so the migration is per repository with no flag day.

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

**With linting, integration tests and stable Go matrix:**
```yaml
jobs:
  go-check:
    uses: duynhlab/gha-workflows/.github/workflows/go-check.yml@main
    with:
      command-test: 'go test -race -coverprofile=coverage.out -covermode=atomic ./...'
      lint: true
      lint-version: 'v2.12.2'
      test-stable: true
      integration: true
    secrets: inherit
```

> To feed integration coverage into SonarCloud, pair `integration: true` here with `integration-coverage: true` on [`sonarqube.yml`](#sonarqubeyml).

---

# Security

For the security policy, rollout phases, ruleset settings and exception process, see [go-security.md](go-security.md).

## gitleaks.yml

**Purpose:** Secret scanning for source code and git history
**Trigger:** Caller-defined (recommended on PR + push)
**Features:** Gitleaks CLI binary (MIT, open-source), auto PR diff / full scan, SARIF upload to GitHub Security tab, job summary

> Uses a composite action ([`gitleaks`](actions.md#gitleaks)) that downloads the gitleaks binary directly. No license key required — works for both personal and organization repositories.

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

**PR block, push warn:**
```yaml
jobs:
  gitleaks:
    uses: duynhlab/gha-workflows/.github/workflows/gitleaks.yml@main
    continue-on-error: ${{ github.event_name != 'pull_request' }}
    with:
      config-path: '.gitleaks.toml' # optional
    permissions:
      contents: read
      security-events: write
```

> Be careful adding an `if:` guard (for example excluding tags) when other jobs `needs:` this one — a skipped dependency skips every downstream job, which can silently turn a tag build into a no-op.

---

## go-security.yml

**Purpose:** Single security entrypoint for Go services — CodeQL + Govulncheck + Dependency Review behind one stable gate
**Trigger:** Caller-defined (recommended: PR, push to main, and a weekly schedule)
**Features:** Nested reusable workflows, per-language SARIF categories, `Security Gate` check for branch rulesets, no secrets required

> Call this instead of wiring `codeql.yml` / `govulncheck.yml` / `dependency-review.yml` into every service.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `go-version-file` | string | `"go.mod"` | No | Path to `go.mod`, used to pick the Go version |
| `go-working-directory` | string | `"."` | No | Directory for Go tooling (monorepo / nested modules) |
| `codeql-queries` | string | `"security-extended"` | No | CodeQL query suite |
| `codeql-build-mode` | string | `"autobuild"` | No | `autobuild` or `manual` (for generated code / custom builds) |
| `codeql-manual-build-command` | string | `""` | No | Build command, required when `codeql-build-mode: manual` |
| `scan-actions` | boolean | `false` | No | Also run CodeQL against the repo's own GitHub Actions workflows |
| `govulncheck-enforce` | boolean | `true` | No | Fail on govulncheck findings. `false` = report-only baselining |
| `govulncheck-version` | string | `"v1.6.0"` | No | Pinned `golang.org/x/vuln` version |
| `govulncheck-build-tags` | string | `""` | No | Comma-separated build tags |
| `dependency-review-severity` | string | `"critical"` | No | `low`, `moderate`, `high`, or `critical` |
| `dependency-review-scopes` | string | `"runtime"` | No | Dependency scopes to block on |
| `allowed-licenses` | string | `""` | No | License allowlist. Empty = license check skipped |
| `license-enforce` | boolean | `false` | No | Block on license violations instead of warning |
| `allowed-ghsas` | string | `""` | No | Advisory IDs waived in Dependency Review. Each needs an owner, ticket and expiry |
| `allowed-dependencies-licenses` | string | `""` | No | purls exempt from the license policy |

### Jobs

| Job | Runs when | Blocking |
|-----|-----------|----------|
| `codeql-go` | Always | Execution only — severity via Code Scanning merge protection |
| `codeql-actions` | `scan-actions: true` | Execution only |
| `govulncheck` | Always | Yes, when `govulncheck-enforce: true` |
| `dependency-review` | `pull_request` only | Yes (vulnerabilities); licenses per `license-enforce` |
| `security-gate` | Always (`if: always()`) | **Yes — require this check** |

### Usage

**Pull request:**
```yaml
jobs:
  go-security:
    uses: duynhlab/gha-workflows/.github/workflows/go-security.yml@main
    with:
      go-version-file: go.mod
      govulncheck-enforce: true
      dependency-review-severity: critical
    permissions:
      actions: read
      contents: read
      pull-requests: read
      security-events: write
```

> **Do not pass `secrets: inherit`** — this workflow needs no secrets. The caller must grant the full permission set above, because permissions can only be kept or reduced down a reusable-workflow call chain.

**Baselining a repo that is not clean yet:**
```yaml
    with:
      govulncheck-enforce: false   # report-only; fix findings, then flip to true
```

---

## codeql.yml

**Purpose:** Run one CodeQL analysis for one language
**Features:** Per-language SARIF category (`/language:<lang>`), `security-extended` by default, `persist-credentials: false`

> Usually consumed through `go-security.yml`. Call it directly when you need a language that isn't part of the Go bundle — for example scanning a workflow-only repository with `language: actions`.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `language` | string | - | **Yes** | CodeQL language, e.g. `go`, `actions` |
| `build-mode` | string | - | **Yes** | `autobuild` (compiled), `none` (interpreted/config), or `manual` |
| `queries` | string | `"security-extended"` | No | Query suite |
| `go-version-file` | string | `"go.mod"` | No | Used only when `language: go` |
| `manual-build-command` | string | `""` | No | Required when `build-mode: manual` |

### Usage

```yaml
jobs:
  codeql-actions:
    uses: duynhlab/gha-workflows/.github/workflows/codeql.yml@main
    with:
      language: actions
      build-mode: none
    permissions:
      actions: read
      contents: read
      security-events: write
```

> A green CodeQL job does **not** mean the repo has no alerts. Enforce alert severity with a Code Scanning merge protection rule in the branch ruleset.

---

## govulncheck.yml

**Purpose:** Find known vulnerabilities in Go dependencies that are actually reachable from your code
**Features:** Call-graph analysis (low noise), pinned tool version, real blocking gate, SARIF upload under category `govulncheck`

> `govulncheck -format sarif` **always exits 0**, even with findings. This workflow therefore runs text mode for the exit code and SARIF mode for the report, and a final step owns the pass/fail decision.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `go-version-file` | string | `"go.mod"` | No | Path to `go.mod` (never `go-version: stable` — results depend on the toolchain) |
| `working-directory` | string | `"."` | No | Directory to scan |
| `package-pattern` | string | `"./..."` | No | Package pattern |
| `govulncheck-version` | string | `"v1.6.0"` | No | Pinned `golang.org/x/vuln` version |
| `build-tags` | string | `""` | No | Comma-separated build tags |
| `include-tests` | boolean | `false` | No | Also analyze test files |
| `enforce` | boolean | `true` | No | Fail on findings. `false` = report-only |

### Outputs

| Output | Description |
|--------|-------------|
| `exit-code` | `0` = clean, `3` = vulnerabilities found, other = tool/build error |
| `status` | `clean`, `vulnerable`, or `error` |

### Usage

```yaml
jobs:
  govulncheck:
    uses: duynhlab/gha-workflows/.github/workflows/govulncheck.yml@main
    with:
      enforce: true
    permissions:
      contents: read
      security-events: write
```

> A tool or build error (any exit code other than `0` or `3`) fails the job **even when `enforce: false`** — a scan that did not run is not a clean scan.

---

## dependency-review.yml

**Purpose:** Block pull requests that introduce dependencies with known vulnerabilities, and audit licenses
**Trigger:** `pull_request` only — self-skips on push/schedule (there is no dependency diff to review)
**Features:** Split vulnerability gate and license audit, least-privilege (`contents: read` only)

> **Prerequisites:** Dependency Graph enabled; private repos need GitHub Code Security / Advanced Security; self-hosted runners must support the Node 24 runtime used by v5. `pull_request_target` is deliberately not supported.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `fail-on-severity` | string | `"critical"` | No | `low`, `moderate`, `high`, or `critical` |
| `fail-on-scopes` | string | `"runtime"` | No | `unknown`, `runtime`, `development` |
| `allowed-licenses` | string | `""` | No | License allowlist. **Empty skips the license check** — keep it empty until Legal approves a policy |
| `license-enforce` | boolean | `false` | No | `false` = warn only, `true` = block |
| `allowed-ghsas` | string | `""` | No | Waived advisory IDs. Each needs an owner, ticket and expiry |
| `allowed-dependencies-licenses` | string | `""` | No | purls exempt from the license policy |

### Usage

```yaml
jobs:
  dependency-review:
    uses: duynhlab/gha-workflows/.github/workflows/dependency-review.yml@main
    with:
      fail-on-severity: critical
    permissions:
      contents: read
```

---

# Container images

`docker-build-go.yml` and `docker-build-node.yml` take **identical inputs and outputs** — the shared contract below. They differ only in the **tagging strategy**. Both build the image locally (`--load`), scan it with Trivy, and push to GHCR only if the scan passes, so a vulnerable image never reaches the registry.

### Shared inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `image-name` | string | - | **Yes** | Docker image name (without registry prefix) |
| `dockerfile` | string | `"Dockerfile"` | No | Path to Dockerfile |
| `context` | string | `"."` | No | Docker build context |
| `push` | boolean | `false` | No | Push image to registry |
| `platforms` | string | `"linux/amd64,linux/arm64"` | No | Target platforms of the pushed index |
| `build-args` | string | `""` | No | Build-time variables |
| `tags` | string | `""` | No | Custom tags (comma-separated); empty = default tagging |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `sbom` | boolean | `false` | No | Generate SBOM attestation |
| `scan-before-push` | boolean | `true` | No | Scan the image with Trivy before pushing |
| `scan-severity` | string | `"CRITICAL,HIGH"` | No | Severities scanned and **reported** in the summary |
| `scan-block-severity` | string | `"CRITICAL"` | No | Severities that **block the push**. Everything else in `scan-severity` is report-only |
| `scan-ignore-unfixed` | boolean | `true` | No | Ignore vulnerabilities without a fix available |
| `scan-trivyignores` | string | `""` | No | Path to a `.trivyignore` (relative to the build context) for suppressing known upstream CVEs |
| `scan-exit-code` | string | `"1"` | No | **Deprecated — no longer gates the push.** Superseded by `scan-block-severity`; kept only for backward compatibility |

> `scan-severity` and `scan-block-severity` are two different knobs. By default `CRITICAL,HIGH` are reported but only `CRITICAL` blocks — this avoids builds getting stuck on base-image HIGH CVEs that have no upstream fix yet. To block on HIGH too, set `scan-block-severity: 'CRITICAL,HIGH'`.

### Shared outputs

| Output | Description |
|--------|-------------|
| `tags` | Generated image tags (newline-separated) |
| `digest` | Image digest (`sha256:...`) |
| `scan-status` | Pre-push scan result: `pass`, `fail`, or `skipped` |

---

## docker-build-go.yml

**Purpose:** Build a Go service Docker image, scan it, and push to GHCR only if clean
**Tagging:** **Immutable only** — `sha-<short>` for branch pushes, `pr-<n>` for pull requests, `X.Y.Z` + `X.Y` for `v*` tags. No `:latest`, no branch-named tags.

The immutable tagging is deliberate: it pairs with digest-pinned deployments, so a redeploy always resolves to the exact artifact that was scanned and signed.

### Usage

**With scan-before-push (default, recommended):**
```yaml
jobs:
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-go.yml@main
    with:
      image-name: my-service
      push: true
      # scan-before-push: true          # default
      # scan-severity: 'CRITICAL,HIGH'  # default — reported
      # scan-block-severity: 'CRITICAL' # default — blocks the push
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

> `--load` (used for local scanning) only supports single-platform builds, so the Trivy gate exercises the **runner's** architecture only — amd64 on `ubuntu-latest`. The default `platforms` is multi-arch, so the arm64 layers of the pushed index go out unscanned; the gate blocks the push on an amd64 finding, and an arm64-only regression would not be caught. Pass `platforms: linux/amd64` if you need the scanned artifact to be the whole pushed artifact.
>
> Cross-compile in the Dockerfile, or the arm64 leg runs the entire builder stage under QEMU: `FROM --platform=$BUILDPLATFORM …`, `ARG TARGETOS TARGETARCH`, then `GOARCH=${TARGETARCH} go build`. With it the arm64 leg costs under a minute; without it, minutes to tens of minutes.

---

## docker-build-node.yml

**Purpose:** Build a Node.js service Docker image, scan it, and push to GHCR only if clean
**Tagging:** branch name, `sha-<short>`, `pr-<n>`, semver on `v*` tags, **plus `:latest` on the default branch**

Inputs and outputs are identical to the [shared contract](#shared-inputs) above.

> Unlike the Go builder, this one publishes mutable tags (`:latest` and branch tags). If you deploy from a mutable tag, the running image can change without a new digest — prefer deploying by digest.

### Usage

```yaml
jobs:
  build:
    uses: duynhlab/gha-workflows/.github/workflows/docker-build-node.yml@main
    with:
      image-name: my-node-service
      push: true
    secrets: inherit
    permissions:
      contents: read
      packages: write
      actions: read
```

---

## docker-sign.yml

**Purpose:** Sign Docker images using Cosign (keyless / OIDC)
**Features:** Keyless signing via Sigstore OIDC, signs all tags for a given digest

> Chain after any builder that outputs `tags` and `digest`. Because the builder only pushes images that pass the scan, a failed scan means `docker-sign` is skipped automatically.

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

> **Reporting only** — this is no longer the security gate. The gate lives inside the builders via `scan-before-push`. Use this workflow for detailed SARIF reporting and Google Sheets tracking after the image has been pushed.

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `image-ref` | string | - | **Yes** | Full image reference (e.g. `ghcr.io/org/app@sha256:...`) |
| `severity` | string | `"CRITICAL,HIGH"` | No | Comma-separated severity levels to scan |
| `exit-code` | string | `"1"` | No | `0` = warn only, `1` = fail on vulnerabilities |
| `ignore-unfixed` | boolean | `true` | No | Skip vulnerabilities without an available fix |
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

**Post-push report (non-blocking):**
```yaml
jobs:
  trivy-report:
    needs: [build]
    if: needs.build.outputs.scan-status == 'pass'
    uses: duynhlab/gha-workflows/.github/workflows/trivy-scan.yml@main
    with:
      image-ref: ghcr.io/${{ github.repository }}/my-app@${{ needs.build.outputs.digest }}
      severity: 'CRITICAL,HIGH,MEDIUM'
      exit-code: '0'
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

> When using Google Sheets, create a tab named `security-scan`. Each scan appends a row with: Timestamp, Workflow URL, Repository, Image, Critical, High, Medium, Low, Status, Branch, Author.

---

# Release

## goreleaser.yml

**Purpose:** Build Go release binaries with GoReleaser and publish them to a GitHub Release
**Trigger:** Caller-defined, intended for `v*` tags
**Features:** Archives, combined `checksums.txt`, cosign keyless signature over the checksums, syft SBOMs

The `.goreleaser.yaml` config lives in the **calling** repository. The version is derived from the git tag (plus `-X main.version` ldflags); there is no `build-info.env` sidecar.

Published artifacts follow the standard GoReleaser supply-chain layout:

| Artifact | Description |
|----------|-------------|
| `<repo>-<version>-linux-amd64.tar.gz` | Archive containing `bin/<repo>` plus LICENSE/README |
| `checksums.txt` | Single combined checksum file |
| `checksums.txt.sig` / `checksums.txt.pem` | Cosign keyless signature over the checksums |
| SBOM(s) | syft, one per archive |

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `go-version` | string | `""` | No | Go version override; empty reads the `go` directive from `go-version-file` |
| `go-version-file` | string | `"go.mod"` | No | `go.mod` that sets the Go version when `go-version` is empty, so the release binary uses the toolchain CI tested |
| `goreleaser-version` | string | `"~> v2"` | No | GoReleaser **CLI** version constraint |

> `goreleaser-version` constrains the GoReleaser CLI (v2), which is independent of the `goreleaser-action` major pinned inside the workflow. Do not try to "align" the two numbers.

### Permissions

The caller must grant at least:

```yaml
permissions:
  contents: write   # create the Release and upload assets
  id-token: write   # cosign keyless (OIDC) signing
```

No explicit secrets are required — the workflow uses the automatic `GITHUB_TOKEN`.

### Usage

```yaml
jobs:
  release-binary:
    if: startsWith(github.ref, 'refs/tags/v')
    uses: duynhlab/gha-workflows/.github/workflows/goreleaser.yml@main
    permissions:
      contents: write
      id-token: write
```

---

# Code quality

## sonarqube.yml

**Purpose:** SonarCloud code analysis
**Features:** Go coverage integration (unit + optional integration), Quality Gate check, configurable exclusions

### Inputs

| Parameter | Type | Default | Required | Description |
|-----------|------|---------|----------|-------------|
| `project-key` | string | - | **Yes** | SonarCloud project key (e.g. `org_project-name`) |
| `organization` | string | - | **Yes** | SonarCloud organization |
| `sources` | string | `"."` | No | Comma-separated source directories |
| `exclusions` | string | `"**/vendor/**,**/node_modules/**,**/*_test.go,**/testdata/**"` | No | Patterns excluded from analysis entirely |
| `coverage-exclusions` | string | *(see below)* | No | Paths excluded from the **coverage % only** — still analyzed for bugs and smells |
| `coverage-path` | string | `"coverage.out"` | No | Path to Go coverage file |
| `artifact-name` | string | `"coverage-report"` | No | Artifact name containing the coverage report |
| `integration-coverage` | boolean | `false` | No | Merge `coverage-integration.out` from `go-check.yml`'s integration job into the report |
| `runs-on` | string | `"ubuntu-latest"` | No | Runner type |
| `quality-gate-wait` | boolean | `true` | No | Wait for Quality Gate result |
| `quality-gate-timeout` | number | `300` | No | Quality Gate polling timeout (seconds) |
| `fail-on-quality-gate` | boolean | `true` | No | Fail the job if the Quality Gate fails |

Default `coverage-exclusions` — bootstrap, wiring, migrations and mocks, which drag the coverage number down without saying anything about test quality:

```
**/cmd/**,**/internal/core/database.go,**/db/migrations/**,**/mocks/**,**/*_mock.go
```

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
      integration: true
    secrets: inherit

  sonar:
    needs: [go-check]
    uses: duynhlab/gha-workflows/.github/workflows/sonarqube.yml@main
    with:
      project-key: my-org_my-project
      organization: my-org
      integration-coverage: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

---

# Infrastructure

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

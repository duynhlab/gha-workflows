# Configuration

Repository setup required before calling these workflows: secrets, Slack channels, and
CODEOWNERS.

## Required secrets

Set these in the calling repository: **Settings → Secrets and variables → Actions**.

| Secret | Used by | Required | Description |
|--------|---------|----------|-------------|
| `SLACK_BOT_TOKEN` | `pr-checks.yml`, `status.yml` | **Yes** | Slack bot token for notifications |
| `SONAR_TOKEN` | `sonarqube.yml` | **Yes** | SonarCloud authentication token |
| `GSHEET_CLIENT_EMAIL` | `status.yml`, `trivy-scan.yml` | No | Google service account email (Sheets reporting) |
| `GSHEET_PRIVATE_KEY` | `status.yml`, `trivy-scan.yml` | No | Google service account private key (Sheets reporting) |
| `COPILOT_PAT` | `renovate-review.yml` | **Yes** | Fine-grained PAT with the *Copilot Requests* permission and public-repository read; must carry no write scopes |

Workflows that need **no secrets at all**: `go-security.yml`, `codeql.yml`, `govulncheck.yml`,
`dependency-review.yml`, `gitleaks.yml`, `tf-lint.yml`. Do not pass `secrets: inherit` to them —
grant permissions instead.

`goreleaser.yml` and `docker-sign.yml` use the automatic `GITHUB_TOKEN` and OIDC; they need
`contents: write` / `packages: write` and `id-token: write` from the caller rather than a secret.

## Security prerequisites

For the security workflows, the calling repository also needs:

- Dependency Graph enabled.
- GitHub Code Security / Advanced Security, if the repository is private.
- CodeQL **Advanced Setup** — switch off Default Setup to avoid duplicate scans.
- Permission to use reusable workflows from this repository (org setting: Actions → Policies).

Full details and the branch ruleset configuration: [go-security.md](go-security.md).

## Slack channels

Optional, per team convention:

| Channel | Purpose |
|---------|---------|
| `#ci-alert` | General CI notifications |
| `#pull-request-main` | PR notifications for `main` |
| `#pull-request-dev` | PR notifications for `dev` |
| `#dev-notifications` | Development updates |

## CODEOWNERS

### How `pr-checks.yml` uses CODEOWNERS

`pr-checks.yml` extracts the global code owners from the caller repo's CODEOWNERS file and tags
them in the Slack PR notification (e.g. `cc @duynhne @duyhenryer @duynebot`).

**Discovery order** (matches [GitHub's precedence](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners#codeowners-file-location)):

1. `.github/CODEOWNERS`
2. `CODEOWNERS` (repo root)
3. `docs/CODEOWNERS`

The workflow sparse-checks out all three paths and uses the first file found. It then reads the
**global owners line** (the line starting with `*`) and extracts all `@user` / `@org/team`
mentions.

If no CODEOWNERS file exists, or no `*` line is found, the Slack message is still sent — just
without owner mentions.

### File format

Each line is a **file pattern** followed by one or more **owners** (`@username` or
`@org/team-name`). Owners must have write access to the repository.

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

- The `*` (wildcard) line defines **default owners** for all files — this is the line
  `pr-checks.yml` reads for Slack mentions.
- Order matters: the **last matching pattern** wins for GitHub's review request.
- Teams use the format `@org/team-name` and must have explicit write access.
- Inline comments start with `#`.
- One pattern per line; multiple owners on the same line.

### Where to place it

Recommended: `.github/CODEOWNERS` — keeps the repo root clean and matches GitHub's
highest-priority lookup path.

> Full syntax: [GitHub docs — About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners).

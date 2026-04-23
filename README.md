# PAT Usage Audit — GitHub Organization Token Visibility

A GitHub Action and shell-based audit tool that queries multiple GitHub APIs to produce a comprehensive report of **Personal Access Token (PAT)** usage across a GitHub organization. Because GitHub does not offer a single "list all PATs" API, this tool combines SAML SSO credential authorizations, the organization audit log, and GraphQL queries to maximize visibility.

---

## Table of Contents

- [Why This Exists](#why-this-exists)
- [How It Works](#how-it-works)
- [Usage](#usage)
- [Data Sources & Coverage](#data-sources--coverage)
- [Required Token Permissions](#required-token-permissions)
- [Setup & Configuration](#setup--configuration)
- [Report Outputs](#report-outputs)
- [Limitations](#limitations)
- [Repository Structure](#repository-structure)
- [Security Considerations](#security-considerations)
- [License](#license)

---

## Why This Exists

GitHub does not expose a single API endpoint to enumerate **all** classic PATs (`ghp_*`) created by members of an organization. Classic PATs are user-owned; org-level visibility is limited to:

- **SAML SSO credential authorizations** — only if SAML SSO is configured on the org.
- **Audit log events** — only on **GitHub Enterprise Cloud**.
- **Fine-grained PAT management endpoints** — require a **GitHub App** installation token (not a classic PAT).

This tool bridges those gaps by aggregating every available signal into a single Markdown + JSON report.

---

## How It Works

Published as a **GitHub Marketplace Action** powered by `actions/github-script@v9` — no bash scripts, no shell dependencies. All logic runs as inline JavaScript via Octokit.

The action runs as a **composite action** with 2 steps:

1. **Collect Data** — Fetches repos, members, SAML SSO credentials, audit log events, and scans org workflows for custom secret usage. All data written as JSON to `./reports/`.
2. **Generate Report** — Builds a per-user PAT activity matrix, generates a Markdown report, and writes a rich job summary to the Actions UI.

> **Key design choice:** The report only shows **users with PAT activity** — not all org members. A user appears only if they show up in SAML SSO credentials, audit log PAT events, or token-authenticated access events.

---

## Usage

### From the GitHub Marketplace

Use this action in any repository:

```yaml
name: PAT Audit
on:
  schedule:
    - cron: '0 8 * * 1'
  workflow_dispatch:

env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: 'true'

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: jackgkafaty/GitHub_PAT_AUDIT@v1
        with:
          org-pat: ${{ secrets.ORG_LEVEL_PAT }}
          # org-name: my-org        # optional, defaults to repository owner
          # lookback-days: '30'     # optional, defaults to 30

      - uses: actions/upload-artifact@v4
        with:
          name: pat-audit-report
          path: reports/
          retention-days: 90
```

### Action Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `org-pat` | **Yes** | — | Fine-grained PAT — see [Required Token Permissions](#required-token-permissions) |
| `org-name` | No | Repository owner | GitHub organization slug |
| `lookback-days` | No | `30` | Days to look back in the audit log |

### Action Outputs

| Output | Description |
|---|---|
| `pat-active-users` | Number of users with PAT activity detected |
| `report-path` | Path to the reports directory (`reports`) |

### Run from This Repository

This repository includes a workflow at `.github/workflows/pat-audit.yml` that uses the action locally (`uses: ./`). It runs weekly and supports manual dispatch.

To trigger: **Actions → PAT Audit → Run workflow**.

### Run Locally

```bash
export ORG_PAT="github_pat_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export ORG_NAME="your-org"
export LOOKBACK_DAYS=30

bash scripts/pat-audit.sh
```

The report will be written to `./reports/`.

---

## Data Sources & Coverage

| Data Source | API Endpoint | What It Shows | Requirements |
|---|---|---|---|
| SAML SSO Credential Authorizations | `GET /orgs/{org}/credential-authorizations` | PATs & SSH keys authorized for SSO: user, token last 8 chars, scopes, auth/access/expiry dates | SAML SSO enabled (any plan) |
| Audit Log — PAT Lifecycle | `GET /orgs/{org}/audit-log?phrase=action:personal_access_token` | Token creation, approval, denial, revocation events | Enterprise Cloud |
| Audit Log — Token Auth Events | `GET /orgs/{org}/audit-log?include=all` | API calls made via PAT/bot (actor, action, repo, access type) | Enterprise Cloud |
| Workflow Secret Scanner | Code Search API + Contents API | Workflow files using custom secrets (non-GITHUB_TOKEN) — indicates PAT usage | Contents (read) |
| Actor Summary | Aggregated from audit log | Per-user event count, bot flag, repos accessed, timeline | Enterprise Cloud |
| Repositories | GraphQL `organization.repositories` | Full repo inventory with privacy, archive status, last push | Metadata (read) |
| Members | GraphQL `organization.membersWithRole` | All members with roles (used internally for cross-referencing, not displayed) | Members (read) |

### Coverage by Org Configuration

| Configuration | Classic PATs Visible? | Fine-Grained PATs Visible? | Access Events Visible? |
|---|---|---|---|
| **Enterprise Cloud + SAML SSO** | Yes — credential authorizations | Audit log + GitHub App | Yes — audit log |
| **Enterprise Cloud, no SAML** | Audit log only | Audit log + GitHub App | Yes — audit log |
| **Team/Pro + SAML SSO** | Yes — credential authorizations | GitHub App only | No audit log API |
| **Team/Pro, no SAML** | No | GitHub App only | No |
| **Free plan** | No | GitHub App only | No |

---

## Required Token Permissions

This tool uses a **fine-grained Personal Access Token (FGPAT)** scoped to your organization. Fine-grained PATs are preferred over classic PATs because they enforce read-only access, mandatory expiration, and granular permission boundaries.

> **Requirement:** The token owner must be an **organization owner**. Fine-grained PATs cannot escalate beyond the user's own org role — the permissions below grant read access within that role.

### Organization Permissions

| Permission | Access Level | Why It's Needed |
|---|---|---|
| **Administration** | Read | Read SAML SSO credential authorizations (`GET /orgs/{org}/credential-authorizations`), query the audit log (`GET /orgs/{org}/audit-log`) |
| **Members** | Read | List organization members with roles via GraphQL (`membersWithRole`) |

### Repository Permissions

The token must be scoped to **All repositories** in the organization.

| Permission | Access Level | Why It's Needed |
|---|---|---|
| **Metadata** | Read (granted by default) | List org repositories via GraphQL (name, visibility, archive status, timestamps) |
| **Contents** | Read | Read workflow file contents for secret scanning (`GET /repos/{owner}/{repo}/contents/{path}`), search workflows via Code Search API |

### API Endpoints and Permission Mapping

| Endpoint | Method | Permission Required |
|---|---|---|
| `GET /orgs/{org}/credential-authorizations` | GET | Organization: Administration (read) |
| `GET /orgs/{org}/audit-log` | GET | Organization: Administration (read) |
| `POST /graphql` — `organization.membersWithRole` | POST | Organization: Members (read) |
| `POST /graphql` — `organization.repositories` | POST | Repository: Metadata (read) |
| `GET /search/code` | GET | Repository: Contents (read) |
| `GET /repos/{owner}/{repo}/contents/{path}` | GET | Repository: Contents (read) |
| `GET /user` | GET | No additional permissions required |
| `GET /orgs/{org}` | GET | Organization: Members (read) |
| `GET /rate_limit` | GET | No additional permissions required |

---

## Setup & Configuration

### 1. Create a Fine-Grained PAT

1. Go to [GitHub Settings → Developer Settings → Personal Access Tokens → Fine-grained tokens](https://github.com/settings/personal-access-tokens/new).
2. Set **Resource owner** to your target organization.
3. Set **Repository access** to **All repositories**.
4. Under **Organization permissions**, grant:
   - **Administration** → Read
   - **Members** → Read
5. Under **Repository permissions**, grant:
   - **Contents** → Read
   - (**Metadata** → Read is granted by default)
6. Set an appropriate expiration (90 days or less recommended).
7. Copy the generated token.

### 2. Configure Repository Secrets & Variables

In the repository running this action:

| Secret / Variable | Type | Value |
|---|---|---|
| `ORG_LEVEL_PAT` | **Secret** | The fine-grained PAT created above |
| `GITHUB_ORG_NAME` | **Variable** | Your GitHub organization slug (e.g., `octodemo`) |

### 3. Environment Variables

The audit script uses the following environment variables:

| Variable | Required | Default | Description |
|---|---|---|---|
| `ORG_PAT` | Yes | — | Fine-grained PAT — see [Required Token Permissions](#required-token-permissions) |
| `ORG_NAME` | Yes | — | GitHub organization slug |
| `LOOKBACK_DAYS` | No | `30` | Number of days to look back in the audit log |
| `GITHUB_API_VERSION` | No | `2026-03-10` | GitHub API version header |

---

## Report Outputs

After execution, the `reports/` directory contains:

| File | Format | Description |
|---|---|---|
| `pat-audit-report.md` | Markdown | Human-readable consolidated report with all sections |
| `user_pat_matrix.json` | JSON | Per-user PAT activity matrix (only PAT-active users) |
| `repositories.json` | JSON | All org repositories with metadata |
| `members.json` | JSON | All org members (used for cross-referencing roles) |
| `credential_authorizations.json` | JSON | SAML SSO authorized credentials (PATs + SSH keys) |
| `audit_pat_events.json` | JSON | Audit log events for PAT lifecycle |
| `audit_token_access.json` | JSON | Audit log events with token-based authentication markers |
| `actor_summary.json` | JSON | Aggregated per-actor activity summary |
| `workflow_secrets.json` | JSON | Workflows using custom secrets (PAT detection) |

---

## Limitations

1. **No universal PAT listing** — There is no GitHub API to list all classic PATs across an organization. Coverage depends on SAML SSO and Enterprise Cloud.
2. **Fine-grained PAT enumeration requires a GitHub App** — The `/orgs/{org}/personal-access-tokens` endpoint requires a GitHub App installation token; it does not accept user PATs.
3. **Audit log requires Enterprise Cloud** — Free and Team plans do not have API access to the audit log.
4. **Token values are never exposed** — The audit log shows `token_last_eight` or `hashed_token`, never the full token (by design).
5. **SAML SSO credential authorizations require SAML** — If your org doesn't use SAML SSO, this data source returns nothing.
6. **Rate limits** — Large organizations may hit GitHub API rate limits. The script uses pagination with safety limits (max 50 pages per endpoint).

---

## Repository Structure

```
PatUsage/
├── action.yml                     # Composite action (marketplace)
├── .github/
│   └── workflows/
│       └── pat-audit.yml          # Workflow using the composite action
├── README.md                      # This file
├── FEASIBILITY-REVIEW.md          # API feasibility analysis
├── scripts/                       # Bash scripts for local use
│   ├── lib.sh                     # Shared functions
│   └── pat-audit.sh               # Local runner (sequential)
└── reports/
    └── .gitkeep                   # Placeholder (generated at runtime)
```

---

## Security Considerations

- **Use a fine-grained PAT** — fine-grained tokens enforce read-only access, mandatory expiration, and can be scoped to a single organization. They are strictly preferred over classic PATs for this tool.
- **Store the PAT as an encrypted repository secret** — never commit it to the repository.
- **Use a dedicated service account** — avoid using a personal PAT; create a dedicated org-owner service account for audit operations.
- **Rotate the PAT regularly** — set a short expiration (90 days or less) and rotate on schedule. Fine-grained PATs require an expiration date by default.
- **Principle of least privilege** — all permissions granted are read-only. The token cannot modify org settings, repository contents, or membership.
- **Report artifacts may contain sensitive metadata** — restrict access to workflow artifacts and avoid publishing reports publicly.

---

## License

This project is provided as-is for internal organizational use.

# Claude Workload Identity Federation (WIF) for GitHub Actions

How the AI workflows in `shared-workflows` — `ai-triage.yml` and `ai-assistant.yml` — authenticate to the Claude API without a static `ANTHROPIC_API_KEY`.

## Overview

WIF lets a GitHub Actions job exchange GitHub's short-lived **OIDC ID token** for a short-lived Anthropic access token. Nothing secret is stored in the repository: the IDs involved are identifiers, not credentials.

```
GitHub runner ──(id-token: write)──► GitHub OIDC JWT (aud=https://api.anthropic.com, ~5 min)
      │
      └──► POST https://api.anthropic.com/v1/oauth/token
               (JWT + federation_rule_id + organization_id [+ service_account_id, workspace_id])
                    │
                    └──► Anthropic access token (sk-ant-oat01-..., ~10 min) ──► Claude API
```

## Which GitHub token is used

- **Used:** the GitHub Actions **OIDC ID token** — a JWT signed by `https://token.actions.githubusercontent.com`, requested via `ACTIONS_ID_TOKEN_REQUEST_URL` / `ACTIONS_ID_TOKEN_REQUEST_TOKEN`. These env vars exist only when the job has `permissions: id-token: write`.
- **Not used for Claude auth:** `GITHUB_TOKEN` (`github.token`) or a PAT. `GITHUB_TOKEN` is still passed to the action, but only for GitHub API operations (e.g. `gh issue comment`).

`claude-code-action` requests the OIDC token with audience `https://api.anthropic.com`, writes it under `${{ runner.temp }}/claude-workload-identity/`, and performs the exchange itself.

### Identity carried in a reusable workflow

Both `ai-triage.yml` and `ai-assistant.yml` are reusable workflows (`on: workflow_call`) invoked from consumer repos:

```yaml
uses: avioconsulting/shared-workflows/.github/workflows/ai-triage.yml@main
uses: avioconsulting/shared-workflows/.github/workflows/ai-assistant.yml@main
```

The OIDC token is therefore issued for the **caller repository's run**, not for `shared-workflows`. The workflow that actually runs is identified by the `job_workflow_ref` claim:

| Claim                 | `ai-triage.yml`                                                                   | `ai-assistant.yml`                                                                   |
|-----------------------|-----------------------------------------------------------------------------------|--------------------------------------------------------------------------------------|
| Caller trigger        | `issues` (`opened`, `labeled`)                                                    | `issue_comment` (`created`)                                                          |
| `sub`                 | `repo:avioconsulting/<repo>:ref:refs/heads/main`                                  | `repo:avioconsulting/<repo>:ref:refs/heads/main`                                     |
| `repository_owner`    | `avioconsulting`                                                                  | `avioconsulting`                                                                     |
| `repository_owner_id` | `26659507`                                                                        | `26659507`                                                                           |
| `event_name`          | `issues`                                                                          | `issue_comment`                                                                      |
| `job_workflow_ref`    | `avioconsulting/shared-workflows/.github/workflows/ai-triage.yml@refs/heads/main` | `avioconsulting/shared-workflows/.github/workflows/ai-assistant.yml@refs/heads/main` |

Both events run on the caller's default-branch ref, so `sub` is **identical** for the two workflows. `job_workflow_ref` (plus `event_name` as a secondary check) is what distinguishes them.

## 1. Anthropic setup (Claude Console)

Go to **Settings → Workload identity → Connect workload** and select **GitHub Actions** (`platform.claude.com/settings/workload-identity-federation`). The wizard creates:

### Federation issuer (`fdis_...`, once per org)

```json
{
  "name": "github-actions",
  "issuer_url": "https://token.actions.githubusercontent.com",
  "jwks": { "type": "discovery" }
}
```

Discovery mode means Anthropic refreshes GitHub's signing keys automatically.

### Service account (`svac_...`)

The identity the workflow acts as inside your Anthropic org.

### Federation rule (`fdrl_...`)

Controls which workflow runs may obtain a token. Single-repo example:

```json
{
  "name": "gha-main",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "repo:avioconsulting/your-repo:ref:refs/heads/main",
    "audience": "https://api.anthropic.com",
    "claims": { "repository_owner": "avioconsulting" }
  },
  "target": { "type": "service_account", "service_account_id": "svac_..." },
  "workspace_id": "wrkspc_...",
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

### Federation rules for the shared workflows (one per workflow)

A per-repo `subject_prefix` doesn't fit reusable workflows called from many repos, and `sub` alone can't tell triage from assistant. Create **two rules**, each pinned to its own `job_workflow_ref`, so a token minted for one workflow can never authenticate the other.

**Triage rule** — `ANTHROPIC_TRIAGE_FEDERATION_RULE_ID`

```json
{
  "name": "gha-ai-triage",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "repo:avioconsulting/",
    "audience": "https://api.anthropic.com",
    "claims": {
      "repository_owner": "avioconsulting",
      "job_workflow_ref": "avioconsulting/shared-workflows/.github/workflows/ai-triage.yml@refs/heads/main",
      "event_name": "issues"
    }
  },
  "target": { "type": "service_account", "service_account_id": "svac_triage..." },
  "workspace_id": "wrkspc_...",
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

**Assistant rule** — `ANTHROPIC_ASSISTANT_FEDERATION_RULE_ID`

```json
{
  "name": "gha-ai-assistant",
  "issuer_id": "fdis_...",
  "match": {
    "subject_prefix": "repo:avioconsulting/",
    "audience": "https://api.anthropic.com",
    "claims": {
      "repository_owner": "avioconsulting",
      "job_workflow_ref": "avioconsulting/shared-workflows/.github/workflows/ai-assistant.yml@refs/heads/main",
      "event_name": "issue_comment"
    }
  },
  "target": { "type": "service_account", "service_account_id": "svac_assistant..." },
  "workspace_id": "wrkspc_...",
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 1800
}
```

Design notes:

- **One issuer, two rules.** Both rules reference the same GitHub Actions issuer (`fdis_...`).
- **Separate service accounts.** Usage and spend are attributed per workflow, and either can be revoked independently. Optionally place triage in its own workspace with a lower spend limit.
- **Lifetime per workload.** Triage is a short, read-mostly run (600 s). The assistant's Fix job builds and runs longer agent turns (e.g. 1800 s).
- **Pinned to `main`.** `@refs/heads/main` in `job_workflow_ref` means a modified copy of a workflow on another branch or in a fork cannot obtain a token.
- **Verify claim values.** After the first run, confirm the exact `job_workflow_ref` / `event_name` strings in the Console's authentication history and that the rule matches them.

Also note your **Organization ID** (Settings → Organization).

## 2. GitHub setup

Store the IDs as repository or organization **variables** (not secrets):

| Variable | Example |
|---|---|
| `ANTHROPIC_ORGANIZATION_ID` | `00000000-0000-0000-0000-000000000000` |
| `ANTHROPIC_TRIAGE_FEDERATION_RULE_ID` | `fdrl_...` (triage rule) |
| `ANTHROPIC_ASSISTANT_FEDERATION_RULE_ID` | `fdrl_...` (assistant rule) |
| `ANTHROPIC_SERVICE_ACCOUNT_ID` (optional) | `svac_...` |
| `ANTHROPIC_WORKSPACE_ID` (if rule spans workspaces) | `wrkspc_...` |

**Caller workflows must grant `id-token: write`.** A reusable workflow cannot get more permissions than its caller gives it; if the caller's job-level `permissions` omit `id-token`, the `id-token: write` declared inside the shared workflow is downgraded to `none` and the OIDC fetch fails.

Caller for triage:

```yaml
name: AI Issue Triage
on:
  issues:
    types: [opened, labeled]
jobs:
  Triage:
    uses: avioconsulting/shared-workflows/.github/workflows/ai-triage.yml@main
    permissions:
      contents: read
      issues: write
      id-token: write
    with:
      triage-labels: bug,enhancement,question
    secrets: inherit
```

Caller for the assistant:

```yaml
name: AI Maintenance Assistant
on:
  issue_comment:
    types: [created]
jobs:
  Assistant:
    uses: avioconsulting/shared-workflows/.github/workflows/ai-assistant.yml@main
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
    secrets: inherit
```

## 3. Workflow configuration

### With `claude-code-action` (current approach in `ai-triage.yml` and `ai-assistant.yml`)

```yaml
permissions:
  contents: read
  issues: write
  id-token: write   # required for the OIDC token

steps:
  - uses: anthropics/claude-code-action@<pinned-sha>
    with:
      # ai-triage.yml uses ANTHROPIC_TRIAGE_FEDERATION_RULE_ID
      # ai-assistant.yml uses ANTHROPIC_ASSISTANT_FEDERATION_RULE_ID
      anthropic_federation_rule_id: ${{ vars.ANTHROPIC_ASSISTANT_FEDERATION_RULE_ID }}
      anthropic_organization_id: ${{ vars.ANTHROPIC_ORGANIZATION_ID }}
      # optional:
      # anthropic_service_account_id: ${{ vars.ANTHROPIC_SERVICE_ACCOUNT_ID }}
      # anthropic_workspace_id: ${{ vars.ANTHROPIC_WORKSPACE_ID }}
      # anthropic_oidc_audience: https://api.anthropic.com   # default
      github_token: ${{ github.token }}
```

Action inputs:

- `anthropic_federation_rule_id` — required
- `anthropic_organization_id` — required
- `anthropic_service_account_id` — add if the exchange fails without it
- `anthropic_workspace_id` — needed when the rule covers multiple workspaces
- `anthropic_oidc_audience` — override only if the rule expects a non-default audience

**Do not** also set `anthropic_api_key` or `claude_code_oauth_token`: a static credential takes precedence and federation is silently skipped. With federation, `classify_inline_comments` is skipped.

### With an SDK / script

```yaml
env:
  ANTHROPIC_FEDERATION_RULE_ID: fdrl_...
  ANTHROPIC_ORGANIZATION_ID: <org-uuid>
  ANTHROPIC_SERVICE_ACCOUNT_ID: svac_...
  ANTHROPIC_WORKSPACE_ID: wrkspc_...
  ANTHROPIC_IDENTITY_TOKEN_FILE: /tmp/gha-jwt
steps:
  - name: Fetch GitHub OIDC token
    run: |
      curl -sS -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
        "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=https://api.anthropic.com" \
        | jq -r .value > "$ANTHROPIC_IDENTITY_TOKEN_FILE"
  - run: python your_script.py   # anthropic.Anthropic() picks up the env vars
```

The SDK exchanges the JWT on first use and refreshes automatically. GitHub's JWT expires in ~5 minutes; for long jobs, re-run the fetch step to keep the file current.

## 4. Hardening (applied in `ai-triage.yml` and `ai-assistant.yml`)

Prevent the agent from reading or reusing the OIDC credentials:

```json
{
  "permissions": { "deny": ["Read(/${{ runner.temp }}/claude-workload-identity/**)", "Read(//proc/**)"] },
  "sandbox": {
    "enabled": true,
    "failIfUnavailable": true,
    "filesystem": { "denyRead": ["${{ runner.temp }}/claude-workload-identity", "/proc"] },
    "credentials": { "envVars": [
      { "name": "ACTIONS_ID_TOKEN_REQUEST_URL", "mode": "deny" },
      { "name": "ACTIONS_ID_TOKEN_REQUEST_TOKEN", "mode": "deny" },
      { "name": "ANTHROPIC_IDENTITY_TOKEN_FILE", "mode": "deny" }
    ]}
  }
}
```

Build steps that run project code also `unset ACTIONS_ID_TOKEN_REQUEST_URL ACTIONS_ID_TOKEN_REQUEST_TOKEN` as defense in depth.

Rule-scoping guidance:

- `repo:your-org/*` alone matches **every repo** and, without a `ref` constraint, **fork PR runs** — anyone opening a PR could get a token.
- Pin the repo, add `claims.ref` (e.g. `refs/heads/main`), keep `repository_owner`, and for reusable workflows pin `job_workflow_ref` — one rule per workflow.
- For deploy jobs, match `repo:org/repo:environment:production` and protect the environment with required reviewers.

## 5. Troubleshooting

- A failed exchange returns an opaque `401 authentication_error` / `Authentication failed`. Check **Settings → Workload identity → History** in the Console for the deny reason.
- Most common: `match_subject_prefix` — the `sub` suffix varies by event (`ref:...`, `pull_request`, `environment:...`).
- Missing `id-token: write` (in the job **or** the caller workflow) → no OIDC env vars, action can't fetch a token.
- Wrong rule ID for the workflow (e.g. triage using the assistant rule) → `job_workflow_ref` / `event_name` claim mismatch.
- Rule spans multiple workspaces → `workspace_id` required.
- Success returns `access_token` starting with `sk-ant-oat01-` plus `expires_in`.

## References

- [Use WIF with GitHub Actions](https://platform.claude.com/docs/en/manage-claude/wif-providers/github-actions)
- [Workload Identity Federation overview](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation)
- [GitHub OIDC subject claims](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#example-subject-claims)
- `claude-code-action` docs: `docs/setup.md` → Workload Identity Federation
- Implementation: [`.github/workflows/ai-triage.yml`](.github/workflows/ai-triage.yml), [`.github/workflows/ai-assistant.yml`](.github/workflows/ai-assistant.yml)

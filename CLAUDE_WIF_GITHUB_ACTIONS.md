# Claude Workload Identity Federation (WIF) for GitHub Actions

How direct Triage, Analyze, and Fix authenticate to the Claude API without a
static `ANTHROPIC_API_KEY`. The separate gh-aw triage workflow still uses an API
key.

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

The workflows are called from consumer repos with `on: workflow_call`. The OIDC token is issued for the **caller repository's run**, not for `shared-workflows`. Match the exact `job_workflow_ref` for each enabled workflow:

| Workflow | `event_name` | `job_workflow_ref` | Actions variable |
| --- | --- | --- | --- |
| Triage | `issues` or `issue_comment` (via `condition`) | `avioconsulting/shared-workflows/.github/workflows/ai-triage.yml@refs/heads/main` | `ANTHROPIC_TRIAGE_FEDERATION_RULE_ID` |
| Analyze | `issue_comment` | `avioconsulting/shared-workflows/.github/workflows/ai-analyze.yml@refs/heads/main` | `ANTHROPIC_ANALYZE_FEDERATION_RULE_ID` |
| Fix | `issue_comment` | `avioconsulting/shared-workflows/.github/workflows/ai-fix.yml@refs/heads/main` | `ANTHROPIC_FIX_FEDERATION_RULE_ID` |

All these events run on the caller's default-branch ref, so `sub` can be identical for runs from the same repository. Restrict the caller to approved repositories and protected default branches. An earlier rule for the combined workflow does not match the Analyze or Fix path.

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

The `sub` claim alone cannot distinguish Triage, Analyze, and Fix. Create a
separate rule for each enabled workflow (up to three), and match its exact
`job_workflow_ref` and `event_name`. Triage runs on both `issues` (issue opened)
and `issue_comment` (`@claude triage`). `match.claims` matches exactly one string
value, so remove `event_name` from `claims` and allow both values in
`match.condition`, a CEL expression over `claims` that must also pass. The examples below show one approved
caller; constrain additional callers and their protected default branches
explicitly rather than allowing the entire organization.

**Triage rule** — `ANTHROPIC_TRIAGE_FEDERATION_RULE_ID`


```json
{
  "name": "github-ai-triage",
  "description": "AI Triage Workflow in GitHub Actions. When issues are created on AVIO Repos, this rule will match.",
  "issuer_id": "fdis_01...",
  "issuer_name": "github-actions",
  "match": {
    "subject_prefix": null,
    "audience": null,
    "claims": null,
    "condition": "claims.sub.startsWith(\"repo:avioconsulting/*\") && claims.repository_owner_id == 26659507 && claims.repository_owner == \"avioconsulting\" &&  \nclaims.refs == \"refs/heads/main\" && \nclaims.event_name in ['issues', 'issue_comment']"
  },
  "attributes": null,
  "target": {
    "type": "service_account",
    "service_account_id": "svac_01...",
    "service_account_name": "github-ci-caller"
  },
  "workspace_id": "wrkspc_01...",
  "applies_to_all_workspaces": false,
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

**Fix rule** — `ANTHROPIC_FIX_FEDERATION_RULE_ID`. For Analyze, create a
separate rule using `ANTHROPIC_ANALYZE_FEDERATION_RULE_ID` in the caller and
the exact Analyze path from the table above.

```json
{
  "name": "github-ai-assistant",
  "description": "Used for Fix and Analyze workflows in GitHub Actions",
  "issuer_id": "fdis_01...",
  "issuer_name": "github-actions",
  "match": {
    "subject_prefix": "repo:avioconsulting/*",
    "audience": null,
    "claims": {
      "ref": "\"refs/heads/main\"",
      "event_name": "issue_comment",
      "repository_owner": "avioconsulting",
      "repository_owner_id": "26659507"
    },
    "condition": null
  },
  "attributes": null,
  "target": {
    "type": "service_account",
    "service_account_id": "svac_01...",
    "service_account_name": "github-ci-caller"
  },
  "workspace_id": "wrkspc_01...",
  "applies_to_all_workspaces": false,
  "oauth_scope": "workspace:developer",
  "token_lifetime_seconds": 600
}
```

Design notes:

- **One issuer, one rule per enabled workflow.** The rules reference the same GitHub Actions issuer (`fdis_...`).
- **Separate service accounts.** Usage and spend are attributed per workflow, and each identity can be revoked independently. Optionally place triage in its own workspace with a lower spend limit.
- **Lifetime per workload.** Triage is a short, read-mostly run (600 s). Fix builds and runs longer agent turns (e.g. 1800 s).
- **Match the selected ref.** The examples call `@main`, which appears as `@refs/heads/main` in `job_workflow_ref`. When piloting on another branch or pinning to a reviewed SHA, update the rule to match the ref actually used.
- **Verify claim values.** After the first run, confirm the exact `job_workflow_ref` / `event_name` strings in the Console's authentication history and that the rule matches them.

Also note your **Organization ID** (Settings → Organization).

## 2. GitHub setup

Store the IDs as repository or organization **variables** (not secrets):

| Variable | Example |
|---|---|
| `ANTHROPIC_ORGANIZATION_ID` | `00000000-0000-0000-0000-000000000000` |
| `ANTHROPIC_TRIAGE_FEDERATION_RULE_ID` | `fdrl_...` (triage rule) |
| `ANTHROPIC_ANALYZE_FEDERATION_RULE_ID` | `fdrl_...` (Analyze rule) |
| `ANTHROPIC_FIX_FEDERATION_RULE_ID` | `fdrl_...` (Fix rule) |

**Caller workflows must grant `id-token: write`.** A reusable workflow cannot
get more permissions than its caller grants. In each workflow, only the agent job
requests OIDC; `Gate`, `Publish`, `PublishFix`, and `ReportFailure`
do not.

Caller for triage:

```yaml
name: AI Issue Triage
on:
  issues:
    types: [opened]
  issue_comment:
    types: [created]
jobs:
  Triage:
    uses: avioconsulting/shared-workflows/.github/workflows/ai-triage.yml@main
    permissions:
      contents: read
      issues: write
      id-token: write
    with:
      classification-labels: bug,enhancement,question
```

Use the separate [Analyze and Fix caller examples](README.md#setup). WIF does not require `secrets: inherit`.

## 3. Workflow configuration

### With `claude-code-action` (Analyze example)

```yaml
permissions:
  contents: read
  issues: write
  id-token: write   # required for the OIDC token

steps:
  - uses: anthropics/claude-code-action@v1
    with:
      anthropic_federation_rule_id: ${{ vars.ANTHROPIC_ANALYZE_FEDERATION_RULE_ID }}
      anthropic_organization_id: ${{ vars.ANTHROPIC_ORGANIZATION_ID }}
      github_token: ${{ github.token }}
```

Action inputs:

- `anthropic_federation_rule_id` — required
- `anthropic_organization_id` — required

The action also supports `anthropic_service_account_id`,
`anthropic_workspace_id`, and `anthropic_oidc_audience`, but the current
reusable workflows do not forward these inputs or corresponding caller
variables. Use rules that do not require those overrides.

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

## 4. Hardening

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

Fix unsets the OIDC request variables in the trusted default-branch warmup as
defense in depth; the job still has OIDC permission. Changed project code runs
through the sandboxed offline verifier, which refuses to run if OIDC request
variables or federation files are visible. Publication runs in a separate job
without OIDC and does not run project code.

Rule-scoping guidance:

- `repo:your-org/*` alone matches **every repo** and, without a `ref` constraint, **fork PR runs** — anyone opening a PR could get a token.
- Pin the repo, add `claims.ref` (e.g. `refs/heads/main`), keep `repository_owner`, and for reusable workflows pin `job_workflow_ref` — one rule per workflow.
- For deploy jobs, match `repo:org/repo:environment:production` and protect the environment with required reviewers.

## 5. Troubleshooting

- A failed exchange returns an opaque `401 authentication_error` / `Authentication failed`. Check **Settings → Workload identity → History** in the Console for the deny reason.
- Most common: `match_subject_prefix` — the `sub` suffix varies by event (`ref:...`, `pull_request`, `environment:...`).
- Missing `id-token: write` (in the job **or** the caller workflow) → no OIDC env vars, action can't fetch a token.
- Wrong rule ID for the workflow (e.g. Analyze using the Fix rule) → `job_workflow_ref` / `event_name` claim mismatch.
- Rule spans multiple workspaces → the exchange may require an explicit workspace ID; use a single-workspace rule with the current shared workflows.
- Success returns `access_token` starting with `sk-ant-oat01-` plus `expires_in`.

## References

- [Use WIF with GitHub Actions](https://platform.claude.com/docs/en/manage-claude/wif-providers/github-actions)
- [Workload Identity Federation overview](https://platform.claude.com/docs/en/manage-claude/workload-identity-federation)
- [GitHub OIDC subject claims](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect#example-subject-claims)
- `claude-code-action` docs: `docs/setup.md` → Workload Identity Federation
- Implementation: [Triage](.github/workflows/ai-triage.yml), [Analyze](.github/workflows/ai-analyze.yml), and [Fix](.github/workflows/ai-fix.yml)

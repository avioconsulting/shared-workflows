# Github Actions - Shared Workflows
A repository to store the Github Actions shared workflows

## AI framework maintenance

The Triage, Analyze, and Fix workflows use the official
[Claude Code Action](https://github.com/anthropics/claude-code-action/tree/main/examples).

| Workflow | Trigger in the calling repository | Result |
| --- | --- | --- |
| `ai-triage.yml` | Issue opened by a user with write access, or a new issue comment starting with `@claude triage` from one | Labels and a concise triage comment |
| `ai-analyze.yml` | New human-authored issue comment starting with `@claude analyze` | Analysis comment |
| `ai-fix.yml` | New human-authored issue comment starting with `@claude fix` or `@claude implement` | Branch and draft PR, or an issue reply if blocked |

### Setup

1. Follow [Use WIF with GitHub Actions](https://platform.claude.com/docs/en/manage-claude/wif-providers/github-actions).
   In the Claude Console, register the GitHub OIDC issuer and create a
   [federation rule for each enabled workflow](CLAUDE_WIF_GITHUB_ACTIONS.md).
   Match the `https://api.anthropic.com` audience, approved calling repositories and
   their protected default branches, and the exact reusable `job_workflow_ref`
   (including the shared workflow ref). Match `event_name` to `issue_comment` for
   Analyze and Fix. Triage runs on both `issues` and `issue_comment`; because
   `claims` matches one exact value, allow both with the rule's CEL `condition`:
   `claims.event_name in ['issues', 'issue_comment']`. Configure workspace
   spending limits.
2. In **Settings → Secrets and variables → Actions → Variables** of each
   **calling repository**, set `ANTHROPIC_ORGANIZATION_ID`,
   `ANTHROPIC_TRIAGE_FEDERATION_RULE_ID`, `ANTHROPIC_ANALYZE_FEDERATION_RULE_ID`,
   and `ANTHROPIC_FIX_FEDERATION_RULE_ID` as applicable. These identifiers are
   not secrets. Organization-level variables scoped to approved callers also work.
   Add `id-token: write` to each caller job below and remove its API-key mapping
   or `secrets: inherit` if no other secrets are needed. Pilot against a reviewed
   shared-workflow ref with matching Console rules before moving existing callers.
3. Precreate `bug`, `enhancement`, `question`, `ai-triaged`, and `need information`
   labels in the calling repository. For custom classifications, create those
   labels and describe their meaning in repository guidance or label descriptions.
   Triage skips labels that do not exist. The `needs-triage` label is no longer used.
4. Add repository guidance in `AGENTS.md` and a `CLAUDE.md` that imports `@AGENTS.md`.
   Keep framework-specific triage requirements in that guidance and issue templates.
5. For automatic fix PRs, enable **Settings → Actions → General → Workflow
   permissions → Allow GitHub Actions to create and approve pull requests** in
   the calling repository, subject to organization policy. The workflow creates
   PRs but does not approve or merge them.

These workflows use **Anthropic API billing**, charged to the organization
associated with the federation rule. Claude subscriptions do not fund this API
usage. GitHub Actions runner usage is accounted for separately.

The shared repository must be accessible to the caller. The examples use `@main`;
use the branch containing these workflows for a pilot, and a reviewed SHA for rollout.

Create `.github/workflows/ai-triage.yml` in the framework repository:

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

Create `.github/workflows/ai-analyze.yml` in the framework repository:

```yaml
name: AI Maintenance Analyze
on:
  issue_comment:
    types: [created]
jobs:
  Analyze:
    uses: avioconsulting/shared-workflows/.github/workflows/ai-analyze.yml@main
    permissions:
      contents: read
      issues: write
      id-token: write
```

To enable fixes, create `.github/workflows/ai-fix.yml` separately:

```yaml
name: AI Maintenance Fix
on:
  issue_comment:
    types: [created]
jobs:
  Fix:
    uses: avioconsulting/shared-workflows/.github/workflows/ai-fix.yml@main
    permissions:
      contents: write
      issues: write
      pull-requests: write
      id-token: write
    with:
      java-version: '17'
      java-distribution: temurin
```

Fix defaults to Temurin Java 8. Set `java-version` to the version your project
requires; the default-branch baseline and offline verification use the same JDK.

You can enable Analyze or Fix independently. The old `ai-assistant.yml` entry
point is removed in this revision. Before rolling out the shared-workflow
change, identify callers of that path (including `@main` callers), configure
federation for the new paths, and coordinate the caller updates.

If the default-branch Maven build needs private dependencies, explicitly map a
read-only JFrog credential pair under the `Fix` caller job. The names on the
right are examples; use your repository's actual secret names. Do not use
`secrets: inherit`.

```yaml
    secrets:
      JFROG_USER: ${{ secrets.JFROG_USER }}
      JFROG_PASSWORD: ${{ secrets.JFROG_PASSWORD }}
```

Fix configures Maven server entries for `avio-releases` and
`avio-mulesoft-ee-releases` that reference these JFROG credentials as AVIO REPO credentials. It mirrors the `mulesoft-ee-releases` repository ID
through AVIO Artifactory using `avio-mulesoft-ee-releases`. If migrating a caller
that used direct Mule EE credentials, use the read-only JFrog pair with access
to that mirror instead; Fix does not accept `MULE_EE_*` secrets. Do not write
resolved credentials to files on the runner.

### Inputs and execution limits

These workflows accept an optional `claude-model` input. Triage defaults to
`claude-haiku-5-5`; Analyze and Fix default to `claude-sonnet-5-5`.
To debug a run, set `claude-show-full-output: true` in the calling job's `with:`
block and run the workflow with debug enabled. This optional boolean defaults to `false`. Full output includes tool
results, which may contain sensitive data.

Callers that override `model` must rename that input to `claude-model`.

| Workflow | Additional inputs | Max turns | Claude CLI budget | Configured timeout |
| --- | --- | --- | --- | --- |
| `ai-triage.yml` | `classification-labels` (default `bug,enhancement,question`), `info-label` (default `need information`), `triaged-label` (default `ai-triaged`) | 12 | $1 | Gate: 5 minutes; Triage: 10 minutes; Publish: 5 minutes |
| `ai-analyze.yml` | None | 20 | $3 | Gate: 5 minutes; Analyze: 20 minutes |
| `ai-fix.yml` | `java-version` (default `8`), `java-distribution` (default `temurin`) | 30 | $6 | Gate: 5 minutes; Fix: 60 minutes; publication: 10 minutes |

`classification-labels` lists the labels the agent may choose as a classification.
`info-label` and `triaged-label` are operational labels managed by the workflow;
keep them out of `classification-labels`. All labels must already exist; triage
skips missing labels. The former `triage-labels` input was removed, and a caller
that still passes it fails; rename it to `classification-labels`.

None of these workflows exposes an `enabled` input. Disable a workflow by
disabling its calling workflow in GitHub Actions or gating the caller job with
an `if` condition. CLI budgets apply to individual agent invocations, not aggregate
workflow, repository, or organization spending. Configure API spending limits
separately.

Fix currently declares `maven-args` but does not use it for warmup or verification.

### Triage behavior

Triage runs in three jobs. `Gate` checks the requester's write access. `Triage` runs
the agent with read-only issue access: it reads the issue, labels, repository
guidance and issue templates, searches for duplicates with `gh issue list` and
`gh issue view`, and writes a classification and a comment to files. `Publish`
validates that result and applies it, so label rules are enforced by the workflow
rather than the prompt.

| Event | Requester | Result |
| --- | --- | --- |
| Issue opened | Opener has write access | Initial triage |
| Issue opened | Opener is not an owner, member, or collaborator | Job skipped; a collaborator can comment `@claude triage` |
| Issue opened | Opener is a member or collaborator without write access | Run notice only; nothing is posted |
| `@claude triage [input]` comment | Commenter has write access | Retriage using the input |
| `@claude triage` comment | Commenter is not an owner, member, or collaborator | Job skipped |
| `@claude triage` comment | Commenter is a member or collaborator without write access | Run fails; nothing is posted |

The agent:

- Chooses one classification from `classification-labels`, or none when unclear.
- Asks at most three questions, only for information that blocks the next step,
  assessing bugs, enhancements and questions differently.
- Runs at most two duplicate searches and inspects at most three candidates. It
  links potential duplicates but never closes or labels them.
- Writes a 100–180 word comment (maximum 250) with **Summary** (**Update** on
  retriage), **Information needed**, **Potential duplicates** and **Next step**,
  omitting empty sections.

On retriage, text after `@claude triage` takes priority over the issue text and
earlier triage. The agent treats questions answered since the last triage as
resolved and asks only what is still missing. Use it to triage an outside
reporter's issue, correct a classification, or refresh triage after the reporter
replies.

`Publish`:

- Uses only labels listed in `classification-labels` that exist in the repository.
- For initial triage, posts at most one comment per issue (a `github-actions[bot]`
  comment containing `<!-- ai-issue-triage:v1`) and never replaces an existing
  classification.
- For retriage, always posts a new comment that links the request, may replace the
  classification, and removes `info-label` when nothing is still missing.
- Adds `info-label` when information is needed and `triaged-label` after each
  triage. Labels that do not exist are skipped.

`ai-triaged` records that triage ran; it is not acceptance, prioritization, or
proof that the issue is reproducible. Search `is:issue is:open -label:ai-triaged`
to find issues, such as those from outside reporters, that still need
`@claude triage`.

Triage runs are serialized per issue. Triage failures are not reported on the
issue, to avoid noise on reporters' issues.

### Assistant commands and permissions

Post one of these commands on an issue:

| Command | Behavior |
| --- | --- |
| `@claude triage` or `@claude triage <input>` | Retriage the issue using the input; see [Triage behavior](#triage-behavior) |
| `@claude analyze why this fails` | Investigate and comment (under 400 words) with a summary, linked evidence, assumptions, a file-level change plan, test cases, risks, and open questions |
| `@claude fix this bug` | Make focused code changes, run relevant checks, and create a draft PR with an issue reply |
| `@claude implement this feature` | Use the same workflow and checks to implement a feature |

Each workflow's `Gate` job matches its supported commands at the very start of
the comment, with no leading whitespace. Use the command by itself or add a
literal space before optional task details; a tab or newline immediately after
the command is not a separator. Matching is case-insensitive. A bare `@claude`,
unsupported commands such as `@claude review`, and mentions elsewhere in the
comment do not start an agent job.

Only newly created, human-authored **issue** comments are eligible. PR conversation
comments, inline reviews, and comment edits are excluded.

Authorization has two layers:

1. **Pre-filter in the job condition.** The `Gate` job runs only when the author's
   `author_association` (the comment author, or the issue author for a newly
   opened issue) is `OWNER`, `MEMBER`, or `COLLABORATOR`. Other authors are
   skipped without starting a runner. This is not a permission check: members
   and collaborators with read or triage access pass it.
2. **Write-access check.** The `Gate` job's *Authorize requester* step, which is
   the same in all three workflows, calls the GitHub API for the requester (the
   event sender) and requires `write` or `admin` permission. For a command it
   fails the run if the check fails, and otherwise adds an eyes reaction before
   any agent job starts. For a newly opened issue, triage instead skips quietly
   with a run notice. The Claude Code Action also checks write access before running
   an agent.

An org member whose membership is private may appear as `CONTRIBUTOR` and be
skipped by the pre-filter. Verify on a pilot repository; if this happens, make the
membership public or add `CONTRIBUTOR` to the pre-filter list.

Each job receives only the permissions it needs:

| Job | Contents | Issues | Pull requests | OIDC |
| --- | --- | --- | --- | --- |
| Gate (all three workflows) | None | Write | None | No |
| Triage | Read | Read | None | Yes |
| Publish (Triage) | None | Write | None | No |
| Analyze | Read | Write (posting step only) | None | Yes |
| Fix | Read | Read | Read | Yes |
| PublishFix | Write | Write | Write | No |
| ReportFailure (Analyze and Fix) | None | Write | None | No |

Same-run artifact downloads do not require an additional `actions: read` grant. All agents
use explicit automation prompts, avoiding interactive branch setup. Checkout
credentials remain unpersisted. No agent posts to GitHub itself: each writes its
result to a file, and a later workflow step publishes it. The Analyze agent's
tools are limited to `Read`, `Glob`, `Grep`, and `Write`; a final step posts
`analysis.md` with a link to the triggering comment and an
`<!-- ai-analyze:v1 ... -->` marker. The Triage agent additionally has read-only
`gh issue list` and `gh issue view` for duplicate search. If Analyze or Fix fails,
`ReportFailure` comments on the issue with a link to the run; cancelled runs are
not reported.

### Fix publication

`Fix` runs the trusted default-branch baseline before Claude starts and keeps
warmed dependencies on the same runner for offline verification. It unsets
OIDC request variables for the warm-up shell, but the job itself has OIDC
permission. Optional Maven credentials are scoped to the warm-up step and
unset before a Gradle build. Claude sees only the baseline result, not the
build output. The Claude sandbox denies `AVIO_REPO_USER` and `AVIO_REPO_PASSWORD`,
and the offline verifier refuses to run if either is visible. This relies on the
trusted baseline not persisting credentials on the shared runner. Fix does not
upload dependencies or save or restore a cross-run Maven cache. If an earlier
revision created Maven caches, remove those caches in the calling repositories.
Publication is handled by subsequent workflow steps:

1. Claude reads the triggering issue/comment and discussion, implements a focused
   fix, and runs relevant checks. It is instructed to leave changes uncommitted
   locally rather than create branches, push, open PRs, or post comments itself.
   The agent job has only read permissions on GitHub. The offline verifier uses
   dependencies and plugins already warmed by the baseline. If a fix requires a new
   dependency or version, the agent is instructed to request clarification.
2. Claude writes `fix-pr-metadata.json` (status, plus title and commit message for
   `ready`) and
   `fix-pr-body.md` under `runner.temp/fix-agent`, outside the checkout. The requested body
   includes the change summary, tests and actual check results, limitations, and
   `Fixes #<issue-number>`.
3. After a successful agent conclusion, the job uploads a binary-capable patch,
   the checked-out base SHA, and the metadata as a one-day artifact. Available
   offline verification logs are uploaded separately for one day. The separate
   PublishFix job checks out a clean default branch and downloads that handoff.
   An `actions/github-script` step requires status `ready`, `needs_clarification`,
   or `no_changes` and a nonempty body. Only `ready` proceeds to patch application
   and PR creation, after validating nonempty single-line title/commit-message
   fields. A title or commit message longer than 72 characters is shortened at a
   word boundary, with a warning, rather than failing the run. PR publication stops if
   the default branch has moved since the agent's checkout; request a fresh fix.
   The publisher checks and applies the patch without running project code.
4. `peter-evans/create-pull-request@v8.1.1` commits and pushes the changes to
   `fix/issue-<issue-number>` for issues labeled `bug`, `feat/issue-<issue-number>`
   for issues labeled `enhancement`, or `chore/issue-<issue-number>` otherwise,
   and opens a draft PR against the default branch using the agent-generated metadata.
   If both labels are present, `bug` takes precedence. Subsequent runs can update
   the existing PR on the selected branch. No new PR is created when there is no
   diff against the base.
5. `gh issue comment` posts the PR link on the issue, or a no-diff message with the
   agent's explanation when no PR was created.

For `needs_clarification`, the publisher posts the agent's questions and skips
patch application and PR creation, even when the handoff contains partial edits.
When there are partial edits, the comment links to the patch bundle and workflow
run, gives the approximate expiration time, and includes a hidden marker with
the run and artifact IDs.
Reply with `@claude fix` or `@claude implement` followed by the answers to start
a fresh run that reads the updated discussion. A plain reply does not trigger the assistant. For `no_changes`, the
publisher posts the explanation without applying a patch or creating a PR.

**CI triggering:** publication uses the repository's `GITHUB_TOKEN`. Its pushes
do not start `push` workflows. Matching `pull_request` workflows for `opened`,
`synchronize`, or `reopened` events [require approval to run](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow).
Approve those runs from the PR or use your repository's manual CI path. Fully
automatic follow-on CI would require a publication change, such as an
appropriately scoped GitHub App token; Fix does not expose a publication-token
secret. Remember that `pull_request` branch filters match the base branch.

### Pilot boundaries

Try triage (including `@claude triage` on an outside reporter's issue), `@claude analyze`, and `@claude fix` on a trusted issue first. Confirm
labels, analysis, the fix branch, draft PR metadata, the issue reply, and the
chosen CI triggering arrangement before relying on the workflow. Local workflow
linting does not verify live GitHub/model execution.

Triage, Analyze, and Fix use an `author_association` pre-filter, their own
write-access check, and the Claude Code Action's built-in write-access checks.
Confirm that a maintainer whose org membership is private is not skipped. Fix executes changed project code only through its sandboxed offline
verifier. Confirm on the pilot runner that the verifier cannot reach the network,
the OIDC request variables, or the federation token/cache. Confirm an unapproved
workflow ref is denied in Claude Console authentication history. Write-capable
publication runs on a separate runner without OIDC and consumes only the
patch/report and metadata. Human review and required checks still govern merges.

For a Maven project with private dependencies, confirm the baseline warms them
on a fresh runner and verification succeeds offline without JFrog credentials.

## Adding workflows to a new project

### Maven Java Build

Create/Add a `.github/workflows/build.yml` file within the project.

```yaml
name: Maven Build and Release

on:
  push:
    branches:
      - 'main'
      - 'chore/**'
      - 'feat/**'
  pull_request:
    branches:
      - 'main'

jobs:
  Build-Maven:
    uses: avioconsulting/shared-workflows/.github/workflows/maven-build.yml@main
    secrets: inherit
    # with:
    #   include-mule-ee-repo: false
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   include-test-results: true
    #   maven-args: -X

  Release-Maven:
    needs: Build-Maven
    uses: avioconsulting/shared-workflows/.github/workflows/maven-release.yml@main
    secrets: inherit
    with:
      app-version: ${{ needs.Build-Maven.outputs.app-version }}
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   publish-maven-central: true
    #   maven-args: -X
    #   main-branch: main

  Post-Release-Maven:
    needs: [Build-Maven, Release-Maven]
    uses: avioconsulting/shared-workflows/.github/workflows/maven-post-release.yml@main
    secrets: inherit
    with:
      app-version: ${{ needs.Build-Maven.outputs.app-version }}
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   maven-args: -X
    #   pr-reviewers: adesjardin, manikmagar, kkingavio
```

### Maven mule-plugin Build

The difference with a mule-plugin build, is that a mule-plugin needs to be deployed to maven central.

```yaml
name: Maven Build and Release for Mule-Plugin

on:
  push:
    branches:
      - 'main'
      - 'chore/**'
      - 'feat/**'
  pull_request:
    branches:
      - 'main'

jobs:
  Build-Maven:
    uses: avioconsulting/shared-workflows/.github/workflows/maven-build.yml@main
    secrets: inherit
    with:
      include-mule-ee-repo: true
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   include-test-results: true
    #   maven-args: -X

  Release-Maven:
    needs: Build-Maven
    uses: avioconsulting/shared-workflows/.github/workflows/maven-release.yml@main
    secrets: inherit
    with:
      app-version: ${{ needs.Build-Maven.outputs.app-version }}
      publish-maven-central: true
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   maven-args: -X
    #   main-branch: main

  Post-Release-Maven:
    needs: [Build-Maven, Release-Maven]
    uses: avioconsulting/shared-workflows/.github/workflows/maven-post-release.yml@main
    secrets: inherit
    with:
      app-version: ${{ needs.Build-Maven.outputs.app-version }}
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   maven-args: -X
    #   main-branch: main
    #   pr-reviewers: adesjardin, manikmagar, kkingavio
```

### Gradle Java Build

```yaml
name: Gradle Build and Release

on:
  push:
    branches:
      - 'main'
      - 'chore/**'
      - 'feat/**'
  pull_request:
    branches:
      - 'main'

jobs:
  Build-Gradle:
    uses: avioconsulting/shared-workflows/.github/workflows/gradle-build.yml@main
    secrets: inherit
    # with:
    #   include-mule-ee-repo: false
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   include-test-results: true
    #   gradle-args: -X

  Release-Gradle:
    needs: Build-Gradle
    uses: avioconsulting/shared-workflows/.github/workflows/gradle-release.yml@main
    secrets: inherit
    with:
      app-version: ${{ needs.Build-Gradle.outputs.app-version }}
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   gradle-args: -X
    #   main-branch: main

  Post-Release-Gradle:
    needs: [Build-Gradle, Release-Gradle]
    uses: avioconsulting/shared-workflows/.github/workflows/gradle-post-release.yml@main
    secrets: inherit
    with:
      app-version: ${{ needs.Build-Gradle.outputs.app-version }}
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   main-branch: main
    #   version-project: mule-linter-core
    #   pr-reviewers: adesjardin, manikmagar, kkingavio
```

### UV Python Build
Must use [UV](https://docs.astral.sh/uv/guides/install-python/) as package manager

```yaml
name: Build and Release for Python Project

on:
  push:
    branches:
      - 'main'
      - 'chore/**'
      - 'feat/**'
  pull_request:
    branches:
      - 'main'

jobs:
  Build-Python:
    secrets: inherit
    uses: avioconsulting/shared-workflows/.github/workflows/python-build.yml@main
    with: 
      # python-version: '3.12'
      # include-test-results: true

  Release-Python:
    secrets: inherit
    needs: Build-Python
    uses: avioconsulting/shared-workflows/.github/workflows/python-release.yml@main
    with:
      app-version: ${{ needs.Build-Python.outputs.app-version }}
      # main-branch: 'main'

  Post-Release-Python:
    secrets: inherit
    needs: [Build-Python, Release-Python]
    uses: avioconsulting/shared-workflows/.github/workflows/python-post-release.yml@main
    with:
      app-version: ${{ needs.Build-Python.outputs.app-version }}
      # python-version: '3.12'
      # main-branch: 'main'
      # pr-reviewers: adesjardin, manikmagar, kkingavio
```

## Shared Workflow Details

### `maven-build.yml`

* Runs on any branches defined by the project build.yml, which should include `main`, `chore/*`, `feat/*`
* Executes initial setup, then `.mvnw compile`, `.mvnw verify` and then publishes the unit test results

| Variable               | Description                                                                                     | Type    | Required | Default       | Input/Output |
|------------------------|-------------------------------------------------------------------------------------------------|---------|----------|---------------|--------------|
| `include-mule-ee-repo` | Adds the Mule EE repo to settings.xml for java setup.  This is needed for mule plugin projects. | boolean | false    | false         | input        |
| `java-distribution`    | The Java Distribution to use                                                                    | string  | false    | adopt-hotspot | input        |
| `java-version`         | The Java Version to use                                                                         | string  | false    | 8             | input        |
| `include-test-results` | A flag to run publish-unit-test-result                                                          | boolean | false    | true          | input        |
| `maven-args`           | Maven arguments appended to maven calls                                                         | string  | false    | n/a           | input        |
| `app-version`          | The application version                                                                         | string  | n/a      | n/a           | output       |


### `maven-release.yml`

* Only runs on the `main` branch
* Executes initial setup, then runs JReleaser, and captures the JReleaser output. Optionally it will release the artifact to maven central

| Variable                | Description                                                         | Type    | Required | Default       | Input/Output |
|-------------------------|---------------------------------------------------------------------|---------|----------|---------------|--------------|
| `app-version`           | Application version to release                                      | string  | true     | n/a           | input        |
| `publish-maven-central` | Publish artifact to maven central, enable this for mule-plugins     | boolean | false    | false         | input        |
| `java-distribution`     | The Java Distribution to use                                        | string  | false    | adopt-hotspot | input        |
| `java-version`          | The Java Version to use                                             | string  | false    | 8             | input        |
| `main-branch`           | Main branch name, allows override for repos that still use ‘master’ | string  | false    | main          | input        |
| `maven-args`            | Maven arguments appended to maven calls                             | string  | false    | n/a           | input        |

The Maven build runs `verify` with tests enabled by default. Subsequent staging
and publishing deploys pass `-DskipTests=true -DskipITs=true` to skip Surefire
and Failsafe respectively. Projects that start services for integration tests
should use `skipITs` to skip that setup and teardown as well.

For app-authored next-snapshot PRs, pass `release-app-id` and the
`RELEASE_APP_PRIVATE_KEY` secret to `maven-post-release.yml`. Without an ID,
existing callers continue to use `GITHUB_TOKEN`. The optional
`maven-snapshot-check.yml` workflow returns `verified=true` only for the AVIO
release bot's exact next-patch edit to a single-module root POM. Keep normal
builds on `main` and run full PR checks unless that output is true.

Gradle callers can opt into the same release bot via `gradle-post-release.yml`
with `release-app-id` and `RELEASE_APP_PRIVATE_KEY`. To enable the lightweight
PR check, call `gradle-snapshot-check.yml` with `version-files` listing every
expected Semver plugin `version.properties` path (one per line). The complete
diff must bump all listed files to the same next patch snapshot. Other layouts
or edits take the normal build path. As with Maven, keep full builds on `main`
and exclude generated snapshot branches from duplicate push builds.

Maven release workflow requires the following [Organization Secrets](https://github.com/organizations/avioconsulting/settings/secrets/actions) to be shared with the target repository :

- JRELEASER_GPG_PASSPHRASE
- JRELEASER_GPG_PUBLIC_KEY
- JRELEASER_GPG_SECRET_KEY
- MAVEN_GPG_PASSPHRASE
- MAVEN_GPG_PRIVATE_KEY
- MULE_EE_PASSWORD (if Mule EE is needed)
- MULE_EE_USERNAME (if Mule EE is needed)

Depending on the target maven deployment repository, configure the target repository secrets and maven POM.

**NOTE: Do not set both sets of secrets on a repository, build may fail.**

If you are using [avio-mule-modules-parent](https://github.com/avioconsulting/avio-mule-modules-parent), check its documentation for how to configure Maven POM.

#### Deploying to Maven Central

Assign following secrets to the target repository -
- OSSRH_PASSWORD
- OSSRH_USERNAME

Distribution Management section - 

```xml
    <distributionManagement>
        <repository>
            <id>ossrh</id>
            <url>${nexus.url}/service/local/staging/deploy/maven2/</url>
        </repository>
        <snapshotRepository>
            <id>ossrh</id>
            <url>${nexus.url}/content/repositories/snapshots</url>
        </snapshotRepository>
    </distributionManagement>
```

### Deploying to AVIO's JFrog Artifactory

Assign following Secrets to the target repository -
- MAVEN_PUBLISH_USERNAME
- MAVEN_PUBLISH_PASSWORD

Distribution Management section -

```xml
  <distributionManagement>
    <repository>
      <id>maven-publish</id>
      <name>AVIO Releases Repository</name>
      <url>https://avio.jfrog.io/artifactory/mulesoft-mvn-local/</url>
    </repository>
    <snapshotRepository>
      <id>maven-publish</id>
      <name>AVIO Snapshots Repository</name>
      <url>https://avio.jfrog.io/artifactory/mulesoft-mvn-local/</url>
    </snapshotRepository>
  </distributionManagement>
```

### `maven-post-release.yml`

* Only runs on `main` branch, if the `app-version` does not include 'SNAPSHOT'
* Executes initial setup, increments the pom version, then creates a pull request for the version changes

| Variable            | Description                                                         | Type   | Required | Default                           | Input/Output |
|---------------------|---------------------------------------------------------------------|--------|----------|-----------------------------------|--------------|
| `app-version`       | Application version that was released                               | string | true     | n/a                               | input        |
| `java-distribution` | The Java Distribution to use                                        | string | false    | adopt-hotspot                     | input        |
| `java-version`      | The Java Version to use                                             | string | false    | 8                                 | input        |
| `main-branch`       | Main branch name, allows override for repos that still use ‘master’ | string | false    | main                              | input        |
| `pr-reviewers`      | Users to be included in the post-release pull request               | string | false    | adesjardin, manikmagar, kkingavio | input        |


### `gradle-build.yml`

* Runs on any branches defined by the project build.yml, which should include `main`, `chore/*`, `feat/*`
* Executes initial setup, then `.mvnw compile`, `.mvn verify` and then publishes the unit test results

| Variable               | Description                                                                       | Type    | Required | Default       | Input/Output |
|------------------------|-----------------------------------------------------------------------------------|---------|----------|---------------|--------------|
| `java-distribution`    | The Java Distribution to use                                                      | string  | false    | adopt-hotspot | input        |
| `java-version`         | The Java Version to use                                                           | string  | false    | 8             | input        |
| `include-test-results` | A flag to run publish-unit-test-result                                            | boolean | false    | true          | input        |
| `version-project`      | For use with multi-module projects, this is used to determine application version | boolean | false    | n/a           | input        |
| `gradle-args`          | Gradle arguments appended to gradle calls                                         | string  | false    | n/a           | input        |
| `app-version`          | The application version                                                           | string  | n/a      | n/a           | output       |


### `gradle-release.yml`

* Only runs on the `main` branch
* Executes initial setup, then runs JReleaser, and captures the JReleaser output.

The Gradle release workflow uses JReleaser 1.26.0 and the Central Publisher Portal.
Consumers must migrate their `jreleaser.yml` deployer from `nexus2` to
`mavenCentral`, for example:

```yaml
deploy:
  maven:
    mavenCentral:
      maven-central:
        active: RELEASE
        url: https://central.sonatype.com/api/v1/publisher
        applyMavenCentralRules: true
        stagingRepositories:
          - build/staging-deploy
```

Use the actual staging directory (or directories) produced by the consuming
project's `./gradlew publish`. Remove Nexus2-only `closeRepository`,
`releaseRepository`, and `snapshotUrl` settings. This configuration publishes
releases only; snapshot publication requires a separate snapshot deployer.

For JReleaser-managed snapshots, consumers may add a `nexus2` deployer with
`active: SNAPSHOT`, `snapshotSupported: true`, and `snapshotUrl` pointing to
`https://central.sonatype.com/repository/maven-snapshots/`. Its `url` may use the
same URL. For SNAPSHOT versions this adapter uses Maven repository PUT requests,
not legacy OSSRH staging APIs. Keep the `mavenCentral` deployer release-only.
The namespace must have snapshots enabled in Central Portal.

The `MAVEN_PUBLISH_USERNAME` and `MAVEN_PUBLISH_PASSWORD` organization secrets
must contain **Central Portal user-token credentials**, as in Maven publishing.
They are exposed as `JRELEASER_MAVENCENTRAL_USERNAME` and
`JRELEASER_MAVENCENTRAL_PASSWORD`; legacy OSSRH tokens will not work.
The same credentials are also exposed as `JRELEASER_NEXUS2_USERNAME` and
`JRELEASER_NEXUS2_PASSWORD` for snapshot deployers. Normal publishing remains
restricted to the exact configured main branch.

For testing on a non-main branch, pass `dry-run: true`. This stages artifacts
locally and runs `full-release --dry-run`: it does not publish to Central or
create a GitHub release. Pull requests remain excluded. Use a non-SNAPSHOT
`app-version` and matching Gradle version override to exercise the release-only
deployer, for example `gradle-args: '-Dversion.semver=2.0.0-centralmigrationtest'`
for projects using the semver plugin. Signing and Portal credentials are still
required for CI validation. A dry run does not prove server-side authentication
or publication succeeds.

| Variable            | Description                                                         | Type   | Required | Default       | Input/Output |
|---------------------|---------------------------------------------------------------------|--------|----------|---------------|--------------|
| `app-version`       | Application version to release                                      | string | true     | n/a           | input        |
| `java-distribution` | The Java Distribution to use                                        | string | false    | adopt-hotspot | input        |
| `java-version`      | The Java Version to use                                             | string | false    | 8             | input        |
| `main-branch`       | Main branch name, allows override for repos that still use ‘master’ | string | false    | main          | input        |
| `gradle-args`       | Gradle arguments appended to gradle calls                           | string | false    | n/a           | input        |

Additional input: `dry-run` (boolean, optional, default `false`). Real releases
require an exact match of `refs/heads/<main-branch>`; dry runs may run on other
branches without relaxing the real-release guard.

Gradle release workflow requires the following [Organization Secrets](https://github.com/organizations/avioconsulting/settings/secrets/actions) to be shared with the target repository :

- MAVEN_GPG_PASSPHRASE
- MAVEN_GPG_PUBLIC_KEY
- MAVEN_GPG_PRIVATE_KEY
- MAVEN_PUBLISH_PASSWORD
- MAVEN_PUBLISH_USERNAME

The Maven signing secrets are mapped to JReleaser's `JRELEASER_GPG_*`
environment variables. The private key must be an exported PGP secret key in a
format supported by JReleaser's in-memory signer (ASCII-armored or Base64-encoded),
not a filesystem path. No organization secret values need to be copied or exposed.

### `gradle-post-release.yml`

* Only runs on `main` branch, if the `app-version` does not include 'SNAPSHOT'
* Executes initial setup, increments the version, then creates a pull request for the version changes

| Variable            | Description                                                                       | Type    | Required | Default                           | Input/Output |
|---------------------|-----------------------------------------------------------------------------------|---------|----------|-----------------------------------|--------------|
| `app-version`       | Application version that was released                                             | string  | true     | n/a                               | input        |
| `java-distribution` | The Java Distribution to use                                                      | string  | false    | adopt-hotspot                     | input        |
| `java-version`      | The Java Version to use                                                           | string  | false    | 8                                 | input        |
| `main-branch`       | Main branch name, allows override for repos that still use ‘master’               | string  | false    | main                              | input        |
| `version-project`   | For use with multi-module projects, this is used to determine application version | boolean | false    | n/a                               | input        |
| `pr-reviewers`      | Users to be included in the post-release pull request                             | string  | false    | adesjardin, manikmagar, kkingavio | input        |

### `python-build.yml`

* Runs on any branches defined by the project build.yml, which should include `main`, `chore/*`, `feat/*`
* Executes initial setup, tests with pytest, and then publishes the unit test results

| Variable               | Description                                                                       | Type    | Required | Default       | Input/Output |
|------------------------|-----------------------------------------------------------------------------------|---------|----------|---------------|--------------|
| `python-version`       | The Python Version to use                                                         | string  | false    | 3.12          | input        |
| `include-test-results` | A flag to publish unit test results                                               | boolean | false    | true          | input        |
| `app-version`          | The application version                                                           | string  | n/a      | n/a           | output       |

### `python-release.yml`

* Only runs on the `main` branch
* Executes downloads artifacts, then runs JReleaser, and captures the JReleaser output.

| Variable            | Description                                                         | Type   | Required | Default       | Input/Output |
|---------------------|---------------------------------------------------------------------|--------|----------|---------------|--------------|
| `app-version`       | Application version to release                                      | string | true     | n/a           | input        |
| `main-branch`       | Main branch name, allows override for repos that still use ‘master’ | string | false    | main          | input        |

Python release workflow requires the following [Organization Secrets](https://github.com/organizations/avioconsulting/settings/secrets/actions) to be shared with the target repository:

- JRELEASER_GPG_PASSPHRASE
- JRELEASER_GPG_PUBLIC_KEY
- JRELEASER_GPG_SECRET_KEY

### `python-post-release.yml`

* Only runs on `main` branch, if the `app-version` does not include '.dev'
* Executes initial setup, increments the patch version and adds `.dev1`, then creates a pull request for the version changes

| Variable            | Description                                                         | Type   | Required | Default                           | Input/Output |
|---------------------|---------------------------------------------------------------------|--------|----------|-----------------------------------|--------------|
| `app-version`       | Application version to release                                      | string | true     | n/a                               | input        |
| `python-version`    | The Python Version to use                                           | string | false    | 3.12                              | input        |
| `main-branch`       | Main branch name, allows override for repos that still use ‘master’ | string | false    | main                              | input        |
| `pr-reviewers`      | Users to be included in the post-release pull request               | string | false    | adesjardin, manikmagar, kkingavio | input        |

### Publish docs to GitHub Pages

Create/Add a `.github/workflows/github-pages.yml` file within the project.

```yaml
# Simple workflow for deploying static content to GitHub Pages
name: Deploy static content to Pages

on:
  # Runs on pushes targeting the default branch
  # Temporary on current branch
  push:
    branches:
      - main

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

jobs:
  Build-Maven:
    uses: avioconsulting/shared-workflows/.github/workflows/github-pages.yml@feat/add-github-pages
    secrets: inherit
    # with:
    #   java-distribution: temurin
    #   java-version: 8
    #   pages-directory: './target/docs'
```

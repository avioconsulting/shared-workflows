# Github Actions - Shared Workflows
A repository to store the Github Actions shared workflows

## AI framework maintenance

The Triage, Analyze, and Fix workflows use the official
[Claude Code Action](https://github.com/anthropics/claude-code-action/tree/main/examples).

| Workflow | Trigger in the calling repository | Result |
| --- | --- | --- |
| `ai-triage.yml` | Human-authored issue opened, or `needs-triage` label added by a writer | Labels and a concise triage comment |
| `ai-analyze.yml` | New human-authored issue comment starting with `@claude analyze` | Analysis comment |
| `ai-fix.yml` | New human-authored issue comment starting with `@claude fix` | Fix branch and draft PR, or an issue reply if blocked |

### Setup

1. Follow [Use WIF with GitHub Actions](https://platform.claude.com/docs/en/manage-claude/wif-providers/github-actions).
   In the Claude Console, register the GitHub OIDC issuer and create a
   [federation rule for each enabled workflow](CLAUDE_WIF_GITHUB_ACTIONS.md).
   Match the `https://api.anthropic.com` audience, approved calling repositories and
   their protected default branches, and the exact reusable `job_workflow_ref`
   (including the shared workflow ref). Match `event_name` to `issues` for triage
   and `issue_comment` for Analyze and Fix. Configure workspace spending limits.
2. In **Settings → Secrets and variables → Actions → Variables** of each
   **calling repository**, set `ANTHROPIC_ORGANIZATION_ID`,
   `ANTHROPIC_TRIAGE_FEDERATION_RULE_ID`, `ANTHROPIC_ANALYZE_FEDERATION_RULE_ID`,
   and `ANTHROPIC_FIX_FEDERATION_RULE_ID` as applicable. These identifiers are
   not secrets. Organization-level variables scoped to approved callers also work.
   Add `id-token: write` to each caller job below and remove its API-key mapping
   or `secrets: inherit` if no other secrets are needed. Pilot against a reviewed
   shared-workflow ref with matching Console rules before moving existing callers.
3. Precreate `bug`, `enhancement`, `question`, `ai-triaged`, `need information`,
   and `needs-triage` labels in the calling repository. For custom classifications,
   create those labels and describe their meaning in repository guidance or label
   descriptions. Adding `needs-triage` is the maintainer approval for issues opened
   by users without write access.
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

These workflows accept an optional `model` input. Triage defaults to
`claude-haiku-5-5`; Analyze and Fix default to `claude-sonnet-5-5`.

| Workflow | Additional inputs | Max turns | Claude CLI budget | Configured timeout |
| --- | --- | --- | --- | --- |
| `ai-triage.yml` | `triage-labels`: comma-separated classification labels | 12 | $1 | 10 minutes |
| `ai-analyze.yml` | None | 30 | $6 | Acknowledge: 5 minutes; Analyze: 35 minutes |
| `ai-fix.yml` | `java-version` (default `8`), `java-distribution` (default `temurin`) | 30 | $6 | Acknowledge: 5 minutes; Fix: 60 minutes; publication: 10 minutes |

The triage caller above explicitly sets `triage-labels` to
`bug,enhancement,question`. The workflow's current default also includes
`ai-triaged` and `"need information"`; these are operational labels, so keep them
out of the classification input. The prompt handles those operational labels
separately. All labels must already exist; triage skips missing labels.

None of these workflows exposes an `enabled` input. Disable a workflow by
disabling its calling workflow in GitHub Actions or gating the caller job with
an `if` condition. CLI budgets apply to individual agent invocations, not aggregate
workflow, repository, or organization spending. Configure API spending limits
separately.

Fix currently declares `maven-args` but does not use it for warmup or verification.

### Triage behavior

The triage prompt uses repository guidance and existing label descriptions, and
requests the following behavior:

- Prefer one supported classification, preserve existing labels, and abstain when
  the classification is unclear.
- Request `need information` only when missing details block the next step.
  Assess bugs, enhancements, and questions differently; ask at most three focused
  questions rather than requiring reproduction steps for every issue.
- Search open and closed issues using at most two targeted searches and inspect
  at most three strong duplicate candidates. Link potential duplicates with an
  explanation; do not close issues or label them as duplicates.
- Produce a comment normally 100–180 words, capped at 250, with a summary,
  necessary clarification, supported duplicate links, and one concrete next step.
- Skip closed issues and avoid source changes, project-code execution, and fixes.

Triage identifies its previous comment using the workflow bot's identity
and `<!-- ai-issue-triage:v1 -->` marker. On a rerun it skips another comment and
completes only justified missing label additions. It adds `ai-triaged` after
posting or confirming an existing triage comment.

Issues opened by a user with repository write access are triaged automatically.
The workflow checks the event sender's write access
before checkout, and the Claude Code Action also checks it before triage.
For other issue creators, a user with write access can add `needs-triage` to
request triage. Users with triage-only access cannot authorize a run. Runs
are serialized per issue.

### Assistant commands and permissions

Post one of these commands on an issue:

| Command | Behavior |
| --- | --- |
| `@claude analyze why this fails` | Investigate and comment with evidence, confirmed findings versus hypotheses, a suggested fix scope and regression test, compatibility risks, and missing information |
| `@claude fix this bug` | Make focused code changes, run relevant checks, and create a draft PR with an issue reply |

Each workflow's `Acknowledge` job matches its own command at the very start of
the comment, with no leading whitespace. Use the command by itself or add a
literal space before optional task details; a tab or newline immediately after
the command is not a separator. Matching is case-insensitive. A bare `@claude`,
unsupported commands such as `@claude review`, and mentions elsewhere in the
comment do not start an agent job.

Only newly created, human-authored **issue** comments are eligible. PR conversation
comments, inline reviews, and comment edits are excluded. Both workflows check
repository write access before adding an eyes reaction and before checking out
the project. The Claude Code Action also checks write access before running
either agent.

Each job receives only the permissions it needs:

| Job | Contents | Issues | Pull requests | OIDC |
| --- | --- | --- | --- | --- |
| Acknowledge (both workflows) | None | Write | None | No |
| Analyze | Read | Write | None | Yes |
| Fix | Read | Read | Read | Yes |
| PublishFix | Write | Write | Write | No |

Same-run artifact downloads do not require an additional `actions: read` grant. Both agents
use explicit automation prompts, avoiding interactive branch setup. Checkout
credentials remain unpersisted. Like triage, Analyze receives `GH_TOKEN`
and posts findings directly using `gh issue comment`. Its instructions permit
only one analysis comment as a GitHub change and prohibit file edits and
project-code execution. Its tool allowlist permits file reads, `gh issue view`,
and `gh issue comment`. The contents-read token cannot push repository changes;
the issues-write permission is broader than the comment-only instruction.

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
   fields and a title of at most 256 characters. PR publication stops if
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
Reply with `@claude fix` and the answers to start a fresh run that reads the updated
discussion. A plain reply does not trigger the assistant. For `no_changes`, the
publisher posts the explanation without applying a patch or creating a PR.

**CI triggering:** publication uses the repository's `GITHUB_TOKEN`. Its pushes
do not start `push` workflows. Matching `pull_request` workflows for `opened`,
`synchronize`, or `reopened` events [require approval to run](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow).
Approve those runs from the PR or use your repository's manual CI path. Fully
automatic follow-on CI would require a publication change, such as an
appropriately scoped GitHub App token; Fix does not expose a publication-token
secret. Remember that `pull_request` branch filters match the base branch.

### Pilot boundaries

Try triage, `@claude analyze`, and `@claude fix` on a trusted issue first. Confirm
labels, analysis, the fix branch, draft PR metadata, the issue reply, and the
chosen CI triggering arrangement before relying on the workflow. Local workflow
linting does not verify live GitHub/model execution.

Triage, Analyze, and Fix use the Claude Code Action's built-in write-access
checks. Fix executes changed project code only through its sandboxed offline
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

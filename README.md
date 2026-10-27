# Github Actions - Shared Workflows
A repository to store the Github Actions shared workflows

## AI framework maintenance

Two reusable workflows use the official [Claude Code Action](https://github.com/anthropics/claude-code-action/tree/main/examples).
A third provides an alternative triage implementation using [GitHub Agentic Workflows (gh-aw)](https://github.github.com/gh-aw/):

| Workflow | Trigger in the calling repository | Result |
| --- | --- | --- |
| `ai-triage.yml` | Human-authored issue opened | Labels and a concise triage comment |
| `ai-assistant.yml` | New conversation comment mentioning `@claude` | The action's normal analysis, fix, or review interaction |
| `ai-triage-gh-aw.lock.yml` | Human-authored issue opened | Labels and a workflow-managed triage comment, published through safe outputs |

Choose one triage implementation per calling repository.

### Setup

1. Create a dedicated API key in the [Anthropic Console](https://console.anthropic.com/)
   and configure appropriate API spending limits.
2. In the **calling framework repository**, open **Settings → Secrets and variables
   → Actions → New repository secret**. Store the key as `ANTHROPIC_API_KEY`.
   Alternatively, create an organization-level Actions secret with the same name
   and grant the calling repositories access. Pass the secret explicitly as shown
   below; the reusable workflow declares the secret but does not store its value.
3. Precreate `bug`, `enhancement`, `question`, `ai-triaged`, and `need information`
   labels in the calling repository. For custom classifications, create those
   labels and describe their meaning in repository guidance or label descriptions.
4. Add repository guidance in `AGENTS.md` and a `CLAUDE.md` that imports `@AGENTS.md`.
   Keep framework-specific triage requirements in that guidance and issue templates.

All three workflows are configured for **Anthropic API billing**, charged to the
organization associated with the API key. Claude Pro/Max subscriptions do not fund
this API usage. GitHub Actions runner usage is accounted for separately.

The shared repository must be accessible to the caller. The examples use `@main`;
use the branch containing these workflows for a pilot, and a reviewed SHA for rollout.

Create `.github/workflows/ai-triage.yml` in the framework repository:

```yaml
name: AI Issue Triage
on:
  issues:
    types: [opened]
jobs:
  Triage:
    uses: avioconsulting/shared-workflows/.github/workflows/ai-triage.yml@main
    permissions:
      contents: read
      issues: write
    with:
      triage-labels: bug,enhancement,question
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Create `.github/workflows/ai-assistant.yml` in the framework repository:

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
    # with:
    #   enabled: false
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Inputs and execution limits

All three workflows accept an optional `model` input, defaulting to
`claude-sonnet-4-5-20250929`. Pass named secrets, not `secrets: inherit`.

| Workflow | Additional inputs | Max turns | Claude CLI budget | Configured timeout |
| --- | --- | --- | --- | --- |
| `ai-triage.yml` | `triage-labels`: comma-separated classification labels | 12 | $1 | 10 minutes |
| `ai-assistant.yml` | `enabled`: boolean, default `true` | 30 | $6 | 35 minutes |
| `ai-triage-gh-aw.lock.yml` | No additional user-facing inputs; labels configured in the Markdown source | 12 | $1 | 10 minutes for agent execution |

The direct-triage caller above explicitly sets `triage-labels` to
`bug,enhancement,question`. The workflow's current default also includes
`ai-triaged` and `"need information"`; these are operational labels, so keep them
out of the classification input. The prompt handles those operational labels
separately. All labels must already exist; triage skips missing labels.

Only the assistant supports `enabled: false`. Disable either triage implementation
by disabling its calling workflow in GitHub Actions or gating the caller job with
an `if` condition. CLI budgets apply to individual agent invocations, not aggregate
workflow, repository, or organization spending. Configure API spending limits
separately; gh-aw can run additional model-backed processing outside the main
triage invocation.

### Triage behavior

Both triage prompts use repository guidance and existing label descriptions, and
request the following behavior:

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

Direct triage identifies its previous comment using the workflow bot's identity
and `<!-- ai-issue-triage:v1 -->` marker. On a rerun it skips another comment and
completes only justified missing label additions. It adds `ai-triaged` after
posting or confirming an existing triage comment. The gh-aw version instead
submits an updated comment for automatic replacement and requests additive labels
through safe outputs, which are processed after the agent finishes.

### Commands and pilot boundaries

Use comments such as `@claude analyze why this fails`, `@claude fix this bug`, or
`@claude review this PR`. These are task directions, not a custom command parser.
The assistant uses the action's built-in context, write-access checks, bot checks,
comments, and branch handling. Any mention can trigger it, including quoted examples.
Only newly created main-conversation comments are selected by the caller above.
Analyze and review requests are instructed to avoid file changes and project-code
execution; fix requests may change code and run relevant checks. Reviews are
limited to five high-confidence actionable findings.

Fixes use the action's normal branch and PR-creation-link behavior; this wrapper
does not automatically open a draft PR. Follow the returned link to open the PR,
then let existing CI and human review gate merging. Review instructions request
advisory comments only; they are not a separate policy-enforcement layer.

**Pilot scope:** try this on a trusted same-repository issue/PR first. There are no
custom authorization scripts, structured-output publishers, stale-head guards,
separate publication credentials, daily quotas, or isolated test runners. Claude
can use tools and execute project commands in the model-authenticated job.
Triage allows outside contributors through `allowed_non_write_users` and gives
Claude issue-write access directly, as in the official example. Tool/label limits
are instructions and permission configuration, not an untrusted-code sandbox.
Monitor API usage and use the disabling options described above.

To prove the integration, open one issue, request analysis and a fix, open the PR
using Claude's link, and request a review. Confirm comments, labels, branch output,
normal CI, and built-in authorization on the pilot. Live GitHub/model execution
is not verified by local workflow linting.

### gh-aw triage comparison

[`ai-triage-gh-aw.md`](.github/workflows/ai-triage-gh-aw.md) implements the same
triage task with Claude and the main-agent limits listed above. Unlike the
direct-action version, its agent has read-only
GitHub access; gh-aw publishes through built-in safe outputs with a fixed label
allowlist and replacement of its older triage comment. No custom scripts are used.

The Markdown is the editable source. GitHub runs the generated
[`ai-triage-gh-aw.lock.yml`](.github/workflows/ai-triage-gh-aw.lock.yml).
Use gh-aw v0.92.0 to recompile source changes:

```sh
gh aw compile ai-triage-gh-aw
```

Commit both the Markdown source and regenerated lock workflow.
`.github/aw/actions-lock.json` records the compiler's action pin;
`.gitattributes` marks generated workflow files. Do not hand-edit the lock workflow.

To compare in a pilot, replace the triage caller job above with this job rather
than running both against the same issue:

```yaml
  Triage:
    uses: avioconsulting/shared-workflows/.github/workflows/ai-triage-gh-aw.lock.yml@main
    permissions:
      actions: read
      contents: read
      issues: write
      pull-requests: write
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

The `model` input is supported; there is no `enabled` or `triage-labels` input.
To customize labels, update both the `safe-outputs.add-labels.allowed` list and
the triage instructions in the Markdown source, then recompile. Keep operational
and classification label meanings distinct. All additions are requested in one
`add_labels` call, and one `add_comment` call submits the complete triage comment.

The additional caller permissions accommodate gh-aw's generated status and
safe-output jobs, not its read-only agent. Use `gh aw compile` for compiler
validation. Older actionlint versions may not understand generated constructs
such as `concurrency.queue` and `job.workflow_*`.

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

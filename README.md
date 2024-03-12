# Github Actions - Shared Workflows
A repository to store the Github Actions shared workflows

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
    # with:
    #   include-mule-ee-repo: false
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   include-test-results: true
    #   maven-args: -X

  Release-Maven:
    needs: Build-Maven
    uses: avioconsulting/shared-workflows/.github/workflows/maven-release.yml@main
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
    with:
      include-mule-ee-repo: true
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   include-test-results: true
    #   maven-args: -X

  Release-Maven:
    needs: Build-Maven
    uses: avioconsulting/shared-workflows/.github/workflows/maven-release.yml@main
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
    # with:
    #   include-mule-ee-repo: false
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   include-test-results: true
    #   gradle-args: -X

  Release-Gradle:
    needs: Build-Gradle
    uses: avioconsulting/shared-workflows/.github/workflows/gradle-release.yml@main
    with:
      app-version: ${{ needs.Build-Gradle.outputs.app-version }}
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   gradle-args: -X
    #   main-branch: main

  Post-Release-Gradle:
    needs: [Build-Gradle, Release-Gradle]
    uses: avioconsulting/shared-workflows/.github/workflows/gradle-post-release.yml@main
    with:
      app-version: ${{ needs.Build-Gradle.outputs.app-version }}
    #   java-distribution: adopt-hotspot
    #   java-version: 8
    #   main-branch: main
    #   version-project: mule-linter-core
    #   pr-reviewers: adesjardin, manikmagar, kkingavio
```
### Mule CloudHub Build
* Full [Github flow](https://docs.github.com/en/get-started/using-github/github-flow) release process for a MuleSoft application utilizing AVIO's Mule Deploy Library to deploy to CloudHub 1.0. 
* To take advantage of the Mule shared workflow you will need to complete the following steps:
  - Configure your application to be deployed with the Mule Deploy Library. 
    * Include the plugin in your project's pom.xml
    * Create and configure a muleDeploy.groovy file in your project's root directory
    * [refer to full documentation here](https://github.com/avioconsulting/mule-deploy-library)
  - Create a jreleaser.yaml file for `early-access` and official releases. 
  - Create/Add a `.github/workflows/build.yml` file within the project similar to the following:

```yaml
name: Mule Build and Release

on:
  # will kick off a build on any push to the following patterns. If it's a non-main branch then only tests will run
  push:
    branches:
      - 'main'
      - 'fix/**'
      - 'feature/**'
      - 'hotfix/**'

  # to kick off manually
  workflow_dispatch:
    inputs:
      tag:
        description: 'Version corresponding to the tag you wish to deploy (eg. v1.0.99011924)'
        required: true
        type: string
      deploy-directly-to-prod:
        description: |
          Set to true if you would like to deploy directly to PROD in an emergency situation. 
          Subject to environment deployment protections
        required: false
        type: boolean
        default: false

  # for running tests when a PR is created or push to fix/hotfix/feature branch
  pull_request:
    types:
      - opened
    branches:
      - 'main'

jobs:
  Mule-Pipeline:
    uses: avioconsulting/shared-workflows/.github/workflows/mule-main.yml@feature/main
    secrets: inherit
    with:
      tag-name: ${{ inputs.tag-name }}
      # Organization name that is the target for deployment
      anypoint-org-name: 'AVIO IT'
      dev-env: 'DEV'
      uat-env: 'QA'
      prod-env: 'PROD'
      # Arguments to add to the end of the invocation of the mule deploy library
      mule-deploy-args: |
        -DmuleDeploy.addParam=${{ secrets.ADD_PARAM }}
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

- GIT_TOKEN
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

| Variable            | Description                                                         | Type   | Required | Default       | Input/Output |
|---------------------|---------------------------------------------------------------------|--------|----------|---------------|--------------|
| `app-version`       | Application version to release                                      | string | true     | n/a           | input        |
| `java-distribution` | The Java Distribution to use                                        | string | false    | adopt-hotspot | input        |
| `java-version`      | The Java Version to use                                             | string | false    | 8             | input        |
| `main-branch`       | Main branch name, allows override for repos that still use ‘master’ | string | false    | main          | input        |
| `gradle-args`       | Gradle arguments appended to gradle calls                           | string | false    | n/a           | input        |

Gradle release workflow requires the following [Organization Secrets](https://github.com/organizations/avioconsulting/settings/secrets/actions) to be shared with the target repository :

- GIT_TOKEN
- JRELEASER_GPG_PASSPHRASE
- JRELEASER_GPG_PUBLIC_KEY
- JRELEASER_GPG_SECRET_KEY
- MAVEN_GPG_PASSPHRASE
- MAVEN_GPG_PRIVATE_KEY
- OSSRH_PASSWORD
- OSSRH_USERNAME

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

### `mule-main.yml`
| Variable            | Description                                                                       | Type    | Required | Default                           | Input/Output |
|---------------------|-----------------------------------------------------------------------------------|---------|----------|-----------------------------------|--------------|
| `anypoint-org-name`       | Organization name to target deployment in CloudHub                                             | string  | true     | n/a                               | input        |
| `dev-env` | Name of corresponding Mule and Github development environment                                                      | string  | true    | n/a                     | input        |
| `uat-env`      | Name of corresponding Mule and Github uat environment                                                            | string  | true    | n/a                                 | input        |
| `prod-env`       | Name of corresponding Mule and Github production environment                | string  | true    | n/a                              | input        |
| `tag-name`   | Tag name to use for deployment. Used with manual kickoff. If blank string then `early-access` will be used | string | true    | n/a                               | input        |
| `mule-deploy-args`      | Additional arguments that should be provided during deployment using the mule deploy library invocation                             | string  | false    | `''` | input        |

### `mule-build-test.yml`
* Runs Munit tests or builds jar file and creates `early-access` release

| Variable            | Description                                                                       | Type    | Required | Default                           | Input/Output |
|---------------------|-----------------------------------------------------------------------------------|---------|----------|-----------------------------------|--------------|
| `test-only`       | Select true if you only want to run tests. False will run tests, build, and publish a snapshot release                                             | boolean  | false     | true                               | input        |
| `mule-env` | Environment to test against and build jar file                                                   | string  | true    | n/a                     | input        |
| `mule-deploy-args`      | Additional arguments that should be provided during deployment using the mule deploy library invocation                             | string  | false    | `''` | input        |
| `released-version`      | The version of the application that was deployed. Automatically set to build number + ddmmyy                             | string  | n/a    | n/a | output        |


### `mule-deploy.yml`
* Gets Mule deployable jar from release and deploys to specified environment

| Variable            | Description                                                                       | Type    | Required | Default                           | Input/Output |
|---------------------|-----------------------------------------------------------------------------------|---------|----------|-----------------------------------|--------------|
| `anypoint-org-name`       | Organization name to target deployment in CloudHub                                             | string  | true     | n/a                               | input        |
| `mule-env` | Target environment for deployment                                                   | string  | true    | n/a                     | input        |
| `tag-name`   | Tag name to retrieve corresponding release. If blank string then `early-access` will be used | string | true    | n/a                               | input        |
| `mule-deploy-args`      | Additional arguments that should be provided during deployment using the mule deploy library invocation                             | string  | false    | `''` | input        |

### `mule-post-release.yml`
* Creates official release after successful deployment

| Variable            | Description                                                                       | Type    | Required | Default                           | Input/Output |
|---------------------|-----------------------------------------------------------------------------------|---------|----------|-----------------------------------|--------------|
| `early-access-tag-name`       | Name of the early-access release name that corresponds to the snapshot release that was just successfully deployed to prod                                         | string  | false     | `'early-access'`                              | input        |
| `version-to-release` | Pom version of the application that was just deployed to production. Generally the output of mule-build-test.yml                                                   | string  | true    | n/a                     | input        |

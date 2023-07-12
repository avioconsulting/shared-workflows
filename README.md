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


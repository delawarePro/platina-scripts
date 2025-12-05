# Intro

This repository contains a set of [YAML templates](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-includes) that can be used to quickly setup pipelines in Azure DevOps.

We don't have support for GitHub Actions (yet), but we're open to suggestions!

# Usage

## dotnet CI pipeline

### Branch Conventions

The [ci.branch.conventions.vars.yml](https://github.com/delawarePro/platina-scripts/blob/master/azure/pipelines/dotnet/ci.branch.conventions.vars.yml) template contains a set of rules that are applied to determine which steps to execute when running a CI build.
Based on the branch name, we set the values of the `withRelease`, `prerelease` and `noTests` variables. These are used by [ci.yml](https://github.com/delawarePro/platina-scripts/blob/master/azure/pipelines/dotnet/ci.yml).

The default convention is:
- When running a prerelease build, we publish artifacts and nugets with the specifiek prerelease version. Whether tests are skipped or not depends on the provided variable.
- When running the CI on a prerelease branch (refs/heads/prereleases/...), we publish artifacts and nugets with a prerelease version, but we skip tests.
- When running a CI build on the main branch or a release branch (refs/heads/releases/...), we publish stable artifacts and nugets, but we skip tests.
- In all other cases (e.g when running a PR build on a feature branch), we run all tests but we don't publish artifacts or nugets.

### Assumptions

The `ci.yml` is taking the following assumptions:
- `nuget.config` is located in the root of the project. A specific path can be configured via the parameters.
- When building nugets, it's recommended to use the [Platina Build](https://dev.azure.com/platina-dlwr/Dlw.Platina/_git/Dlw.Platina?path=/DevOps/src/Build) utilities.

### Examples

#### Basic CI build

The following pipeline is the most basic dotnet restore, build, test pipeline you can create:

```yaml
# Define parameters that can be used to tweak pipeline behaviour on run.
parameters:

# Flag to skip tests.
- name: NoTests
  type: boolean
  default: false
  displayName: 'Skip tests.'

# Add a name to push prerelease packages. This name is used as a version suffix.
- name: Prerelease
  type: string
  default: none
  displayName: 'Pre-release to trigger preparing packages.'

# Trigger pipeline whenever we push to main, master, release or prerelease branch.
trigger:
  batch: true
  branches:
    include:
    - main
    - master
    - releases/*
    - prereleases/*

# Use a Microsoft-hosted pool with Linux image.
pool:
  vmImage: 'ubuntu-latest'

# Add the Platina Scripts GitHub repo (https://github.com/delawarePro/platina-scripts/) as a resource to this pipeline. This allows us to use the Platina YAML templates.
resources:
  repositories:
  - repository: scripts
    type: github
    name: delawarePro/platina-scripts
    ref: 'refs/tags/2.5.0'
    # Name of an Azure DevOps Service Connection that is connected with GitHub.
    endpoint: 'GitHub'

variables:

# Detect Platina conventions based on branch naming.
# This will set 'noTests', 'prerelease' and 'withRelease' variables.
- template: azure/pipelines/dotnet/ci.branch.conventions.vars.yml@scripts
  parameters:
    noTests: ${{ parameters.NoTests }}
    prerelease: ${{ parameters.Prerelease }}

steps:
- template: azure/pipelines/dotnet/ci.yml@scripts
  parameters:
    # Solution to build.
    solution: 'Dlw.MyApp.sln'

    # Test arguments needed to support MSTest.Sdk (for now).
    testArguments: -p:TestingPlatformCommandLineArguments="--report-trx --results-directory $(Agent.TempDirectory) --coverage"
    testCoverageArguments: ''
```

#### CI build with dotnet publish

The following pipeline will add a 'dotnet publish' step and will upload the output to the build artifacts:

```yaml
# Define parameters that can be used to tweak pipeline behaviour on run.
parameters:

# Flag to skip tests.
- name: NoTests
  type: boolean
  default: false
  displayName: 'Skip tests.'

# Add a name to push prerelease packages. This name is used as a version suffix.
- name: Prerelease
  type: string
  default: none
  displayName: 'Pre-release to trigger preparing packages.'

# Trigger pipeline whenever we push to main, master, release or prerelease branch.
trigger:
  batch: true
  branches:
    include:
    - main
    - master
    - releases/*
    - prereleases/*

# Use a Microsoft-hosted pool with Linux image.
pool:
  vmImage: 'ubuntu-latest'

# Add the Platina Scripts GitHub repo (https://github.com/delawarePro/platina-scripts/) as a resource to this pipeline. This allows us to use the Platina YAML templates.
resources:
  repositories:
  - repository: scripts
    type: github
    name: delawarePro/platina-scripts
    ref: 'refs/tags/2.5.0'
    # Name of an Azure DevOps Service Connection that is connected with GitHub.
    endpoint: 'GitHub'

variables:

# Detect Platina conventions based on branch naming.
# This will set 'noTests', 'prerelease' and 'withRelease' variables.
- template: azure/pipelines/dotnet/ci.branch.conventions.vars.yml@scripts
  parameters:
    noTests: ${{ parameters.NoTests }}
    prerelease: ${{ parameters.Prerelease }}

steps:
- template: azure/pipelines/dotnet/ci.yml@scripts
  parameters:
    # Solution to build.
    solution: 'Dlw.MyApp.sln'

    # Test arguments needed to support MSTest.Sdk (for now).
    testArguments: -p:TestingPlatformCommandLineArguments="--report-trx --results-directory $(Agent.TempDirectory) --coverage"
    testCoverageArguments: ''

    # Run dotnet publish, when 'withRelease' is true, artifacts will be uploaded to build pipeline.
    withPublish: true

    # Whether to publish the release artifacts (including containers, nugets, ...).
    withRelease: ${{ eq(variables.withRelease, true) }}
```

#### CI build with dotnet pack

The following pipeline will add a 'dotnet pack' step and will upload the generated nugets to an external NuGet feed.

```yaml
# Define parameters that can be used to tweak pipeline behaviour on run.
parameters:

# Flag to skip tests.
- name: NoTests
  type: boolean
  default: false
  displayName: 'Skip tests.'

# Add a name to push prerelease packages. This name is used as a version suffix.
- name: Prerelease
  type: string
  default: none
  displayName: 'Pre-release to trigger preparing packages.'

# Trigger pipeline whenever we push to main, master, release or prerelease branch.
trigger:
  batch: true
  branches:
    include:
    - main
    - master
    - releases/*
    - prereleases/*

# Use a Microsoft-hosted pool with Linux image.
pool:
  vmImage: 'ubuntu-latest'

# Add the Platina Scripts GitHub repo (https://github.com/delawarePro/platina-scripts/) as a resource to this pipeline. This allows us to use the Platina YAML templates.
resources:
  repositories:
  - repository: scripts
    type: github
    name: delawarePro/platina-scripts
    ref: 'refs/tags/2.5.0'
    # Name of an Azure DevOps Service Connection that is connected with GitHub.
    endpoint: 'GitHub'

variables:

# Detect Platina conventions based on branch naming.
# This will set 'noTests', 'prerelease' and 'withRelease' variables.
- template: azure/pipelines/dotnet/ci.branch.conventions.vars.yml@scripts
  parameters:
    noTests: ${{ parameters.NoTests }}
    prerelease: ${{ parameters.Prerelease }}

steps:
- template: azure/pipelines/dotnet/ci.yml@scripts
  parameters:
    # Solution to build.
    solution: 'Dlw.MyApp.sln'

    # Test arguments needed to support MSTest.Sdk (for now).
    testArguments: -p:TestingPlatformCommandLineArguments="--report-trx --results-directory $(Agent.TempDirectory) --coverage"
    testCoverageArguments: ''

    # Run 'dotnet pack' to generate nugets. When 'withRelease' is true, nugets will pushed to the internal (or external) feed.
    withPack: true

    # Name of the Azure DevOps Artifacts feed to push nugets to.
    #internalFeedName: ''
    # Name of service connection that can be used to push nugets to an external feed.
    externalFeedCredentials: 'PlatinaArtifacts'

    # Whether to publish the release artifacts (including containers, nugets, ...).
    withRelease: ${{ eq(variables.withRelease, true) }}
```

#### CI build with docker container generation

// TODO

## dotnet CD pipeline

// TODO

## Javascript CI pipeline

// TODO
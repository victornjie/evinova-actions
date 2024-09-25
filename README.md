# Evinova GitHub Actions Workflows

This repository contains re-usable workflows used by the Evinova GitHub projects.

# How to integrate
The first step is to setup your project to be invoked by the build using Yarn/package.json.
The build & release process depends on two primary commands being available in the consuming repository:

```bash
yarn install --frozen-lockfile
```

and:

```bash
yarn package
```

As long as the consuming repository can run the above two commands to perform all the build, test, verification, it can be setup to use these reusable evinova-actions. The application code language under build can be any language, as long as the above two commands are implemented from the top level root repository. These reusable build actions have so far been used for Typescript, Python and Java projects successfully.

To make it easier to setup a new repository, a template GitHub project has been setup with the minimum setup. Clone from this template repository or copy the contents to your existing repository: [EPM Project Template](https://github.com/evinova/epm-project-template)

## Usage
Every repository wishing to use these workflows should create the following 4 files in their `./github/workflows` folder:

- lint-pr-title.yaml - contains the workflow definition for enforcing PR title matches Conventional Commits specification
- build-app.yaml - contains the workflow definition for any PR or push to main build process
- build-release.yaml - contains the workflow definition that will build, tag and perform a release version commit back to the main branch.
- deploy-app.yaml - contains the workflow definition for defining the deployment steps given a tag and environment name

Refer to the sections for each below for examples of each setup.


###  lint-pr-title.yaml
Example content: 
```yaml
# GitHub actions workflow triggered on a pull request's 
# branch everytime the PR is opened or updated.
#
# Verifies the PR's title conforms to the Conventional Commits
# specification: https://www.conventionalcommits.org/
name: lint-pr-title
run-name: 'lint-pr-title: ${{ github.ref }}'

concurrency:
  # isolate to single PR/branch
  group: $${ github.repository }}/${{ github.workflow }}/${{ github.ref }}
  # cancel any in-progress runs for the same PRs
  cancel-in-progress: true

on:
  # run on any PRs on any branch
  pull_request:
    types: [ opened, reopened, edited, synchronize ]

jobs:
  lint-pr-title:
    uses: evinova/evinova-actions/.github/workflows/reusable-lint-pr-title.yaml@v1.0.0
    secrets: inherit
```


###  build-app.yaml
Example content: 
```yaml
name: build-app
run-name: 'build-app: ${{ github.ref }}'

concurrency:
  # isolate to single PR/branch
  # github.ref is in the format:
  #   - refs/heads/<branch_name> : for branches
  #   - refs/pull/<pr_number>/merge : for pull requests
  #   - refs/tags/<tag_name> : , for tags 
  group: $${ github.repository }}/${{ github.workflow }}/${{ github.ref }}
  # cancel any in progress runs on for the same 
  cancel-in-progress: true

on:
  workflow_dispatch:

  # run on all pull-requests on all branches
  pull_request:
    branches:
    - '**'

  # run on pushes to any of these branches
  push:
    branches:
    - main
    - master
    - develop
    - release-*
    - hotfix-*

jobs:
  build-app:
    uses: evinova/evinova-actions/.github/workflows/reusable-build-app.yaml@v1.0.0
    secrets: inherit
```

###  build-release.yaml
Example content: 
```yaml
# GitHub actions workflow triggered manually from GitHub Actions website.
# This workflow validates, tests, packages and tags a services type project
# using a either an auto-increment or forced, patch, minor or 
# major semantic version update.
name: build-release
run-name: 'build-release: ${{ inputs.version_type }}'

concurrency:
  group: $${ github.repository }}/${{ github.workflow }}/${{ github.ref }}
  cancel-in-progress: true

on:
  workflow_dispatch:
    inputs:
      version_type:
        description: 'Release Type to Create (auto, patch, minor, major)'
        required: true
        default: 'auto'
        type: choice
        options:
        - auto
        - patch
        - minor
        - major
      # deploy_to_dev:
      #   description: 'Deploy to DEV on completion?'
      #   required: true
      #   default: true
      #   type: boolean

jobs:
  release:
    uses: evinova/evinova-actions/.github/workflows/reusable-build-release.yaml@v1.0.0
    with: 
      version_type: ${{ inputs.version_type }}
      deploy_to_dev: false 
    secrets: inherit
```


###  deploy-app.yaml
Example content: 
```yaml
# GitHub actions workflow triggered manually from GitHub Actions website.
name: deploy-app
run-name: 'deploy-app: ${{ inputs.environment }} ${{ inputs.environment_subdomain }} ${{ inputs.version_tag }}'

concurrency:
  group: $${ github.repository }}/${{ github.workflow }}/${{ github.ref }}/${{ inputs.environment }}-${{ inputs.environment_subdomain }}
  cancel-in-progress: true

on:
  workflow_dispatch:
    inputs:
      version_tag:
        description: 'Version Tag to Deploy'
        required: true
        type: string
      environment:
        description: 'Target Environment'
        required: true
        default: 'dev'
        type: choice
        options:
        - dev
        - sit
        - ppt
        - prod
      environment_subdomain:
        description: 'Optional Environment Deployment URL Subdomain'
        required: false
        type: string
        default: ''
  workflow_call:
    inputs:
      version_tag:
        description: 'Version Tag to Deploy'
        required: true
        type: string
      environment:
        description: 'Target Environment'
        required: true
        type: string
      environment_subdomain:
        description: 'Optional Environment Deployment URL Subdomain'
        required: false
        type: string
        default: ''

jobs:
  deploy_app:
    uses: evinova/evinova-actions/.github/workflows/reusable-deploy-app-service.yaml@v1.0.0
    with: 
      version_tag: ${{ inputs.version_tag }}
      environment: ${{ inputs.environment }}
    secrets: inherit
```

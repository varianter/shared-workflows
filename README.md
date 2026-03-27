# Variant Workflows

Collection of reusable workflows used for deploying on Variant AKS.

## Required secrets

The following secrets must be configured in your repository (or inherited from the organization):

| Secret                                   | Required                     | Description                                                        |
| ---------------------------------------- | ---------------------------- | ------------------------------------------------------------------ |
| `VARIANT_ACR_CONTAINER_PUSH_USERNAME`    | Always                       | Username with push access to the Azure Container Registry          |
| `VARIANT_ACR_CONTAINER_PUSH_SECRET`      | Always                       | Password or token with push access to the Azure Container Registry |
| `VARIANT_GITHUB_REPO_UPDATER_APP_ID`     | When `enable-gitops: 'true'` | GitHub App ID used to update the GitOps repository                 |
| `VARIANT_GITHUB_REPO_UPDATER_PRIVATEKEY` | When `enable-gitops: 'true'` | GitHub App private key used to update the GitOps repository        |

Using `secrets: inherit` in the consumer workflow is the recommended way to pass these through.

## Build and Deploy

Build dockerfile, push to ACR and deploying. Will update infra repo with new image tag in true gitops sense.

### Trigger on release

```
name: Build and Deploy

on:
  release:
    types:
      - published

jobs:
  use-shared-workflow:
    uses: varianter/shared-workflows/.github/workflows/build-and-deploy-shared.yaml@main
    secrets: inherit
    with:
      environment: prod
      acr-url: variantplatformacr.azurecr.io
      image-name: cool-project
      docker-context: .
      dockerfile: ./Dockerfile
      gitops-repository: infrastructure-as-code
      gitops-directory-path: deployments/prod/workloads/cool-project
      gitops-workload-name: dash
      gitops-file-to-update: values.yaml
      gitops-yq-path: .image.tag
      gitops-continue-on-error: 'true'
      enable-gitops: 'true'

```

### Manual trigger

Supporting several envs and configuration.

```
name: Build and Deploy

on:
  push:
    branches:
      - feature/dockerize
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment to deploy'
        required: true
        default: dev
        type: choice
        options:
          - lab
          - dev
          - prod
jobs:
  use-shared-workflow:
    uses: varianter/shared-workflows/.github/workflows/build-and-deploy-shared.yaml@main
    secrets: inherit
    with:
      environment: ${{ inputs.environment || 'dev' }}
      acr-url: variantplatformacr.azurecr.io
      image-name: cool-project
      docker-context: .
      dockerfile: ./Dockerfile
      build-args: |
        NEXT_PUBLIC_API_BASE="https://myapi.api/api/v1"
      gitops-repository: infrastructure-as-code
      gitops-directory-path: deployments/${{ inputs.environment || 'dev' }}/workloads/cool-project
      gitops-workload-name: dash
      gitops-file-to-update: values.yaml
      gitops-yq-path: .image.tag
      gitops-continue-on-error: 'true'
      enable-gitops: 'true'
```

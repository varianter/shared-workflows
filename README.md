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

## Check Argo CD (after deploying)

Waits for the app to roll out in Argo CD, read-only and **without any secrets**: the job authenticates with its GitHub
OIDC token, which Entra exchanges for a read-only Argo CD token. It works from any `varianter` repo, public or private.

Run it after `build-and-deploy-shared` and pass the GitOps commit that deploy pushed, so the check cannot pass on the
previous, still-healthy deploy:

```
jobs:
  build-and-deploy:
    uses: varianter/shared-workflows/.github/workflows/build-and-deploy-shared.yaml@main
    secrets: inherit
    with:
      # ...as above...

  verify:
    needs: build-and-deploy
    permissions:
      id-token: write # required: the reusable workflow cannot grant itself this
    uses: varianter/shared-workflows/.github/workflows/check-argocd-shared.yaml@main
    with:
      app: cool-project-dev # Argo CD Application name
      expected-revision: ${{ needs.build-and-deploy.outputs.gitops-commit-sha }}
```

| Input               | Required | Description                                                                                                                                                                  |
| ------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `app`               | Yes      | Argo CD Application name, e.g. `tid-dev`                                                                                                                                     |
| `expected-revision` | No       | GitOps commit that must be deployed. Passes once Argo CD has successfully synced this commit **or a later one**. Empty skips the check (only health is verified, with a warning) |
| `timeout-seconds`   | No       | Default `600`                                                                                                                                                                |

Notes:

- Must be referenced as `@main`. Access is granted to this exact file on `main` (federated credential in
  `varianter/infrastructure-as-code`, `environments/prod/cluster/argocd-ci-github.tf`). Renaming it requires updating that first.
- `gitops-commit-sha` is empty when nothing changed or the GitOps push failed. Then there is nothing to wait for; guard the job
  with `if: needs.build-and-deploy.outputs.gitops-commit-sha != ''` if you want it skipped (or to fail) in that case.
- "Later commit" is decided by commit time as reported by Argo CD (the GitOps repo is private), so it assumes a linear history.
- The identity is read-only but shared: any `varianter` repo can read any Argo CD app. The workflow only prints health/sync
  summaries, never raw application data.

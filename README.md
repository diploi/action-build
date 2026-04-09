# Diploi Builder Action

A GitHub Action that builds and pushes Docker images for components in a [Diploi](https://diploi.com/) project.

## Overview

Diploi projects follow a monorepo structure where each component (e.g. a Next.js frontend, a Node.js API) lives in its own folder. This action handles the Docker build and push for a single component, and is intended to be used as part of an automatically generated CI/CD workflow driven by a matrix of components.

## Usage

This action is used as part of an auto-generated `Build.yaml` workflow in your Diploi project repository. The workflow is structured in two jobs:

1. **Define components** — reads your `diploi.yaml` and outputs a build matrix
2. **Build components** — runs a parallel build for each component in the matrix

```yaml
name: Build Components

on:
  push:
    branches:
      - '*'

jobs:
  define-components:
    name: Define Components
    runs-on: ubuntu-latest
    outputs:
      components: ${{ steps.diploi-meta.outputs.components }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - id: diploi-meta
        name: Diploi meta
        uses: diploi/action-components@v1.8

  run-builds:
    name: Build ${{ matrix.name }} ${{ matrix.stage }}
    runs-on: ubuntu-24.04-arm
    needs: define-components
    strategy:
      fail-fast: false
      matrix:
        include: ${{ fromJSON(needs.define-components.outputs.components) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      - name: Diploi build
        uses: diploi/action-build@v1.11
        with: ${{ matrix }}
        env:
          project: ${{ secrets.DIPLOI_REGISTRY_PROJECT }}
          registry: ${{ secrets.DIPLOI_REGISTRY_HOSTNAME }}
          username: ${{ secrets.DIPLOI_REGISTRY_USERNAME }}
          password: ${{ secrets.DIPLOI_REGISTRY_PASSWORD }}
```

The four `DIPLOI_REGISTRY_*` secrets are provisioned automatically by Diploi when you connect your GitHub repository.

The build matrix is derived from your `diploi.yaml`. Each component in the matrix corresponds to a folder in your repository, and the action builds either `Dockerfile` (production) or `Dockerfile.dev` (development) depending on the build type.

## Inputs

These are passed automatically via `with: ${{ matrix }}` and do not need to be set manually.

| Input        | Required | Description                                                                          |
| ------------ | -------- | ------------------------------------------------------------------------------------ |
| `folder`     | Yes      | The folder containing the component and its Dockerfile                               |
| `buildArgs`  | Yes      | Static `KEY=VALUE` pairs passed to the Docker build as `ARG`s                        |
| `registry`   | Yes      | Hostname of the Diploi container registry                                            |
| `username`   | Yes      | Registry username                                                                    |
| `password`   | Yes      | Registry password                                                                    |
| `identifier` | No       | Component identifier, used to construct the image name                               |
| `project`    | No       | Project name in the Diploi registry                                                  |
| `name`       | No       | Human-readable name for the build                                                    |
| `type`       | No       | Build type (`main`, `dev`, `main-dev`) — controls which Dockerfile and tags are used |

## Passing GitHub variables into the build

If your Dockerfile relies on build-time configuration (e.g. API URLs, feature flags), you can pass [static ENV values](https://docs.diploi.com/reference/diploi-yaml#static-values) via the `env` field in your `diploi.yaml`. These are forwarded to Docker as `ARG` values.

Add the variables to the `env` of the relevant component in your `diploi.yaml`:

```yaml
components:
  - name: Next.js
    identifier: next
    package: https://github.com/diploi/component-nextjs#v16.1.2
    env:
      include:
        - value: test
          name: TEST
```

Then declare the corresponding `ARG` instructions in your `Dockerfile`:

```dockerfile
ARG TEST
```

And expose them as ENV values if required:

```dockerfile
ENV TEST_VALUE=$TEST
```

## Passing GitHub secrets into the build

For sensitive values such as private registry tokens or API keys needed at build time, use the `secrets` environment variable in your workflow. These are passed as [Docker build secrets](https://docs.docker.com/build/building/secrets/), which are mounted only during the build and never stored in the image layers.

Add a `secrets` entry to the `env:` block in your workflow:

```yaml
- name: Diploi build
  uses: diploi/action-build@v1.11
  with: ${{ matrix }}
  env:
    project: ${{ secrets.DIPLOI_REGISTRY_PROJECT }}
    registry: ${{ secrets.DIPLOI_REGISTRY_HOSTNAME }}
    username: ${{ secrets.DIPLOI_REGISTRY_USERNAME }}
    password: ${{ secrets.DIPLOI_REGISTRY_PASSWORD }}
    secrets: |
      NPM_TOKEN=${{ secrets.NPM_TOKEN }}
      GITHUB_TOKEN=${{ secrets.MY_GITHUB_TOKEN }}
```

Then consume the secrets in your `Dockerfile` using `--mount=type=secret`:

```dockerfile
RUN --mount=type=secret,id=NPM_TOKEN \
    NPM_TOKEN=$(cat /run/secrets/NPM_TOKEN) npm install
```

Unlike static ENVs, Docker build secrets are not baked into the image and will not appear in the image history.

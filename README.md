> **Warning**
> This action is archived in favor of using:
> 1. the new [elao.app.docker recipe](https://manala.github.io/manala-recipes/recipes/elao.app.docker/)
> 1. the embedded recipe Github actions or [integration tasks config](https://manala.github.io/manala-recipes/recipes/elao.app.docker/#github-actions)

# Manala CI Action

This action provisions a Manala CI environment generated from the ["elao.app" recipe](https://manala.github.io/manala-recipes/recipes/elao.app/)
and allows to execute CLI commands inside.

Use-cases:

- Execute your tests suites
- Create releases for your application
- Deploy your application

using Github Actions workflows.

## Usage

### Pre-requisites

Create a workflow `.yaml` file in your repositories `.github/workflows` directory.  
An [example workflow](#example-workflow) is available below.  
For more information, reference the GitHub Help Documentation for [Creating a workflow file](https://help.github.com/en/articles/configuring-a-workflow#creating-a-workflow-file).

This action relies on the Manala ["elao.app" recipe](https://manala.github.io/manala-recipes/recipes/elao.app/) recipe,
which generates a `Dockerfile` for your CI environment and `docker-compose.yaml` file for your services.

It will build and start Manala CI container and related services,
allowing you to execute CLI commands in a full-fledged environment according to your project needs.

### Inputs

| arg | description | mandatory | default | example |
| :---| :--- | :---: | :---: | --- |
| `cmd` | The CLI command to execute | no | `echo ...` | `make test@integration` | 
| `working-directory` | The working dir for the command, relative to your workspace dir | no | current workspace dir | `'api'` | 
| `docker-compose-file` | The docker-compose.yaml file used to start services | no | `".manala/github/docker-compose.yaml"` | | 

### Basic usage:

```yaml
steps:
  # [...]

  - name: 'Run tests'
    uses: Elao/manala-ci-action@main
    with:
      cmd: make test@integration
```

The first time the action is called, it builds the container and start services 
before calling your command, which may take some minutes.  
Subsequent calls to the action will execute the command inside directly.

We'd recommend splitting the first call and subsequent ones, so the first one is
clearly identified as the Manala CI setup:

```yaml
steps:
  # [...]

  - name: 'Setup container with Manala'
    uses: Elao/manala-ci-action@main

  # [...]

  - name: 'Run tests'
    uses: Elao/manala-ci-action@main
    with:
      cmd: make test@integration
```

### Cache

The container is built each time the action is called for the first time in a job.  
In order to speed up this part, you can use [satackey/action-docker-layer-caching](https://github.com/satackey/action-docker-layer-caching):

```yaml
steps:

  # [...]

  - name: 'Cache Docker containers layers'
    uses: satackey/action-docker-layer-caching@v0.0.10
    with:
      key: docker-cache-{hash}
      restore-keys: docker-cache-

  - name: 'Setup container with Manala'
    uses: Elao/manala-ci-action@main
```

### Example workflow

A concrete example for a Symfony app with:

- **cache** for the Docker container built, allowing to speedup the workflow execution.  
  Cache samples for Composer, simple-phpunit and Yarn/NPM are also included.  
  The `${{ secrets.CACHE_VERSION }}` can be configured in your project secrets, 
  allowing to invalidate cache easily on need.
- **timeout** samples to limit the workflow execution. Adapt to your needs.
- **skipped workflow** in case of a Draft PR or a PR labelled with "WIP".
- **autocancel** to automatically cancel a previous workflow
  (e.g: when pushing new commits or amending commits to a PR).

```yaml
name: Tests

on:
  push:
    branches:
      - master
    paths:
      - .github/**
      - .manala/**
      - src/**
      # Complete according to your project / workflow needs
  pull_request:
    types: [ opened, synchronize, reopened, ready_for_review ]
    paths:
      - .github/**
      - .manala/**
      - src/**
      # Complete according to your project / workflow needs

jobs:

  tests:
    name: 'Tests'
    runs-on: ubuntu-latest
    timeout-minutes: 15
    # Do not run on WIP or Draft PRs
    if: "!github.event.pull_request || (!contains(github.event.pull_request.labels.*.name, 'WIP') && github.event.pull_request.draft == false)"

    steps:

      - name: 'Cancel Previous Runs'
        uses: styfle/cancel-workflow-action@0.5.0
        with:
          access_token: ${{ github.token }}

      - name: 'Checkout'
        uses: actions/checkout@v2

      - name: 'Cache composer dependencies'
        uses: actions/cache@v2
        with:
          path: .manala/.cache/docker/composer
          key: composer-${{ secrets.CACHE_VERSION }}-${{ hashFiles('composer.lock') }}

      - name: 'Cache phpunit install dir'
        uses: actions/cache@v2
        with:
          path: bin/.phpunit
          key: phpunit-${{ secrets.CACHE_VERSION }}-${{ hashFiles('phpunit.xml.dist') }}

      - name: 'Cache yarn/npm dependencies'
        uses: actions/cache@v2
        with:
          path: |
            .manala/.cache/docker/yarn
            .manala/.cache/docker/npm
          key: yarn-${{ secrets.CACHE_VERSION }}-${{ hashFiles('yarn.lock') }}

      - name: 'Cache Docker containers layers (including Manala CI)'
        uses: satackey/action-docker-layer-caching@v0.0.11
        with:
          key: docker-cache-${{ secrets.CACHE_VERSION }}-{hash}
          restore-keys: docker-cache-${{ secrets.CACHE_VERSION }}
          
      - name: 'Setup container with Manala'
        uses: Elao/manala-ci-action@main
        timeout-minutes: 6

      - name: 'Install dependencies & setup project'
        uses: Elao/manala-ci-action@main
        timeout-minutes: 3
        with:
          cmd: |
            echo "::group::Install project"
              make install@integration
            echo "::endgroup::"

            echo "::group::Install phpunit"
              bin/phpunit install
            echo "::endgroup::"

      - name: 'Run tests'
        uses: Elao/manala-ci-action@main
        timeout-minutes: 5
        with:
          cmd: make test.phpunit@integration
```

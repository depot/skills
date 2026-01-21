# CI/CD Integration Guide

Complete guides for integrating Depot with all major CI/CD platforms.

## Table of Contents
- [Authentication Methods](#authentication-methods)
- [GitHub Actions](#github-actions)
- [CircleCI](#circleci)
- [GitLab CI](#gitlab-ci)
- [Buildkite](#buildkite)
- [AWS CodeBuild](#aws-codebuild)
- [Google Cloud Build](#google-cloud-build)
- [Jenkins](#jenkins)
- [Bitbucket Pipelines](#bitbucket-pipelines)
- [Travis CI](#travis-ci)
- [Docker Bake in CI](#docker-bake-in-ci)

---

## Authentication Methods

### OIDC Trust Relationships (Recommended)
No static secrets. CI provider exchanges its token for a short-lived Depot token.

**Supported providers**: GitHub Actions, CircleCI, Buildkite

**Setup**:
1. Go to Project Settings in Depot dashboard
2. Add OIDC trust relationship
3. Specify provider and repository details

### Project Tokens
Scoped to a specific project. Ideal for CI when OIDC isn't available.

**Setup**:
1. Go to Project Settings > Tokens
2. Create new token
3. Store as `DEPOT_TOKEN` secret in CI

### User Tokens
Tied to your account. Works across all accessible projects.

**Setup**:
1. Go to Account Settings > API Tokens
2. Create new token
3. Store as `DEPOT_TOKEN` secret

---

## GitHub Actions

### Using depot/build-push-action (Recommended)

```yaml
name: Build
on: push

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # Required for OIDC
    steps:
      - uses: actions/checkout@v4

      - uses: depot/build-push-action@v1
        with:
          project: <your-project-id>
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

### Using depot CLI directly

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: depot/setup-action@v1

      - run: depot build -t myapp:${{ github.sha }} . --push
        env:
          DEPOT_PROJECT_ID: <your-project-id>
```

### With Project Token (no OIDC)

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: depot/setup-action@v1

      - run: depot build -t myapp:${{ github.sha }} . --push
        env:
          DEPOT_TOKEN: ${{ secrets.DEPOT_TOKEN }}
          DEPOT_PROJECT_ID: <your-project-id>
```

### Multi-Platform Build

```yaml
- uses: depot/build-push-action@v1
  with:
    project: <your-project-id>
    platforms: linux/amd64,linux/arm64
    push: true
    tags: myapp:latest
```

### Build, Test, Then Push

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: depot/setup-action@v1

      # Build and save to Depot Registry
      - run: depot build -t myapp:${{ github.sha }} . --save --metadata-file=build.json
        env:
          DEPOT_PROJECT_ID: <your-project-id>

      # Pull for testing
      - run: |
          BUILD_ID=$(jq -r '.["depot.build"]' build.json)
          depot pull -t myapp:test $BUILD_ID
        env:
          DEPOT_PROJECT_ID: <your-project-id>

      # Run tests
      - run: docker run myapp:test npm test

      # Push to production registry
      - run: |
          BUILD_ID=$(jq -r '.["depot.build"]' build.json)
          depot push -t ghcr.io/org/myapp:${{ github.sha }} $BUILD_ID
        env:
          DEPOT_PROJECT_ID: <your-project-id>
```

### Push to Multiple Registries

```yaml
- uses: depot/build-push-action@v1
  with:
    project: <your-project-id>
    push: true
    tags: |
      ghcr.io/${{ github.repository }}:${{ github.sha }}
      docker.io/myorg/myapp:${{ github.sha }}
      ${{ secrets.AWS_ACCOUNT }}.dkr.ecr.us-east-1.amazonaws.com/myapp:${{ github.sha }}
```

### Generate SBOM

```yaml
- uses: depot/build-push-action@v1
  with:
    project: <your-project-id>
    push: true
    tags: myapp:latest
    sbom: true
    sbom-dir: ./sbom-output
```

---

## CircleCI

### With OIDC (Recommended)

```yaml
version: 2.1

jobs:
  build:
    machine:
      image: ubuntu-2204:current
    steps:
      - checkout
      - run:
          name: Install Depot CLI
          command: curl -L https://depot.dev/install-cli.sh | sudo env DEPOT_INSTALL_DIR=/usr/local/bin sh
      - run:
          name: Build and push
          command: depot build -t myapp:${CIRCLE_SHA1} . --push
          environment:
            DEPOT_PROJECT_ID: <your-project-id>

workflows:
  build:
    jobs:
      - build:
          context:
            - depot-oidc  # Context with OIDC trust configured
```

### With Project Token

```yaml
version: 2.1

jobs:
  build:
    machine:
      image: ubuntu-2204:current
    steps:
      - checkout
      - run:
          name: Install Depot CLI
          command: curl -L https://depot.dev/install-cli.sh | sudo env DEPOT_INSTALL_DIR=/usr/local/bin sh
      - run:
          name: Build and push
          command: depot build -t myapp:${CIRCLE_SHA1} . --push
          environment:
            DEPOT_TOKEN: ${DEPOT_TOKEN}
            DEPOT_PROJECT_ID: <your-project-id>
```

### Docker Executor (requires setup_remote_docker)

```yaml
jobs:
  build:
    docker:
      - image: cimg/base:current
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.24
      - run:
          name: Install Depot CLI
          command: curl -L https://depot.dev/install-cli.sh | sudo env DEPOT_INSTALL_DIR=/usr/local/bin sh
      - run:
          name: Build
          command: depot build -t myapp:${CIRCLE_SHA1} . --push
```

---

## GitLab CI

### Basic Setup

```yaml
stages:
  - build

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DEPOT_TOKEN: ${DEPOT_TOKEN}
    DEPOT_PROJECT_ID: <your-project-id>
  before_script:
    - apk add --no-cache curl
    - curl -L https://depot.dev/install-cli.sh | sh
    - export PATH="$HOME/.depot/bin:$PATH"
  script:
    - depot build -t myapp:${CI_COMMIT_SHA} . --push
```

### Push to GitLab Container Registry

```yaml
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DEPOT_TOKEN: ${DEPOT_TOKEN}
    DEPOT_PROJECT_ID: <your-project-id>
  before_script:
    - apk add --no-cache curl
    - curl -L https://depot.dev/install-cli.sh | sh
    - export PATH="$HOME/.depot/bin:$PATH"
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - depot build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA . --push
```

### Multi-Platform

```yaml
build:
  script:
    - depot build --platform linux/amd64,linux/arm64 -t myapp:${CI_COMMIT_SHA} . --push
```

---

## Buildkite

### With OIDC

```yaml
steps:
  - label: ":docker: Build"
    command:
      - curl -L https://depot.dev/install-cli.sh | sh
      - export PATH="$HOME/.depot/bin:$PATH"
      - depot build -t myapp:$BUILDKITE_COMMIT . --push
    env:
      DEPOT_PROJECT_ID: <your-project-id>
    plugins:
      - docker-login#v2.1.0:
          username: myuser
          password-env: DOCKER_PASSWORD
```

### With Project Token

```yaml
steps:
  - label: ":docker: Build"
    command:
      - curl -L https://depot.dev/install-cli.sh | sh
      - export PATH="$HOME/.depot/bin:$PATH"
      - depot build -t myapp:$BUILDKITE_COMMIT . --push
    env:
      DEPOT_TOKEN: ${DEPOT_TOKEN}
      DEPOT_PROJECT_ID: <your-project-id>
```

---

## AWS CodeBuild

```yaml
version: 0.2

env:
  variables:
    DEPOT_PROJECT_ID: <your-project-id>
  secrets-manager:
    DEPOT_TOKEN: depot:token

phases:
  install:
    commands:
      - curl -L https://depot.dev/install-cli.sh | sh
      - export PATH="$HOME/.depot/bin:$PATH"

  pre_build:
    commands:
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com

  build:
    commands:
      - depot build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/myapp:$CODEBUILD_RESOLVED_SOURCE_VERSION . --push
```

---

## Google Cloud Build

```yaml
steps:
  - name: 'gcr.io/cloud-builders/curl'
    entrypoint: 'bash'
    args:
      - '-c'
      - 'curl -L https://depot.dev/install-cli.sh | sh'

  - name: 'gcr.io/cloud-builders/docker'
    entrypoint: 'bash'
    args:
      - '-c'
      - |
        export PATH="$HOME/.depot/bin:$PATH"
        depot build -t gcr.io/$PROJECT_ID/myapp:$COMMIT_SHA . --push
    env:
      - 'DEPOT_TOKEN=${_DEPOT_TOKEN}'
      - 'DEPOT_PROJECT_ID=<your-project-id>'

substitutions:
  _DEPOT_TOKEN: ''  # Set via --substitutions or Secret Manager
```

---

## Jenkins

### Pipeline

```groovy
pipeline {
    agent any

    environment {
        DEPOT_TOKEN = credentials('depot-token')
        DEPOT_PROJECT_ID = '<your-project-id>'
    }

    stages {
        stage('Install Depot') {
            steps {
                sh 'curl -L https://depot.dev/install-cli.sh | sh'
            }
        }

        stage('Build') {
            steps {
                sh '''
                    export PATH="$HOME/.depot/bin:$PATH"
                    depot build -t myapp:${GIT_COMMIT} . --push
                '''
            }
        }
    }
}
```

---

## Bitbucket Pipelines

```yaml
image: atlassian/default-image:4

pipelines:
  default:
    - step:
        name: Build and Push
        script:
          - curl -L https://depot.dev/install-cli.sh | sh
          - export PATH="$HOME/.depot/bin:$PATH"
          - depot build -t myapp:$BITBUCKET_COMMIT . --push
        services:
          - docker
        caches:
          - docker

definitions:
  services:
    docker:
      memory: 3072
```

Set `DEPOT_TOKEN` and `DEPOT_PROJECT_ID` as repository variables.

---

## Travis CI

```yaml
language: generic

services:
  - docker

env:
  global:
    - DEPOT_PROJECT_ID=<your-project-id>

before_install:
  - curl -L https://depot.dev/install-cli.sh | sh
  - export PATH="$HOME/.depot/bin:$PATH"

script:
  - depot build -t myapp:$TRAVIS_COMMIT . --push
```

Set `DEPOT_TOKEN` as an encrypted environment variable.

---

## Docker Bake in CI

### GitHub Actions with depot/bake-action

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - uses: depot/bake-action@v1
        with:
          project: <your-project-id>
          files: docker-bake.hcl
          push: true
```

### Build, Test, Push Pattern

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      build-id: ${{ steps.build.outputs.build-id }}
    steps:
      - uses: actions/checkout@v4
      - uses: depot/setup-action@v1

      - id: build
        run: |
          depot bake -f docker-bake.hcl --save --metadata-file=bake.json
          echo "build-id=$(jq -r '.["depot.build"]' bake.json)" >> $GITHUB_OUTPUT
        env:
          DEPOT_PROJECT_ID: <your-project-id>

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: depot/setup-action@v1
      - uses: depot/pull-action@v1
        with:
          project: <your-project-id>
          build-id: ${{ needs.build.outputs.build-id }}
          target: app
          tag: myapp:test

      - run: docker run myapp:test npm test

  push:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - uses: depot/setup-action@v1
      - run: |
          depot push --project $DEPOT_PROJECT_ID -t ghcr.io/org/myapp:latest ${{ needs.build.outputs.build-id }}
        env:
          DEPOT_PROJECT_ID: <your-project-id>
```

---

## Best Practices

1. **Use OIDC when available** - No secrets to manage or rotate
2. **Share cache between CI and local** - Same project ID enables shared caching
3. **Use `--save` for test workflows** - Build once, test, then push
4. **Multi-platform in CI** - Native ARM builds without emulation overhead
5. **Leverage `depot bake`** - Parallel multi-image builds with deduplication

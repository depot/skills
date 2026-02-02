---
name: setup
description: Install Depot CLI, authenticate, and initialize projects
metadata:
  tags: setup, install, login, auth, oidc, token
---

## Install the CLI

**macOS:**
```bash
brew install depot/tap/depot
```

**Linux:**
```bash
curl -L https://depot.dev/install-cli.sh | sh
```

**Windows (PowerShell):**
```powershell
iwr https://depot.dev/install-cli.ps1 -useb | iex
```

## Authenticate Locally

```bash
depot login
```

Opens a browser to authenticate with your Depot account. Stores credentials locally for future use.

**Verify authentication:**
```bash
depot projects list
```

If you see your projects, authentication is working.

## Initialize a Project

```bash
depot init
```

Creates `depot.json` in the current directory, linking it to a Depot project. Commit this file to your repository.

**Verify setup:**
```bash
depot build -t test:latest . --load
```

Look for `[depot]` prefixed output confirming Depot builders are in use.

## CI/CD Authentication

### OIDC (Recommended)

Use OIDC trust relationships for providers that support it. No static secrets to rotate.

| Provider | OIDC Support | Setup |
|----------|--------------|-------|
| GitHub Actions | ✅ Yes | Add trust relationship in Depot, use `id-token: write` permission |
| CircleCI | ✅ Yes | Add trust relationship in Depot project settings |
| Buildkite | ✅ Yes | Add trust relationship in Depot project settings |
| GitLab CI | ❌ No | Use project token |
| Jenkins | ❌ No | Use project token |
| AWS CodeBuild | ❌ No | Use project token |
| Bitbucket Pipelines | ❌ No | Use project token |
| Travis CI | ❌ No | Use project token |

### GitHub Actions with OIDC

```yaml
jobs:
  build:
    permissions:
      contents: read
      id-token: write  # Required for OIDC
    steps:
      - uses: actions/checkout@v4
      - uses: depot/build-push-action@v1
        with:
          context: .
          tags: myapp:latest
```

### CircleCI with OIDC

```yaml
jobs:
  build:
    docker:
      - image: cimg/base:current
    steps:
      - checkout
      - run:
          name: Install Depot
          command: curl -L https://depot.dev/install-cli.sh | sh
      - run:
          name: Build
          command: depot build -t myapp:latest .
```

### Project Tokens (For Providers Without OIDC)

1. Go to Depot project settings → Tokens
2. Create a new project token
3. Add as `DEPOT_TOKEN` secret in your CI provider

```yaml
# GitLab CI example
build:
  image: alpine
  before_script:
    - curl -L https://depot.dev/install-cli.sh | sh
  script:
    - depot build -t myapp:latest .
  variables:
    DEPOT_TOKEN: $DEPOT_TOKEN  # From CI/CD variables
```

```groovy
// Jenkins example
pipeline {
    environment {
        DEPOT_TOKEN = credentials('depot-token')
    }
    stages {
        stage('Build') {
            steps {
                sh 'curl -L https://depot.dev/install-cli.sh | sh'
                sh 'depot build -t myapp:latest .'
            }
        }
    }
}
```

## Troubleshooting Setup

| Issue | Solution |
|-------|----------|
| `depot: command not found` | Ensure install location is in PATH, or run with full path |
| `401 Unauthorized` | Run `depot login` again, or check `DEPOT_TOKEN` is set correctly |
| `No project found` | Run `depot init` or set `DEPOT_PROJECT_ID` environment variable |
| OIDC fails in CI | Verify trust relationship matches repo/org, check required permissions |

# Depot CLI Reference

## Installation

**macOS (Homebrew):**
```bash
brew install depot/tap/depot
```

**Linux:**
```bash
curl -L https://depot.dev/install-cli.sh | sh
```

**Specific version:**
```bash
curl -L https://depot.dev/install-cli.sh | sh -s VERSION_NUMBER
```

## Authentication

### Local Development
```bash
depot login
```
Authenticates with your Depot account and stores a user token locally.

### Environment Variable
```bash
export DEPOT_TOKEN=your_token_here
```

### CI/CD - Project Tokens
Create project tokens in Project Settings. Set `DEPOT_TOKEN` environment variable in CI.

### CI/CD - OIDC Trust Relationships
For GitHub Actions, CircleCI, Buildkite, and RWX. No static secrets required.
GitHub Actions requires `id-token: write` permission.

## Core Build Commands

### depot build

Drop-in replacement for `docker buildx build`. All `docker buildx build` flags are supported.

```bash
# Basic build
depot build -t repo/image:tag .

# Build and load to local Docker
depot build -t repo/image:tag . --load

# Build and push to registry
depot build -t repo/image:tag . --push

# Multi-platform build
depot build --platform linux/amd64,linux/arm64 -t repo/image:tag . --push

# Build with secrets
depot build --secret id=mysecret,src=secret.txt -t repo/image:tag .

# Build with SSH agent forwarding
depot build --ssh default -t repo/image:tag .

# Build with build args
depot build --build-arg VERSION=1.0.0 -t repo/image:tag .

# Save to Depot Registry (no external registry needed)
depot build -t repo/image:tag . --save
```

**Key flags:**
- `--load` - Download image to local Docker daemon
- `--push` - Push to container registry
- `--save` - Save to Depot Registry for later retrieval
- `--platform` - Target platforms (e.g., `linux/amd64,linux/arm64`)
- `--project` - Specify Depot project ID
- `--token` - Authentication token (overrides DEPOT_TOKEN)
- `--lint` - Lint Dockerfile during build

### depot bake

Builds multiple targets from HCL, JSON, or Docker Compose files.

```bash
# Build from docker-bake.hcl
depot bake -f docker-bake.hcl

# Build from docker-compose.yml
depot bake -f docker-compose.yml

# Build specific targets
depot bake -f docker-bake.hcl target1 target2

# Override variables
depot bake -f docker-bake.hcl --set "*.platform=linux/amd64,linux/arm64"
```

## Depot Registry Commands

### depot pull
Pull image from Depot Registry by build ID.
```bash
depot pull --project <project-id> -t image:tag <build-id>
```

### depot push
Push image from Depot Registry to another registry.
```bash
depot push --project <project-id> -t registry.example.com/image:tag <build-id>
```

### depot pull-token
Generate a 1-hour read-only token for pulling from Depot Registry.
```bash
depot pull-token --project <project-id>
```

## Project Management

### depot init
Initialize a project in current directory, creates `depot.json`.
```bash
depot init
```

### depot projects create
```bash
depot projects create --region us-east-1 "my-project"
```

### depot projects list
List all projects in your organization.

### depot list builds
```bash
depot list builds --project <project-id> --output json
```

## Cache Management

### depot cache reset
Force a new empty cache volume.
```bash
depot cache reset --project <project-id>
```

## Configuration

### depot configure-docker
Install Depot as Docker CLI plugin and set as default builder.
```bash
depot configure-docker
```

Uninstall:
```bash
depot configure-docker --uninstall
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `DEPOT_TOKEN` | Authentication token |
| `DEPOT_PROJECT_ID` | Default project ID |
| `DEPOT_NO_SUMMARY_LINK` | Suppress build summary links |

## Project Resolution Order

1. `--project` flag
2. `DEPOT_PROJECT_ID` environment variable
3. Interactive prompt (if terminal)
4. `depot.json` in current directory

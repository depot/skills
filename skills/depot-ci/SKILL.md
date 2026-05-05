---
name: depot-ci
description: >
  Configures and manages Depot CI, a drop-in replacement for GitHub Actions that runs workflows
  entirely within Depot. Use when migrating GitHub Actions workflows to Depot CI, running
  `depot ci migrate`, managing Depot CI secrets and variables, running workflows with
  `depot ci run`, debugging Depot CI runs with `depot ci run list`, `depot ci status`,
  `depot ci logs`, or `depot ci ssh`, checking
  workflow compatibility, or understanding Depot CI capabilities. Also use when the user
  mentions .depot/ directory, depot ci commands, or asks about running GitHub Actions workflows
  on Depot's infrastructure without GitHub-hosted runners.
---

# Depot CI

Depot CI is a programmable CI system for engineers and agents. Workflows in Depot CI run entirely on Depot compute with built-in job visibility, debuggability, and control. GitHub Actions is the first syntax Depot CI supports: migrate your existing GitHub Actions workflows, and get fast, reliable runs on optimized infrastructure.

## Architecture

Three subsystems: **compute** (provisions and executes work), **orchestrator** (schedules multi-step workflows, handles dependencies), **GitHub Actions parser** (translates Actions YAML into orchestrator workflows). The system is fully programmable.

## Org Context Check for Multi-Org Users

If a user belongs to multiple organizations, before setup/migration or if CI commands can't find expected workflows, verify Depot org context first:

```bash
# Check current org ID
depot org show

# List orgs the user belongs to
depot org list

# Option A: switch default org for this shell/session
depot org switch <org-id>

# Option B: keep current org and target explicitly per command
depot ci run --org <org-id> --workflow .depot/workflows/ci.yml
```

Use `--org <org-id>` when the workflow/repo lives in a different org than the current default.

## Getting Started

### 1. Install the Depot Code Access GitHub App

Depot dashboard → Settings → GitHub Code Access → Connect to GitHub

(If you've used Claude Code on Depot, this may already be installed.)

### 2. Migrate workflows

```bash
depot ci migrate
```

This interactive wizard:

1. Checks that the Depot Code Access app is installed and configured.
1. Discovers all workflows in `.github/workflows/` and analyzes each for Depot CI compatibility.
1. Copies selected workflows to `.depot/workflows/` with inline corrections and comments.
1. Copies local actions from `.github/actions/` to `.depot/actions/`.
1. Detects secrets and variables referenced in workflows and prints next steps for importing them.

Your `.github/` directory is untouched, so workflows run in both GitHub and Depot simultaneously.

**Warning:** Workflows that cause side effects (deploys, artifact updates) will execute twice.

#### Migrate subcommands

The migrate command can also be run as individual steps:

```bash
# Check installation and auth
depot ci migrate preflight

# Copy and transform workflows to .depot/workflows/
depot ci migrate workflows

# Import GitHub Actions secrets and variables into Depot CI
depot ci migrate secrets-and-vars
```

#### Migrate flags

| Flag              | Description                                 |
| ----------------- | ------------------------------------------- |
| `-y, --yes`       | Non-interactive, migrate all workflows      |
| `--overwrite`     | Overwrite existing `.depot/` directory      |
| `--org <id>`      | Organization ID (required if multiple orgs) |
| `--token <token>` | Depot API token                             |

### 3. Import secrets and variables

```bash
depot ci migrate secrets-and-vars
```

This creates and runs a one-shot GitHub Actions workflow on a temporary branch that reads your existing secrets and variables and imports them into Depot CI. The branch is safe to delete afterwards.

You can also add secrets and variables manually with `depot ci secrets add` and `depot ci vars add` (see below).

#### Migrate Secrets-and-Vars flags

| Flag               | Description                                                                        |
| ------------------ | ---------------------------------------------------------------------------------- |
| `-y, --yes`        | Skip preview and confirmation prompts                                              |
| `--branch`         | Override the branch name used for the migration workflow                           |
| `--secrets <name>` | Secret name to include; can be repeated to select multiple. Omit to include all.   |
| `--vars <name>`    | Variable name to include; can be repeated to select multiple. Omit to include all. |
| `--org <id>`       | Organization ID (required if multiple orgs)                                        |
| `--token <token>`  | Depot API token                                                                    |

### 4. Manual setup (without migrate command)

Create `.depot/workflows/` and `.depot/actions/` directories manually. Copy workflow files from `.github/workflows/`. Configure secrets via the CLI.

## Managing Secrets

Secrets can be org-wide or scoped to a specific repository. They can also have **variants**: multiple values for the same name that resolve based on workflow context (repository, branch, workflow file, GitHub environment). The CLI only handles repo scoping; variants with branch, workflow, or environment rules must be created in the dashboard. See the **Secret and Variable Variants** section below for resolution rules.

```bash
# Add (prompts for value securely if --value omitted)
depot ci secrets add SECRET_NAME
depot ci secrets add SECRET_NAME --value "$NPM_TOKEN" --description "NPM auth token"

# Add repo-scoped secret
depot ci secrets add SECRET_NAME --repo owner/repo --value "$NPM_TOKEN"

# List (names and metadata only, no values)
depot ci secrets list
depot ci secrets list --output json
depot ci secrets list --repo owner/repo    # Also show repo-specific secrets

# Remove
depot ci secrets remove SECRET_NAME
depot ci secrets remove SECRET_NAME --force          # Skip confirmation
depot ci secrets remove SECRET_NAME --repo owner/repo  # Remove repo-scoped secret
```

## Credential Safety Guardrails

Treat credentials as sensitive input and never echo them back in outputs.

- For non-interactive flows, pass secret values via environment variables (for example: `--value "$NPM_TOKEN"`), not literals.
- Prefer interactive secret prompts (`depot ci secrets add SECRET_NAME`) over command-line secret values.
- Do not hardcode secrets or tokens in commands, scripts, workflow YAML, logs, or examples.
- Use CI secret stores for `DEPOT_TOKEN` and other credentials; pass at runtime only.
- Avoid force/non-interactive destructive flags unless explicitly requested by the user.
- Before running credential-affecting commands, confirm scope (org, repo, workflow) and intended target.

## Managing Variables

Non-secret config values accessible as `${{ vars.VARIABLE_NAME }}`. Unlike secrets, values can be read back. Variables can be org-wide or scoped to a specific repository, just like secrets.

```bash
# Add org-wide variable
depot ci vars add VAR_NAME --value "some-value"

# Add repo-scoped variable
depot ci vars add VAR_NAME --value "some-value" --repo owner/repo

# List
depot ci vars list
depot ci vars list --output json
depot ci vars list --repo owner/repo    # Also show repo-specific variables

# Remove
depot ci vars remove VAR_NAME
depot ci vars remove VAR_NAME --force
depot ci vars remove VAR_NAME --repo owner/repo
```

## Secret and Variable Variants

A variant is another secret or variable with the same name but different repository scope or access rules. Variants let one name resolve to different values depending on the workflow context, for example a different `DATABASE_URL` for `production` vs `staging`, or for one repo vs all repos.

Variants are created and managed in the dashboard ([Depot CI workflows](https://depot.dev/orgs/_/workflows) → Settings → Secrets/Variables → Add variant). The CLI only supports repository scoping, so anything beyond that requires the dashboard.

### Access rule kinds

Limit when a variant is selected by combining one or more rules:

- **Repository**: select when the workflow runs in a specific repository.
- **Branch**: select when the branch name matches; supports glob patterns like `release/*`.
- **Workflow**: select when the workflow file matches; supports glob patterns like `deploy-*.yaml`.
- **Environment**: select when the job's GitHub `environment` field matches exactly. Provides compatibility with GitHub Environment Secrets. Jobs without an `environment` field never match environment rules.

Within a single rule kind, alternatives broaden availability (`branch=main` OR `branch=release/*`). Across kinds, the variant must satisfy all rule kinds (`branch=main` AND `environment=production`).

### Resolution priority

When multiple variants match a job, Depot picks the most specific:

1. Environment rules win over all others. An org-wide variant with an environment rule beats a repo-scoped variant with no environment rule.
2. Repository scope wins over branch and workflow rules, but not over environment rules.
3. Branch and workflow rules: literal matches win over globs, narrower globs win over broader globs (`release/v2` beats `release/*`).

For a repo to override an org-wide environment variant, create a variant with both the repository scope **and** the environment rule.

To preview which variant resolves for a given workflow context, open the dashboard secret or variable create/edit page, expand the **Secret variants** or **Variable variants** list, and enter a sample context (repo, branch, workflow file, environment). The matching variant is highlighted.

## Running Workflows

```bash
# Run a workflow
depot ci run --workflow .depot/workflows/ci.yml

# Run a workflow in a specific org (for multi-org users)
depot ci run --org <org-id> --workflow .depot/workflows/ci.yml

# Run specific jobs only
depot ci run --workflow .depot/workflows/ci.yml --job build --job test

# Run a job and connect via SSH
depot ci run --workflow .depot/workflows/ci.yml --job build --ssh

# Debug with tmate session after step N (requires single --job)
depot ci run --workflow .depot/workflows/ci.yml --job build --ssh-after-step 3
```

The CLI auto-detects uncommitted changes vs. the default branch, uploads a patch to Depot Cache, and injects a step to apply it after checkout, so your local working state runs without needing a push.

Use `--ssh` or `--ssh-after-step` on `depot ci run` to start a debug session when launching a new run. Use `depot ci ssh` (below) to connect to an already-running job.

## Dispatching Workflows

`depot ci dispatch` triggers a workflow via `workflow_dispatch` from the terminal. Inputs are validated against the workflow's declared input schema (required inputs must be supplied, typed inputs are coerced).

```bash
# Dispatch on a branch
depot ci dispatch --repo depot/cli --workflow deploy.yml --ref main

# Pass inputs (repeatable)
depot ci dispatch --repo depot/cli --workflow deploy.yml --ref main \
  --input environment=staging --input dry_run=true

# JSON output (returns the new run ID)
depot ci dispatch --repo depot/cli --workflow deploy.yml --ref main --output json
```

`--workflow` takes the workflow file's **basename** (for example `deploy.yml`), not the full path `.depot/workflows/deploy.yml`. This matches GitHub's `workflow_dispatch` API convention.

### `dispatch` flags

| Flag                    | Description                                                                |
| ----------------------- | -------------------------------------------------------------------------- |
| `--repo <owner/repo>`   | Target GitHub repository (required)                                        |
| `--workflow <filename>` | Workflow file basename, for example `deploy.yml` (required)                |
| `--ref <branch-or-tag>` | Branch or tag name to run the workflow on (required)                       |
| `--input <key>=<value>` | Workflow input as `key=value`; repeatable                                  |
| `--output json`         | Output the RPC response as JSON                                            |
| `--org <id>`            | Organization ID                                                            |
| `--token <token>`       | Depot API token                                                            |

## Custom Images

Build a custom image once and reuse it across jobs to skip repeated setup steps.

### Build the image

Use `depot/snapshot-action` (Depot CI only, not compatible with GitHub Actions):

```yaml
jobs:
  build-image:
    runs-on: depot-ubuntu-latest
    steps:
      - run: sudo apt-get install -y your-tool
      - uses: depot/snapshot-action@v1
        with:
          image: <org-id>.registry.depot.dev/my-ci-image:latest
```

### Use the image

Reference it in any Depot CI job with the `runs-on` object syntax:

```yaml
jobs:
  test:
    runs-on:
      size: 2x8
      image: <org-id>.registry.depot.dev/my-ci-image:latest
    steps:
      - uses: actions/checkout@v4
```

Available sizes: `2x8`, `4x16`, `8x32`, `16x64`, `32x128` (CPUs x RAM in GB).

**Constraints:** Images get pushed to and must be pulled from the Depot registry (`registry.depot.dev`), external registries are not supported.

## Parallel Steps

Depot CI supports running steps concurrently within a single job using `parallel:` blocks. This reduces job duration to the slowest branch rather than the sum of all steps. This is a Depot CI-specific feature, it is not compatible with GitHub Actions runners.

Use `parallel:` inside `steps:` with individual steps or `sequential:` groups. Each branch starts from the same job state; step outputs, environment variable and `$GITHUB_PATH` changes from all branches are merged back when the block completes.

```yaml
# Run lint, typecheck, and tests concurrently
steps:
  - uses: actions/checkout@v4
  - name: Install dependencies
    run: pnpm install
  - parallel:
      - name: Lint
        run: pnpm lint
      - name: Typecheck
        run: pnpm type-check
      - name: Test
        run: pnpm test
```

Use `sequential:` inside `parallel:` to group steps that must run in order within one branch:

```yaml
- parallel:
    - sequential:
        - name: Build
          run: npm run build
        - name: Test
          run: npm test
    - name: Lint
      run: npm run lint
```

Control failure behavior with `fail-fast:`. Defaults to `true` which cancels remaining steps in a parallel block. `false` will instead let all steps in the parallel block run to completion:

```yaml
- fail-fast: false
  parallel:
    - name: Lint
      run: pnpm lint
    - name: Typecheck
      run: pnpm type-check
```

**Limitations:**

- `parallel:` cannot be nested inside another `parallel:` (use `sequential:` inside `parallel:` instead)
- Step `id` values must be unique across the entire job (including even in different parallel blocks)

## SSH into Running Jobs

Connect to a running CI job via interactive terminal for debugging.

```bash
# Connect directly using a job ID
depot ci ssh <job-id>

# Connect to a specific job in a run
depot ci ssh <run-id> --job build

# Auto-select job when there's only one
depot ci ssh <run-id>

# Print SSH connection details for automation
depot ci ssh <run-id> --info --output json
```

The command waits up to 5 minutes for the job sandbox to be provisioned if it hasn't started yet.

### SSH flags

| Flag              | Description                                           |
| ----------------- | ----------------------------------------------------- |
| `--job <key>`     | Job key to connect to (required for multi-job runs)   |
| `--info`          | Print SSH details instead of connecting interactively |
| `-o, --output`    | Output format for `--info` (`json`)                   |
| `--org <id>`      | Organization ID                                       |
| `--token <token>` | Depot API token                                       |

## Checking Status and Logs

```bash
# Check run status (shows workflows -> jobs -> attempts hierarchy)
depot ci status <run-id>

# Fetch logs (accepts run ID, job ID, or attempt ID)
depot ci logs <run-id>
depot ci logs <attempt-id>

# Specify a job when the run has multiple jobs
depot ci logs <run-id> --job test

# Disambiguate when multiple workflows share the same job key
depot ci logs <run-id> --job build --workflow ci.yml
```

When given a run or job ID, `depot ci logs` resolves to the latest attempt automatically.

### Logs flags

| Flag                | Description                                             |
| ------------------- | ------------------------------------------------------- |
| `--job <key>`       | Job key to select (required when run has multiple jobs) |
| `--workflow <path>` | Workflow path to filter jobs (for example, `ci.yml`)    |
| `--org <id>`        | Organization ID                                         |
| `--token <token>`   | Depot API token                                         |

## Listing Runs and Triage Flow

`depot ci run list` is the primary entrypoint for debugging active/recent CI activity across workflows.

```bash
# List runs (defaults to queued + running)
depot ci run list

# Filter by status (repeatable)
depot ci run list --status failed
depot ci run list --status finished --status failed

# Filter by repository and commit SHA prefix
depot ci run list --repo depot/api --sha abc123

# Filter by trigger event
depot ci run list --trigger workflow_dispatch

# Filter failed runs for a pull request (--pr requires --repo)
depot ci run list --repo depot/api --status failed --pr 42

# Limit number of results
depot ci run list -n 5

# Machine-readable output for tooling/agents
depot ci run list --output json
```

### `run list` flags

| Flag                  | Description                                                                          |
| --------------------- | ------------------------------------------------------------------------------------ |
| `-n <int>`            | Number of runs to return (default `50`)                                              |
| `--status <name>`     | Filter by status; repeatable: `queued`, `running`, `finished`, `failed`, `cancelled` |
| `--repo <owner/repo>` | Filter by repository                                                                 |
| `--sha <prefix>`      | Filter by commit SHA prefix                                                          |
| `--trigger <event>`   | Filter by trigger event, for example `push` or `workflow_dispatch`                   |
| `--pr <number>`       | Filter by pull request number (requires `--repo`)                                    |
| `-o, --output`        | Output format (`json`)                                                               |
| `--org <id>`          | Organization ID                                                                      |
| `--token <token>`     | Depot API token                                                                      |

### Inspecting a single run

```bash
# Print a flat record for one run (org, repo, status, trigger, ref, sha, head sha, timestamps)
depot ci run show <run-id>
depot ci run get <run-id>            # alias

# JSON output
depot ci run show <run-id> --output json
```

Use `depot ci status <run-id>` instead when you need the full workflow/job/attempt hierarchy.

### Debugging failed runs

```bash
# Find failed runs
depot ci run list --status failed -n 10

# Pull logs directly (auto-selects job if only one)
depot ci logs <run-id>

# Specify job when there are multiple
depot ci logs <run-id> --job build

# Use status to inspect the full workflow/job/attempt hierarchy when needed
depot ci status <run-id>
```

Use `--output json` on `depot ci run list` for machine-readable output.

## Listing and Inspecting Workflows

`depot ci run list` returns runs (one entry per triggering event); `depot ci workflow list` returns workflows (one entry per workflow execution within a run, with per-job counts). Use `depot ci workflow list` when you want to filter by workflow `--name` (for example `deploy`), or to see job-count breakdowns rather than run-level status.

```bash
# List recent workflows (default 50, max 200)
depot ci workflow list
depot ci workflow ls                  # alias

# Filter by workflow name
depot ci workflow list --name deploy

# Filter by repo, status, head SHA, and pull request
depot ci workflow list --repo depot/api --status failed --sha abc123 --pr 42

# JSON output
depot ci workflow list --output json
```

### `workflow list` flags

| Flag                  | Description                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------- |
| `-n <int>`            | Number of recent workflows to return (default `50`, max `200`)                                    |
| `--name <name>`       | Filter by workflow name                                                                           |
| `--repo <owner/repo>` | Filter by repository                                                                              |
| `--status <name>`     | Filter by status; repeatable: `queued`, `running`, `finished`, `failed`, `cancelled`              |
| `--trigger <event>`   | Filter by trigger event, for example `push` or `workflow_dispatch`                                |
| `--sha <prefix>`      | Filter by head SHA prefix                                                                         |
| `--pr <number>`       | Filter by pull request number                                                                     |
| `-o, --output`        | Output format (`json`)                                                                            |
| `--org <id>`          | Organization ID                                                                                   |
| `--token <token>`     | Depot API token                                                                                   |

### Inspecting a single workflow

```bash
# Show parent run context, executions, jobs, and attempts
depot ci workflow show <workflow-id>
depot ci workflow get <workflow-id>   # alias

# JSON output
depot ci workflow show <workflow-id> --output json
```

## Cancelling, Rerunning, and Retrying Workflows

These verbs operate on workflows (or individual jobs within a workflow), not whole runs. They give the CLI parity with the dashboard for workflow- and job-level mutations.

### `depot ci cancel`

Cancels either an entire workflow (and all its jobs) with `--workflow <workflow-id>`, or a single job with `--job <job-id>`. Exactly one of `--workflow` or `--job` must be set; they are mutually exclusive. With `--job`, the CLI resolves the containing workflow from run status automatically. Workflows or jobs already in a terminal state (finished, failed, cancelled) cannot be cancelled and return an error.

```bash
# Cancel an entire workflow (and all its jobs)
depot ci cancel <run-id> --workflow <workflow-id>

# Cancel a single job (workflow is resolved automatically)
depot ci cancel <run-id> --job <job-id>

# JSON output
depot ci cancel <run-id> --workflow <workflow-id> --output json
```

| Flag              | Description                                                  |
| ----------------- | ------------------------------------------------------------ |
| `--workflow <id>` | Workflow ID to cancel (mutually exclusive with `--job`)      |
| `--job <id>`      | Job ID to cancel (mutually exclusive with `--workflow`)      |
| `--output json`   | Output the RPC response as JSON                              |
| `--org <id>`      | Organization ID                                              |
| `--token <token>` | Depot API token                                              |

### `depot ci rerun`

Re-runs every job in a workflow that has reached a terminal state. Creates a new attempt for each job. For runs that contain only one workflow, the CLI resolves it automatically; for multi-workflow runs, pass `--workflow <id>`. Rerunning a workflow that is still running returns a precondition error: cancel it first.

```bash
# Rerun the (single) workflow in a run
depot ci rerun <run-id>

# Rerun a specific workflow in a multi-workflow run
depot ci rerun <run-id> --workflow <workflow-id>

# JSON output
depot ci rerun <run-id> --output json
```

| Flag              | Description                                                                |
| ----------------- | -------------------------------------------------------------------------- |
| `--workflow <id>` | Workflow ID to rerun (required when the run contains multiple workflows)   |
| `--output json`   | Output the RPC response as JSON                                            |
| `--org <id>`      | Organization ID                                                            |
| `--token <token>` | Depot API token                                                            |

### `depot ci retry`

Retries a single failed or cancelled job with `--job <job-id>`, or every failed/cancelled job in a workflow with `--failed`. Exactly one of `--job` or `--failed` must be set; they are mutually exclusive. With `--job`, the containing workflow is resolved automatically from run status. With `--failed` on a multi-workflow run, `--workflow <id>` is required. Each retry creates a new attempt; previous attempts remain visible in `depot ci status`.

```bash
# Retry a single job
depot ci retry <run-id> --job <job-id>

# Retry every failed/cancelled job in the (single) workflow
depot ci retry <run-id> --failed

# Retry every failed/cancelled job in a specific workflow
depot ci retry <run-id> --failed --workflow <workflow-id>

# JSON output
depot ci retry <run-id> --job <job-id> --output json
```

| Flag              | Description                                                                        |
| ----------------- | ---------------------------------------------------------------------------------- |
| `--job <id>`      | Job ID to retry (mutually exclusive with `--failed`)                               |
| `--failed`        | Retry every failed/cancelled job in the workflow (mutually exclusive with `--job`) |
| `--workflow <id>` | Workflow ID; required with `--failed` when the run has multiple workflows          |
| `--output json`   | Output the RPC response as JSON                                                    |
| `--org <id>`      | Organization ID                                                                    |
| `--token <token>` | Depot API token                                                                    |

## Compatibility with GitHub Actions

### Supported

#### Workflow level

`name`, `run-name`, `on`, `permissions`, `env`, `concurrency`, `defaults`, `jobs`, `on.workflow_call` (with inputs, outputs, secrets)

#### Triggers

`push` (branches, tags, paths), `pull_request` (branches, paths), `pull_request_target`, `schedule`, `workflow_call`, `workflow_dispatch` (with inputs), `workflow_run`, `merge_group`

#### Job level

`name`, `needs`, `if`, `permissions`, `outputs`, `env`, `defaults`, `timeout-minutes`, `concurrency`, `strategy` (matrix, fail-fast, max-parallel), `continue-on-error`, `container`, `services`, `uses` (reusable workflows), `with`, `secrets`, `secrets.inherit`, `steps`

#### Step level

`id`, `name`, `if`, `uses`, `run`, `shell`, `with`, `env`, `working-directory`, `continue-on-error`, `timeout-minutes`

#### Permissions

`actions`, `checks`, `contents`, `id-token`, `metadata`, `pull_requests`, `statuses`, `workflows`

#### Expressions

`github`, `env`, `vars`, `secrets`, `needs`, `strategy`, `matrix`, `steps`, `job`, `runner`, `inputs` contexts. Functions: `always()`, `success()`, `failure()`, `cancelled()`, `case()`, `contains()`, `startsWith()`, `endsWith()`, `format()`, `join()`, `toJSON()`, `fromJSON()`, `hashFiles()`

#### Action types

JavaScript (Node 12/16/20/24), Composite, Docker

### Not Supported

- **Cross-repo reusable workflows**: `uses` referencing workflows in other repositories is not supported. Local reusable workflows work.
- **Fork-triggered PRs**: `pull_request` and `pull_request_target` from forks not supported yet
- **Non-Ubuntu runner labels**: all non-Depot labels silently treated as `depot-ubuntu-latest` (no error, runs on Ubuntu)
- **Deployment environments**: the `environment` field is not supported
- **GitHub-specific event triggers**: `branch_protection_rule`, `check_run`, `check_suite`, `create`, `delete`, `deployment`, `deployment_status`, `discussion`, `discussion_comment`, `fork`, `gollum`, `image_version`, `issue_comment`, `issues`, `label`, `milestone`, `page_build`, `public`, `pull_request_comment`, `pull_request_review`, `pull_request_review_comment`, `registry_package`, `release`, `repository_dispatch`, `status`, `watch`

### Runner labels

Depot CI sandboxes are x86_64 only. There is no Arm, macOS, or Windows support: those Depot GitHub Actions runner labels (`depot-ubuntu-*-arm`, `depot-macos-*`, `depot-windows-*`) are not compatible with Depot CI.

Supported labels:

| Label                   | Sandbox size | CPUs | RAM    |
| ----------------------- | ------------ | ---- | ------ |
| `depot-ubuntu-latest`   | `2x8`        | 2    | 8 GB   |
| `depot-ubuntu-24.04`    | `2x8`        | 2    | 8 GB   |
| `depot-ubuntu-24.04-4`  | `4x16`       | 4    | 16 GB  |
| `depot-ubuntu-24.04-8`  | `8x32`       | 8    | 32 GB  |
| `depot-ubuntu-24.04-16` | `16x64`      | 16   | 64 GB  |
| `depot-ubuntu-24.04-32` | `32x128`     | 32   | 128 GB |
| `depot-ubuntu-24.04-64` | `64x256`     | 64   | 256 GB |

Any label Depot CI can't parse is silently treated as `depot-ubuntu-latest`.

## Directory Structure

```
your-repo/
├── .github/
│   ├── workflows/     # Original GHA workflows (keep running)
│   └── actions/       # Local composite actions
├── .depot/
│   ├── workflows/     # Depot CI copies of workflows
│   └── actions/       # Depot CI copies of local actions
```

## Common Mistakes

| Mistake                                       | Fix                                                                                      |
| --------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Removing `.github/workflows/` after migration | Keep them during transition to verify Depot CI parity                                    |
| Using cross-repo reusable workflows           | Not supported yet, inline the workflow or copy it locally                                |
| Setting secrets without `--repo` when needed  | Use `--repo owner/repo` for repo-specific secret overrides                               |
| Running in the wrong org context              | Check `depot org show`, list with `depot org list`, then switch org or pass `--org <id>` |
| Forgetting `--org` flag with multiple orgs    | Migration or run commands may miss the expected repo/workflow; specify `--org <id>`      |
| Workflows with `runs-on: windows-latest`      | Treated as `depot-ubuntu-latest`, may fail                                               |

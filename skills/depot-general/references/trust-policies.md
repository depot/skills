# Project OIDC trust policies

Use `depot projects trust-policy` to manage which CI providers can authenticate to a Depot project. Supply an organization API token through `--token` or `DEPOT_TOKEN`. Select the project with `--project-id`, `DEPOT_PROJECT_ID`, or `depot.json`; add `--output json` for structured output.

```bash
depot projects trust-policy list --project-id <project-id> --output json
depot projects trust-policy add --project-id <project-id> --provider github --repository-owner acme --repository api
depot projects trust-policy update <policy-id> --project-id <project-id> --allow-public-pull-requests=false
depot projects trust-policy remove <policy-id> --project-id <project-id>
```

List returns one page. `--page-size` accepts 1–250 and defaults to 250. Pass `nextPageToken` with `--page-token`, keeping the project and optional `--provider` filter unchanged.

| Provider | Required identity flags | Optional restriction |
| --- | --- | --- |
| `github` | `--repository-owner`, `--repository` | `--allow-public-pull-requests` permits public pull requests from forks |
| `gitlab` | `--namespace-id`, `--gitlab-project-id` (numeric IDs) | `--protected-refs-only` restricts authentication to protected refs |
| `circleci` | `--organization-uuid`, `--project-uuid` | None |
| `buildkite` | `--organization-slug`, `--pipeline-slug` | None |

Adding an existing policy returns its saved settings without changing restrictions. Use `update` with the applicable restriction flag and an explicit `true` or `false`; only supplied flags change. Provider identities cannot be edited: add the replacement and remove the old policy. Removing a policy revokes its authorization without a confirmation prompt, so confirm the intended project and policy before execution.

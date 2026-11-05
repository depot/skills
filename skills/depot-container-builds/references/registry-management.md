# Depot Registry management

Use `depot registry` for organization-wide container registry management. Supply an organization API token through `--token` or `DEPOT_TOKEN`; stored login tokens are not used. The token selects the organization. Output is JSON.

Repository arguments are paths such as `team/app`, without the hostname, organization prefix, tag, or digest. Quote wildcard scopes. Delete, revoke, and retention-set commands require `--yes` and never prompt; execute them only for the user's intended targets.

## Repositories and images

```bash
depot registry repository list --query team/ --page 1 --page-size 50
depot registry repository create team/app
depot registry repository create team/base --upstream-id <id> --upstream-repository library/alpine
depot registry image list team/app --tag-status tagged --query release
depot registry image inspect team/app latest
```

Repository, image, and token lists return one page with pagination metadata. `--page` starts at 1; `--page-size` defaults to 25 and accepts 1–100. Image `--tag-status` accepts `all`, `tagged`, or `untagged`. Image inspection returns manifest bytes as base64 in JSON.

```bash
depot registry tag delete team/app old-release --yes
depot registry image delete team/app --tag old-release --tag staging --yes
depot registry repository delete team/old-app --yes
```

Image deletion accepts repeatable `--tag` or `--digest`; deleting a digest removes all associated tags. Inspect the JSON `failures` field: an error exit can follow partial deletion. Repository deletion removes its images. Use the separate `depot image` commands for project build images.

## Registry access tokens

```bash
depot registry token create release-pull --permission pull --scope 'team/*'
depot registry token list --query release
depot registry token revoke <token-id> --yes
```

Creation requires `--permission pull|push` and 1–50 repeatable `--scope` values. Optional flags: `--description` and `--expires-at <future-RFC3339-timestamp>`; omitted expiry means no expiration. Organization-token creation must be enabled for the organization. The new secret appears once in output; store it securely and keep it out of logs. Listings contain metadata only.

## Retention

```bash
depot registry retention get team/app
depot registry retention set team/app --keep-count 10 --yes
depot registry retention delete team/app --yes
```

Scopes can be repository paths or patterns such as `'team/**'`. `get` returns only the explicit policy, not inherited settings. `set` requires exactly one positive `--keep-count` or `--keep-days` and can delete older images. Deleting a policy removes its override; inherited retention can still apply and deleted images are not restored.

## Pull-through upstreams

```bash
depot registry upstream list
depot registry upstream create private-images --url registry.example.com --username <username> --password-stdin < password.txt
depot registry upstream update <upstream-id> --password-stdin < password.txt
depot registry upstream delete <upstream-id> --yes
```

Keep credentials out of `--url`. Basic authentication is the default. For ECR, use `--auth-type ecr --role-arn <arn>` instead of username/password credentials. Upstream updates change only supplied `--url`, `--username`, `--role-arn`, or `--password-stdin` fields; an explicitly empty username or password clears the value. Authentication type cannot be changed. Listings never return passwords.

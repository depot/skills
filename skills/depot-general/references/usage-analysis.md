# Usage analysis

Use an organization API token or read-only organization token in `DEPOT_TOKEN` or `--token`. These commands do not prompt for login or read stored login credentials. The token determines the organization.

## Daily series and totals

```bash
depot usage series --product depot-ci --start 2026-11-01T00:00:00Z --end 2026-11-08T00:00:00Z
depot usage series --project <project-id> --start 2026-11-01T00:00:00Z --end 2026-11-08T00:00:00Z
```

Output is JSON. Repeat `--product` to select products: `container-builds`, `github-actions`, `depot-ci`, `test-results`, `storage`, or `agent-sandboxes`. Use `--project` instead of `--product` for a project's container-build and layer-cache series.

Supply both `--start` and `--end` as RFC3339 timestamps. The start is inclusive, the end exclusive, and the maximum range is 93 days. Without dates, the range starts at the current UTC month's beginning and ends now. Daily buckets use UTC.

## Attribute usage

```bash
depot usage breakdown --product depot-ci --group-by workflow --repo owner/repo --start 2026-11-01T00:00:00Z --end 2026-11-08T00:00:00Z
```

Supported combinations:

| `--product` | `--group-by` |
|-------------|--------------|
| `container-builds` | `project` |
| `github-actions` | `repository`, `runner`, `job` |
| `depot-ci` | `repository`, `workflow` |
| `test-results` | `repository` |

Filter container builds with repeatable `--project`; filter other products with repeatable `--repo owner/name`. For GitHub Actions job groups, use repeatable `--workflow-path`. For Depot CI workflow groups, use repeatable `--workflow-key` with the stable key returned by the API. Other combinations are rejected.

Each call returns one page. Set `--page-size` from 1 to 500 (default 100), then pass `nextPageToken` as `--page-token` with unchanged dates and filters. Continue even after an empty page when a token remains. Prefer explicit dates when paginating. Pages are not a fixed snapshot; late usage can change ordering.

## Interpret results

- Inspect coverage fields, including `availableFrom`, `unavailable`, `partialHistory`, and `note`, before treating zero as no usage.
- Keep product units separate. Elapsed seconds, vCPU-seconds, and GitHub Actions billable seconds normalized to a 2-vCPU runner are not interchangeable. Usage quantities are not invoice amounts.
- Preserve JSON integer strings when handling large counts. Do not coerce every value to a floating-point number.
- Use returned period storage averages; do not sum daily averages.
- Use `depot usage current` for the current billing period. It accepts no date flags, and its period can differ from the calendar-month default above.
- Use `depot usage report` for legacy export rows only. It can truncate at 5,000 rows and excludes Depot CI and registry storage; prefer paginated breakdowns for attribution.
- Use `depot usage agents` for completed remote-agent session compute, running-session counts, and recent sessions. These sessions cover a subset of sandbox usage.

API details: https://depot.dev/docs/api/usage-analysis

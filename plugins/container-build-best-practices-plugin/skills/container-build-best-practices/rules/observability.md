---
name: observability
description: Use build logs and metrics to diagnose build performance
metadata:
  tags: logs, metrics, observability, cache
---

## Logs

- Logs show step status with `cached` or `deduplicated` badges.
- Filter steps by status or search step names to find slow stages.
- Share build logs via the **Share build** link when collaborating.

## Metrics

- Build details show resource utilization and OOM warnings.
- Step-level metrics highlight regressions in duration or cache hits.
- Use the build list charts to spot sudden cache or duration changes.

## Performance Investigation

- Check logs to identify the slowest steps.
- Use metrics to see cache hit drops or resource saturation.
- Inspect the build context size when builds spike unexpectedly.

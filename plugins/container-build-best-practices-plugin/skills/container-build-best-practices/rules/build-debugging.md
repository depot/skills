---
name: build-debugging
description: Debug failed builds by fetching build steps and logs via the Depot API
metadata:
  tags: debug, logs, failed, api, steps, errors
---

## Get Build ID

Find the build ID from:
- Build output: `Build: https://depot.dev/orgs/<org>/projects/<project>/builds/<build-id>`
- Metadata file: `depot build --metadata-file=build.json` then `jq -r '."depot.build"' build.json`
- Build list: `depot list builds --project <project-id>`

## Debug via Web UI

The fastest way to debug a failed build:
1. Open the build URL in the Depot dashboard
2. Failed steps are highlighted in red
3. Click the step to expand logs
4. Use **Share build** to share with teammates

## Debug via API

Use the Depot Build Service API to programmatically fetch build steps and logs.

### API Token Setup

API calls require an **API token** (different from project tokens used in CI).

**Get your organization settings URL:**
```bash
# Get your organization ID
depot org show

# Then visit: https://depot.dev/orgs/<org-id>/settings
```

**Create the token:**
1. Go to `https://depot.dev/orgs/<org-id>/settings`
2. Under **API Tokens**, enter a description
3. Click **Create token**
4. Set as environment variable:

```bash
export DEPOT_API_TOKEN=your_api_token_here
```

API tokens grant access to manage projects and builds within your organization, including full push/pull permissions to the Depot Registry.

### Get Build Steps

Fetch all steps for a build to find which one failed:

**Node.js:**
```typescript
import {createPromiseClient} from '@connectrpc/connect';
import {createConnectTransport} from '@connectrpc/connect-node';
import {BuildService} from '@depot/api/depot/build/v1/build_connect';

const transport = createConnectTransport({
  baseUrl: 'https://api.depot.dev',
  httpVersion: '2',
});

const client = createPromiseClient(BuildService, transport);

const response = await client.getBuildSteps(
  {buildId: 'BUILD_ID', projectId: 'PROJECT_ID', pageSize: 100},
  {headers: {Authorization: `Bearer ${process.env.DEPOT_API_TOKEN}`}},
);

// Find failed steps
const failedSteps = response.steps.filter(step => step.error);
for (const step of failedSteps) {
  console.log(`Failed step: ${step.name}`);
  console.log(`Digest: ${step.digest}`);
  console.log(`Error: ${step.error}`);
}
```

**Go:**
```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "os"

    "connectrpc.com/connect"
    buildv1 "github.com/depot/api/go/depot/build/v1"
    "github.com/depot/api/go/depot/build/v1/buildv1connect"
)

func main() {
    client := buildv1connect.NewBuildServiceClient(
        http.DefaultClient,
        "https://api.depot.dev",
    )

    req := connect.NewRequest(&buildv1.GetBuildStepsRequest{
        BuildId:   "BUILD_ID",
        ProjectId: "PROJECT_ID",
        PageSize:  100,
    })
    req.Header().Set("Authorization", "Bearer "+os.Getenv("DEPOT_API_TOKEN"))

    resp, err := client.GetBuildSteps(context.Background(), req)
    if err != nil {
        panic(err)
    }

    for _, step := range resp.Msg.Steps {
        if step.Error != "" {
            fmt.Printf("Failed step: %s\nDigest: %s\nError: %s\n",
                step.Name, step.Digest, step.Error)
        }
    }
}
```

### Get Logs for a Failed Step

Once you have the step digest, fetch the logs:

**Node.js:**
```typescript
const logsResponse = await client.getBuildStepLogs(
  {
    buildId: 'BUILD_ID',
    projectId: 'PROJECT_ID',
    buildStepDigest: failedStep.digest,
    pageSize: 1000,
  },
  {headers: {Authorization: `Bearer ${process.env.DEPOT_API_TOKEN}`}},
);

for (const line of logsResponse.lines) {
  console.log(line.text);
}
```

**Go:**
```go
logsReq := connect.NewRequest(&buildv1.GetBuildStepLogsRequest{
    BuildId:         "BUILD_ID",
    ProjectId:       "PROJECT_ID",
    BuildStepDigest: failedStep.Digest,
    PageSize:        1000,
})
logsReq.Header().Set("Authorization", "Bearer "+os.Getenv("DEPOT_API_TOKEN"))

logsResp, err := client.GetBuildStepLogs(context.Background(), logsReq)
if err != nil {
    panic(err)
}

for _, line := range logsResp.Msg.Lines {
    fmt.Println(line.Text)
}
```

## Complete Debug Script

**Node.js script to find and print failed step logs:**
```typescript
import {createPromiseClient} from '@connectrpc/connect';
import {createConnectTransport} from '@connectrpc/connect-node';
import {BuildService} from '@depot/api/depot/build/v1/build_connect';

async function debugBuild(buildId: string, projectId: string) {
  const transport = createConnectTransport({
    baseUrl: 'https://api.depot.dev',
    httpVersion: '2',
  });

  const client = createPromiseClient(BuildService, transport);
  const headers = {Authorization: `Bearer ${process.env.DEPOT_API_TOKEN}`};

  // Get all build steps
  const stepsResponse = await client.getBuildSteps(
    {buildId, projectId, pageSize: 100},
    {headers},
  );

  // Find failed steps
  const failedSteps = stepsResponse.steps.filter(step => step.error);

  if (failedSteps.length === 0) {
    console.log('No failed steps found');
    return;
  }

  // Get logs for each failed step
  for (const step of failedSteps) {
    console.log(`\n=== Failed Step: ${step.name} ===`);
    console.log(`Error: ${step.error}\n`);

    const logsResponse = await client.getBuildStepLogs(
      {buildId, projectId, buildStepDigest: step.digest, pageSize: 1000},
      {headers},
    );

    for (const line of logsResponse.lines) {
      console.log(line.text);
    }
  }
}

// Usage: DEPOT_API_TOKEN=xxx npx ts-node debug.ts BUILD_ID PROJECT_ID
debugBuild(process.argv[2], process.argv[3]);
```

## Common Failure Patterns

| Log Pattern | Likely Cause | Solution |
|-------------|--------------|----------|
| `COPY failed: file not found` | File missing from build context | Check `.dockerignore`, verify file path |
| `RUN npm install` fails | Network or dependency issue | Add cache mount, check package.json |
| `OOMKilled` | Out of memory | Increase builder size or optimize build |
| `unauthorized: authentication required` | Registry auth failed | Run `docker login` before build |
| `context canceled` | Build timeout or interruption | Check for resource constraints |

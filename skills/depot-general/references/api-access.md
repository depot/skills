# API Access

Protocol: Connect framework (gRPC + HTTP JSON). SDKs: `@depot/sdk-node` (Node.js), `depot/depot-go` (Go).

```javascript
import {depot} from '@depot/sdk-node'
const headers = { Authorization: `Bearer ${process.env.DEPOT_TOKEN}` }

// List projects
const result = await depot.core.v1.ProjectService.listProjects({}, {headers})

// Create a build
const build = await depot.build.v1.BuildService.createBuild(
  {projectId: '<id>'}, {headers}
)
```

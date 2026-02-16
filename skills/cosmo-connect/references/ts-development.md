# TypeScript Development

## Plugin Development (Bun)

Available since wgc@0.96.0. Plugins run on Bun runtime.

### Structure

```
plugin/
├── bin/grpc-health-check/proto/health/v1/health.proto
└── src/plugin-server.ts
```

`plugin-server.ts` provides health check, Unix socket, stdout handshake.

### Implementation

```typescript
import * as grpc from '@grpc/grpc-js';
import { MyServiceService, IMyServiceServer } from '../generated/service_grpc_pb.js';
import { QueryRequest, QueryResponse } from '../generated/service_pb.js';
import { PluginServer } from './plugin-server.js';

export class MyService implements IMyServiceServer {
    [name: string]: grpc.UntypedHandleCall;

    queryData(
        call: grpc.ServerUnaryCall<QueryRequest, QueryResponse>,
        callback: grpc.sendUnaryData<QueryResponse>
    ) {
        const response = new QueryResponse();
        // ... set fields
        callback(null, response);
    }
}

function run() {
    const pluginServer = new PluginServer();
    pluginServer.addService(MyServiceService, new MyService());
    pluginServer.serve().catch((error) => {
        console.error('Failed to start plugin server:', error);
        process.exit(1);
    });
}

run();
```

### Debugging

Build with `--debug`. Debug via:
- **Bun web debugger:** URL in `plugin_stderr.log`
- **VS Code:** Bun extension + WebSocket URL from `plugin_stderr.log`

## Connect RPC Service (Fastify)

### Implementation

```typescript
import type { ConnectRouter } from "@connectrpc/connect";
import { Service } from "./proto/service/v1/service_pb.js";

export default (router: ConnectRouter) => {
    router.service(Service, {
        queryGetProject: async (req) => ({
            getProject: { id: req.id, name: "My Project" },
        }),
    });
};
```

### Build & Run

```bash
npm install
npm run bootstrap    # Generates proto, buf code, router config, downloads router
npm run start        # Starts gRPC service (port 50051) + Router (port 3002)
```

### Generated Files

- `src/graph/schema.graphql` - GraphQL schema (source of truth)
- `src/proto/service/v1/` - Generated protobuf files
- `router.compose.yaml` - Router composition config
- `router.config.yaml` - Router runtime config
- `router.execution.config.json` - Generated router execution config

## Required Versions

| Tool | Version |
|------|---------|
| Bun | ^1.2.15 |
| Node | ^22.11.0 |

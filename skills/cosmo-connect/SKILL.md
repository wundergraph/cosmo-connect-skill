---
name: cosmo-connect
description: >
  Create and modify APIs using WunderGraph Cosmo Connect. Covers gRPC plugin development (Go/TypeScript),
  standalone gRPC services, schema-to-proto workflows, CLI commands (wgc), local router setup,
  graph.yaml composition, field resolvers, batching, federation entities, and deployment.
  Use when building Cosmo Connect subgraphs, plugins, gRPC services, or setting up a local Cosmo Router.
---

# Cosmo Connect

Cosmo Connect lets you use GraphQL Federation without running GraphQL servers. Define an Apollo-compatible Subgraph Schema, compile it to protobuf, implement it as gRPC.

## Decision: Plugin vs gRPC Service

| Factor | Router Plugin | gRPC Service |
|--------|--------------|--------------|
| Languages | Go, TypeScript (Bun) | Any gRPC language |
| Deployment | Co-located with router | Independent microservice |
| Scaling | Coupled to router | Independent |
| Latency | Minimal (IPC) | Network overhead |
| Team autonomy | Low | High |

**Use plugins** for simple integrations, lowest latency, wrapping legacy APIs.
**Use gRPC services** for team ownership, independent scaling, language flexibility.

## Core Workflow

```
1. Define GraphQL Schema (standard Subgraph Schema)
2. Generate Protobuf + Mapping (wgc CLI)
3. Implement gRPC logic
4. Configure Router
5. Deploy and test
```

### Plugin Workflow

```bash
# Scaffold (Go)
wgc router plugin init my-plugin -p myproject --language go

# Or scaffold (TypeScript)
wgc router plugin init my-plugin -p myproject --language ts

# Edit schema: my-plugin/src/schema.graphql
# Generate proto + code:
wgc router plugin generate ./my-plugin

# Implement resolvers in src/main.go or src/plugin.ts
# Test:
wgc router plugin test ./my-plugin

# Build:
wgc router plugin build ./my-plugin

# Publish to Cosmo Cloud:
wgc router plugin publish ./my-plugin
```

### gRPC Service Workflow

```bash
# Scaffold
wgc grpc-service init --template golang-connect-rpc --directory ./my-service
# Or: --template typescript-connect-rpc-fastify

# Edit schema, then generate:
wgc grpc-service generate -i ./schema.graphql -o ./generated

# Implement service, then publish:
wgc grpc-service publish my-service --schema ./schema.graphql --generated ./generated
```

## Go Plugin Implementation Pattern

```go
package main

import (
    "log"
    service "github.com/wundergraph/cosmo/plugin/generated"
    routerplugin "github.com/wundergraph/cosmo/router-plugin"
    "google.golang.org/grpc"
)

func main() {
    pl, err := routerplugin.NewRouterPlugin(func(s *grpc.Server) {
        s.RegisterService(&service.MyService_ServiceDesc, &MyServiceImpl{})
    })
    if err != nil {
        log.Fatalf("failed to create router plugin: %v", err)
    }
    pl.Serve()
}
```

## TypeScript Plugin Implementation Pattern

```typescript
import * as grpc from '@grpc/grpc-js';
import { MyServiceService, IMyServiceServer } from '../generated/service_grpc_pb.js';
import { PluginServer } from './plugin-server.js';

class MyService implements IMyServiceServer {
    [name: string]: grpc.UntypedHandleCall;
    queryData(call, callback) {
        const response = new QueryResponse();
        callback(null, response);
    }
}

const pluginServer = new PluginServer();
pluginServer.addService(MyServiceService, new MyService());
pluginServer.serve();
```

## Go gRPC Service Pattern (Connect RPC)

```go
type Service struct{}

func (s *Service) QueryGetProject(ctx context.Context, req *connect.Request[pb.QueryGetProjectRequest]) (*connect.Response[pb.QueryGetProjectResponse], error) {
    return connect.NewResponse(&pb.QueryGetProjectResponse{
        GetProject: &pb.Project{Id: req.Msg.Id, Name: "My Project"},
    }), nil
}
```

## TypeScript gRPC Service Pattern (Connect RPC + Fastify)

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

## Field Resolvers

For fields with arguments, use `@connect__fieldResolver`:

```graphql
type Foo {
    id: ID!
    bar(baz: String!): String! @connect__fieldResolver(context: "id")
}
```

Generates a dedicated `ResolveFooBar` RPC. Batching is automatic.
**Critical:** Return results in the exact same order as provided context elements.

## Local Router Setup (No Cloud)

Set `ROUTER_CONFIG_PATH` to run in static config mode — no controlplane or API token needed.

```bash
# 1. Create graph.yaml declaring subgraphs (HTTP, gRPC standalone, or plugin)
# 2. Compose into execution config:
wgc router compose -i ./graph.yaml -o ./config.json

# 3. Run router:
ROUTER_CONFIG_PATH=./config.json go run ./cmd/router/main.go

# For plugins, also set:
PLUGINS_ENABLED=true PLUGINS_PATH=./plugins
```

### graph.yaml Subgraph Shapes

```yaml
# Standard GraphQL (HTTP)
- name: my-subgraph
  routing_url: http://localhost:4001/graphql
  schema:
    file: ./path/to/schema.graphqls

# Connect - Standalone gRPC
- name: my-grpc-subgraph
  routing_url: dns:///localhost:4011
  grpc:
    schema_file: ./path/to/src/schema.graphql
    proto_file: ./path/to/generated/service.proto
    mapping_file: ./path/to/generated/mapping.json

# Connect - Plugin
- name: my-plugin-subgraph
  plugin:
    version: 0.0.1
    path: ./path/to/plugin-dir
```

### Router Config (config.yaml)

```yaml
version: "1"
plugins:
  enabled: true
  path: "plugins"
```

## GraphQL Feature Support

Supported: Query, Mutation, Entity Lookups (single/multiple/compound keys), Scalar/Complex args, Enums, Interfaces, Unions, Recursive types, Nested objects, Lists, Entity definitions, Key directives, External fields, Field resolvers (with batching).

Not yet supported: Nested key entity lookups, @requires, Subscriptions, Nullable list items (protobuf constraint).

## References

- For CLI command details, see [references/cli-reference.md](references/cli-reference.md)
- For Go development details (plugins + HTTP client + logging + tracing), see [references/go-development.md](references/go-development.md)
- For TypeScript development details, see [references/ts-development.md](references/ts-development.md)
- For local development setup (router, graph.yaml, standalone/plugin step-by-step, troubleshooting), see [references/local-development.md](references/local-development.md)
- For internal ConnectRPC architecture, see [references/architecture.md](references/architecture.md)

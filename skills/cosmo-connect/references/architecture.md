# Internal ConnectRPC Architecture

## Component Communication

```
Proto definitions (proto/wg/cosmo/)
    | buf generate
    v
Generated clients (@wundergraph/cosmo-connect, connect-go/)
    |
  CLI (Node.js)     Studio (Next.js)     Controlplane (Fastify)
  createPromise     useQuery()           fastifyConnectPlugin
  Client()          connect-query        router.service(...)
```

Wire protocol: `POST /wg.cosmo.platform.v1.PlatformService/MethodName`
Content-Type: `application/connect+proto` or `+json`

## Services

### PlatformService (~150+ RPCs)
Main API for CLI and Studio. Domains: Namespaces, Federated Graphs, Subgraphs, Monographs, Contracts, Feature Flags, Proposals, Schema Checks, Overrides, Subgraph Linking, Router Tokens, API Keys, Organization, RBAC, Webhooks, Integrations, SSO/OIDC, Analytics, Linting, Pruning, Cache Warmer, Billing, Persisted Ops, Plugins.

### NodeService
Router self-registration: `SelfRegister` RPC.

### GraphQLMetricsService
Router metrics export: `PublishGraphQLMetrics`, `PublishAggregatedGraphQLMetrics`.

## Code Generation

| Config | Target | Output |
|--------|--------|--------|
| `buf.ts.gen.yaml` | TypeScript | `connect/src/` |
| `buf.router.go.gen.yaml` | Go (Router) | `router/gen/proto/` |
| `buf.connect-go.go.gen.yaml` | Go (connect-go) | `connect-go/gen/proto/` |
| `buf.graphqlmetrics.go.gen.yaml` | Go (Metrics) | `graphqlmetrics/gen/proto/` |

Commands: `pnpm generate` (TS), `make generate-go` (Go), `make generate` (both).

## CLI Client

```typescript
// cli/src/core/client/client.ts
const transport = createConnectTransport({
    baseUrl: opts.baseUrl,       // default: https://cosmo-cp.wundergraph.com
    httpVersion: '1.1',
    sendCompression: compressionBrotli,
});

return {
    platform: createPromiseClient(PlatformService, transport),
    node: createPromiseClient(NodeService, transport),
};
```

Auth: `Authorization: Bearer <COSMO_API_KEY>` per-call header.

## Controlplane Handler Pattern

```typescript
// controlplane/src/core/bufservices/subgraph/linkSubgraph.ts
export function linkSubgraph(opts, req, ctx) {
    return handleError(ctx, logger, async () => {
        const authContext = await opts.authenticator.authenticate(ctx.requestHeader);
        // validate, authorize, execute, return
        return { response: { code: EnumStatusCode.OK } };
    });
}
```

Handlers in: `controlplane/src/core/bufservices/<domain>/`

## Studio Client

Uses `@connectrpc/connect-web`, cookie auth (`credentials: "include"`), `useHttpGet: true` for caching.

## Router gRPC Connector

`router/pkg/grpcconnector/` - NOT ConnectRPC, uses standard gRPC:
- `grpcremote.RemoteGRPCProvider` - remote endpoint
- `grpcplugin.GRPCPlugin` - local HashiCorp go-plugin
- `grpcplugin.GRPCOCIPlugin` - OCI container plugin

## Environment Variables

| Variable | Default |
|----------|---------|
| `COSMO_API_URL` | `https://cosmo-cp.wundergraph.com` |
| `COSMO_API_KEY` | (required) |
| `COSMO_WEB_URL` | `https://cosmo.wundergraph.com` |
| `CDN_URL` | `https://cosmo-cdn.wundergraph.com` |
| `PLUGIN_REGISTRY_URL` | `cosmo-registry.wundergraph.com` |

## Subgraph Linking

`wgc subgraph link <source> -t <namespace>/<target>` - links source to target for cross-namespace schema check propagation. One link per source (unique constraint). Schema checks on source also run traffic/pruning checks against target's federated graphs.

DB table: `linked_subgraphs` with cascade deletes. RBAC: write on source, read on target.

# Go Development

## Plugin Library

Package `router-plugin` provides:
- Plugin server init via HashiCorp go-plugin
- HTTP client with middleware, tracing, generics
- Structured logging (hclog)
- Automatic panic recovery
- OpenTelemetry tracing

### Plugin Init

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

### HTTP Client

```go
import "github.com/wundergraph/cosmo/router-plugin/httpclient"

client := httpclient.New(
    httpclient.WithBaseURL("https://api.example.com"),
    httpclient.WithTimeout(10 * time.Second),
    httpclient.WithHeader("Accept", "application/json"),
    httpclient.WithMiddleware(httpclient.AuthBearerMiddleware("token")),
    httpclient.WithTracing(),
)

resp, err := client.Get(ctx, "/users/1")
user, err := httpclient.UnmarshalTo[User](resp)
```

Built-in middleware: `AuthBearerMiddleware`, `BasicAuthMiddleware`, `UserAgentMiddleware`.

### Logging

```go
import "github.com/hashicorp/go-hclog"

pl, err := routerplugin.NewRouterPlugin(registerFunc,
    routerplugin.WithLogger(hclog.Info),
)

// In resolvers:
logger := hclog.FromContext(ctx)
logger.Info("Processing request", "project_count", len(req.GetProjects()))
```

### Tracing

```go
pl, err := routerplugin.NewRouterPlugin(registerFunc,
    routerplugin.WithTracing(),
)

// In resolvers:
tracer := otel.Tracer("my-service")
ctx, span := tracer.Start(ctx, "operation-name")
defer span.End()
```

### Debugging

Build with `--debug`, attach Delve/GoLand/VS Code. Process name: `<os>_<arch>`.

## Connect RPC Service Implementation

```go
package service

import (
    "context"
    "connectrpc.com/connect"
    pb "github.com/wundergraph/service/pkg/generated/service/v1"
)

type Service struct{}

func (s *Service) QueryGetProject(ctx context.Context, req *connect.Request[pb.QueryGetProjectRequest]) (*connect.Response[pb.QueryGetProjectResponse], error) {
    return connect.NewResponse(&pb.QueryGetProjectResponse{
        GetProject: &pb.Project{Id: req.Msg.Id, Name: "My Project"},
    }), nil
}
```

### Build & Run (Go gRPC service)

```bash
make                 # Generates proto, buf code, composes config, downloads router
make start           # Starts gRPC service (port 50051) + Router (port 3002)
```

## Required Versions

| Tool | Version |
|------|---------|
| Go | >=1.22.0 |
| protoc | ^29.3 |
| protoc-gen-go | ^1.34.2 |
| protoc-gen-go-grpc | ^1.5.1 |

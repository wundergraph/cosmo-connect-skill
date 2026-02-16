# CLI Reference

## Router Plugin Commands (`wgc router plugin`)

### `wgc router plugin init [options] <name>`
Scaffold a new plugin project.

| Option | Description | Default |
|--------|-------------|---------|
| `-p, --project <project>` | Creates minimal router project with plugin | (none) |
| `-d, --directory <directory>` | Directory to create in | `.` |
| `-l, --language <language>` | `go` or `ts` | `go` |

Creates: `src/schema.graphql`, `src/main.go` (or `plugin.ts`), test file, `generated/mapping.json`, `generated/service.proto`, `generated/service.proto.lock.json`, Makefile, Dockerfile, README.

With `--project`, also creates `config.yaml` and `graph.yaml`.

### `wgc router plugin generate [directory] [options]`
Generate proto, mapping, and gRPC code from schema.

| Option | Description | Default |
|--------|-------------|---------|
| `--go-module-path <path>` | Go module path | `github.com/wundergraph/cosmo/plugin` |
| `--skip-tools-installation` | Skip tool install | `false` |
| `--force-tools-installation` | Force tool install | `false` |

### `wgc router plugin build [options] [directory]`
Generate code and compile to platform-specific binaries.

| Option | Description | Default |
|--------|-------------|---------|
| `--generate-only` | Generate only, don't compile | `false` |
| `--go-module-path <path>` | Go module path | (default) |
| `--debug` | Build with debug symbols | `false` |
| `--platform [platforms...]` | Target platforms | Host platform |
| `--all-platforms` | Build all platforms | `false` |

Platforms: `linux-amd64`, `linux-arm64`, `darwin-amd64`, `darwin-arm64`, `windows-amd64`

### `wgc router plugin test [options] [directory]`
Run plugin tests.

### `wgc router plugin publish [directory]`
Build Docker image, push to registry, publish schema.

| Option | Description | Default |
|--------|-------------|---------|
| `--name [string]` | Plugin subgraph name | (from dir) |
| `-n, --namespace [string]` | Namespace | `"default"` |
| `--platform [platforms...]` | Docker platforms | `linux/amd64` + current |
| `--label [labels...]` | Labels `key=value` | |
| `--fail-on-composition-error` | Fail on composition error | `false` |
| `--fail-on-admission-webhook-error` | Fail on webhook error | `false` |

Requires Docker Buildx with container builder.

### `wgc router plugin create [pluginName] --label [labelName]`
Create plugin subgraph on control plane (without publishing).

### `wgc router plugin delete <name> [-f, --force]`
Delete plugin subgraph from control plane.

---

## gRPC Service Commands (`wgc grpc-service`)

### `wgc grpc-service init [options]`

| Option | Description | Default |
|--------|-------------|---------|
| `-t, --template <template>` | Template name | `typescript-connect-rpc-fastify` |
| `-d, --directory <directory>` | Output directory | `.` |

Templates: `golang-connect-rpc`, `typescript-connect-rpc-fastify`

### `wgc grpc-service list-templates`
List available templates.

### `wgc grpc-service generate [options] [service-name]`

| Option | Description | Default |
|--------|-------------|---------|
| `-i, --input <path>` | GraphQL schema file (required) | |
| `-o, --output <path>` | Output directory | `.` |
| `-p, --package-name <name>` | Proto package name | `service.v1` |
| `-g, --go-package <name>` | Go package option | |

Output: `service.proto`, `service.mapping.json`, `service.proto.lock.json`

### `wgc grpc-service create <name> -r <routing-url> [options]`
Create gRPC subgraph on control plane.

Routing URL format: `dns:///localhost:4001/`

### `wgc grpc-service publish <name> --schema <path> --generated <path> [options]`
Publish gRPC subgraph. Auto-creates if not exists.

Required: `--schema` (GraphQL file), `--generated` (folder with proto, mapping, lock files).

### `wgc grpc-service delete <name> [-f, --force]`
Delete gRPC subgraph.

---

## Composition

### `wgc router compose`
Local composition: combines subgraph schemas into router execution config.

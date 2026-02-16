# Cosmo Connect Skill for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that teaches Claude how to build APIs using [WunderGraph Cosmo Connect](https://cosmo-docs.wundergraph.com/cli/router/plugin).

## What it covers

- gRPC plugin development (Go / TypeScript)
- Standalone gRPC services (Connect RPC)
- Schema-to-protobuf workflows
- CLI commands (`wgc`)
- Local router setup
- `graph.yaml` composition
- Field resolvers, batching, federation entities
- Deployment

## Install

```
/plugin marketplace add wundergraph/cosmo-connect-skill
/plugin install cosmo-connect@wundergraph-cosmo-connect
```

## Usage

Once installed, Claude Code automatically activates the skill when you work on Cosmo Connect tasks. You can also invoke it explicitly:

```
Use the cosmo-connect skill to scaffold a Go plugin for my user service
```

## License

Apache-2.0

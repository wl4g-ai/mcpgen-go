# Go MCP server Generator from OpenAPI 3.x specification

Generate production-ready Model Context Protocol (MCP) servers from OpenAPI specs. Each API operation becomes an AI tool that forwards requests to your upstream service.

## Building

```sh
make
# binary: bin/mcpgen
```

## Quick Start

### 1. Generate the MCP server

```sh
./bin/mcpgen -i testdata/example_confluence_oas_v3.1.yaml -o myconfluence-mcp
cd myconfluence-mcp
```

This produces a complete Go project with tools for every operation:

```
myconfluence-mcp/
├── .credentials                 # file-based token (set MCP_UPSTREAM_TOKEN_FILE)
├── main.go                      # entry point (stdio/http transport)
├── client.sh                    # quick curl-based test script
├── Makefile                     # build / run / clean / test
├── myconfluence-mcp             # compiled binary
└── internal/
    ├── mcpserver/server.go      # MCP server setup + tool registration
    ├── helpers/                 # ForwardRequest, logging, parameter parsing
    └── mcptools/                # one file per API operation
        ├── GetPage.go
        ├── CreatePage.go
        ├── UpdatePage.go
        ├── DeletePage.go
        ├── SearchContent.go
        └── ...
```

### 2. Start the server

The server defaults to httpbin.org which echoes requests — great for quick verification:

```sh
./myconfluence-mcp --transport http --port 8080 -v 1
# MCP_UPSTREAM_ENDPOINT=https://httpbin.org/anything
```

Set your actual upstream to enable real API calls:

```sh
export MCP_UPSTREAM_ENDPOINT=https://example.atlassian.net/wiki/rest/api
# Option 1: pass token via env var
MCP_UPSTREAM_TOKEN=your-token ./myconfluence-mcp --transport http --port 8080 -v 1

# Option 2: read token from file (safer, no shell history exposure)
echo -n "your-token" > .credentials
MCP_UPSTREAM_TOKEN_FILE=.credentials ./myconfluence-mcp --transport http --port 8080 -v 1
```

### 3. Test with client.sh

```sh
./client.sh list-tools
./client.sh call GetPage '{"id": "123456"}'
```

## Agent Integration

### Local Mode (stdio)

Run the MCP server as a child process — recommended for local development.

### OpenCode

`~/.config/opencode/config.json`:

```json
{
  "mcp": {
    "myconfluence": {
      "type": "local",
      "command": ["bash", "-c", "./myconfluence-mcp"],
      "args": ["--transport", "stdio"],
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      },
      "enabled": true
    }
  }
}
```

### Claude Code

`~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "myconfluence-r": {
      "command": "./myconfluence-mcp",
      "args": ["--transport", "stdio"],
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      }
    }
  }
}
```

### Claude Desktop

`~/.config/claude-desktop/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "myconfluence-r": {
      "command": ["bash", "-c", "./myconfluence-mcp"],
      "args": ["--transport", "stdio"],
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      }
    }
  }
}
```

### Codex CLI

`~/.codex/config.yaml`:

```yaml
mcp:
  servers:
    myconfluence-r:
      command: ./myconfluence-mcp
      args: ["--transport", "stdio"]
      env:
        MCP_UPSTREAM_ENDPOINT: https://example.atlassian.net/wiki/rest/api
        MCP_UPSTREAM_TOKEN: your-token
        MCP_UPSTREAM_TOKEN_FILE: "/path/to/fallback/.credentials"
```

### Cursor

`~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "myconfluence-r": {
      "command": "./myconfluence-mcp",
      "args": ["--transport", "stdio"],
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      }
    }
  }
}
```

### Remote Mode (HTTP)

Run the server separately and connect agents via HTTP transport. Suitable for shared instances, cloud deployments, or when agent cannot spawn local processes.

Start the server:

```sh
export MCP_UPSTREAM_ENDPOINT=https://example.atlassian.net/wiki/rest/api
export MCP_UPSTREAM_TOKEN=your-token
./myconfluence-mcp --transport http --port 8080 -v 1
```

### OpenCode (remote)

`~/.config/opencode/config.json`:

```json
{
  "mcp": {
    "myconfluence-r": {
      "type": "remote",
      "url": "http://localhost:8080/mcp",
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      }
    }
  }
}
```

### Claude Code (remote)

`~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "myconfluence-r": {
      "url": "http://localhost:8080/mcp",
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      }
    }
  }
}
```

### Claude Desktop (remote)

`~/.config/claude-desktop/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "myconfluence-r": {
      "url": "http://localhost:8080/mcp",
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      }
    }
  }
}
```

### Codex CLI (remote)

`~/.codex/config.yaml`:

```yaml
mcp:
  servers:
    myconfluence-r:
      url: http://localhost:8080/mcp
      env:
        MCP_UPSTREAM_ENDPOINT: https://example.atlassian.net/wiki/rest/api
        MCP_UPSTREAM_TOKEN: your-token
        MCP_UPSTREAM_TOKEN_FILE: "/path/to/fallback/.credentials"
```

### Cursor (remote)

`~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "myconfluence-r": {
      "url": "http://localhost:8080/mcp",
      "env": {
        "MCP_UPSTREAM_ENDPOINT": "https://example.atlassian.net/wiki/rest/api",
        "MCP_UPSTREAM_TOKEN": "your-token",
        "MCP_UPSTREAM_TOKEN_FILE": "/path/to/fallback/.credentials"
      }
    }
  }
}
```

## Generator CLI

```sh
./bin/mcpgen -i spec.yaml -o output-dir [--includes /path1,/path2] [--excludes /path3]
```

| Flag | Description |
|---|---|
| `-i, --input` | Path to the OpenAPI specification file (JSON or YAML) |
| `-o, --output` | Path to the output MCP server directory |
| `--includes` | Comma-separated OpenAPI paths to include (omit for all) |
| `--excludes` | Comma-separated OpenAPI paths to exclude |
| `--validation` | Enable OpenAPI validation |

### Path filtering

Use `--includes` and `--excludes` to control which API paths generate MCP tools. Paths match the OpenAPI path keys (e.g., `/wiki/rest/api/page`, `/space/{id}`). A path appearing in both flags triggers an error.

```sh
# Only generate tools for pages and spaces
./bin/mcpgen -i spec.yaml -o mymcp --includes "/wiki/rest/api/page,/wiki/rest/api/space"

# Generate all tools except health checks
./bin/mcpgen -i spec.yaml -o mymcp --excludes "/health,/status"
```

### Tool name truncation

Long `operationId` values are automatically truncated to 125 characters with a hash suffix to preserve uniqueness, ensuring compatibility with MCP tool name limits.

## Server CLI

| Flag | Description | Default |
|---|---|---|
| `--transport <stdio\|http>` | Transport mode | `stdio` |
| `--port <number>` | HTTP server port | `8080` |
| `-v, --verbose <0-10>` | Request logging verbosity | `0` |

### Logging levels

| Level | Output |
|---|---|
| `0` | Silent |
| `1` | HTTP access log: `[http] sid=- 200 POST /mcp (1ms)` |
| `2` | MCP request log: `[mcp] tool=SearchContent args={...}`, upstream method + URL |
| `3` | + upstream query params |
| `5` | + request/response headers |
| `7` | + request/response body |
| `9` | + pretty-printed JSON body |
| `10` | Same as 9 (full debug) |

### Environment variables

| Variable | Description |
|---|---|
| `MCP_UPSTREAM_ENDPOINT` | Base URL of the upstream API (default: `https://httpbin.org/anything`) |
| `MCP_UPSTREAM_TOKEN` | Bearer token for upstream auth (fallback when no Authorization header from client) |
| `MCP_UPSTREAM_TOKEN_FILE` | Path to a file containing the bearer token (alternative to `MCP_UPSTREAM_TOKEN`) |

### Token retrieval priority

The server tries to obtain a Bearer token in this order:

1. Authorization header from the client's HTTP request (forwarded)
2. `MCP_UPSTREAM_TOKEN` environment variable
3. `MCP_UPSTREAM_TOKEN_FILE` (read from file — ideal for Kubernetes secrets)
4. macOS Keychain (`security find-generic-password -s mcpgen-upstream -wa ""`)
5. Windows Credential Manager (`cmdkey /get:mcpgen-upstream`)

## License

[MIT License](LICENSE)

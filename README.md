# mcpgen: Transform OpenAPI APIs into AI Agent Tools

**mcpgen** is a command-line tool that generates production-ready Model Context Protocol (MCP) servers from OpenAPI specifications, exposing your existing APIs as tools for AI agents.

## Installation

```sh
go install github.com/lyeslabs/mcpgen/cmd/mcpgen@latest
```

## Quick Start

### 1. Prepare your OpenAPI spec

```yaml
# todoopenapi.yaml
openapi: 3.0.3
info:
  title: Todo API
  version: v1.0.0
servers:
  - url: https://api.example.com/v1
paths:
  /todos:
    get:
      operationId: listTodos
      summary: List all todo items
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, completed, in-progress]
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          description: A list of todo items.
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Todo'
    post:
      operationId: createTodo
      summary: Create a new todo item
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [title]
              properties:
                title:
                  type: string
                priority:
                  type: string
                  enum: [low, medium, high]
      responses:
        '201':
          description: Todo item created.
  /todos/{todoId}:
    get:
      operationId: getTodoById
      summary: Get a specific todo item
      parameters:
        - name: todoId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: The requested todo item.
    put:
      operationId: updateTodoById
      summary: Update an existing todo item
      parameters:
        - name: todoId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                title:
                  type: string
                status:
                  type: string
                  enum: [pending, in-progress, completed]
      responses:
        '200':
          description: Todo item updated.
    delete:
      operationId: deleteTodoById
      summary: Delete a todo item
      parameters:
        - name: todoId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '204':
          description: Todo item deleted.
components:
  schemas:
    Todo:
      type: object
      required: [id, title, status]
      properties:
        id:
          type: string
          format: uuid
        title:
          type: string
        status:
          type: string
          enum: [pending, in-progress, completed]
```

### 2. Generate the MCP server

```sh
mcpgen --input todoopenapi.yaml --output mytodoserver
cd mytodoserver
```

This produces a complete, buildable Go project:

```
mytodoserver/
├── main.go                      # entry point (stdio/http transport)
├── client.sh                    # quick curl-based test script
├── go.mod / go.sum              # auto-generated module
├── server                       # compiled binary
└── internal/
    ├── mcpserver/server.go      # MCP server setup + tool registration
    ├── helpers/
    │   ├── client.go            # ForwardRequest, parameter helpers
    │   └── request_log.go       # Request logging with verbosity levels
    └── mcptools/
        ├── CreateTodo.go        # per-tool input schema + handler
        ├── ListTodos.go
        ├── GetTodoById.go
        ├── UpdateTodoById.go
        └── DeleteTodoById.go
```

### 3. Start the server

```sh
# Point to your actual upstream API
UPSTREAM_ENDPOINT=https://api.example.com/v1 \
  ./server --transport http --port 8080
```

## Server Configuration

### CLI flags

| Flag | Description | Default |
|---|---|---|
| `--transport stdio\|http` | Transport mode | `stdio` |
| `--port <number>` | HTTP server port | `8080` |
| `--v <0-10>` | Request logging verbosity | `0` |

### Environment variables

| Variable | Description |
|---|---|
| `UPSTREAM_ENDPOINT` | Base URL of the upstream API to forward requests to |
| `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY` | Proxy configuration (uses `http.ProxyFromEnvironment`) |
| `NO_PROXY` | Hosts to exclude from proxy |

### Logging levels (`-v`)

| Level | Output |
|---|---|
| `0` | Silent (default) |
| `1` | nginx-style access log: `[http] 200 POST /mcp (1ms)` |
| `2` | + method + URL for upstream requests |
| `3-4` | + query parameters |
| `5-6` | + request/response headers |
| `7-8` | + request body |
| `9-10` | + pretty-printed JSON body |

```sh
# Quick — just one line per request
UPSTREAM_ENDPOINT=https://api.example.com/v1 ./server --transport http -v 1

# Full debug
UPSTREAM_ENDPOINT=https://api.example.com/v1 ./server --transport http --port 9090 -v 9
```

## Testing with client.sh

Every generated project includes a `client.sh` script for quick manual testing:

```sh
# Start the server
UPSTREAM_ENDPOINT=https://api.example.com/v1 ./server --transport http --port 8080

# In another terminal:
./client.sh                          # Show help with example commands
./client.sh list-tools              # List all available tools
./client.sh call ListTodos '{"limit": 20, "status": "pending"}'
./client.sh call CreateTodo '{"body": {"title": "Buy groceries", "priority": "medium"}}'
./client.sh call GetTodoById '{"todoId": "550e8400-e29b-41d4-a716-446655440000"}'

# Custom server URL
MCP_SERVER_URL=http://localhost:9090/mcp ./client.sh list-tools
```

> **Note:** If your environment has `ALL_PROXY`/`HTTP_PROXY` set, `client.sh` automatically uses `--noproxy '*'` to bypass the proxy for localhost connections.

## How It Works

`mcpgen` reads your OpenAPI spec and generates for each API operation:

1. **Input schema** — JSON Schema constant describing the tool's arguments
2. **Response templates** — Markdown documentation for LLM context
3. **Tool registration** — `NewFooMCPTool()` function
4. **Handler function** — `FooHandler()` that forwards requests to the upstream API via `ForwardRequest()`

You focus on customizing handler logic; `mcpgen` handles all the MCP boilerplate.

## License

[MIT License](LICENSE)

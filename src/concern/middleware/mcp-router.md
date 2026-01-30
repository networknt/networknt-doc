# MCP Router

The MCP (Model Context Protocol) Router in `light-4j` allows binding LLM tools to HTTP endpoints, enabling an AI model to interact with your services via the MCP standard. It supports both Server-Sent Events (SSE) for connection establishment and HTTP POST for JSON-RPC messages.

## Configuration

The configuration for the MCP Router is defined in `mcp-router.yml` (or `mcp-router.json/yaml`). It corresponds to the `McpConfig.java` class.

### Configuration Properties

| Property | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | boolean | `true` | Enable or disable the MCP Router Handler. |
| `ssePath` | string | `/mcp/sse` | The endpoint path for establishing the MCP Server-Sent Events (SSE) connection. |
| `messagePath` | string | `/mcp/message` | The endpoint path for sending MCP JSON-RPC messages (POST). |
| `tools` | list | `null` | A list of tools exposed by this router. Each tool maps to a downstream HTTP service. |

### Configuration Example (mcp-router.yml)

```yaml
# Enable MCP Router Handler
enabled: true

# Path for MCP Server-Sent Events (SSE) endpoint
ssePath: /mcp/sse

# Path for MCP JSON-RPC message endpoint
messagePath: /mcp/message

# Define tools exposed by this router
tools:
  - name: getWeather
    description: Get the weather for a location
    host: https://api.weather.com
    path: /v1/current
    method: GET
    # inputSchema is optional; default is type: object
```

## Architecture

The MCP Router implements the MCP HTTP transport specification:

1.  **Connection**: Clients connect to the `ssePath` (default `/mcp/sse`) via GET to establish an SSE session. The server responds with an endpoint event containing the URI for message submission.
2.  **Messaging**: Clients send JSON-RPC 2.0 messages to the `messagePath` (default `/mcp/message`) via POST.
3.  **Tool Execution**: The router translates `tools/call` requests into HTTP requests to the configured downstream services (`host` + `path`).

## Supported Methods

The router supports the following MCP JSON-RPC methods:

*   `initialize`: Returns server capabilities (tools list changed notification) and server info.
*   `notifications/initialized`: Acknowledgment from the client.
*   `tools/list`: Lists all configured tools with their names, descriptions, and input schemas.
*   `tools/call`: Executes a specific tool by name. The `arguments` in the request are passed to the downstream service.

## Usage

Register the `McpHandler` in your `handler.yml` chain. Ensure the `ssePath` and `messagePath` are accessible.

Example 
```yaml
paths:
  - path: '/mcp/sse'
    method: 'GET'
    exec:
      - mcp-router
  - path: '/mcp/message'
    method: 'POST'
    exec:
      - mcp-router
```

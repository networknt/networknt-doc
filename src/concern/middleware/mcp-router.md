# MCP Router

The MCP (Model Context Protocol) Router in `light-4j` allows binding LLM tools to HTTP endpoints, enabling an AI model to interact with your services via the MCP standard. It supports both Server-Sent Events (SSE) for connection establishment and HTTP POST for JSON-RPC messages.

## Configuration

The configuration for the MCP Router is defined in `mcp-router.yml` (or `mcp-router.json/yaml`). It corresponds to the `McpConfig.java` class.

### Configuration Properties

| Property | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | boolean | `true` | Enable or disable the MCP Router Handler. |
| `path` | string | `/mcp` | The unified endpoint path for both connection establishment (SSE via GET) and JSON-RPC messages (POST). |
| `tools` | list | `null` | A list of tools exposed by this router. Each tool maps to a downstream HTTP service. |

### Configuration Example (mcp-router.yml)

```yaml
# Enable MCP Router Handler
enabled: true

# Path for MCP endpoint (Streamable HTTP)
path: /mcp

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

The MCP Router implements the MCP HTTP transport specification using a **Single Path** architecture:

1.  **Connection**: Clients connect to the configured `path` (default `/mcp`) via **GET** to establish an SSE session. The server generates a unique session ID and responds with an `endpoint` event containing the URI for message submission (e.g., `/mcp?sessionId=uuid`).
2.  **Messaging**: Clients send JSON-RPC 2.0 messages to the same `path` (default `/mcp`) via **POST**. The session context is maintained via the `sessionId` query parameter or header.
3.  **Tool Execution**: The router translates `tools/call` requests into HTTP requests to the configured downstream services (`host` + `path`).

## Supported Methods

The router supports the following MCP JSON-RPC methods:

*   `initialize`: Returns server capabilities (tools list changed notification) and server info.
*   `notifications/initialized`: Acknowledgment from the client.
*   `tools/list`: Lists all configured tools with their names, descriptions, and input schemas.
*   `tools/call`: Executes a specific tool by name. The `arguments` in the request are passed to the downstream service.

## Usage

Register the `McpHandler` in your `handler.yml` chain.

Example:
```yaml
paths:
  - path: '/mcp'
    method: 'GET'
    exec:
      - mcp-router
  - path: '/mcp'
    method: 'POST'
    exec:
      - mcp-router
```

## Verification with verify_mcp.py

To assist with testing and verification, a Python script `verify_mcp.py` is available in the `light-gateway` root directory (and potentially other project roots). This script simulates an MCP client to ensure the router is correctly configured and routing requests.

### Prerequisite

Install the required Python packages:

```bash
pip install requests sseclient-py
```

### Running the Script

Run the script against your running gateway instance (default `https://localhost:8443`):

```bash
python3 verify_mcp.py
```

### What it does

1.  **Connects to SSE**: Initiates a GET request to `/mcp` and listens for server sent events. It confirms receipt of the `endpoint` event.
2.  **Initializes**: Sends a POST request with the `initialize` JSON-RPC method to verify protocol negotiation.
3.  **Lists Tools**: Sends a `tools/list` request to verify that configured tools are being correctly exposed by the router.

If successful, you will see output confirming the SSE connection, the endpoint event, and the JSON responses for initialization and tool listing.

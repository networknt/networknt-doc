# MCP Audit Logging

The `light-4j-mcp` framework extends the standard Light-4j `AuditHandler` to natively support capturing fine-grained Model Context Protocol (MCP) data. 

Because standard MCP traffic is structured as JSON-RPC over HTTP/SSE, simply logging the raw `requestBody` and `responseBody` would result in bulky and noisy logs (capturing boilerplate messages like `initialize`, `notifications/initialized`, and ping events). Instead, the `McpHandler` intelligently bridges the gap by intercepting specific tool execution calls and forwarding only the most relevant operational data to the `AuditHandler`.

## How It Works

When a client makes a JSON-RPC request to execute an MCP tool (using the `tools/call` method):
1. The `McpHandler` identifies the tool call and parses its payload.
2. It extracts the tool name and argument parameters.
3. Once the tool finishes executing (or if it encounters an error, missing tool, or access-control denial), the `McpHandler` captures the final output.
4. It injects this processed data into the `AUDIT_INFO` metadata attached to the HTTP exchange.
5. Finally, the downstream `AuditHandler` seamlessly formats and outputs these fields into your centralized `audit.log`.

## Configuration

To enable MCP auditing, you need to configure your handler chain and instruct the `AuditHandler` to output the new fields.

### 1. Update `audit.yml`

In your `audit.yml` (or via externalized `values.yml`), augment the `audit` array to include the three new MCP-specific keys: `toolName`, `toolArguments`, and `toolResult`.

```yaml
# audit.yml
audit.audit:
  - client_id
  - user_id
  - endpoint
  - serviceId
  - toolName
  - toolArguments
  - toolResult
```

*(Note: The latest `audit-config` module provides these defaults automatically, but you may need to explicitly add them if overriding via `values.yml` or if using custom templates).*

### 2. Update `handler.yml`

Ensure that the `AuditHandler` is placed *before* the `McpHandler` in the execution chain for your MCP endpoints. This allows the standard audit timer to start properly and attach the completion listener that ultimately writes the log to disk.

```yaml
# handler.yml
paths:
  - path: '/ctrl/mcp'
    method: 'post'
    exec:
      - audit
      - mcp
```

## Example Log Output

When a tool is executed, the `AuditHandler` will combine standard metadata with the custom fields and output a structured JSON log entry similar to this:

```json
{
  "timestamp": "2026-04-11T12:00:00.123-0400",
  "serviceId": "com.networknt.mcp-router-1.0.0",
  "X-Correlation-Id": "c4ca4238a0b923820dcc509a6f75849b",
  "endpoint": "/ctrl/mcp@post",
  "toolName": "search_database",
  "toolArguments": "{\"query\":\"select * from users\",\"limit\":10}",
  "toolResult": "{\"status\":\"success\",\"rows\":10}",
  "statusCode": 200,
  "responseTime": 105
}
```

## Security & Data Masking

Because `toolArguments` and `toolResult` are seamlessly passed to the standard `AuditHandler`, they natively inherit Light-4j's built-in robust mask capabilities. If masking is enabled (`audit.mask: true`), any sensitive data passed through the MCP payloads will be sanitized according to your standard security configurations before being written to disk.

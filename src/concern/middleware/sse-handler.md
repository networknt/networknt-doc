# SSE Handler

The SSE (Server-Sent Events) Handler in `light-4j` allows the server to push real-time updates to the client over a standard HTTP connection. This handler manages the connection lifecycle and keep-alive messages.

## Configuration

The configuration for the SSE Handler is defined in `sse.yml` (or `sse.json/yaml`). It corresponds to the `SseConfig.java` class.

### Configuration Properties

| Property | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | boolean | `true` | Enable or disable the SSE Handler. |
| `path` | string | `/sse` | The default endpoint path for the SSE Handler. |
| `keepAliveInterval` | int | `10000` | The keep-alive interval in milliseconds. Used for the default path or if no specific interval is defined for a prefix. |
| `pathPrefixes` | list | `null` | A list of path prefixes with specific keep-alive intervals. See example below. |
| `metricsInjection` | boolean | `false` | If true, injects metrics for the downstream API response time (useful in sidecar/gateway modes). |
| `metricsName` | string | `router-response` | The name used for metrics categorization if injection is enabled. |

### Configuration Example (sse.yml)

```yaml
# Enable SSE Handler
enabled: true

# Default SSE Endpoint Path
path: /sse

# Default keep-alive interval in milliseconds
keepAliveInterval: 10000

# Define path prefix related configuration properties as a list of key/value pairs.
# If request path cannot match to one of the pathPrefixes or the default path, the request will be skipped.
pathPrefixes:
  - pathPrefix: /sse/abc
    keepAliveInterval: 20000
  - pathPrefix: /sse/def
    keepAliveInterval: 40000

# Metrics injection (optional, for gateway/sidecar usage)
metricsInjection: false
metricsName: router-response
```

## Usage

Register the `SseHandler` in your `handler.yml` chain. When a client connects to the configured path (e.g., `/sse`), the handler will establish a persistent connection and manage keep-alive signals based on the `keepAliveInterval`.

### Path Matching

1.  **Prefix Matching**: If `pathPrefixes` are configured, the handler checks if the request path starts with any of the defined prefixes. If a match is found, the specific `keepAliveInterval` for that prefix is used.
2.  **Exact Matching**: If no prefix matches, the handler checks if the request path exactly matches the default `path` (`/sse`).
3.  **Passthrough**: If neither matches, the request is passed to the next handler in the chain.

### Traceability

The handler supports traceability via the `X-Traceability-Id` header or an `id` query parameter to track connections in the `SseConnectionRegistry`.

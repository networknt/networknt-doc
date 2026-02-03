# External Service Handler

The External Service Handler (`ExternalServiceHandler`) enables the gateway or service to access third-party services, potentially through a corporate proxy (forward proxy).

## Introduction

In enterprise environments, internal services often need to call external APIs (e.g., Salesforce, Slack) but are restricted by firewalls and MUST go through a corporate web gateway (like McAfee Web Gateway). The default `RouterHandler` or `ProxyHandler` may not support this specific forward proxy configuration easily or may be too heavy-weight.

The `ExternalServiceHandler` uses the JDK 11+ `HttpClient` to make these calls. It supports:
*   Configuring a forward proxy (host and port).
*   Routing requests based on path prefixes to different external hosts.
*   URL rewriting.
*   Custom timeouts per path.
*   Connection retries.

## Configuration

The handler is configured via `external-service.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `false` |
| `proxyHost` | The hostname of the corporate proxy/gateway. | `null` |
| `proxyPort` | The port of the corporate proxy. | `443` |
| `connectTimeout` | Connection timeout in milliseconds. | `3000` |
| `timeout` | Request timeout in milliseconds. | `5000` |
| `enableHttp2` | Use HTTP/2 for the external connection. | `false` |
| `pathPrefixes` | A list of objects defining path-to-host mapping and specific timeouts. | `[]` |
| `pathHostMappings` | Only legacy simple mapping. Use `pathPrefixes` instead. | `null` |
| `urlRewriteRules` | Regex rules to rewrite the URL path before sending to upstream. | `[]` |

### `external-service.yml` Example

```yaml
enabled: true
# Corporate Proxy Settings
proxyHost: proxy.corp.example.com
proxyPort: 8080

# Global Timeouts
connectTimeout: 2000
timeout: 5000

# Mapping Paths to External Hosts
pathPrefixes:
  - pathPrefix: /sharepoint
    host: https://sharepoint.microsoft.com
    timeout: 10000
  - pathPrefix: /salesforce
    host: https://api.salesforce.com
    timeout: 5000
  - pathPrefix: /openai
    host: https://api.openai.com

# URL Rewriting (Optional)
urlRewriteRules:
  - /openai/(.*) /$1
```

## Features

### Forward Proxy Support
If `proxyHost` and `proxyPort` are configured, all outgoing requests will be routed through this proxy. This is essential for accessing the internet from within a DMZ or secure corporate network.

### Path-Based Routing
The handler intercepts requests matching a configured prefix (e.g., `/sharepoint`) and routes them to the corresponding target host (e.g., `https://sharepoint.microsoft.com`).

*   **pathPrefixes**: The recommended config, allowing specific timeouts per route.
*   **pathHostMappings**: Simple space-separated string mappings (Legacy).

### URL Rewriting
You can rewrite the path before it is sent to the target. For example, stripping the `/openai` prefix so that a request to `/openai/v1/chat` becomes `/v1/chat` at the target `https://api.openai.com`.

### Metrics Injection
Like the Proxy Handler, this handler can inject metrics into the `MetricsHandler` to track the latency of external calls separately from internal processing.

### Body Handling
It supports buffering request bodies for POST, PUT, and PATCH methods to forward them to the external service.

## Usage

1.  Register the handler in `handler.yml`.

    ```yaml
    handlers:
      - com.networknt.proxy.ExternalServiceHandler@external
    ```

2.  Add it to the default chain or specific paths. Since it works based on path prefixes, it is often placed in the default chain *before* the router or other terminal handlers, so it can intercept specific traffic.

    ```yaml
    chains:
      default:
        - exception
        - metrics
        - header
        - external  # <--- Intercepts configured paths (e.g. /sharepoint)
        - prefix
        - router
    ```

    If the request path matches one of the `pathPrefixes`, the `ExternalServiceHandler` takes over, makes the remote call, and ends the exchange (it acts as a terminal handler for those requests). If the path does not match, it calls `Handler.next(exchange, next)` to pass the request down the chain.

## Limitations

*   It acts as a simple pass-through. Complex logic (aggregation, transformation) should be handled by an orchestrator or specialized handler.
*   It creates a new `HttpRequest` for each call but reuses the `HttpClient`.

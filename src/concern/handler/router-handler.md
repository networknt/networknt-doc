# Router Handler

The `RouterHandler` is a key component in the [Light-Router](https://github.com/networknt/light-router) and [http-sidecar](https://github.com/networknt/http-sidecar) services. It acts as a reverse proxy request handler, forwarding incoming requests to downstream services based on service discovery and routing rules.

## Introduction

In a microservices architecture, a gateway or sidecar often needs to route requests to various backend services. The `RouterHandler` fulfills this role by:

1.  **Proxying**: It uses a `LoadBalancingRouterProxyClient` to forward requests.
2.  **Protocol Support**: It supports HTTP/1.1 and HTTP/2, with optional TLS (HTTPS) for downstream connections.
3.  ** rewriting**: It offers powerful capabilities to rewrite URLs, methods, headers, and query parameters before forwarding the request.
4.  **Resilience**: It handles connection pooling, retries, and timeouts (both global and path-specific).

## Configuration

The handler is configured via the `router.yml` file. This configuration file allows you to fine-tune the connection behavior and define rewrite rules.

### Key Configuration Options

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `http2Enabled` | Use HTTP/2 for downstream connections. | `true` |
| `httpsEnabled` | Use TLS (HTTPS) for downstream connections. | `true` |
| `maxRequestTime` | Global timeout (in ms) for downstream requests. | `1000` |
| `rewriteHostHeader` | Rewrite the `Host` header to match the target service. | `true` |
| `connectionsPerThread` | Number of cached connections per thread in the pool. | `10` |
| `maxConnectionRetries` | Number of times to retry a failed connection. | `3` |
| `hostWhitelist` | List of allowed target hosts (supports regex). | `[]` |

### Path-Specific Timeouts

You can override the global `maxRequestTime` for specific paths using `pathPrefixMaxRequestTime`.

```yaml
pathPrefixMaxRequestTime:
  /v1/heavy-processing: 5000
  /v2/report: 10000
```

### Rewrite Rules

The router supports extensive rewrite rules to adapt legacy clients or unify API surfaces.

**URL Rewrite**
Rewrite specific paths using Regex.
```yaml
urlRewriteRules:
  # Regex Pattern -> Replacement
  - /listings/(.*)$ /listing.html?listing=$1
```

**Method Rewrite**
Convert methods (e.g., for clients that don't support DELETE/PUT).
```yaml
methodRewriteRules:
  # Endpoint Pattern -> Source Method -> Target Method
  - /v1/pets/{petId} GET DELETE
```

**Header & Query Param Rewrite**
Rename keys or replace values for headers and query parameters.
```yaml
headerRewriteRules:
  /v1/old-api:
    - oldK: X-Old-Header
      newK: X-New-Header

queryParamRewriteRules:
  /v1/search:
    - oldK: q
      newK: query
```

### Example `router.yml`

```yaml
# Router Configuration
http2Enabled: true
httpsEnabled: true
maxRequestTime: 2000
rewriteHostHeader: true
reuseXForwarded: false
maxConnectionRetries: 3
connectionsPerThread: 20

# Host Whitelist (Regex)
hostWhitelist:
  - 192\.168\..*
  - networknt\.com

# Metrics
metricsInjection: true
metricsName: router-response
```

## Usage

The `RouterHandler` is typically used as the terminal handler in a gateway or sidecar chain. It effectively "exits" the Light-4j processing chain and hands the request over to the external service.

### `handler.yml`

```yaml
handlers:
  - com.networknt.router.RouterHandler@router
  # ... other handlers

chains:
  default:
    - correlation
    - limit
    - router # The request is forwarded here
```

## Metrics Injection

If `metricsInjection` is enabled, the router can inject response time metrics into the configured Metrics Handler. This helps in tracking the latency introduced by downstream services separately from the gateway's own processing time.

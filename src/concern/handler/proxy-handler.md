# Proxy Handler

The Proxy Handler (`LightProxyHandler`) is a core component of the Light-Gateway and Sidecar patterns. It acts as a reverse proxy, forwarding incoming requests to backend services (downstream APIs) and returning the responses to the client.

## Introduction

In a microservices architecture, a reverse proxy is often needed to route traffic from the internet to internal services, or to act as a sidecar handling cross-cutting concerns (security, metrics, etc.) for a specific service instance. The `LightProxyHandler` is built on top of Undertow's `ProxyHandler` and provides features like load balancing, connection pooling, and protocol translation (HTTP/1.1 to HTTP/2).

## Configuration

The handler is configured via `proxy.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the proxy handler. | `true` |
| `hosts` | Comma-separated list of downstream target URIs (e.g., `http://localhost:8080,https://api.example.com`). | `http://localhost:8080` |
| `http2Enabled` | Use HTTP/2 to connect to the backend. Requires all targets to support HTTPS and HTTP/2. | `false` |
| `connectionsPerThread` | Number of connections per IO thread to the target server. | `20` |
| `maxRequestTime` | Timeout in milliseconds for the proxy request. | `1000` |
| `rewriteHostHeader` | Rewrite the `Host` header to the target's host. | `true` |
| `forwardJwtClaims` | If `true`, decodes the JWT and injects claims as a JSON string header `jwtClaims` to the backend. | `false` |
| `metricsInjection` | If `true`, injects proxy response time metrics into the `MetricsHandler`. | `false` |

### `proxy.yml` Example

```yaml
enabled: true
http2Enabled: false
hosts: http://localhost:8081,http://localhost:8082
connectionsPerThread: 20
maxRequestTime: 5000
rewriteHostHeader: true
reuseXForwarded: false
maxConnectionRetries: 3
maxQueueSize: 0
forwardJwtClaims: true
metricsInjection: true
metricsName: proxy-response
```

## Features

### Load Balancing
The handler supports client-side load balancing if multiple URLs are provided in `hosts`. It uses a round-robin strategy by default via `LoadBalancingProxyClient`.

### Protocol Support
*   **HTTP/1.1**: Default behavior.
*   **HTTP/2**: Enable `http2Enabled` to communicate with backends via HTTP/2 (requires HTTPS).
*   **HTTPS**: Supports invalid certificate verification (for internal self-signed certs) if configured in `client.yml` (TLS verification disabled).

### JWT Claims Forwarding
If `forwardJwtClaims` is enabled, the handler extracts the JWT from the `Authorization` header, decodes the claims (without signature verification, assuming it was verified by `SecurityHandler` previously), and injects them as a JSON header `jwtClaims` to the downstream service. This is useful for backend services that need user context but don't want to re-verify or parse JWTs.

### Metrics Injection
The handler can measure the time taken for the downstream call (including network latency) and inject this data into the `MetricsHandler`. This allows distinguishing between the gateway's overhead and the backend's processing time.

## Usage

In `handler.yml`, register the `LightProxyHandler`.

```yaml
handlers:
  - com.networknt.proxy.LightProxyHandler@proxy
```

Then use it in paths, typically as the terminal handler.

```yaml
paths:
  - path: '/v1/pets'
    method: 'get'
    exec:
      - security
      - metrics
      - proxy
```

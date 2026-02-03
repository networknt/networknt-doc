# Health Check Handler

The Health Check Handler (`HealthGetHandler`) manages the server's health endpoint. It is critical for load balancers (like F5, AWS ALB), container orchestrators (Kubernetes Liveness/Readiness probes), and the Light Control Plane to verify that the service is running and capable of handling requests.

## Introduction

A running process does not always mean a functioning service. The `HealthGetHandler` provides a lightweight endpoint to confirm application status. It can return a simple "OK" string or a JSON object, and optionally verify downstream dependencies before reporting healthy.

## Configuration

The handler is configured via `health.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the health check. | `true` |
| `useJson` | If `true`, returns `{"result": "OK"}`. If `false`, returns plain `OK`. | `false` |
| `timeout` | Timeout in milliseconds for the check (important when checking downstream). | `2000` |
| `downstreamEnabled` | If `true`, verify a downstream service before returning OK. | `false` |
| `downstreamHost` | URL of the downstream service (e.g., `http://localhost:8081` for a sidecar). | `http://localhost:8081` |
| `downstreamPath` | Path of the downstream health check. | `/health` |

### `health.yml` Example

```yaml
enabled: true
useJson: true
timeout: 500
# For services needing to verify a sidecar (e.g. light-proxy, kafka-sidecar)
downstreamEnabled: false
downstreamHost: http://localhost:8081
downstreamPath: /health
```

## Usage

To use the health handler, register it in your `handler.yml` and map it to a path (typically `/health`).

### `handler.yml` Configuration

1.  **Register the Handler**:
    ```yaml
    handlers:
      - com.networknt.health.HealthGetHandler@health
    ```

2.  **Define the Path**:
    ```yaml
    paths:
      - path: '/health'
        method: 'get'
        exec:
          - health
      # Secure endpoint for Control Plane (optional)
      - path: '/adm/health/${server.serviceId}'
        method: 'get'
        exec:
          - security
          - health
    ```

### Scenarios

1.  **Kubernetes Probes**: Configure `livenessProbe` and `readinessProbe` in your `deployment.yaml` to hit `/health`.
2.  **Load Balancers**: Configure the health monitor to check `/health` and expect "OK" (or JSON if configured).
3.  **Control Plane**: The Control Plane uses a secure path (e.g., `/adm/health/...`) with service discovery to monitor specific instances.

## Downstream Check

If your service relies heavily on a sidecar (like `http-sidecar` or `kafka-sidecar`) or a specific downstream API, you can enable `downstreamEnabled`.
*   The handler will make a request to `downstreamHost` + `downstreamPath`.
*   If the downstream call fails or times out, the health check returns an error, preventing traffic from being routed to this broken instance.

**Note**: For complex dependency checking (Database, Redis, etc.), it is recommended to extend this handler or implement a custom `HealthCheck` interface if using a specific framework, though `HealthGetHandler` is the standard entry point for Light-4j.

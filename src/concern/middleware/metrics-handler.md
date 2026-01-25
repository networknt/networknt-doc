# Metrics Handler

The metrics handler collects API runtime information and reports it periodically to a time-series database. This module is essential for monitoring performance, troubleshooting production issues, and understanding usage patterns in a microservices environment.

### Introduction

The metrics handler captures performance data during the request/response lifecycle. It reports this information to a remote server using a "push" model. Currently, the module supports two primary implementations:
*   **MetricsHandler**: Reports to InfluxDB.
*   **APMMetricsHandler**: Reports to Broadcom APM.

Additionally, some metrics servers like [Prometheus](/concern/prometheus/) pull data from the framework, while Grafana is typically used to visualize the collected data from two perspectives:
1.  **Client-oriented**: Shows API usage and runtime info from the client's perspective.
2.  **API-oriented**: Shows which clients are calling a specific API and their performance characteristics.

### Configuration (metrics.yml)

The behavior of all push-based metrics handlers is controlled via `metrics.yml`.

```yaml
# Metrics handler configuration shared by all push metrics handlers.
---
# Enable or disable the metrics handler.
enabled: ${metrics.enabled:true}

# Enable JVM MBean monitoring (CPU and Memory usage).
enableJVMMonitor: ${metrics.enableJVMMonitor:false}

# Time series database server protocol (http or https).
serverProtocol: ${metrics.serverProtocol:http}

# Database or metrics server hostname.
serverHost: ${metrics.serverHost:localhost}

# Database port number (e.g., 8086 for InfluxDB).
serverPort: ${metrics.serverPort:8086}

# Metrics server request path (optional, used by Broadcom APM).
serverPath: ${metrics.serverPath:/apm/metricFeed}

# Time series database name.
serverName: ${metrics.serverName:metrics}

# Database user.
serverUser: ${metrics.serverUser:admin}

# Database password.
serverPass: ${metrics.serverPass:admin}

# Reporting interval and reset period in minutes.
reportInMinutes: ${metrics.reportInMinutes:1}

# Category name for centralized database (e.g., http-sidecar, light-gateway).
productName: ${metrics.productName:http-sidecar}

# Optional tags for granular reporting
sendScopeClientId: ${metrics.sendScopeClientId:false}
sendCallerId: ${metrics.sendCallerId:false}
sendIssuer: ${metrics.sendIssuer:false}
issuerRegex: ${metrics.issuerRegex:}
```

### Setup

To enable metrics, add the handler to your `handler.yml` file and include it in the default chain. It is usually placed early in the chain to capture the full request latency.

```yaml
handlers:
  - com.networknt.metrics.MetricsHandler@metrics

chains:
  default:
    - exception
    - correlation
    - metrics
    - ...
```

### Metrics Injection

In addition to standard request/response metrics, the following handlers can inject metrics for downstream API calls:
*   **egress-router**: Enabled via `metricsInjection` flag in `router.yml`.
*   **ingress-proxy**: Enabled via `metricsInjection` flag in `proxy.yml`.
*   **lambda-invoker**: Enabled via `metricsInjection` flag in `lambda-invoker.yml`.

### Visualizing Metrics

A common stack for visualization is InfluxDB combined with Grafana. Below is an example `docker-compose.yml` to start both services:

```yaml
influxdb:
  image: influxdb:latest
  container_name: influxdb
  ports:
    - "8083:8083"
    - "8086:8086"

grafana:
  image: grafana/grafana:latest
  container_name: grafana
  ports:
    - "3000:3000"
  links:
    - influxdb
```

### Auto-Registration

The `metrics-config` module automatically registers itself with the `ModuleRegistry` when `MetricsConfig.load()` is called. These settings, including masked passwords, can be verified via the [Server Info](/concern/admin/server-info.md) endpoint.

# Prometheus Metrics

The `prometheus` module in `light-4j` provides a systems monitoring middleware handler that integrates with [Prometheus](https://prometheus.io/). Unlike other metrics modules that "push" data to a central server (e.g., InfluxDB), Prometheus adopts a **pull-based** model where the Prometheus server periodically scrapes metrics from instrumented targets.

### Features

*   **Runtime Collection**: Captures HTTP request counts and response times using the Prometheus dimensional data model.
*   **Dimensionality**: Automatically attaches labels like `endpoint` and `clientId` to every metric, allowing for granular filtering and aggregation in Grafana.
*   **JVM Hotspot Monitoring**: Optional collection of JVM-level metrics including CPU usage, memory pools, garage collection, and thread states.
*   **Standardized Scrape Endpoint**: Provides a dedicated handler to expose metrics in the standard Prometheus text format.
*   **Auto-Registration**: The module automatically registers its configuration with the `ModuleRegistry` during initialization.

### Configuration (prometheus.yml)

The behavior of the Prometheus middleware is controlled by the `prometheus.yml` file.

```yaml
# Prometheus Metrics Configuration
---
# If metrics handler is enabled or not. Default is false.
enabled: ${prometheus.enabled:false}

# If the Prometheus hotspot monitor is enabled or not. 
# includes thread, memory, classloader statistics, etc.
enableHotspot: ${prometheus.enableHotspot:false}
```

### Setup

To enable Prometheus metrics, you need to add two handlers to your `handler.yml`: one to collect the data and another to expose it as an endpoint for scraping.

#### 1. Add Handlers to handler.yml

```yaml
handlers:
  # Captures metrics for every request
  - com.networknt.metrics.prometheus.PrometheusHandler@prometheus
  # Exposes the metrics to the Prometheus server
  - com.networknt.metrics.prometheus.PrometheusGetHandler@getprometheus

chains:
  default:
    - exception
    - prometheus  # Place early in the chain
    - correlation
    - ...
```

#### 2. Define the Scrape Path

You must expose a `GET` endpoint (typically `/v1/prometheus` or `/metrics`) that invoke the `getprometheus` handler.

```yaml
paths:
  - path: '/v1/prometheus'
    method: 'get'
    exec:
      - getprometheus
```

### Collected Metrics

The `PrometheusHandler` defines several core application metrics:

*   **requests_total**: Total number of HTTP requests received.
*   **success_total**: Total number of successful requests (status codes 200-399).
*   **auth_error_total**: Total number of authentication/authorization errors (status codes 401, 403).
*   **request_error_total**: Total number of client-side request errors (status codes 400-499).
*   **server_error_total**: Total number of server-side errors (status codes 500+).
*   **response_time_seconds**: A summary of HTTP response latencies.

### JVM Hotspot Monitoring

When `enableHotspot` is set to `true`, the module initializes the Prometheus `DefaultExports`, which captures:

*   **Process Statistics**: `process_cpu_seconds_total`, `process_open_fds`.
*   **Memory Usage**: `jvm_memory_pool_bytes_used`, `jvm_memory_pool_bytes_max`.
*   **Thread Areas**: `jvm_threads_state`, `jvm_threads_deadlocked`.
*   **Garbage Collection**: `jvm_gc_collection_seconds`.

### Scraping with Prometheus

To pull data from your service, configure your Prometheus server's `scrape_configs` in its configuration file:

```yaml
scrape_configs:
  - job_name: 'light-4j-services'
    scrape_interval: 15s
    metrics_path: /v1/prometheus
    static_configs:
      - targets: ['localhost:8080']
    scheme: https
    tls_config:
      insecure_skip_verify: true
```

### Dependency

Add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>prometheus</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

### Module Registration

The `prometheus` module registers itself with the `ModuleRegistry` automatically when `PrometheusConfig.load()` is called. This registration (along with the current configuration values) can be inspected via the [Server Info](/concern/admin/server-info.md) endpoint.

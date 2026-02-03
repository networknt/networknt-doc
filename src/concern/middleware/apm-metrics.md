# APM Metrics Handler

The APM Metrics Handler (`APMMetricsHandler`) is a specialized metrics handler designed to integrate with Broadcom APM (formerly CA APM). It collects application metrics and sends them to the APM EPAgent (Enterprise Performance Agent) via its RESTful interface.

## Introduction

Many enterprise customers use Broadcom APM for monitoring and performance management. This handler allows light-4j services (microsrevices, gateways, sidecars) to push metrics directly to an EPAgent deployed as a sidecar or a common service in the Kubernetes cluster or on the VM host.

It extends the [AbstractMetricsHandler](TODO:link_to_abstract_metrics) and shares the same core logic for collecting request/response metrics, but implements a specific sender for the APM protocol.

## Configuration

The handler uses the `metrics.yml` configuration file, which is shared by all push-based metrics handlers.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the metrics handler. | `true` |
| `enableJVMMonitor`| Enable JVM metrics collection (CPU, Memory). | `false` |
| `serverProtocol` | Protocol for the EPAgent (http or https). | `http` |
| `serverHost` | Hostname of the EPAgent. | `localhost` |
| `serverPort` | Port of the EPAgent. | `8086` |
| `serverPath` | REST path for the EPAgent metric feed. | `/apm/metricFeed` |
| `productName` | The product/service name used as a category in APM. | `http-sidecar` |
| `reportInMinutes` | Interval in minutes to report metrics. | `1` |

### `metrics.yml` Example

```yaml
enabled: true
enableJVMMonitor: true
serverProtocol: http
# Hostname of the Broadcom APM EPAgent service
serverHost: opentracing.ccaapm.svc.cluster.local
serverPort: 8888
# Default path for EPAgent REST interface
serverPath: /apm/metricFeed
# Name of your service/component in APM
productName: my-service-name
reportInMinutes: 1
```

## Usage

### 1. Register the Handler

In `handler.yml`, you must register the `APMMetricsHandler` specifically. Since there can only be one metrics handler active in the chain usually, you should use the alias `metrics`.

```yaml
handlers:
  - com.networknt.metrics.APMMetricsHandler@metrics
```

### 2. Configure the Chain

Add the `metrics` alias to your default chain. It should be placed early in the chain to capture the full duration of requests.

```yaml
chains:
  default:
    - exception
    - metrics
    - header
    # ... other handlers
```

### 3. Update values.yml

In your deployment's `values.yml`, override the default `metrics.yml` values to point to your actual EPAgent.

```yaml
metrics.serverHost: epagent.monitoring.svc
metrics.serverPort: 8080
metrics.productName: payment-service
```

## Metrics Collected

The handler collects the following metrics:
*   **Response Time**: Duration of the request.
*   **Success Count**: Number of successful requests (2xx).
*   **Error Counts**: 4xx (request errors) and 5xx (server errors).
*   **JVM Metrics**: (Optional) Heap usage, Thread count, CPU usage.

## Tags

Common tags injected into metrics:
*   `api`: Service ID.
*   `env`: Environment (dev, test, prod).
*   `host`: Hostname / Container ID.
*   `clientId`: Calling client ID (if available).
*   `endpoint`: The API endpoint being accessed.

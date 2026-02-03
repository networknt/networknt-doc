# Correlation Handler

The `CorrelationHandler` is a middleware handler designed to ensure that every request contains a unique correlation ID (`X-Correlation-Id`). This ID is propagated across service-to-service calls, allowing for centralized logging and traceability in a microservices architecture.

## Introduction

In a distributed system, a single user request often triggers a chain of calls across multiple services. To debug issues or trace the flow of execution, it is critical to have a unique identifier that ties all these logs together. The `CorrelationHandler` ensures this mechanism by:

1.  **Checking**: Looking for an existing `X-Correlation-Id` header in the incoming request.
2.  **Generating**: If the header is missing and configuration allows, generating a new UUID for the request.
3.  **Logging**: Injecting the Correlation ID (and optional Traceability ID) into the SLF4J MDC (Mapped Diagnostic Context) so it appears in specific log patterns.

## Configuration

The handler is configured via the `correlation.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Indicates if the correlation middleware is enabled. | `true` |
| `autogenCorrelationID` | If `true`, the handler generates a new Correlation ID if missing in the request. | `true` |
| `correlationMdcField` | The key used in the MDC context for the Correlation ID. | `cId` |
| `traceabilityMdcField` | The key used in the MDC context for the Traceability ID. | `tId` |

### Example `correlation.yml`

```yaml
# Correlation Id Handler Configuration
enabled: true
# If set to true, it will auto-generate the correlationID if it is not provided in the request
autogenCorrelationID: true
# MDC context keys usually match your logback.xml pattern
correlationMdcField: cId
traceabilityMdcField: tId
```

## Usage

To use the `CorrelationHandler`, register it in `handler.yml` and add it to the middleware chain. It should be placed early in the chain so that subsequent handlers can benefit from the MDC context.

### `handler.yml`

```yaml
handlers:
  - com.networknt.correlation.CorrelationHandler@correlation
  # ... other handlers

chains:
  default:
    - correlation
    # ... derived handlers
```

## Logging Integration

The handler puts the Correlation ID into the logging context (MDC) using the key defined in `correlationMdcField` (default `cId`). To see this in your logs, your `logback.xml` must utilize this key in its pattern.

### `logback.xml` Example

```xml
<appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
        <!-- The %X{cId} pattern pulls the value from the MDC -->
        <pattern>%d{HH:mm:ss.SSS} [%thread] %X{cId} %-5level %logger{36} - %msg%n</pattern>
    </encoder>
</appender>
```

When properly configured, your logs will include the Correlation ID, making it easy to filter logs in tools like Splunk or ELK (Elasticsearch, Logstash, Kibana).

## Interaction with Traceability

The handler also acknowledges the `X-Traceability-Id` header.
-   If `X-Traceability-Id` is present, it is added to the MDC under the `traceabilityMdcField` (default `tId`).
-   If a new Correlation ID is generated and a Traceability ID exists, the handler logs an info message associating the two IDs: `Associate traceability Id {tId} with correlation Id {cId}`.

## Client Propagation

When making downstream calls using `light-4j` Client modules (like `Http2Client`), it is best practice to propagate the Correlation ID. The Client module provides helper methods (e.g., `propagateHeaders`) to automatically copy the `X-Correlation-Id` from the current request to the downstream client request.

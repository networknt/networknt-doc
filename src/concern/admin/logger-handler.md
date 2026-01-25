# Logger Handler

The `logger-handler` module provides a suite of administrative endpoints to manage logger levels and retrieve log contents at runtime. This allows developers and operators to troubleshoot issues in a running system by adjusting verbosity or inspecting logs without requiring a restart.

**Note:** These administrative features are powerful. In production environments, they **must** be protected by security handlers (e.g., OAuth 2.0 JWT verification with specific scopes) to prevent unauthorized access. It is highly recommended to use the [Light Controller](https://doc.networknt.com/service/controller/) or [Light Portal](https://portal.networknt.com) for centralized management.

## Features

*   **Runtime Level Adjustment**: Change logging levels for existing loggers or create new ones for specific packages/classes.
*   **Log Content Inspection**: Retrieve log entries directly from service instances for UI display or analysis.
*   **Pass-Through Support**: In gateway or sidecar deployments, these handlers can pass requests through to backend services (even if they use different frameworks like Spring Boot).
*   **Auto-Registration**: The module automatically registers its configuration with the `ModuleRegistry` for visibility.

## Configuration (logging.yml)

The behavior of the logging handlers is controlled via `logging.yml`.

```yaml
# Logging endpoint configuration
---
# Indicate if the logging info is enabled or not.
enabled: ${logging.enabled:true}

# Default time period backward in milliseconds for log content retrieval.
# Default is 10 minutes (600,000 ms).
logStart: ${logging.logStart:600000}

# Downstream configuration (useful in sidecar/gateway scenarios)
downstreamEnabled: ${logging.downstreamEnabled:false}

# Downstream API host (e.g., the backend service IP)
downstreamHost: ${logging.downstreamHost:http://localhost:8081}

# Framework of the downstream service (Light4j, SpringBoot, etc.)
downstreamFramework: ${logging.downstreamFramework:Light4j}
```

## Set Up

To enable the logging endpoints, add the following to your `handler.yml`:

```yaml
handlers:
  - com.networknt.logging.handler.LoggerGetHandler@getLogger
  - com.networknt.logging.handler.LoggerPostHandler@postLogger
  - com.networknt.logging.handler.LoggerGetNameHandler@getLoggerName
  - com.networknt.logging.handler.LoggerGetLogContentsHandler@getLogContents

paths:
  - path: '/adm/logger'
    method: 'GET'
    exec:
      - admin  # Security handler
      - getLogger
  - path: '/adm/logger'
    method: 'POST'
    exec:
      - admin
      - postLogger
  - path: '/adm/logger/{loggerName}'
    method: 'GET'
    exec:
      - getLoggerName
  - path: '/adm/logger/content'
    method: 'GET'
    exec:
      - admin
      - getLogContents
```

## Usage

### 1. Get Logger Levels
Retrieves all configured loggers and their current levels.

**Endpoint:** `GET /adm/logger`
```bash
curl -k https://localhost:8443/adm/logger
```
**Example Response:**
```json
[
  {"name": "ROOT", "level": "ERROR"},
  {"name": "com.networknt", "level": "TRACE"}
]
```

### 2. Update Logger Levels
Changes the logging level for specific loggers.

**Endpoint:** `POST /adm/logger`
```bash
curl -k -H "Content-Type: application/json" -X POST \
  -d '[{"name": "com.networknt.handler", "level": "DEBUG"}]' \
  https://localhost:8443/adm/logger
```

### 3. Get Log Contents
Retrieves actual log messages from the application's file appenders.

**Endpoint:** `GET /adm/logger/content`
**Query Parameters:**
*   `loggerName`: Filter by logger name.
*   `loggerLevel`: Filter by minimal level (e.g., `TRACE`).
*   `startTime` / `endTime`: Epoch milliseconds.
*   `limit` / `offset`: Pagination controls (default 100/0).

```bash
curl -k 'https://localhost:8443/adm/logger/content?loggerLevel=DEBUG&limit=10'
```

## Pass-Through (Sidecars/Gateways)

When using `http-sidecar` or `light-gateway`, you can target the backend service's loggers by adding the `X-Adm-PassThrough: true` header to your request.

*   If the backend is **Light4j**, the request is passed as-is.
*   If the backend is **SpringBoot**, the sidecar transforms the request to target Spring Boot Actuator endpoints (e.g., `/actuator/loggers`).

**Example for Spring Boot Backend:**
```yaml
logging.downstreamEnabled: true
logging.downstreamHost: http://localhost:8080
logging.downstreamFramework: SpringBoot
```

```bash
curl -H "X-Adm-PassThrough: true" https://localhost:9445/adm/logger
```

## Dependency

Add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>logger-handler</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

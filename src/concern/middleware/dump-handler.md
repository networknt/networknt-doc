# Dump Handler

The `DumpHandler` is a middleware handler used to log the detailed request and response information. It is primarily used for debugging and troubleshooting in development or testing environments.

> **Warning:** detailed logging of requests and responses can be very slow and may consume significant storage. It is generally **not recommended** for production environments unless explicitly needed for brief diagnostic sessions.

## Introduction

Why requests fail is not always obvious. Sometimes, you need to see exactly what headers, cookies, or body content were sent by the client, or exactly what the server responded with. The `DumpHandler` captures this information and writes it to the logs.

## Configuration

The handler is configured via the `dump.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Indicates if the dump middleware is globally enabled. | `false` |
| `mask` | If true, sensitive data may be masked (implementation dependent). | `false` |
| `logLevel` | Sent logs at this level (`TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`). | `INFO` |
| `indentSize` | Number of spaces for indentation in the log output. | `4` |
| `useJson` | If true, logs request/response details as a JSON object (ignoring `indentSize`). | `false` |
| `requestEnabled` | Global switch to enable/disable dumping of request data. | `false` |
| `responseEnabled` | Global switch to enable/disable dumping of response data. | `false` |

### Detailed Configuration (`request` & `response`)

You can granularly control which parts of the HTTP message are logged using the `request` and `response` objects in the config.

**Request Options:**
-   `url`: Log the request URL.
-   `headers`: Log headers.
-   `cookies`: Log cookies.
-   `queryParameters`: Log query parameters.
-   `body`: Log the request body.
-   **Filters**: `filteredHeaders`, `filteredCookies`, `filteredQueryParameters` allow you to exclude specific sensitive keys.

**Response Options:**
-   `headers`: Log headers.
-   `cookies`: Log cookies.
-   `statusCode`: Log the HTTP status code.
-   `body`: Log the response body.

### Example `dump.yml`

```yaml
# Dump Handler Configuration
enabled: true
logLevel: INFO
indentSize: 4
useJson: false
mask: false

# Enable Request Dumping
requestEnabled: true
request:
  url: true
  headers: true
  filteredHeaders:
    - Authorization
    - X-Correlation-Id
  cookies: true
  filteredCookies:
    - JSESSIONID
  queryParameters: true
  filteredQueryParameters:
    - password
  body: true

# Enable Response Dumping
responseEnabled: true
response:
  headers: true
  cookies: true
  body: true
  statusCode: true
```

## Usage

To use the `DumpHandler`, register it in `handler.yml` and add it to the middleware chain.

**Placement:** It is highly recommended to place the `DumpHandler` **after** the `BodyHandler`. If placed before `BodyHandler`, the request body input stream might not be readable or might be consumed prematurely if not handled correctly.

### `handler.yml`

```yaml
handlers:
  - com.networknt.dump.DumpHandler@dump
  # ... other handlers

chains:
  default:
    - correlation
    - body
    - dump  # Place after BodyHandler
    # ... other handlers
```

## Logging

The handler writes to the standard SLF4J logger. You should configure your `logback.xml` to capture these logs, potentially directing them to a separate file to avoid cluttering your main application logs.

### `logback.xml` Example

```xml
<appender name="DUMP_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>target/dump.log</file>
    <encoder>
        <pattern>%-5level [%thread] %date{ISO8601} %msg%n</pattern>
    </encoder>
    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
        <fileNamePattern>target/dump.log.%i.zip</fileNamePattern>
        <minIndex>1</minIndex>
        <maxIndex>5</maxIndex>
    </rollingPolicy>
    <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
        <maxFileSize>10MB</maxFileSize>
    </triggeringPolicy>
</appender>

<!-- Logger for the dump handler package -->
<logger name="com.networknt.dump" level="INFO" additivity="false">
    <appender-ref ref="DUMP_FILE"/>
</logger>
```

## Output Example

```text
INFO  [XNIO-1 task-1] 2026-02-03 10:15:30 DumpHelper.java:23 - Http request/response information:
request:
    url: /v1/pets
    headers:
        Host: localhost:8080
        Content-Type: application/json
    body: {
        "id": 123,
        "name": "Sparky"
    }
response:
    statusCode: 200
    headers:
        Content-Type: application/json
    body: {
        "status": "success"
    }
```

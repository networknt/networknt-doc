# Audit Handler

The `AuditHandler` is a middleware component responsible for logging request and response details for auditing purposes. It captures important information such as headers, execution time, status codes, and even request/response bodies if configured.

The audit logs are typically written to a separate log file (e.g., `audit.log`) via slf4j/logback, allowing them to be ingested by centralized logging systems like ELK or Splunk.

## Configuration

The configuration is managed by `AuditConfig` and corresponds to `audit.yml` (or `audit.json`/`audit.properties`).

### Configuration Properties

| Property | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | boolean | `true` | Enable or disable the Audit Handler. |
| `mask` | boolean | `true` | Enable masking of sensitive data in headers and audit fields. |
| `statusCode` | boolean | `true` | Include response status code in the audit log. |
| `responseTime` | boolean | `true` | Include response time (latency) in milliseconds. |
| `auditOnError` | boolean | `false` | If true, only log when the response status code is >= 400. If false, log every request. |
| `logLevelIsError` | boolean | `false` | If true, logs at ERROR level. If false, logs at INFO level. |
| `timestampFormat` | string | `null` | Custom format for the timestamp (e.g., `yyyy-MM-dd'T'HH:mm:ss.SSSZ`). If null, uses epoch timestamp. |
| `headers` | list | `[...]` | List of request headers to include in the log. Default: `X-Correlation-Id`, `X-Traceability-Id`, `caller_id`. |
| `audit` | list | `[...]` | List of specific audit fields to include. Default: `client_id`, `user_id`, `scope_client_id`, `endpoint`, `serviceId`. |
| `requestBodyMaxSize` | int | `4096` | Max size of request body to log (if `requestBody` is in `audit` list). Truncated if larger. |
| `responseBodyMaxSize` | int | `4096` | Max size of response body to log (if `responseBody` is in `audit` list). Truncated if larger. |

### Configuration Example (audit.yml)

```yaml
# Enable Audit Logging
enabled: ${audit.enabled:true}

# Enable mask in the audit log
mask: ${audit.mask:true}

# Output response status code
statusCode: ${audit.statusCode:true}

# Output response time
responseTime: ${audit.responseTime:true}

# when auditOnError is true:
#  - it will only log when status code >= 400
# when auditOnError is false:
#  - it will log on every request
auditOnError: ${audit.auditOnError:false}

# log level; by default it is set to info. If you want to change it to error, set to true.
logLevelIsError: ${audit.logLevelIsError:false}

# timestamp format
timestampFormat: ${audit.timestampFormat:}

# headers to audit
headers: ${audit.headers:X-Correlation-Id, X-Traceability-Id,caller_id}

# audit fields. Add requestBody or responseBody here to enable payload logging.
audit: ${audit.audit:client_id, user_id, scope_client_id, endpoint, serviceId}
```

### Values Example (values.yml)

You can customize the audit behavior per environment using `values.yml`. For example, to enable full body logging in a dev environment:

```yaml
# audit.yml
audit.audit:
  - client_id
  - user_id
  - endpoint
  - requestBody
  - responseBody
  - queryParameters
  - pathParameters
```

## Logging Body and Parameters

The `AuditHandler` supports logging additional details if they are added to the `audit` list in the configuration:

-   `requestBody`: Logs the request body.
-   `responseBody`: Logs the response body.
-   `queryParameters`: Logs query parameters.
-   `pathParameters`: Logs path parameters.
-   `requestCookies`: Logs request cookies.

**Note**: Enabling body logging (`requestBody` or `responseBody`) can significantly impact performance and is generally not recommended for production unless `auditOnError` is enabled or for specific debugging purposes.

## Usage

Register the `AuditHandler` in your `handler.yml` chain. It is usually placed near the beginning of the chain to capture the start time and ensure it runs even if downstream handlers fail.

```yaml
handler.handlers:
  .
  - com.networknt.audit.AuditHandler@audit

handler.chains.default:
  - exception
  - traceability
  - correlation
  - audit
  .
```

## Logback Configuration

The `AuditHandler` writes to a logger named `audit`. You should configure a specific appender in your `logback.xml` to route these logs to a separate file or system.

Example `logback.xml` snippet:

```xml
<appender name="audit" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>log/audit.log</file>
    <encoder>
        <pattern>%msg%n</pattern>
    </encoder>
    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
        <fileNamePattern>log/audit.log.%i.zip</fileNamePattern>
        <minIndex>1</minIndex>
        <maxIndex>10</maxIndex>
    </rollingPolicy>
    <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
        <maxFileSize>10MB</maxFileSize>
    </triggeringPolicy>
</appender>

<logger name="audit" level="INFO" additivity="false">
    <appender-ref ref="audit"/>
</logger>
```

## Log Format

The audit log is written as a JSON object.

Example output:
```json
{
  "timestamp": "2023-10-27T10:00:00.123-0400",
  "serviceId": "com.networknt.petstore-1.0.0",
  "X-Correlation-Id": "12345",
  "endpoint": "/v1/pets@get",
  "statusCode": 200,
  "responseTime": 45
}
```

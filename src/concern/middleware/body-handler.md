# Body Handler

The `BodyHandler` is a middleware component designed primarily for `light-rest-4j` services. It automatically parses the incoming request body for methods like POST, PUT, and PATCH, and attaches the parsed object to the exchange for subsequent handlers to consume.

**Note:** This handler is **not** suitable for `light-gateway` or `http-sidecar` as those services typically stream the request body directly to downstream services without consuming it. For gateway scenarios, use `RequestBodyInterceptor` and `ResponseBodyInterceptor` if inspection is required.

## Functionality

The handler inspects the `Content-Type` header and parses the body accordingly:

1.  **application/json**:
    -   Parsed into a `Map` (if starts with `{`) or `List` (if starts with `[`).
    -   Attached to exchange with key `BodyHandler.REQUEST_BODY`.
    -   Optionally caches the raw string body to `BodyHandler.REQUEST_BODY_STRING` if `cacheRequestBody` is enabled (useful for auditing).

2.  **text/plain**:
    -   Parsed into a `String`.
    -   Attached to exchange with key `BodyHandler.REQUEST_BODY`.

3.  **multipart/form-data** / **application/x-www-form-urlencoded**:
    -   Parsed into `FormData`.
    -   Attached to exchange with key `BodyHandler.REQUEST_BODY`.

4.  **Other / Missing Content-Type**:
    -   Attached as an `InputStream`.
    -   Attached to exchange with key `BodyHandler.REQUEST_BODY`.

## Configuration

The configuration is managed by `BodyConfig` and corresponds to `body.yml` (or `body.json`/`body.properties`).

### Configuration Properties

| Property | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | boolean | `true` | Enable or disable the Body Handler. |
| `cacheRequestBody` | boolean | `false` | Cache the request body as a string. Required if you want to audit the request body. |
| `cacheResponseBody` | boolean | `false` | Cache the response body as a string. Required if you want to audit the response body. |
| `logFullRequestBody` | boolean | `false` | If true, logs the full request body (debug only, not for production). |
| `logFullResponseBody` | boolean | `false` | If true, logs the full response body (debug only, not for production). |

### Configuration Example (body.yml)

```yaml
# Enable body parse flag
enabled: ${body.enabled:true}

# Cache request body as a string along with JSON object.
# Required for audit logging of request body.
cacheRequestBody: ${body.cacheRequestBody:false}

# Cache response body as a string.
# Required for audit logging of response body.
cacheResponseBody: ${body.cacheResponseBody:false}

# Log full bodies for debugging (use carefully)
logFullRequestBody: ${body.logFullRequestBody:false}
logFullResponseBody: ${body.logFullResponseBody:false}
```

## Usage

Register the `BodyHandler` in your `handler.yml` chain. It should be placed early in the chain, before any validation or business logic handlers that need access to the body.

```yaml
handler.handlers:
  .
  - com.networknt.body.BodyHandler@body

handler.chains.default:
  .
  - exception
  - correlation
  - body
  .
```

### Retrieving the Body

In your business handler or subsequent middleware:

```java
import com.networknt.body.BodyHandler;

// ...

// Get the parsed body (Map, List, String, FormData, or InputStream)
Object body = exchange.getAttachment(BodyHandler.REQUEST_BODY);

if (body instanceof List) {
    // Handle JSON List
} else if (body instanceof Map) {
    // Handle JSON Map
} else {
    // Handle other types
}
```

### Retrieving the Cached String

If `cacheRequestBody` is enabled:

```java
// Get the raw JSON string
String bodyString = exchange.getAttachment(BodyHandler.REQUEST_BODY_STRING);
```

# Response Injection Middleware

The Response Injection middleware (`ResponseInterceptorInjectionHandler`) allows developers to inject `ResponseInterceptor` implementations into the request/response chain. These interceptors can modify the response headers and body before it is sent back to the client.

## Introduction

Modification of the response body in an asynchronous, non-blocking server like Undertow is complex because the response is often streamed. This middleware handles the complexity by injecting a custom `SinkConduit` (`ModifiableContentSinkConduit`) when necessary. This allows interceptors to buffer the response, modify it, and then send the modified content to the client.

## Configuration

The handler is configured via `response-injection.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `true` |
| `appliedBodyInjectionPathPrefixes` | A list of path prefixes where response body injection should be active. | `[]` |

*   **appliedBodyInjectionPathPrefixes**: Modifying the response body requires buffering the entire response in memory (to replace it). This is an expensive operation and should only be enabled for specific paths.

### `response-injection.yml` Example

```yaml
enabled: true
appliedBodyInjectionPathPrefixes:
  - /v1/pets
  - /v1/details
```

## ResponseInterceptor Interface

To use this middleware, Implement the `ResponseInterceptor` interface.

```java
public interface ResponseInterceptor extends Interceptor {
    /**
     * Indicate if the interceptor requires the response body.
     * @return boolean
     */
    boolean isRequiredContent();

    /**
     * Indicate if execution is async.
     * @return boolean (default false)
     */
    default boolean isAsync() { return false; }

    /**
     * Handle the response.
     * @param exchange The HttpExchange
     * @throws Exception Exception
     */
    void handleRequest(HttpServerExchange exchange) throws Exception;
}
```

*   **isRequiredContent**: Return `true` if you need to modify the response body. This signals the middleware to inject the `ModifiableContentSinkConduit`.
*   **handleRequest**: Implement your transformation logic.

## Compression Handling

If an interceptor requires response content (`isRequiredContent()` returns true), the middleware forces the `Accept-Encoding` header in the **request** to `identity`.
*   This prevents the upstream handler (or backend service) from compressing the response (e.g., GZIP), ensuring the interceptor receives plain text (JSON/XML) to modify.
*   If the response is already compressed (e.g., from a static file handler that ignores Accept-Encoding), the middleware might skip injection to avoid corruption, as indicated by `!isCompressed(exchange)` check.

## Usage

1.  Register the handler in `handler.yml`:
    ```yaml
    handlers:
      - com.networknt.handler.ResponseInterceptorInjectionHandler@responseInjection
    ```

2.  Add it to your chain (usually early in the chain, so it wraps subsequent handlers):
    ```yaml
    chains:
      default:
        - exception
        - metrics
        - responseInjection
        - ...
    ```

## Logic Flow

1.  **Request Flow**:
    *   Handler checks if any registered `ResponseInterceptor` requires content.
    *   If yes, and path matches config, and response is not already compressed:
        *   It wraps the response channel with `ModifiableContentSinkConduit`.
        *   It sets request `Accept-Encoding` to `identity`.
2.  **Response Flow**:
    *   When the downstream handler writes the response, it goes into the `ModifiableContentSinkConduit`.
    *   The conduit buffers the response.
    *   Interceptors are invoked to modify the buffered response.
    *   The modified response is sent to the original sink.

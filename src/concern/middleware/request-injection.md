# Request Injection Middleware

The Request Injection middleware (`RequestInterceptorInjectionHandler`) is designed to inject implementations of the `RequestInterceptor` interface into the request chain. This allows developers to modify request metadata and body transparently before the request reaches the business handler.

## Introduction

In many scenarios, you may need to inspect or modify the request body or headers based on complex rules or external data. While the `request-transformer` middleware handles rule-based transformation, the `request-injection` middleware provides a programmatic way to inject custom logic via the `RequestInterceptor` interface.

A key feature of this handler is its ability to buffer the request body (even for large requests using chunked transfer encoding), allowing interceptors to read and modify the full body content.

## Configuration

The handler is configured via `request-injection.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `true` |
| `appliedBodyInjectionPathPrefixes` | A list of path prefixes where body injection should be active. | `[]` |
| `maxBuffers` | Max number of 16K buffers to use for buffering the body. Default 1024 (16MB). | `1024` |

*   **appliedBodyInjectionPathPrefixes**: Body injection involves buffering the entire request in memory, which is expensive. It should only be enabled for specific paths that require it. For other paths, the interceptors are still called, but `shouldReadBody` will return false, skipping the buffering logic unless an interceptor specifically demands it and the path matches.

### `request-injection.yml` Example

```yaml
enabled: true
maxBuffers: 1024
appliedBodyInjectionPathPrefixes:
  - /v1/pets
  - /v1/store
```

## RequestInterceptor Interface

To use this middleware, you must implement the `RequestInterceptor` interface and register your implementation via the service provider interface (SPI) or `service.yml` if using the SingletonServiceFactory.

```java
public interface RequestInterceptor {
    /**
     * Handle the request.
     * @param exchange The HttpExchange
     * @throws Exception Exception
     */
    void handleRequest(HttpServerExchange exchange) throws Exception;

    /**
     * Indicate if the interceptor requires the request body.
     * @return boolean
     */
    default boolean isRequiredContent() {
        return false;
    }
}
```

*   **handleRequest**: Implement your logic here. You can modify headers, check security, or rewrite the body (if available in attachment).
*   **isRequiredContent**: Return `true` if your interceptor needs to inspect the request body. This combined with `appliedBodyInjectionPathPrefixes` config determines if the handler will buffer the request stream.

## Logic Flow

1.  **Check Config**: The handler takes `RequestInjectionConfig` to check if the current request path matches `appliedBodyInjectionPathPrefixes`.
2.  **Check Interceptors**: It checks if any registered `RequestInterceptor` returns `true` for `isRequiredContent()`.
3.  **Buffer Body**: If both conditions are met (and method has content like POST/PUT), it reads the request channel into a `PooledByteBuffer` array.
4.  **Invoke Interceptors**: It calls `handleRequest()` on all registered interceptors.
    *   Interceptors can access the buffered body via `exchange.getAttachment(AttachmentConstants.BUFFERED_REQUEST_DATA_KEY)`.
5.  **Continue**: After all interceptors run (and assuming none terminated the exchange), the request processing continues to the next handler.

## Payload Size Limit

The `maxBuffers` configuration limits the maximum size of the request body that can be buffered.
*   Buffer size = 16KB.
*   Default `maxBuffers` = 1024.
*   Total max size = 16MB.

If the request body exceeds this limit, the handler will throw a `RequestTooBigException` and return error `ERR10068` (PAYLOAD_TOO_LARGE).

## Usage

Register `com.networknt.handler.RequestInterceptorInjectionHandler` in your `handler.yml`.

```yaml
handlers:
  - com.networknt.handler.RequestInterceptorInjectionHandler@injection
  # ...

chains:
  default:
    - injection
    # ...
```

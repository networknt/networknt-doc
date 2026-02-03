# Content Handler

The `ContentHandler` is a middleware handler designed to manage the `Content-Type` header in HTTP responses. It ensures that a valid content type is always set, either by reflecting the request's `Content-Type` or by applying a configured default.

## Introduction

In some scenarios, backend services or handlers might not explicitly set the `Content-Type` header in their responses. This can lead to clients misinterpreting the response body. The `ContentHandler` solves this by:
1.  Checking if the request has a `Content-Type` header. If so, it uses that same value for the response `Content-Type`.
2.  If the request does not have a `Content-Type`, it sets the response `Content-Type` to a configured default value (usually `application/json`).

This is particularly useful when working with clients that expect specific content types or when enforcing a default format for your API.

## Configuration

The handler is configured via the `content.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Indicates if the content middleware is enabled. | `true` |
| `contentType` | The default content type to be used if the request doesn't specify one. | `application/json` |

### Example `content.yml`

```yaml
# Content middleware configuration
enabled: true
# The default content type to be used in the response if request content type is missing
contentType: application/json
```

## Usage

To use the `ContentHandler`, you need to register it in your `handler.yml` configuration file and add it to the middleware chain.

### `handler.yml`

```yaml
handlers:
  - com.networknt.content.ContentHandler@content
  # ... other handlers

chains:
  default:
    - content
    # ... other handlers
```

## Logic

The `handleRequest` method performs the following logic:

1.  **Check Request Header**: It checks if the `Content-Type` header exists in the incoming request.
2.  **Reflect Context Type**: If the request has a `Content-Type`, the handler sets the **response** `Content-Type` to match the request's `Content-Type`.
3.  **Apply Default**: If the request header is missing, the handler sets the response `Content-Type` to the value specified in `contentType` from the configuration (defaulting to `application/json`).
4.  **Next Handler**: Finally, it passes control to the next handler in the chain.

```java
if (exchange.getRequestHeaders().contains(Headers.CONTENT_TYPE)) {
    exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, exchange.getRequestHeaders().get(Headers.CONTENT_TYPE).element());
} else {
    exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, config.getContentType());
}
Handler.next(exchange, next);
```

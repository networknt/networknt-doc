# Encode Decode Middleware

The `encode-decode` module provides two middleware handlers to handle compression and decompression of HTTP request and response bodies. This is crucial for optimizing bandwidth usage and supporting clients that send compressed data.

## Request Decode Handler

The `RequestDecodeHandler` is responsible for handling requests where the body content is compressed (e.g., GZIP or Deflate). It checks the `Content-Encoding` header and automatically wraps the request stream to decompress the content on the fly, allowing subsequent handlers to read the plain content.

### Configuration

The handler is configured via `request-decode.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `false` |
| `decoders` | A list of supported compression schemes. | `["gzip", "deflate"]` |

### Usage

Register `com.networknt.decode.RequestDecodeHandler` in your `handler.yml`. It should be placed early in the chain, before any handler that needs to read the request body (like `BodyHandler`).

```yaml
handlers:
  - com.networknt.decode.RequestDecodeHandler@decode
  # ... other handlers

chains:
  default:
    - decode
    - body
    # ...
```

## Response Encode Handler

The `ResponseEncodeHandler` is responsible for compressing the response body before sending it to the client. It automatically negotiates the compression method based on the client's `Accept-Encoding` header.

### Configuration

The handler is configured via `response-encode.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `false` |
| `encoders` | A list of supported compression schemes. | `["gzip", "deflate"]` |

### Usage

Register `com.networknt.encode.ResponseEncodeHandler` in your `handler.yml`. It is typically placed early in the chain so that it wraps the response exchange for all subsequent handlers.

```yaml
handlers:
  - com.networknt.encode.ResponseEncodeHandler@encode
  # ... other handlers

chains:
  default:
    - encode
    # ... other handlers
```

## Summary

*   **Request Decode**: Decompresses incoming request bodies (Client -> Server).
*   **Response Encode**: Compresses outgoing response bodies (Server -> Client).

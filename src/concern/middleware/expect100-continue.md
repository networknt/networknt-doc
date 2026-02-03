# Expect100 Continue Handler

The `Expect100ContinueHandler` is a middleware handler designed to manage the HTTP `Expect: 100-continue` protocol. This protocol allows a client to send a header with the request headers to ask the server if it is willing to accept the request body before sending it.

## Introduction

Some HTTP clients (like `curl` or Apache HttpClient) automatically send an `Expect: 100-continue` header when the request body is large (usually > 1024 bytes). They then wait for the server to reply with `HTTP/1.1 100 Continue` before transmitting the body.

If the backend or the gateway does not handle this properly, the client might hang waiting for the 100 response, or the connection might be mishandled. This handler provides proper support for this protocol within the Light-4j handler chain.

## Logic Flow

When the handler detects `Expect: 100-continue` in the request header:

1.  **Check Ignored Paths**: If the request path matches `ignoredPathPrefixes`, the handler simply **removes** the `Expect` header and lets the request proceed. This is useful if downstream services don't support or understand the header.
2.  **Check In-Place Paths**: If the request path matches `inPlacePathPrefixes`, the handler immediately sends a `100 Continue` response to the client and then **removes** the header. This signals the client to start sending the body immediately.
3.  **Default Behavior**: If neither of the above applies, it adds a request wrapper (`Expect100ContinueConduit`) and a response commit listener to manage the 100-continue handshake transparently during the request lifecycle.

## Configuration

The handler is configured via `expect-100-continue.yml`.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `false` |
| `ignoredPathPrefixes` | List of path prefixes where the header should be removed without sending a 100 response. | `[]` |
| `inPlacePathPrefixes` | List of path prefixes where a 100 response is sent immediately, then the header is removed. | `[]` |

### `expect-100-continue.yml`

```yaml
# Expect100Continue Handler Configuration
enabled: true

# Remove Expect header for these paths (simulating it never existed)
ignoredPathPrefixes:
  - /v1/old-service

# Send 100 Continue immediately for these paths
inPlacePathPrefixes:
  - /v1/upload
```

## Usage

Register `com.networknt.expect100continue.Expect100ContinueHandler` in your `handler.yml`. It should be placed early in the chain, before any handler that reads the body.

### `handler.yml`

```yaml
handlers:
  - com.networknt.expect100continue.Expect100ContinueHandler@expect
  # ... other handlers

chains:
  default:
    - expect
    # ...
```

## Why use this handler?
Without this handler, Undertow (the underlying server) might handle 100-continue automatically in a way that doesn't fit a proxy/gateway scenario, or downstream services might reject the request with the Expect header. This handler gives you fine-grained control over how to negotiate this protocol.

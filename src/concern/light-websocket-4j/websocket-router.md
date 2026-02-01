# WebSocket Router

The `WebSocketRouterHandler` is a middleware handler designed to route WebSocket connections to downstream services in a microservices architecture. It sits in the request chain and identifies WebSocket handshake requests either by a specific header (`service_id`) or by matching the request path against configured prefixes.

When a request is identified as a WebSocket request targeted for routing, this handler performs the WebSocket handshake (upgrading the connection) and establishes a proxy connection to the appropriate downstream service. It manages the bi-directional traffic between the client and the downstream service.

## Configuration

The configuration for the WebSocket Router Handler is located in `websocket-router.yml`.

```
# Light websocket router configuration
# Enable WebSocket Router Handler
enabled: ${websocket-router.enabled:true}
# Map of path prefix to serviceId for routing purposes when service_id header is missing.
pathPrefixService: ${websocket-router.pathPrefixService:}
```

### Example Configuration

```yaml
# websocket-router.yml
enabled: true
pathPrefixService:
  /ws/chat: chat-service
  /ws/notification: notification-service
```

## Usage

To use the `WebSocketRouterHandler`, you need to register it in your `handler.yml` configuration file. It should be placed in the middleware chain where you want to intercept and route WebSocket requests.

### 1. Register the Handler

Add the fully qualified class name to the `handlers` list in `handler.yml`:

```yaml
handlers:
  - com.networknt.websocket.router.WebSocketRouterHandler@router
  # ... other handlers
```

### 2. Configure the Chain

Add the handler alias `router` or a custom alias if defined to the `default` chain or specific path chains.

```yaml
chains:
  default:
    - exception
    - metrics
    - traceability
    - correlation
    - header
    - router
```

## How it Works

1.  **Request Interception**: The handler checks each incoming request.
2.  **Identification**:
    *   **Header-based**: Checks for the presence of a `service_id` header.
    *   **Path-based**: Checks if the request path matches any entry in the `pathPrefixService` map.
3.  **Handshake & Upgrade**: If matched, the handler delegates to Undertow's `WebSocketProtocolHandshakeHandler` to perform the upgrade.
4.  **Routing**: Upon successful connection (`onConnect`), it looks up the downstream service URL using the `Cluster` and `Service` discovery mechanism based on the `service_id`.
5.  **Proxying**: It establishes a WebSocket connection to the downstream service and pipes messages between the client and the backend.

If the request is not matched as a WebSocket routing request, the handler simply delegates to the next handler in the chain, allowing standard HTTP requests to proceed to other endpoints.

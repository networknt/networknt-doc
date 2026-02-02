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


## Channel Management

The WebSocket Router uses a concept of "Channels" to manage client sessions.

1.  **Channel Group ID**:
    - The router expects a unique identifier for each client connection, typically passed in the `x-group-id` header (internally `WsAttributes.CHANNEL_GROUP_ID`).
    - If this header is missing (e.g., standard browser connections), the router **automatically generates a unique UUID** for the session.

2.  **Connection Mapping**:
    - Each unique Channel Group ID corresponds to a distinct WebSocket connection to the downstream service.
    - If a client connects with a generated UUID, a **new** connection is established to the backend service for that specific session.
    - Messages are proxied exclusively between the client's channel and its corresponding downstream connection.


 This ensures that multiple browser tabs or distinct clients are isolated, each communicating with the backend over its own dedicated WebSocket link.

## Architecture: Router vs. Proxy

You might observe that the `WebSocketRouterHandler` functions primarily as a **Reverse Proxy**: it terminates the client connection and establishes a separate connection to the backend service.

It is named a "Router" because of its role in the system architecture:
1.  **Multiplexing**: It can route different paths (e.g., `/chat`, `/notification`) to different backend services on the same gateway port.
2.  **Service Discovery**: It dynamically resolves the backend URL using the `service_id` and the configured Registry/Cluster (e.g., Consul, Kubernetes), rather than proxying to a static IP address.

Thus, while the *mechanism* is proxying, the *function* is dynamic routing and load balancing of WebSocket traffic.

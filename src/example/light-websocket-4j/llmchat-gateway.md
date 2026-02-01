# LLM Chat Gateway

The `llmchat-gateway` is an example application that demonstrates how to usage `websocket-router` to create a gateway for the [LLM Chat Server](./llmchat-server.md).

It acts as a secure entry point (HTTPS) that proxies WebSocket traffic to the backend chat server.

## Architecture

-   **Gateway**: Listens on HTTPS port 8443. Serves the static UI and routes `/chat` WebSocket connections to the backend.
-   **Backend**: `llmchat-server` running on HTTP port 8080.
-   **Routing**: Uses `WebSocketRouterHandler` to forward messages based on path or serviceId. Also uses `DirectRegistry` for service discovery.

## Configuration

The configuration is located in `src/main/resources/config/values.yml`.

### Server & Handler
Configured for HTTPS on port 8443.

```yaml
server.httpsPort: 8443
server.enableHttp: false
server.enableHttps: true

handler.paths:
  - path: '/'
    method: 'GET'
    exec:
      - resource
  - path: '/chat'
    method: 'GET'
    exec:
      - router
```

### WebSocket Router
Maps requests to the backend service.

```yaml
websocket-router.pathPrefixService:
  /chat: com.networknt.llmchat-1.0.0
```

### Service Registry
Uses `DirectRegistry` to locate the backend server (llmchat-server) at `http://localhost:8080`.

```yaml
service.singletons:
  - com.networknt.registry.Registry:
    - com.networknt.registry.support.DirectRegistry

direct-registry:
  com.networknt.llmchat-1.0.0:
    - http://localhost:8080
```

## Running the Example

1.  **Start Ollama**: Ensure Ollama is running.
2.  **Start Backend**:
    From `light-example-4j/websocket/llmchat-server`:
    ```bash
    mvn exec:java
    ```
3.  **Start Gateway**:
    From `light-example-4j/websocket/llmchat-gateway`:
    ```bash
    mvn exec:java
    ```
    (Note: ensure you have built it first with `mvn clean install`)

## Usage

Open your browser and navigate to **https://localhost:8443**.

-   You might see a security warning because the `server.keystore` uses a self-signed certificate. Accept it to proceed.
-   The chat interface is served from the gateway.
-   When you click "Connect", it opens a secure WebSocket (`wss://`) to the gateway.
-   The gateway routes frames to `llmchat-server`, which invokes the LLM and streams the response back.

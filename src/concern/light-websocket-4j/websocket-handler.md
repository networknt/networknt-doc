# WebSocket Handler

The `WebSocketHandler` is a middleware handler designed to process WebSocket messages on the server side (Light Gateway or Light 4J Service), rather than proxying them to downstream services. It enables the implementation of custom logic, such as Chat bots, GenAI integration, or real-time notifications, directly within the application.

## Configuration

The configuration is located in `websocket-handler.yml`.

| Field | Type | Description | Default |
|---|---|---|---|
| enabled | boolean | Enable or disable the WebSocket Handler. | true |
| pathPrefixHandlers | Map<String, String> | A map where keys are path prefixes (e.g., `/chat`) and values are the fully qualified class names of the handler implementation which must implement `com.networknt.websocket.handler.WebSocketApplicationHandler`. | empty |

### Example Configuration

```yaml
# websocket-handler.yml
enabled: true
pathPrefixHandlers:
  /chat: com.networknt.chat.ChatHandler
  /notifications: com.networknt.notify.NotificationHandler
```

## Implementing a Handler

To handle WebSocket connections, you must implement the `com.networknt.websocket.handler.WebSocketApplicationHandler` interface used in the configuration above.

```java
package com.networknt.chat;

import com.networknt.websocket.handler.WebSocketApplicationHandler;
import io.undertow.websockets.core.*;
import io.undertow.websockets.spi.WebSocketHttpExchange;

public class ChatHandler implements WebSocketApplicationHandler {
    @Override
    public void onConnect(WebSocketHttpExchange exchange, WebSocketChannel channel) {
        channel.getReceiveSetter().set(new AbstractReceiveListener() {
            @Override
            protected void onFullTextMessage(WebSocketChannel channel, BufferedTextMessage message) {
                String data = message.getData();
                // Process message (e.g., send to GenAI, broadcast to other users)
                WebSockets.sendText("Echo: " + data, channel, null);
            }
        });
        channel.resumeReceives();
    }
}
```

## Usage

To use the `WebSocketHandler`, you need to register it in your `handler.yml` configuration file.

### 1. Register the Handler

Add the `WebSocketHandler` to your `handler.yml` configuration:

```yaml
handlers:
  - com.networknt.websocket.handler.WebSocketHandler
```

### 2. Add to Chain

Place it in the middleware chain. It should be placed after security if authentication is required.

```yaml
chains:
  default:
    - exception
    - metrics
    - traceability
    - correlation
    - WebSocketHandler
    # ... other handlers
```

## How It Works

1.  **Matching**: The handler checks if the request path starts with one of the configured `pathPrefixHandlers`.
2.  **Instantiation**: It loads and instantiates the configured handler class singleton at startup.
3.  **Upgrade**: If a request matches, `WebSocketHandler` performs the WebSocket handshake (upgrading HTTP to WebSocket).
4.  **Delegation**: Upon successful connection (`onConnect`), it delegates control to the matched implementation of `WebSocketApplicationHandler`.

If the request does not match any configured prefix, it is passed to the next handler in the chain.

## Maven Dependency

Ensure you have the module dependency in your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>websocket-handler</artifactId>
    <version>${version.light-websocket-4j}</version>
</dependency>
```

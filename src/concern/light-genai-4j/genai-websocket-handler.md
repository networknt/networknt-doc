# GenAI WebSocket Handler

The `genai-websocket-handler` module provides a WebSocket-based interface for interacting with Generative AI models via the `light-genai-4j` library. It manages user sessions, maintains chat history context, and handles the bi-directional stream of messages between the user and the LLM.

## Architecture

This module is designed as an implementation of the `WebSocketApplicationHandler` interface defined in `light-websocket-4j`. It plugs into the `websocket-handler` infrastructure to receive upgraded WebSocket connections.

### Core Components

1.  **GenAiWebSocketHandler**: The main entry point. It handles `onConnect`, manages the WebSocket channel life-cycle, and coordinates message processing.
2.  **SessionManager**: Responsible for creating, retrieving, and validating user chat sessions.
3.  **HistoryManager**: Responsible for persisting and retrieving the conversation history required for LLM context.
4.  **ModelClient**: Wraps the `light-genai-4j` clients (Gemini, OpenAI, etc.) to invoke the model.

## Data Models

### ChatMessage
Represents a single message in the conversation.
```java
public class ChatMessage {
    String role;       // "user", "model", "system"
    String content;    // The text content
    long timestamp;    // Epoch timestamp
}
```

### ChatSession
Represents the state of a conversation.
```java
public class ChatSession {
    String sessionId;
    String userId;
    String model;      // The generic model name (e.g., "gemini-pro")
    Map<String, Object> parameters; // Model parameters (temperature, etc.)
}
```

## Storage Interfaces

To support scaling from a single instance to a clustered environment, session and history management are abstracted behind repository interfaces.

### ChatSessionRepository
```java
public interface ChatSessionRepository {
    ChatSession createSession(String userId, String model);
    ChatSession getSession(String sessionId);
    void deleteSession(String sessionId);
}
```

### ChatHistoryRepository
```java
public interface ChatHistoryRepository {
    void addMessage(String sessionId, ChatMessage message);
    List<ChatMessage> getHistory(String sessionId);
    void clearHistory(String sessionId);
}
```

## Implementations

### In-Memory (Default)
The initial implementation provides in-memory storage using concurrent maps. This is suitable for single-instance deployments or testing.

*   `InMemoryChatSessionRepository`: Uses `ConcurrentHashMap<String, ChatSession>`.
*   `InMemoryChatHistoryRepository`: Uses `ConcurrentHashMap<String, Deque<ChatMessage>>` (or List).

### Future Implementations
The design allows for future implementations without changing the core handler logic:
*   **JDBC/RDBMS**: Persistent storage for long-term history.
*   **Redis**: Shared cache for clustered deployments (session affinity not required).
*   **Hazelcast**: In-memory data grid for distributed caching.

## Configuration
The implementation to use can be configured via `service.yml` (using Light-4J's Service/Module loading) or dedicated config files.

Example `service.yml` for In-Memory:
```yaml
- com.networknt.genai.handler.ChatHistoryRepository:
  - com.networknt.genai.handler.InMemoryChatHistoryRepository
```

## Streaming Behavior

The handler implements a buffering mechanism for the streaming response from the LLM to improve client-side rendering.

### Response Buffering
LLM providers usually stream tokens (words or partial words) as they are generated. If these tokens are sent to the client immediately as individual WebSocket frames, simplistic clients that print each frame on a new line will display a fragmented "word-per-line" output.

To resolve this, the `GenAiWebSocketHandler` buffers incoming tokens and only flushes them to the client when:
1.  A newline character (`\n`) is detected in the buffer.
2.  The stream from the LLM completes.

This ensures that the client receives complete lines or paragraphs, resulting in a cleaner user experience.


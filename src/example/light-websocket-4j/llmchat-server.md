# LLM Chat Server

The `llmchat-server` is an example application demonstrating how to build a real-time Generative AI chat server using `light-genai-4j` and `light-websocket-4j`.

It uses the `genai-websocket-handler` to manage WebSocket connections and orchestrate interactions with an LLM backend (configured for Ollama by default).

## Prerequisites

-   jdk 11 or above
-   maven 3.6.0 or above
-   [Ollama](https://ollama.ai/) running locally (for the default configuration) with a model pulled (e.g., `qwen3:14b` or `llama3`).

## Configuration

The server configuration is consolidated in `src/main/resources/config/values.yml`.

### Server & Handler
The server runs on HTTP port 8080 and defines two main paths:
-   `/`: Serves static web resources (the chat UI).
-   `/chat`: The WebSocket endpoint for chat sessions.

```yaml
handler.paths:
  - path: '/'
    method: 'GET'
    exec:
      - resource
  - path: '/chat'
    method: 'GET'
    exec:
      - websocket
```

### GenAI Client
Dependencies are injected via `service.yml` configuration (in `values.yml`). By default, it uses `OllamaClient`.

```yaml
service.singletons:
  - com.networknt.genai.GenAiClient:
    - com.networknt.genai.ollama.OllamaClient
```

### Ollama Configuration
Configures the connection to the Ollama instance.

```yaml
ollama.ollamaUrl: http://localhost:11434
ollama.model: qwen3:14b
```

### WebSocket Handler
Maps the `/chat` path to the `GenAiWebSocketHandler`.

```yaml
websocket-handler.pathPrefixHandlers:
  /chat: com.networknt.genai.handler.GenAiWebSocketHandler
```

## Running the Server

1.  **Start Ollama**: Ensure Ollama is running and the model configured in `values.yml` is available.
    ```bash
    ollama run qwen3:14b
    ```

2.  **Build and Start**:
    ```bash
    cd light-example-4j/websocket/llmchat-server
    mvn clean install exec:java
    ```

## Usage

### Web UI
Open your browser and navigate to `http://localhost:8080`. You should see a simple chat interface where you can type messages and receive streaming responses from the LLM.

### WebSocket Client
You can also connect using any WebSocket client (like `wscat`):

```bash
wscat -c ws://localhost:8080/chat?userId=user1&model=qwen3:14b
```

Send a message:
```
> Hello
< Assistant: Hi
< Assistant: there!
...
```

The server streams the response token by token (buffered by line/sentence for better display).

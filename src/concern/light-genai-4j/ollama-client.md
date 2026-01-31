# Ollama Client

The `ollama-client` module provides a non-blocking, asynchronous client for interacting with the Ollama API. It is part of the `light-genai-4j` library and leverages the Undertow HTTP client for efficient communication.

## Features

*   **Non-blocking I/O**: Uses Undertow's asynchronous HTTP client for high throughput.
*   **Streaming Support**: Supports streaming responses (NDJSON) from Ollama for chat completions.
*   **Configuration**: externalizable configuration via `ollama.yml`.
*   **Token Management**: N/A (Ollama usually runs locally without auth, but can be proxied).

## Configuration

The client is configured via `ollama.yml` in the `src/main/resources/config` directory (or externalized config folder).

### Properties

| Property | Description | Default |
| :--- | :--- | :--- |
| `ollamaUrl` | The base URL of the Ollama server. | `http://localhost:11434` (example) |
| `model` | The default model to use for requests. | `llama3.2` (example) |

### Example `ollama.yml`

```yaml
ollamaUrl: http://localhost:11434
model: llama3.2
```

## Usage

### Sync Chat (Non-Streaming)

```java
GenAiClient client = new OllamaClient();
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Hello, who are you?"));

// Blocking call (not recommended for high concurrency)
String response = client.chat(messages);
System.out.println(response);
```

### Async Chat (Streaming)

```java
GenAiClient client = new OllamaClient();
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Tell me a story."));

client.chatStream(messages, new StreamCallback() {
    @Override
    public void onEvent(String content) {
        System.out.print(content); // Process token chunk
    }

    @Override
    public void onComplete() {
        System.out.println("\nDone.");
    }

    @Override
    public void onError(Throwable throwable) {
        throwable.printStackTrace();
    }
});
```

## Dependencies

*   `light-4j` core (client, config)
*   `genai-core`

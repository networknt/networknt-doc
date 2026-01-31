# Light GenAI Client Design

## Introduction

The `light-genai-4j` library provides a standardized way for Light-4j applications to interact with various Generative AI (GenAI) providers. By abstracting the underlying client implementations behind a common interface, applications can support dynamic model switching and simplified integration for different environments (e.g., local development vs. production).

## Architecture

The project is structured into a core module and provider-specific implementation modules.

### Modules

1.  **genai-core**: Defines the common interfaces and shared utilities.
2.  **genai-ollama**: Implementation for the Ollama API, suitable for local LLM inference.
3.  **genai-bedrock**: Implementation for AWS Bedrock, suitable for enterprise-grade managed LLMs.

## Interface Design

The core interaction is defined by the `GenAiClient` interface in the `genai-core` module.

```java
package com.networknt.genai;

import java.util.List;

public interface GenAiClient {
    /**
     * Generates a text completion for the given list of chat messages.
     * 
     * @param messages The list of chat messages (history).
     * @return The generated text response from the model.
     */
    String chat(List<ChatMessage> messages);

    /**
     * Generates a text completion stream for the given list of chat messages.
     * 
     * @param messages The list of chat messages (history).
     * @param callback The callback to receive chunks, completion, and errors.
     */
    void chatStream(List<ChatMessage> messages, StreamCallback callback);
}
```

The `StreamCallback` interface:
```java
public interface StreamCallback {
    void onEvent(String content);
    void onComplete();
    void onError(Throwable t);
}
```

This simple interface allows for "drop-in" replacements of the backend model without changing the application logic.

### ChatMessage

A simple POJO to represent a message in the conversation.

```java
public class ChatMessage {
    private String role; // "user", "assistant", "system"
    private String content;
    // constructors, getters, setters
}
```

## Implementations

### Ollama (`genai-ollama`)

Connects to a local or remote Ollama instance.

*   **Configuration**: `ollama.yml`
    *   `ollamaUrl`: URL of the Ollama server (e.g., `http://localhost:11434`).
    *   `model`: The model name to use (e.g., `llama3.1`, `mistral`).
*   **Protocol**: Uses the `/api/generate` endpoint via HTTP/2.

### AWS Bedrock (`genai-bedrock`)

Connects to Amazon Bedrock using the AWS SDK for Java v2.

*   **Configuration**: `bedrock.yml`
    *   `region`: AWS Region (e.g., `us-east-1`).
    *   `modelId`: The specific model ID (e.g., `anthropic.claude-v2`, `amazon.titan-text-express-v1`).
*   **Authentication**: Uses the standard AWS Default Credentials Provider Chain (Environment variables, Profile, IAM Roles).

### OpenAI (`genai-openai`)

Connects to the OpenAI Chat Completions API.

*   **Configuration**: `openai.yml`
    *   `url`: API endpoint (e.g., `https://api.openai.com/v1/chat/completions`).
    *   `model`: The model to use (e.g., `gpt-3.5-turbo`, `gpt-4`).
    *   `apiKey`: Your OpenAI API key.
*   **Protocol**: Uses standard HTTP/2 (or HTTP/1.1) to send JSON payloads.

### Gemini (`genai-gemini`)

Connects to Google's Gemini models (via Vertex AI or AI Studio).

*   **Configuration**: `gemini.yml`
    *   `url`: API endpoint structure.
    *   `model`: The model identifier (e.g., `gemini-pro`).
    *   `apiKey`: Your Google API key.
*   **Protocol**: REST API with JSON payloads.

### Code Example

The following example demonstrates how to use the interface to interact with a model, regardless of the underlying implementation.

```java
// Logic to instantiate the correct client based on external configuration (e.g. from service.yml or reflection)
GenAiClient client;
if (useBedrock) {
    client = new BedrockClient();
} else if (useOpenAi) {
    client = new OpenAiClient();
} else if (useGemini) {
    client = new GeminiClient();
} else {
    client = new OllamaClient();
}

// Application logic remains agnostic
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Explain quantum computing in 50 words."));
String response = client.chat(messages);
System.out.println(response);

// For subsequent turns:
messages.add(new ChatMessage("assistant", response));
messages.add(new ChatMessage("user", "What about entanglement?"));
String response2 = client.chat(messages);
```

## Future Enhancements
*   **Streaming Support**: Add `generateStream` to support token streaming.
*   **Chat Models**: Add support for structured chat history (System, User, Assistant messages).
*   **Tool Use**: Support for function calling and tool use with models that support it.
*   **More Providers**: Integrations for OpenAI (ChatGPT), Google Vertex AI (Gemini), and others.

## Technical Decisions

### Use of `Http2Client` over JDK `HttpClient`

The implementation uses the light-4j `Http2Client` (wrapping Undertow) instead of the standard JDK `HttpClient` for the following reasons:

1.  **Framework Consistency**: `Http2Client` is the standard client within the light-4j ecosystem. Using it ensures consistent configuration, management, and behavior across all modules of the framework.
2.  **Performance**: It leverages the non-blocking I/O capabilities of the underlying Undertow server, sharing the same XNIO worker threads as the server components. This minimizes context switching and optimizes resource usage in a microservices environment.
3.  **Callback Pattern**: The `ClientCallback` and `ChannelListener` patterns are idiomatic to light-4j/Undertow. While they differ from the `CompletableFuture` style of the JDK client, using them maintains architectural uniformity for developers familiar with the framework's internals.
4.  **Integration**: Utilizing the framework's client allows for seamless integration with other light-4j features such as centralized SSL context management, connection pooling, and client-side observability.

For implementations that require vendor-specific logic (like AWS signing), we utilize the official vendor SDKs (e.g., AWS SDK for Java v2 for Bedrock) to handle complex authentication and protocol details efficiently.

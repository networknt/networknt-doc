# Building Real-Time GenAI Applications with Java 21: Introducing Light-GenAI-4J and Light-WebSocket-4J

The landscape of application development is shifting rapidly with the adoption of Generative AI. As enterprises move from experimental scripts to production-grade applications, the requirements for performance, scalability, and maintainability become critical.

Today, we are excited to announce two new modules in the Light-4j ecosystem designed to bridge the gap between high-performance Java microservices and modern AI capabilities: **Light-GenAI-4J** and **Light-WebSocket-4J**.

## The Challenge: Latency and Streaming

Large Language Models (LLMs) are unique compared to traditional APIs. They don't just return a JSON object; they "stream" text token by token. Users expect to see this output in real-time. Traditional synchronous HTTP calls (request/response) often result in poor user experience due to the wait time for the full completion.

While Server-Sent Events (SSE) are a common solution, **WebSockets** offer a robust, bidirectional alternative that is ideal for complex conversational flows where the client and server exchange messages frequently.

## Light-GenAI-4J: The AI Connector

`light-genai-4j` is a new module that provides a unified, high-performance client for interacting with Generative AI models.

### Key Features
-   **Java 21 Ready**: Built to leverage modern Java features.
-   ** unified Interface**: Switch between providers (Ollama, OpenAI, Bedrock, etc.) with configuration changes, not code rewrites.
-   **Light-4j Integration**: seamlessly integrates with the Light-4j middleware chain. You can apply the same Audit, Security, and Metrics handlers to your AI calls as you do for your REST APIs.
-   **Streaming Support**: First-class support for streaming responses, essential for chat applications.

## Light-WebSocket-4J: Real-Time Transport

WebSockets have always been powerful, but implementing them efficiently at scale can be complex. `light-websocket-4j` simplifies this by providing:

-   **Structured Handlers**: A cleaner abstraction for handling Open, Message, Error, and Close events.
-   **Routing**: Route WebSocket messages based on paths or content, similar to REST routing.
-   **Connection Registry**: Built-in management for active connections, making it easier to push messages to specific users or groups.

## Showcase: Building a Real-Time LLM Chatbot

To demonstrate the power of these two modules working together, we've built a complete [LLM Chat Example](https://github.com/networknt/light-example-4j/tree/master/websocket/llmchat-server).

### The Architecture

1.  **Transport**: The user's browser connects via WebSocket to the Light-4j server.
2.  **Handler**: The `GenAiWebSocketHandler` receives the user's prompt.
3.  **AI Client**: The handler uses `OllamaClient` (part of `light-genai-4j`) to send the prompt to a local Ollama instance running the `qwen3:14b` model.
4.  **Streaming**: As Ollama generates tokens, the client captures them via a callback and immediately pushes them down the WebSocket to the browser.

### The Code

Here is a glimpse of how simple the handler logic becomes:

```java
// Inside onFullTextMessage
ollamaClient.chatStream(history, new StreamCallback() {
    @Override
    public void onEvent(String content) {
        // stream the token back to the user immediately
        WebSockets.sendText(content, channel, null);
    }
    
    @Override
    public void onComplete() {
        // handle completion
    }
});
```

Because this runs on Light-4j's non-blocking I/O (powered by XNIO/Undertow), a single server can handle thousands of concurrent chat sessions with minimal resource footprint.

## Deploying with Light-Gateway

For enterprise deployments, we also updated **Light-Gateway** to support these new protocols. You can now place simple, stateless Gateway instances in front of your AI services to handle:
-   **Unified Authentication**: Secure your internal AI models with OAuth2.
-   **Rate Limiting**: Prevent abuse of expensive API tokens.
-   **Cross-Cutting Concerns**: Centralized logging and auditing of all AI interactions.

## Get Started

The code is open source and available on GitHub.

-   **Light-GenAI-4J**: [github.com/networknt/light-genai-4j](https://github.com/networknt/light-genai-4j)
-   **Light-WebSocket-4J**: [github.com/networknt/light-websocket-4j](https://github.com/networknt/light-websocket-4j)
-   **Running Example**: [LLM Chat Server Tutorial](https://github.com/networknt/light-example-4j/tree/master/websocket/llmchat-server)

We believe Java is the perfect language for orchestrating Enterprise AI agents. Give `light-genai-4j` a try and let us know what you build!

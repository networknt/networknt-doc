# Light-genai-4j

`light-genai-4j` is a library that provides integration with various Generative AI models (LLMs) such as Gemini, OpenAI, Bedrock, and Ollama for the light-4j framework.

## Design Decisions

### GenAI WebSocket Handler

The `genai-websocket-handler` module is located in this repository (`light-genai-4j`) rather than `light-websocket-4j`.

**Reasoning:**
*   **Dependency Direction**: This handler implements specific business logic (managing chat history, session context, and invoking LLM clients) that depends on the core GenAI capabilities provided by `light-genai-4j`.
*   **Separation of Concerns**: `light-websocket-4j` is an infrastructure library responsible for the WebSocket transport layer (connection management, routing, hygiene). `light-genai-4j` is the domain library responsible for AI interactions.
*   **Implementation**: This module implements `com.networknt.websocket.handler.WebSocketApplicationHandler` from `light-websocket-4j`, effectively bridging the transport layer with the AI domain layer.

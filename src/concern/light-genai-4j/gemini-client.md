# Gemini Client

The `GeminiClient` is an implementation of `GenAiClient` that interacts with Google's Gemini API. It supports both synchronous chat and asynchronous streaming chat.

## Features

-   **Synchronous Chat**: Sends a prompt to the model and waits for the full response.
-   **Streaming Chat**: streams the response from the model chunk by chunk, suitable for real-time applications.
-   **Non-Blocking I/O**: Uses XNIO and Undertow's asynchronous client to prevent blocking threads during I/O operations.

## Configuration

The client is configured via `gemini.yml`.

### Properties

| Property | Description | Default |
| :--- | :--- | :--- |
| `url` | The Gemini API URL base format. | `https://generativelanguage.googleapis.com/v1beta/models/%s:generateContent` |
| `model` | The model to use (e.g., `gemini-pro`). | `null` |
| `apiKey` | Your Google Cloud API key. | `null` |

### Example `gemini.yml`

```yaml
url: https://generativelanguage.googleapis.com/v1beta/models/%s:generateContent
model: gemini-pro
apiKey: your-google-api-key
```

## Usage

### Injection

You can inject the `GeminiClient` as a `GenAiClient` implementation.

```java
GenAiClient client = new GeminiClient();
```

### Synchronous Chat

```java
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Hello, Gemini!"));
String response = client.chat(messages);
System.out.println(response);
```

### Streaming Chat

```java
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Write a poem."));

client.chatStream(messages, new StreamCallback() {
    @Override
    public void onEvent(String content) {
        System.out.print(content);
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

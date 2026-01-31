# OpenAI Client

The `OpenAiClient` is an implementation of `GenAiClient` that interacts with the OpenAI API (GPT models). It supports both synchronous chat and asynchronous streaming chat.

## Features

-   **Synchronous Chat**: Sends a prompt to the model and waits for the full response.
-   **Streaming Chat**: streams the response from the model token by token, suitable for real-time applications.
-   **Non-Blocking I/O**: Uses XNIO and Undertow's asynchronous client to prevent blocking threads during I/O operations.

## Configuration

The client is configured via `openai.yml`.

### Properties

| Property | Description | Default |
| :--- | :--- | :--- |
| `url` | The OpenAI API URL for chat completions. | `https://api.openai.com/v1/chat/completions` |
| `model` | The model to use (e.g., `gpt-3.5-turbo`, `gpt-4`). | `null` |
| `apiKey` | Your OpenAI API key. | `null` |

### Example `openai.yml`

```yaml
url: https://api.openai.com/v1/chat/completions
model: gpt-3.5-turbo
apiKey: your-openai-api-key
```

## Usage

### Injection

You can inject the `OpenAiClient` as a `GenAiClient` implementation.

```java
GenAiClient client = new OpenAiClient();
```

### Synchronous Chat

```java
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Hello, OpenAI!"));
String response = client.chat(messages);
System.out.println(response);
```

### Streaming Chat

```java
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Write a long story."));

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

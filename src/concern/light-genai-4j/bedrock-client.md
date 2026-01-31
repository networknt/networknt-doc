# Bedrock Client

The `BedrockClient` is an implementation of `GenAiClient` that interacts with AWS Bedrock. It supports both synchronous chat and asynchronous streaming chat via the AWS SDK for Java 2.x.

## Features

-   **Synchronous Chat**: Sends a prompt to the model and waits for the full response.
-   **Streaming Chat**: streams the response from the model chunk by chunk, suitable for real-time applications.
-   **Non-Blocking I/O**: Leverages `BedrockRuntimeAsyncClient` and JDK `CompletableFuture` for non-blocking operations.

## Configuration

The client is configured via `bedrock.yml`.

### Properties

| Property | Description | Default |
| :--- | :--- | :--- |
| `region` | The AWS region where Bedrock is enabled (e.g., `us-east-1`). | `null` |
| `modelId` | The Bedrock model ID to use (e.g., `anthropic.claude-v2`, `amazon.titan-text-express-v1`). | `null` |

### Example `bedrock.yml`

```yaml
region: us-east-1
modelId: anthropic.claude-v2
```

## Usage

### Injection

You can inject the `BedrockClient` as a `GenAiClient` implementation.

```java
GenAiClient client = new BedrockClient();
```

### Synchronous Chat

```java
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Hello, Bedrock!"));
String response = client.chat(messages);
System.out.println(response);
```

### Streaming Chat

```java
List<ChatMessage> messages = new ArrayList<>();
messages.add(new ChatMessage("user", "Tell me a joke."));

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

## AWS Credentials

The `BedrockClient` uses the `DefaultCredentialsProvider` chain from the AWS SDK. Prior to using the client, ensure your environment is configured with valid AWS credentials (e.g., via environment variables, `~/.aws/credentials`, or IAM roles).

# Exception Handler

The `ExceptionHandler` is a critical middleware component that sits at the beginning of the request/response chain. Its primary role is to catch any unhandled exceptions thrown by subsequent handlers, log them, and return a standardized error response to the client.

## Introduction

In Light-4j, exceptions are used to signal errors. While business handlers are encouraged to handle known exceptions locally to provide context-specific responses, the `ExceptionHandler` acts as a safety net (or "last line of defense") to ensure that:
1.  No exception crashes the server or returns a raw stack trace to the client.
2.  All unhandled exceptions result in a proper JSON error response.
3.  Requests are dispatched from the IO thread to the Worker thread (if not already there).

## Thread Model

One important feature of this handler is that it explicitly checks if the request is currently running in the **IO Thread**. If so, it dispatches the request to the **Worker Thread**. 

```java
if (exchange.isInIoThread()) {
    exchange.dispatch(this);
    return;
}
```

This ensures that business logic, which might block or throw exceptions, runs in the worker pool, preventing the non-blocking IO threads from hanging.

## Logic Flow

1.  **Dispatch**: Logic moves execution to a worker thread.
2.  **Chain Execution**: It wraps `Handler.next(exchange, next)` in a `try-catch` block.
3.  **Exception Catching**: If an exception bubbles up, it is categorized and handled:
    *   **FrameworkException** (Runtime): Returns the specific Status code defined in the exception.
    *   **Other RuntimeExceptions**: Returns generic error `ERR10010` (Status 500).
    *   **ApiException** (Checked): Returns the specific Status code defined in the exception.
    *   **ClientException** (Checked): Returns the specific Status code if set, otherwise `ERR10011` (Status 400).
    *   **Other Checked Exceptions**: Returns generic error `ERR10011` (Status 400).

## Configuration

The handler is configured via `exception.yml`. By default, it is enabled.

```yaml
# Exception Handler Configuration
enabled: true
```

## Error Codes

The handler relies on `status.yml` to map error codes to messages.

| Error Code | Status | Description |
| :--- | :--- | :--- |
| `ERR10010` | 500 | RuntimeException: Generic internal server error. |
| `ERR10011` | 400 | UncaughtException: Generic bad request or unhandled checked exception. |

## Usage

This handler should be the **first** (or one of the very first) handlers in your chain defined in `handler.yml`.

### `handler.yml`

```yaml
handlers:
  - com.networknt.exception.ExceptionHandler@exception
  # ... other handlers

chains:
  default:
    - exception
    - metrics
    - trace
    - correlation
    # ... business handlers
```

## Best Practices

*   **Keep it Enabled**: Do not disable this handler in production unless you have a specific reason and alternative exception handling mechanism.
*   **Handle Business Exceptions**: Try to catch and handle semantic exceptions (like "User Not Found") in your business handler to return a more meaningful error code than the generic fallbacks provided here.

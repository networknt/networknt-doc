---
description: Exception Assault Handler
---

# Exception Assault

The `exception-assault` module in `light-chaos-monkey` allows you to inject random exceptions into your application's request processing pipeline. This helps verify that your application and its consumers gracefully handle unexpected failures.

## Core Components

### ExceptionAssaultHandler
The middleware handler responsible for injecting exceptions.
*   **Behavior**: When enabled and triggered (based on the `level` configuration), it throws an `AssaultException`. This exception disrupts the normal request flow, simulating an internal server error or unexpected crash.
*   **Bypass**: Can be configured to bypass the assault logic (e.g., for specific requests or globally until ready).

### ExceptionAssaultConfig
Configuration class that loads settings from `exception-assault.yml`.

## Configuration

The module is configured via `exception-assault.yml`.

**Key Settings:**
*   `enabled`: Enable or disable the handler (default: `false`).
*   `bypass`: If `true`, the assault is skipped even if enabled (default: `true`). Use this to deploy the handler but keep it inactive until needed.
*   `level`: The probability of an attack, defined as "1 out of N requests".
    *   `level: 5` means approximately 1 in 5 requests (20%) will be attacked.
    *   `level: 1` means every request is attacked.

**Example Configuration:**

```yaml
enabled: true
bypass: false
level: 5
```

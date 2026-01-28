# Latency Assault

The `latency-assault` module in `light-chaos-monkey` allows you to inject artificial delays (latency) into your application's request processing. This is useful for testing how your application and its downstream consumers handle slow responses and potential timeouts.

## Core Components

### LatencyAssaultHandler
The middleware handler responsible for injecting latency.
*   **Behavior**: When enabled and triggered (based on the `level` configuration), it calculates a random sleep duration within the configured range and puts the current thread to sleep using `Thread.sleep()`.
*   **Trigger**: The assault is triggered based on a probability defined by the `level`.

### LatencyAssaultConfig
Configuration class that loads settings from `latency-assault.yml`.

## Configuration

The module is configured via `latency-assault.yml`.

**Key Settings:**
*   `enabled`: Enable or disable the handler (default: `false`).
*   `bypass`: If `true`, the assault is skipped even if enabled (default: `true`).
*   `level`: The probability of an attack, defined as "1 out of N requests".
    *   `level: 10` means approximately 1 in 10 requests (10%) will be attacked.
    *   `level: 1` means every request is attacked.
*   `latencyRangeStart`: The minimum delay in milliseconds (default: `1000`).
*   `latencyRangeEnd`: The maximum delay in milliseconds (default: `3000`).

**Example Configuration:**

```yaml
enabled: true
bypass: false
level: 5
latencyRangeStart: 500
latencyRangeEnd: 2000
```

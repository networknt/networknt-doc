---
description: Chaos Monkey Module
---

# Chaos Monkey

The `chaos-monkey` module in `light-chaos-monkey` allows developers and operators to inject various types of failures (assaults) into a running service to test its resilience. It provides API endpoints to query and update assault configurations dynamically at runtime.

## Core Components

### ChaosMonkeyConfig
Configuration class that loads settings from `chaos-monkey.yml`. It defines whether the chaos monkey capability is enabled globally.

### ChaosMonkeyGetHandler
A `LightHttpHandler` that retrieves the current configuration of all registered assault handlers.
*   **Endpoint**: `GET /chaosmonkey` (typically configured via `openapi.yaml` or `handler.yml`)
*   **Response**: A JSON object containing the current configurations for Exception, KillApp, Latency, and Memory assaults.

### ChaosMonkeyPostHandler
A `LightHttpHandler` that allows updating a specific assault configuration on the fly.
*   **Endpoint**: `POST /chaosmonkey?assault={handlerClassName}`
*   **Request**: `assault` query parameter specifying the target handler class (e.g., `com.networknt.chaos.LatencyAssaultHandler`), and a JSON body matching that handler's configuration structure.
*   **Behavior**: Updates the static configuration of the specified assault handler and triggers a re-registration of the module config.

## Assault Types

*   **ExceptionAssault**: Injects exceptions into request handling.
*   **KillappAssault**: Terminates the application instance (use with caution!).
*   **LatencyAssault**: Injects artificial delays (latency) into requests.
*   **MemoryAssault**: Consumes heaps memory to simulate memory pressure/leaks.

## Configuration

The module itself is configured via `chaos-monkey.yml`.

**Key Settings:**
*   `enabled`: Global switch to enable or disable the chaos monkey endpoints (default: `false`).

**Example Configuration:**

```yaml
enabled: true
```

Each assault type has its own specific configuration (e.g., `latency-assault.yml`, `exception-assault.yml`) which controls the probability and specifics of that attack.

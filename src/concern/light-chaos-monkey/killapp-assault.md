# Killapp Assault

The `killapp-assault` module in `light-chaos-monkey` allows you to inject a termination assault into your application. When triggered, it gracefully shuts down the server and then terminates the JVM process. This is the most destructive type of assault and should be used with extreme caution.

## Core Components

### KillappAssaultHandler
The middleware handler responsible for killing the application.
*   **Behavior**: When enabled and triggered (based on the `level` configuration), it calls `Server.shutdown()` to stop the light-4j server and then `System.exit(0)` to terminate the process.
*   **Safety**: Ensure that your deployment environment (e.g., Kubernetes, Docker Swarm) is configured to automatically restart the application after it terminates, otherwise the service will remain down.

### KillappAssaultConfig
Configuration class that loads settings from `killapp-assault.yml`.

## Configuration

The module is configured via `killapp-assault.yml`.

**Key Settings:**
*   `enabled`: Enable or disable the handler (default: `false`).
*   `bypass`: If `true`, the assault is skipped even if enabled (default: `true`).
*   `level`: The probability of an attack, defined as "1 out of N requests".
    *   `level: 10` means approximately 1 in 10 requests (10%) will trigger a shutdown.
    *   `level: 1` means the first request will terminate the app.

**Example Configuration:**

```yaml
enabled: true
bypass: false
level: 100
```

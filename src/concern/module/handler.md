# Handler Module

The `handler` module is the core component of the Light-4j framework responsible for managing the request/response lifecycle. It implements the Chain of Responsibility pattern, allowing developers to compose complex processing flows using small, reusable middleware components.

## core Concepts

### MiddlewareHandler
The `MiddlewareHandler` interface is the contract for all middleware plugins. It allows handlers to be chained together.

```java
public interface MiddlewareHandler extends LightHttpHandler {
    HttpHandler getNext();
    MiddlewareHandler setNext(final HttpHandler next);
    boolean isEnabled();
}
```

*   **Chaining**: Each handler holds a reference to the `next` handler in the chain.
*   **Enabling**: Handlers can be enabled or disabled via configuration.
*   **Execution**: When `handleRequest` is called, the handler performs its logic and then calls `Handler.next(exchange, next)` to pass control to the next handler.

### LightHttpHandler
The `LightHttpHandler` interface extends Undertow's `HttpHandler`. It is recommended for all business handlers to implement this interface. It provides helper methods for:
*   **Standardized Error Responses**: `setExchangeStatus` helps return consistent JSON error responses based on `status.yml`.
*   **Audit Logging**: Automatically integrates with the Audit module to log errors and stack traces if configured.

### HandlerProvider
The `HandlerProvider` interface is used by Service modules to expose a bundle of handlers.
```java
public interface HandlerProvider {
    HttpHandler getHandler();
}
```
This is often used when a library provides a set of API endpoints that need to be injected into the main application (e.g., `ServerInfoGetHandler` or `HealthGetHandler`).

## Handler Orchestration

The `Handler` class is the main orchestrator. It loads the configuration from `handler.yml` and initializes:
1.  **Handlers**: Instantiates handler classes.
2.  **Chains**: Composes lists of handlers into named execution chains.
3.  **Paths**: Maps HTTP methods and path templates to specific executor chains.

When a request arrives, `Handler.start(exchange)` matches the request path against the configured paths.
*   If a match is found, the corresponding chain ID is attached to the exchange, and execution begins.
*   If no path matches, it attempts to execute `defaultHandlers` if configured.
*   If no match and no default handlers, the request processing stops (or falls through to the next external handler).

## Configuration (`handler.yml`)

The configuration file `handler.yml` is central to defining how requests are processed.

### 1. Handlers
Defines all the handler classes used in the application. You can provide an alias using `@` to reference them easily in chains.

```yaml
handlers:
  # Format: com.package.ClassName@alias
  - com.networknt.exception.ExceptionHandler@exception
  - com.networknt.metrics.MetricsHandler@metrics
  - com.networknt.validator.ValidatorHandler@validator
  - com.example.MyBusinessHandler@myHandler
```

### 2. Chains
Defines reusable sequences of handlers. The `default` chain is commonly used for shared middleware.

```yaml
chains:
  default:
    - exception
    - metrics
    - validator
  secured:
    - exception
    - metrics
    - security
    - validator
```

### 3. Paths
Maps specific API endpoints to execution chains.

```yaml
paths:
  - path: '/v1/address'
    method: 'get'
    exec:
      - default
      - myHandler
  - path: '/v1/admin'
    method: 'post'
    exec:
      - secured
      - adminHandler
```

*   **path**: The URI template (e.g., `/v1/pets/{petId}`).
*   **method**: HTTP method (GET, POST, etc.).
*   **exec**: A list of *chains* or *handlers* to execute in order.

### 4. Default Handlers
A fallback chain executed if no path matches. Useful for 404 handling or SPA routing.

```yaml
defaultHandlers:
  - exception
  - cors
  - fileHandler
```

## Best Practices

1.  **Use Aliases**: Always use aliases (e.g., `@exception`) in `handlers` lists. It makes `chains` and `paths` configuration much more readable.
2.  **Granular Chains**: Define common middleware stacks (like `default`, `middleware`, `security`) in `chains` and reuse them in `paths`. This reduces duplication.
3.  **Order Matters**: Place `ExceptionHandler` at the very beginning of the chain to catch errors from all subsequent handlers. Place `MetricsHandler` early to capture accurate timing.

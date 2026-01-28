# RPC Router

The `rpc-router` is a core module of the `light-hybrid-4j` framework. It provides a high-performance routing and validation mechanism that enables a single server instance to host multiple, independent service handlers (RPC-style).

## Overview

In the `light-hybrid-4j` architecture, the `rpc-router` serves as the primary dispatcher for incoming requests. Unlike traditional RESTful routing based on URL paths and HTTP verbs, the RPC router typically uses a single endpoint (e.g., `/api/json`) and determines the target logic based on a `serviceId` (or `cmd`) specified in the request payload.

### Key Responsibilities:
*   **Service Discovery**: Dynamically discovering handlers annotated with `@ServiceHandler` at startup.
*   **Request Dispatching**: Mapping incoming JSON or Form payloads to the correct `HybridHandler`.
*   **Schema Validation**: Validating request payloads against JSON schemas defined in `spec.yaml` files.
*   **Specification Management**: Merging multiple `spec.yaml` files from various service JARs into a single runtime context.
*   **Configuration Management**: Providing a standardized, hot-reloadable configuration via `rpc-router.yml`.

## Core Components

### 1. `SchemaHandler`
The `SchemaHandler` is a middleware handler that performs the initial processing of RPC requests.
- **Spec Loading**: At startup, it scans the classpath for all instances of `spec.yaml` and merges them.
- **Validation**: For every request, it identifies the `serviceId`, retrieves the associated schema, and validates the `data` portion of the payload.
- **Hot Reload**: Supports refreshing the merged specifications at runtime without a server restart.

### 2. `JsonHandler`
The `JsonHandler` is the final dispatcher in the chain for JSON-based RPC calls.
- **Service Execution**: It retrieves the pre-parsed `serviceId` and `data` from the exchange attachments (populated by `SchemaHandler`) and invokes the corresponding `HybridHandler.handle()` method.

### 3. `RpcRouterConfig`
This class manages the configuration found in `rpc-router.yml`. It supports:
- **Standardized Loading**: Singleton-based access via `RpcRouterConfig.load()`.
- **Hot Reload**: Thread-safe configuration refreshing via `RpcRouterConfig.reload()`.
- **Module Info**: Automatic registration with the `ModuleRegistry` for runtime monitoring.

### 4. `RpcStartupHookProvider`
A startup hook that uses **ClassGraph** to scan configured packages for any class implementing `HybridHandler` and bearing the `@ServiceHandler` annotation.

## Configuration (`rpc-router.yml`)

| Property | Default | Description |
| --- | --- | --- |
| `handlerPackages` | `[]` | List of package prefixes to scan for service handlers. |
| `jsonPath` | `/api/json` | The endpoint for JSON-based RPC requests. |
| `formPath` | `/api/form` | The endpoint for Form-based RPC requests. |
| `registerService` | `false` | If enabled, registers each discovered service ID with the discovery registry (e.g., Consul). |

## Request Structure

A typical RPC request to the `rpc-router` looks as follows:

```json
{
  "host": "lightapi.net",
  "service": "petstore",
  "action": "getPetById",
  "version": "1.0.0",
  "data": {
    "id": 123
  }
}
```

The router constructs the internal `serviceId` as `host/service/action/version` (e.g., `lightapi.net/petstore/getPetById/1.0.0`) to locate the handler.

## Implementation Example

### 1. The Service Handler
```java
@ServiceHandler(id = "lightapi.net/petstore/getPetById/1.0.0")
public class GetPetById implements HybridHandler {
    @Override
    public ByteBuffer handle(HttpServerExchange exchange, Object data) {
        Map<String, Object> params = (Map<String, Object>) data;
        // Business logic...
        return NioUtils.toByteBuffer("{\"id\": 123, \"name\": \"Fluffy\"}");
    }
}
```

### 2. The Specification (`spec.yaml`)
Place this in `src/main/resources` of your service module:
```yaml
host: lightapi.net
service: petstore
action:
  - name: getPetById
    version: 1.0.0
    handler: getPetById
    request:
      schema:
        type: object
        properties:
          id: { type: integer }
        required: [id]
```

## Best Practices

1.  **Restrict Scanning**: Always specify `handlerPackages` in `rpc-router.yml` to minimize startup time.
2.  **Contract-First**: Define your `spec.yaml` rigorously. The router uses these schemas to protect your handlers from invalid data.
3.  **Hot Reload**: Use the `/server/info` endpoint to verify that your configuration and specifications have been updated correctly after a reload.

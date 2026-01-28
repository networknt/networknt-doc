# GraphQL Validator

The `graphql-validator` module is a middleware handler for `light-graphql-4j` that validates incoming GraphQL requests before they reach the execution engine. It ensures that requests conform to basic GraphQL protocol requirements.

## Core Components

### ValidatorHandler
The `ValidatorHandler` is the primary middleware. It checks:
1.  **Path**: Ensures the request path matches the configured GraphQL path (default `/graphql`) or subscription path.
2.  **Method**: Verifies that the HTTP method is one of `GET`, `POST`, or `OPTIONS` (for introspection/CORS).
3.  **Body/Query**:
    *   For `GET` requests, it validates that query parameters exist.
    *   For `POST` requests, it parses the JSON body to ensure it is a valid GraphQL request structure.

If any validation fails, it returns an appropriate error code (e.g., `ERR11500` for invalid path, `ERR11501` for invalid method).

### ValidatorConfig
The configuration class backing the validator. It maps to `graphql-validator.yml`.

**Properties:**

| Property | Default | Description |
| :--- | :--- | :--- |
| `enabled` | `true` | Enables or disables the validation middleware. |
| `logError` | `true` | If true, logs validation errors to the server log. |

**Features:**
*   **Singleton Pattern**: Access via `ValidatorConfig.load()`.
*   **Hot Reload**: Supports dynamic reloading via `ValidatorConfig.reload()`.
*   **Module Registry**: Automatic registration for runtime observability.

## Configuration

**graphql-validator.yml**
```yaml
enabled: true
logError: true
```

## Usage

In your `handler.yml`, place the `ValidatorHandler` early in the chain, typically before the GraphQL execution handler but after basic security or CORS handlers.

```yaml
handlers:
  - com.networknt.graphql.validator.ValidatorHandler@validator

paths:
  - path: '/graphql'
    method: 'get'
    handler:
      - validator
      - graphql
  - path: '/graphql'
    method: 'post'
    handler:
      - validator
      - graphql
```

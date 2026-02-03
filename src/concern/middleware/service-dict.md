# Service Dict Handler

The `ServiceDictHandler` is a middleware handler that provides a flexible way to map request paths and methods to `serviceIds`. It bridges the gap between `PathPrefixServiceHandler` (simple prefix mapping) and `PathServiceHandler` (exact Swagger endpoint mapping).

## Introduction

In a microservices gateway, routing requests purely based on path prefixes might be insufficient, and requiring a full OpenAPI specification for `PathServiceHandler` might be overkill or not feasible for some legacy services.

`ServiceDictHandler` offers a middle ground. It allows you to define mappings using a combination of **Path Prefix** and **HTTP Method**. This enables you to route requests to different services based on the operation (e.g., READs go to one service, WRITEs to another) without needing a full API definition.

## Logic Flow

1.  **Intercept Request**: The handler captures the incoming request's URI and HTTP Method.
2.  **Lookup Mapping**: It normalizes the request to key format `pathPrefix@method` and searches for a match in the configuration.
3.  **Inject Header**:
    *   If a match is found, it injects the corresponding `serviceId` into the `service_id` header.
    *   It also populates the audit attachment with the matched endpoint Pattern.
4.  **No Match**: If no mapping is found, it marks the endpoint as `Unknown` in the audit logs.

## Configuration

The handler is configured via the `serviceDict.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `true` |
| `mapping` | A map key-value pair of `pathPrefix@method` -> `serviceId`. | `{}` |

### Configuration Format

The key format in the mapping is `[PathPrefix]@[Method]`. The method should be lowercase.

**Example `serviceDict.yml`**

```yaml
# Service Dictionary Configuration
enabled: true

# Mapping: PathPrefix@Method -> Service ID
mapping:
  # Route all GET requests under /v1/address to address-read service
  /v1/address@get: party.address-read-1.0.0
  
  # Route all POST requests under /v1/address to address-write service
  /v1/address@post: party.address-write-1.0.0
  
  # Route all requests (any method) under /v2/address to address-v2 service
  # Note: logic depends on specific implementation match order if keys overlap
  /v2/address@get: party.address-v2-1.0.0
```

### Supported Formats

Like other handlers, the mapping can be provided as:
1.  **YAML Map**: Standard `key: value` structure.
2.  **JSON String**: `{"/v1/data@get": "service-1"}`.
3.  **Java Map String**: `/v1/data@get=service-1&/v1/data@post=service-2`.

## Usage

To use this handler, register it in `handler.yml` and place it before the `RouterHandler`.

### `handler.yml`

```yaml
handlers:
  - com.networknt.router.middleware.ServiceDictHandler@dict
  # ... other handlers

chains:
  default:
    - correlation
    - dict   # Perform lookup here
    - router
```

## Comparison

| Handler | Matching key | Dependency | Use Case |
| :--- | :--- | :--- | :--- |
| **PathPrefixServiceHandler** | Path Prefix (`/v1/data`) | None | Simple routing where one path = one service. |
| **ServiceDictHandler** | Path + Method (`/v1/data@get`) | None | Routing based on Method (CQRS-style) without Swagger. |
| **PathServiceHandler** | Exact Endpoint (`/v1/data/{id}@get`) | OpenAPI/Swagger | Precise routing for RESTful APIs with specs. |

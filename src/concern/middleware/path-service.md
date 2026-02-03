# Path Service Handler

The `PathServiceHandler` is a middleware handler used to discover the `serviceId` based on the request endpoint (Method + Path). It is similar to the `PathPrefixServiceHandler` but offers finer granularity by leveraging the endpoint information from the audit attachment (populated by OpenAPI/Swagger handlers).

## Introduction

In a microservices architecture, the router needs to know the `serviceId` to route the request to the correct downstream service. While `PathPrefixServiceHandler` uses a simple path prefix, `PathServiceHandler` uses the **exact endpoint** (e.g., `/v1/pets/{petId}@GET`) to map to a `serviceId`.

This is particularly useful when:
1.  You have multiple services sharing the same path prefix but serving different endpoints.
2.  You want to route specific endpoints to different services (e.g., read vs. write separation).

> **Prerequisite**: This handler typically comes **after** the `OpenApiHandler` (or `SwaggerHandler`) in the chain because it relies on the Audit Info attachment (`auditInfo`) where the normalized endpoint string is stored.

## Logic Flow

1.  **Check Service ID**: It first checks if the `service_id` header is already present. If so, it skips logic.
2.  **Retrieve Endpoint**: It retrieves the `endpoint` string from the Audit Info attachment (e.g., `/v1/pets/123@get` -> normalized to `/v1/pets/{petId}@get`).
3.  **Lookup Mapping**: It looks up this endpoint in the configured `mapping`.
4.  **Inject Header**: If a match is found, it injects the mapped `serviceId` into the `service_id` header.

## Configuration

The handler is configured via the `pathService.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `true` |
| `mapping` | A map where the key is the endpoint string and the value is the `serviceId`. | `{}` |

### Configuration Format

The `mapping` can be defined in YAML, JSON, or Java Map string format.

**Endpoint Format**: `[path]@[method]` (method must be lowercase).

**Example `pathService.yml`**:

```yaml
# Path Service Configuration
enabled: true

# Mapping: Endpoint -> Service ID
mapping:
  /v1/address/{id}@get: party.address-1.0.0
  /v2/address@get: party.address-2.0.0
  /v1/contact@post: party.contact-1.0.0
```

## Usage

To use this handler, register it in `handler.yml` and place it **after** the `OpenApiHandler` and **before** the `RouterHandler`.

### `handler.yml`

```yaml
handlers:
  - com.networknt.router.middleware.PathServiceHandler@path
  # ... other handlers

chains:
  default:
    - correlation
    - openapi-handler
    - path   # Place after OpenAPI and before Router
    - router
```

## Comparison

| Feature | PathPrefixServiceHandler | PathServiceHandler |
| :--- | :--- | :--- |
| **Matching** | Path Prefix (`/v1/pets`) | Exact Endpoint (`/v1/pets/{id}@get`) |
| **Dependency** | None | `OpenApiHandler` (for parameter validation & normalization) |
| **Granularity** | Coarse (Service per prefix) | Fine (Service per endpoint) |
| **Performance** | Faster | Slightly slower (depends on OpenAPI parsing) |

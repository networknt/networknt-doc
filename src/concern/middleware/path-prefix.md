# Path Prefix Service Handler

The `PathPrefixServiceHandler` is a middleware handler primarily used in the [Light-Gateway](https://github.com/networknt/light-gateway) or [http-sidecar](https://github.com/networknt/http-sidecar). It maps a request path prefix to a `serviceId`, allowing the router to discover and invoke the correct downstream service without requiring the caller to provide the `service_id` header or perform full OpenAPI validation.

## Introduction

In the Light-4j architecture, the [Router Handler](../handler/router-handler.md) typically relies on a `service_id` header to locate the target service in the service registry (like Consul).

The `PathPrefixServiceHandler` simplifies this by enabling path-based routing. If your downstream services have unique path prefixes (e.g., `/v1/users` -> `user-service`, `/v1/orders` -> `order-service`), this handler can automatically lookup the `serviceId` based on the request URL and inject it into the request header.

## Logic Flow

1.  **Intercept Request**: The handler intercepts the incoming request.
2.  **Match Prefix**: It checks the request path against the configured `mapping` table.
3.  **Inject Header**:
    *   If a matching prefix is found, it retrieves the corresponding `serviceId`.
    *   It injects this `serviceId` into the `service_id` header (if not already present).
    *   It populates the `endpoint` in the audit data for logging/metrics.
4.  **No Match**: If no match is found, it marks the endpoint as `Unknown` in the audit data but allows the request to proceed (best-effort).

## Configuration

The handler is configured via the `pathPrefixService.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `true` |
| `mapping` | A map where the key is the path prefix and the value is the `serviceId`. | `{}` |

### Example `pathPrefixService.yml`

```yaml
# Path Prefix Service Configuration
enabled: true

# Mapping: Path Prefix -> Service ID
mapping:
  /v1/address: party.address-1.0.0
  /v2/address: party.address-2.0.0
  /v1/contact: party.contact-1.0.0
  /v1/pets: petstore-3.0.1
```

### Configuration Formats

The `mapping` can be defined in multiple formats to support different configuration sources (like standard config files or a config server).

**1. YAML Format (Standard):**
```yaml
mapping:
  /v1/address: party.address-1.0.0
  /v2/address: party.address-2.0.0
```

**2. JSON Format (String):**
Useful for environment variables or config servers that expect strings.
```yaml
mapping: '{"/v1/address": "party.address-1.0.0", "/v2/address": "party.address-2.0.0"}'
```

**3. Java Map Format (String):**
Key-value pairs separated by `&`.
```yaml
mapping: /v1/address=party.address-1.0.0&/v2/address=party.address-2.0.0
```

## Usage

To use this handler, register it in `handler.yml` and place it **before** the `RouterHandler`.

### `handler.yml`

```yaml
handlers:
  - com.networknt.router.middleware.PathPrefixServiceHandler@prefix
  # ... other handlers

chains:
  default:
    - correlation
    - metrics
    - prefix   # Place before router
    - router
```

> **Note:** Do not map system endpoints like `/health` or `/server/info` in this handler, as they are common to all services.

## Comparison with PathServiceHandler

*   **PathPrefixServiceHandler**: Simple prefix matching. Does not require OpenAPI specifications. Fast and lightweight. Best for simple routing requirements.
*   **PathServiceHandler**: Requires OpenAPI handlers. Can perform more complex matching based on specific endpoints (Method + Path).

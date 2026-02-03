# Token Handler

The `TokenHandler` is a middleware handler used in the [Light-Gateway](https://github.com/networknt/light-gateway) or [http-sidecar](https://github.com/networknt/http-sidecar). Its primary responsibility is to fetch an OAuth 2.0 access token (Client Credentials Grant) from an authorization server on behalf of the client and inject it into the request header.

## Introduction

In a microservices environment, a service often needs to call other services securely. Instead of implementing token retrieval logic in every service (or when using a language without a mature OAuth client), you can offload this responsibility to the gateway or sidecar.

The `TokenHandler` intercepts outgoing requests, checks if a valid token is cached, and if not, retrieves a new one from the configured OAuth provider derived from `client.yml`. It then attaches this token to the request so the downstream service can authenticate the call.

## Logic Flow

1.  **Resolve Service ID**: The handler first looks for the `service_id` header. This header is typically populated by upstream handlers like:
    *   [PathPrefixServiceHandler](path-prefix.md)
    *   [ServiceDictHandler](service-dict.md)
    *   [PathServiceHandler](path-service.md)
2.  **Filter by Path**: It checks if the request path matches any of the configured `appliedPathPrefixes`. If not, it skips execution.
3.  **Fetch Token**:
    *   It checks an internal cache for a valid JWT associated with the `serviceId`.
    *   If missing or expired, it initiates a Client Credentials Grant request to the OAuth provider defined in `client.yml`.
4.  **Inject Token**:
    *   **Standard**: If the `Authorization` header is empty, it sets `Authorization: Bearer <jwt>`.
    *   **On-Behalf-Of**: If the `Authorization` header already contains a token (e.g., the original user's token), it preserves it and puts the new "scope token" (for service-to-service calls) in the `X-Scope-Token` header.

## Configuration

The handler is configured via `token.yml` for enabling/filtering and `client.yml` for OAuth provider details.

### `token.yml`

```yaml
# Token Handler Configuration
enabled: true

# List of path prefixes where this handler should be active.
# This allows you to selectively apply token injection only for specific routes.
appliedPathPrefixes:
  - /v1/pets
  - /v2/orders
```

### `client.yml`

The OAuth 2.0 provider configuration is managed by the client module.

```yaml
oauth:
  # Enable multiple auth servers if you need different credentials for different services
  multipleAuthServers: true
  token:
    # Default Client Credentials Config
    client_credentials:
      clientId: default_client
      clientSecret: default_secret
      uri: /oauth2/token
    # Service specific overrides (mapped by serviceId)
    serviceIdAuthServers:
      petstore-service:
        clientId: petstore_client
        clientSecret: petstore_secret
        uri: /oauth2/token
```

## Usage

To use this handler, register it in `handler.yml`. It must be placed **after** any handler that resolves the `serviceId` (like `PathPrefixServiceHandler`) and **before** the `RouterHandler`.

### `handler.yml`

```yaml
handlers:
  - com.networknt.router.middleware.TokenHandler@token
  # ... other handlers

chains:
  default:
    - correlation
    - prefix   # Resolves service_id from path
    - token    # Uses service_id to get JWT
    - router   # Forwards request to downstream
```

## Error Handling

If the `service_id` cannot be resolved (because the upstream handler is missing or failed), the `TokenHandler` will return an error `ERR10074` indicating a dependency error.

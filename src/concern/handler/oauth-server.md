# OAuth Server Handler

The `OAuthServerHandler` is a specialized handler designed to simulate an OAuth 2.0 provider endpoint (e.g., `/oauth/token`). Its primary purpose is to facilitate the migration of legacy applications that expect to exchange client credentials for a token, without requiring immediate code changes in those applications.

## Introduction

In a typical migration scenario, you might have legacy clients that are hardcoded to call a specific token endpoint to get an access token before calling an API. The `OAuthServerHandler` intercepts these requests at the gateway.

It supports the `client_credentials` grant type and can operate in two modes:
1.  **Dummy Mode**: Returns a generated dummy token. This is useful when the gateway itself handles authentication/authorization using a different mechanism (e.g., Mutual TLS or a centralized token) and the client just needs *some* token to proceed.
2.  **Pass-Through Mode**: Validates the credentials locally and then fetches a **real** JWT from a backend OAuth 2.0 provider (using `TokenService` configuration) and returns it to the client.

## Configuration

The handler is configured via the `oauthServer.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the handler. | `true` |
| `getMethodEnabled` | Allow token requests via HTTP GET (for legacy support). **Insecure**. | `false` |
| `client_credentials` | A list of valid `clientId:clientSecret` pairs for validation. | `[]` |
| `passThrough` | If `true`, fetches a real token from a downstream provider. If `false`, returns a dummy token. | `false` |
| `tokenServiceId` | The Service ID (in `client.yml`) of the real OAuth provider. Used only if `passThrough` is true. | `light-proxy-client` |

### Example `oauthServer.yml`

```yaml
# OAuth Server Configuration
enabled: true
getMethodEnabled: false
passThrough: true
tokenServiceId: aad-token-service

# List of valid credentials (clientId:clientSecret)
client_credentials:
  - 3848203948:2881938882
  - client_App1:pass1234
```

## Handlers

### OAuthServerHandler
*   **Path**: Typically mapped to `/oauth2/token` or similar.
*   **Method**: `POST`
*   **Content-Type**: `application/json`, `application/x-www-form-urlencoded`, `multipart/form-data`.
*   **Logic**:
    *   Extracts `client_id` and `client_secret` from the request body OR `Authorization: Basic` header.
    *   Validates them against the `client_credentials` list.
    *   If valid, returns a JSON response with `access_token`, `token_type`, and `expires_in`.

### OAuthServerGetHandler
*   **Path**: Configurable (e.g., `/oauth2/token`).
*   **Method**: `GET`
*   **Logic**:
    *   Extracts credentials from Query Parameters (`client_id`, `client_secret`).
    *   **Warning**: This is highly insecure as secrets are exposed in the URL. Use only for legacy migration where absolutely necessary.
    *   Requires `getMethodEnabled: true` in config.

## Usage

To use these handlers, register them in `handler.yml`.

### `handler.yml`

```yaml
paths:
  - path: '/oauth2/token'
    method: 'POST'
    exec:
      - com.networknt.router.OAuthServerHandler
  - path: '/oauth2/token'
    method: 'GET'
    exec:
      - com.networknt.router.OAuthServerGetHandler
```

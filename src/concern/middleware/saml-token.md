# SAML Token Handler

The `SAMLTokenHandler` is a middleware handler designed to support the **SAML 2.0 Bearer Assertion Grant** type (RFC 7522). It allows a client to exchange a SAML 2.0 assertion (and optionally a JWT client assertion) for an OAuth 2.0 access token (JWT) from an authorization server.

## Introduction

In some enterprise environments, legacy identity providers (IdPs) typically issue SAML assertions. However, modern microservices usually require JWTs for authorization. This handler bridges the gap by intercepting requests containing a SAML assertion, exchanging it for a JWT with the authorization server, and then replacing the SAML assertion with the JWT in the `Authorization` header for downstream services.

## Logic Flow

1.  **Intercept Request**: The handler looks for specific headers in the incoming request:
    *   `assertion`: Contains the Base64-encoded SAML 2.0 assertion.
    *   `client_assertion` (Optional): Contains a JWT for client authentication (if required).
2.  **Exchange Token**: It constructs a simplified token request using these assertions and sends it to the configured OAuth 2.0 provider (via `OauthHelper`).
3.  **Process Response**:
    *   **Success**: If the IdP returns a valid access token (JWT), the handler:
        *   Sets the `Authorization` header to `Bearer <access_token>`.
        *   Removes the `assertion` and `client_assertion` headers to clean up the request.
        *   Passes the request to the next handler in the chain.
    *   **Failure**: If the exchange fails, it returns an error response (typically with the status code from the IdP) and terminates the request.

## Configuration

This handler shares configuration with the `TokenHandler` and relies on `client.yml` for OAuth provider details.

### `token.yml`

The handler's enabled status is controlled by the `token.yml` configuration (shared with `TokenHandler`).

```yaml
# Token Handler Configuration
enabled: true
```

### `client.yml`

The details for connecting to the OAuth provider are defined in the `client.yml` file under the `oauth` section.

```yaml
oauth:
  token:
    # URL of the OAuth 2.0 token endpoint
    authorization_code_url: https://localhost:6882/oauth/token
    # Other standard client config...
```

### `security.yml`

HTTP/2 support for the connection to the OAuth provider can be toggled in `security.yml`.

```yaml
# Enable HTTP/2 for OAuth Token Request
oauthHttp2Support: true
```

## Usage

To use this handler, register it in `handler.yml` and place it **before** any handlers that require a valid JWT (like `JwtVerifyHandler`) and usually before the proxy/router handler.

### `handler.yml`

```yaml
handlers:
  - com.networknt.router.middleware.SAMLTokenHandler@saml
  # ... other handlers

chains:
  default:
    - correlation
    - saml   # Exchanges SAML for JWT here
    - router # Forwards request with JWT
```

## Headers

| Header Name | Required | Description |
| :--- | :--- | :--- |
| `assertion` | Yes | The Base64-encoded SAML 2.0 assertion (xml). |
| `client_assertion` | No | A JWT used for client authentication (if using `private_key_jwt` auth). |

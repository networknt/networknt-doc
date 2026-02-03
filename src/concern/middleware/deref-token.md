# DeRef Token Middleware

The `DerefMiddlewareHandler` is designed to handle "By Reference" tokens (opaque tokens) at the edge of your microservices architecture (e.g., in a BFF or Gateway). It exchanges these opaque tokens for "By Value" tokens (JWTs) by communicating with an OAuth 2.0 provider.

## Introduction

In some security architectures, organizations prefer not to expose JWTs (which contain claims and potentially sensitive information) to public clients (browsers, mobile apps). Instead, they issue opaque "Reference Tokens".

However, internal microservices typically rely on JWTs for stateless authentication and authorization. The `DerefMiddlewareHandler` bridges this gap by intercepting requests with reference tokens and "dereferencing" them into JWTs before passing the request to downstream services.

## Logic Flow

The handler performs the following steps:

1.  **Check Header**: It looks for the `Authorization` header.
2.  **Validate Existence**: If the header is missing, it returns an error (`ERR10002`).
3.  **Format Check**:
    *   It checks if the token contains a dot (`.`).
    *   **If existing dot**: It assumes the token is already a JWT (Signed JWTs always have 3 parts separated by dots). The handler ignores it and passes the request to the next handler.
    *   **If no dot**: It treats the token as a Reference Token.
4.  **Dereference**:
    *   It calls the OAuth 2.0 provider (via `OauthHelper`) to exchange the reference token for a JWT.
    *   If the exchange fails or returns an error, it terminates the request with an appropriate error status (`ERR10044` or `ERR10045`).
    *   If successful, it **replaces** the `Authorization` header in the current request with the new `Bearer <JWT>`.
5.  **Proceed**: The request continues to the next handler in the chain with the valid JWT.

## Configuration

The handler is configured via the `deref.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Indicates if the middleware is enabled. | `false` |

### Example `deref.yml`

```yaml
# Dereference Token Middleware Configuration
enabled: true
```

> **Note:** The actual connection details for the OAuth 2.0 provider (URL, credentials) are typically handled by the `client` module configuration and `OauthHelper` internal settings, not directly in `deref.yml`.

## Usage

To use this handler, register it in `handler.yml` and place it **before** the `JwtVerifyHandler` or any other handler that requires a valid JWT.

### `handler.yml`

```yaml
handlers:
  - com.networknt.deref.DerefMiddlewareHandler@deref
  # ... other handlers

chains:
  default:
    - deref
    - security # (JwtVerifyHandler)
    # ... other handlers
```

## Error Codes

-   `ERR10002`: MISSING_AUTH_TOKEN - The `Authorization` header is missing.
-   `ERR10044`: EMPTY_TOKEN_DEREFERENCE_RESPONSE - The OAuth provider returned an empty response.
-   `ERR10045`: TOKEN_DEREFERENCE_ERROR - The OAuth provider returned an error message.

# MSAL Exchange Handler

The `msal-exchange` module in `light-spa-4j` provides a handler to exchange Microsoft Authentication Library (MSAL) tokens for internal application session cookies. This mechanism effectively serves as a Backend-For-Frontend (BFF) authentication layer for Single Page Applications (SPAs).

## Core Components

### MsalTokenExchangeHandler
This middleware handler intercepts requests to specific paths (configured via `msal-exchange.yml`) to perform token exchange or logout operations.
*   **Token Exchange**: Validates the incoming Microsoft Bearer token, performs a token exchange via the OAuth provider, and sets secure, HTTP-only session cookies (`accessToken`, `refreshToken`, `csrf`, etc.) for subsequent requests.
*   **Logout**: Clears all session cookies to securely log the user out.
*   **Session Management**: On subsequent requests, it validates the JWT in the cookie, checks for CSRF token consistency, and handles automatic token renewal if the session is nearing expiration.

### MsalExchangeConfig
Configuration class that loads settings from `msal-exchange.yml`. It supports hot reloading and module registration.

## Configuration

The module is configured via `msal-exchange.yml`.

**Key Settings:**
*   `enabled`: Enable or disable the handler.
*   `exchangePath`: Partial path for triggering token exchange (default: `/auth/ms/exchange`).
*   `logoutPath`: Partial path for triggering logout (default: `/auth/ms/logout`).
*   `cookieDomain` / `cookiePath`: Scope configuration for the session cookies.
*   `cookieSecure`: Whether to mark cookies as Secure (HTTPS only).
*   `sessionTimeout`: Max age for the session cookies.

**Example Configuration:**

```yaml
enabled: true
exchangePath: /auth/ms/exchange
logoutPath: /auth/ms/logout
cookieDomain: localhost
cookiePath: /
cookieSecure: false
sessionTimeout: 3600
rememberMeTimeout: 604800
```

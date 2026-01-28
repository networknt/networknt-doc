# Stateless Auth Handler

The `stateless-auth` module in `light-spa-4j` provides a robust, stateless authentication mechanism for Single Page Applications (SPAs). It handles the OAuth 2.0 Authorization Code flow and manages the resulting tokens using secure, HTTP-only cookies, eliminating the need for client-side storage (like `localStorage`).

## Core Components

### StatelessAuthHandler
This middleware implements the Backend-For-Frontend (BFF) pattern:
*   **Authorization**: Intercepts requests to `/authorization`, exchanges the authorization code for access and refresh tokens from the OAuth 2.0 provider.
*   **Cookie Management**: Stores tokens in secure, HTTP-only cookies (`accessToken`, `refreshToken`) and exposes non-sensitive user info (id, roles) in JavaScript-readable cookies.
*   **CSRF Protection**: Generates and validates Double Submit Cookies (CSRF token in header vs. JWT claim) to prevent Cross-Site Request Forgery.
*   **Token Renewal**: Automatically renews expiring access tokens using the refresh token when the session is nearing timeout (default check at < 1.5 minutes remaining).
*   **Logout**: Handles requests to `/logout` by invalidating all session cookies.

### StatelessAuthConfig
Configuration class that loads settings from `statelessAuth.yml`. Supports hot reloading and module registration.

## Configuration

The module is configured via `statelessAuth.yml`.

**Key Settings:**
*   `enabled`: Enable or disable the handler.
*   `authPath`: Path to handle the authorization code callback (default: `/authorization`).
*   `cookieDomain`: Domain for the session cookies (e.g., `localhost` or your domain).
*   `cookiePath`: Path scope for cookies (default: `/`).
*   `cookieSecure`: Set to `true` for HTTPS environments.
*   `sessionTimeout`: Expiration time for session cookies.
*   `redirectUri`: Where to redirect the SPA after successful login.
*   `enableHttp2`: Whether to use HTTP/2 for backend token calls.

**Social Login Support:**
The configuration also includes sections for configuring social login providers directly if not federating through a central IdP:
*   `googlePath`, `googleClientId`, `googleRedirectUri`
*   `facebookPath`, `facebookClientId`
*   `githubPath`, `githubClientId`

# WebSocket Security Design

WebSockets present unique security challenges compared to standard REST APIs, primarily because the browser's `WebSocket` API does not support custom HTTP headers (like `X-CSRF-TOKEN` or `Authorization`) during the initial connection handshake.

This document outlines the security architecture for WebSockets in the `light-spa-4j` framework, focusing on authentication and CSRF protection.

## Architectual Overview

1.  **Handshake Authentication**: The BFF (Backend-for-Frontend), such as `light-gateway`, validates the request using secure, `HttpOnly` cookies (`accessToken`).
2.  **CSRF Protection**: To prevent CSRF attacks on the handshake request, the BFF requires a matching CSRF token.
3.  **Backend Proxying**: Once authenticated, the BFF establishes a secure server-to-server WebSocket connection to the backend service, propagating claims via standard headers.

## The CSRF Challenge

The standard Double-Submit Cookie pattern for CSRF protection requires the client to send a token in a cookie AND the same token in a custom header. The server compares them to ensure the request originated from a trusted source.

Since the browser `WebSocket` constructor does not allow custom headers, we use the **Sec-WebSocket-Protocol** (Subprotocols) as a secure side-channel.

## Proposed Solution: Subprotocol Side-Channel

### 1. Frontend Implementation (`Chat.tsx`)
The frontend retrieves the `csrf` token from the standard cookie and passes it as a subprotocol string, prefixed with `csrf.`.

```typescript
const csrfToken = cookies.get('csrf');
const protocols = csrfToken ? [`csrf.${csrfToken}`] : [];

// Initializing WebSocket with the token in subprotocols
const socket = new WebSocket(url.toString(), protocols);
```

### 2. BFF Middleware (`StatelessAuthHandler`)
The BFF middleware (e.g., `StatelessAuthHandler` or `MsalTokenExchangeHandler`) extracts the token from the `Sec-WebSocket-Protocol` header during the handshake.

```java
// StatelessAuthHandler.java
String headerCsrf = exchange.getRequestHeaders().getFirst("Sec-WebSocket-Protocol");
if (headerCsrf != null && headerCsrf.startsWith("csrf.")) {
    headerCsrf = headerCsrf.substring(5); // Remove "csrf." prefix
}

// Proceed to compare with token from JWT/Cookie
if (!headerCsrf.equals(jwtCsrf)) {
    throw new Exception("CSRF Validation Failed");
}
```

## Security Advantages
- **No URL Logging**: Unlike query parameters, the `Sec-WebSocket-Protocol` header is not part of the URL and is not recorded in standard web server access logs or browser history.
- **Double-Submit Security**: Maintains the security profile of the existing REST-based CSRF protection.
- **TLS Protection**: The entire handshake is protected by TLS (WSS).

## Backend Propagation
When the BFF proxies the connection to the backend (e.g., `llmchat-server`), it acts as a standard HTTP client. It can then:
- Attach a standard `Authorization: Bearer <JWT>` header.
- Propagate User IDs and other claims via custom `X-` headers.
- Since it is a server-to-server connection, it is not restricted by browser API limitations.

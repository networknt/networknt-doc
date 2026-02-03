# CORS Handler

The `CorsHttpHandler` is a middleware handler that manages Cross-Origin Resource Sharing (CORS) headers. It handles both pre-flight `OPTIONS` requests and actual HTTP requests, ensuring that browsers can securely interact with your API from different origins.

## Introduction

Single Page Applications (SPAs) often run on a different domain or port than the API server they consume. Browsers enforce the Same-Origin Policy, which restricts resources from being loaded from a different origin. To allow this, the API server must implement the CORS protocol.

The `CorsHttpHandler` simplifies this by:
1.  **Handling Pre-flight Requests**: Automatically responding to `OPTIONS` requests with the correct `Access-Control-Allow-*` headers.
2.  **Handling Actual Requests**: Injecting the necessary CORS headers into standard responses so the browser accepts the result.
3.  **Path-Specific Configuration**: allowing different CORS rules for different API paths (useful in gateway scenarios).

## Configuration

The handler is configured via the `cors.yml` file.

| Config Field | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Indicates if the CORS middleware is enabled. | `true` |
| `allowedOrigins` | A list of allowed origins (e.g., `http://localhost:3000`). Wildcards are not supported for security reasons. | `[]` |
| `allowedMethods` | A list of allowed HTTP methods (e.g., `GET`, `POST`, `PUT`, `DELETE`). | `[]` |
| `pathPrefixAllowed` | A map allowing granular configuration per path prefix. Overrides global settings if a path matches. | `null` |

### Example `cors.yml`

```yaml
# CORS Handler Configuration
enabled: true

# Global Configuration
# Allowed origins. Wildcards (*) are NOT supported for security.
allowedOrigins:
  - http://localhost:3000
  - https://my-app.com

# Allowed methods
allowedMethods:
  - GET
  - POST
  - PUT
  - DELETE
  - OPTIONS

# Path-Specific Configuration (Optional)
# Use this if you are running a Gateway and need different rules for different downstream services.
pathPrefixAllowed:
  /v1/pets:
    allowedOrigins:
      - https://petstore.com
    allowedMethods:
      - GET
  /v1/market:
    allowedOrigins:
      - https://market.com
    allowedMethods:
      - POST
```

## Usage

To use the `CorsHttpHandler`, register it in your `handler.yml` configuration file and add it to the middleware chain.

### `handler.yml`

```yaml
handlers:
  - com.networknt.cors.CorsHttpHandler@cors
  # ... other handlers

chains:
  default:
    - cors
    # ... other handlers
```

## Logic

The handler operates with the following logic:

1.  **Check Request**: It determines if the incoming request is a CORS request (contains `Origin` header).
2.  **Match Config**:
    *   It checks `pathPrefixAllowed` first. If the request path matches a configured prefix, it uses that specific configuration.
    *   Otherwise, it uses the global `allowedOrigins` and `allowedMethods`.
3.  **Process Pre-flight (OPTIONS)**:
    *   If the request is an `OPTIONS` request, it validates the `Origin` and `Access-Control-Request-Method`.
    *   It sets the `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Allow-Credentials`, and `Access-Control-Max-Age` headers.
    *   It returns a `200 OK` response immediately, stopping the chain.
4.  **Process Actual Request**:
    *   For non-`OPTIONS` requests, it validates the `Origin`.
    *   It adds the `Access-Control-Allow-Origin` and `Access-Control-Allow-Credentials` headers to the response.
    *   It then passes control to the next handler in the chain.

### Origin Matching
The handler performs strict origin matching. If the `Origin` header matches one of the `allowedOrigins`, that specific origin is reflected in the `Access-Control-Allow-Origin` header. If it does not match, the request may be rejected (403) or handled without CORS headers depending on the flow.

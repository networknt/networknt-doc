# Unified Security

The `UnifiedSecurityHandler` is a powerful middleware that consolidates various security mechanisms into a single handler. It is designed to simplify security configuration, especially in complex environments like a shared light-gateway, where different paths might require different authentication methods.

By using the `UnifiedSecurityHandler`, you avoid chaining multiple security-specific handlers (like `BasicAuthHandler`, `JwtVerifyHandler`, `ApiKeyHandler`) and can manage all security policies in one place.

## Configuration

The configuration for this handler is defined in `unified-security.yml`.

### Example unified-security.yml

```yaml
# Unified security configuration.
---
# Indicate if this handler is enabled.
enabled: ${unified-security.enabled:true}

# Anonymous prefixes configuration. Request paths starting with these prefixes bypass all security.
anonymousPrefixes: ${unified-security.anonymousPrefixes:}

# Path prefix security configuration.
pathPrefixAuths:
  - prefix: /api/v1
    basic: false
    jwt: true
    sjwt: false
    swt: false
    apikey: false
    jwkServiceIds: com.networknt.petstore-1.0.0
  - prefix: /api/v2
    basic: true
    jwt: true
    apikey: true
    jwkServiceIds: service1,service2
```

### Configuration Parameters

| Parameter | Description |
| --- | --- |
| `enabled` | Whether the `UnifiedSecurityHandler` is active. |
| `anonymousPrefixes` | A list of path prefixes that do not require any authentication. |
| `pathPrefixAuths` | A list of configurations defining security policies for specific path prefixes. |

#### Path Prefix Auth Fields

| Field | Description |
| --- | --- |
| `prefix` | The request path prefix this policy applies to. |
| `basic` | Enable Basic Authentication verification. |
| `jwt` | Enable standard JWT verification (with scope check). |
| `sjwt` | Enable Simple JWT verification (signature check only, no scope check). |
| `swt` | Enable Shared Secret Token (SWT) verification. |
| `apikey` | Enable API Key verification. |
| `jwkServiceIds` | Comma-separated list of service IDs for JWK lookup (used if `jwt` is true). |
| `sjwkServiceIds` | Comma-separated list of service IDs for JWK lookup (used if `sjwt` is true). |
| `swtServiceIds` | Comma-separated list of service IDs for introspection servers (used if `swt` is true). |

## How it works

1.  **Anonymous Check**: The handler first checks if the request path matches any prefix in `anonymousPrefixes`. If it does, the request proceeds to the next handler immediately.
2.  **Path Policy Lookup**: The handler iterates through `pathPrefixAuths` to find a match for the request path.
3.  **Authentication Execution**:
    *   If **Basic**, **JWT**, **SJWT**, or **SWT** is enabled, the handler looks for the `Authorization` header.
    *   It identifies the authentication type (Basic vs. Bearer).
    *   For **Bearer** tokens, it determines if the token is a JWT/SJWT or an SWT.
    *   If **SJWT** (Simple JWT) is enabled, it can differentiate between a full JWT (with scopes) and a Simple JWT (without scopes) and invoke the appropriate verification logic.
    *   If multiple methods are enabled (e.g., both JWT and Basic), it attempts to verify based on the provided credentials.
    *   If **ApiKey** is enabled, it checks for the API key in the configured headers/parameters.

## Benefits

*   **Centralized Management**: Configure all security policies for the entire gateway or service in one file.
*   **Reduced Performance Overhead**: Minimizes the number of handlers in the middleware chain.
*   **Flexibility**: Supports different security requirements for different API versions or functional areas.
*   **Dynamic Discovery**: Supports multiple JWK or introspection services per path prefix, enabling integration with multiple identity providers.

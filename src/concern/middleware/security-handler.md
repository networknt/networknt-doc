# Security Handler

The `SecurityHandler` (implemented as `JwtVerifyHandler` in REST, GraphQL, etc.) is responsible for verifying the security tokens (JWT or SWT) in the request header. It is a core component of the light-4j framework and provides distributed policy enforcement without relying on a centralized gateway.

## Configuration

The security configuration is usually defined in `security.yml`, but it can be overridden by framework-specific files like `openapi-security.yml` or `graphql-security.yml`.

### Example security.yml

```yaml
# Security configuration for security module in light-4j.
---
# Enable JWT verification flag.
enableVerifyJwt: ${security.enableVerifyJwt:true}

# Enable SWT verification flag.
enableVerifySwt: ${security.enableVerifySwt:true}

# swt clientId header name.
swtClientIdHeader: ${security.swtClientIdHeader:swt-client}

# swt clientSecret header name. 
swtClientSecretHeader: ${security.swtClientSecretHeader:swt-secret}

# Extract JWT scope token from the X-Scope-Token header and validate the JWT token
enableExtractScopeToken: ${security.enableExtractScopeToken:true}

# Enable JWT scope verification. Only valid when enableVerifyJwt is true.
enableVerifyScope: ${security.enableVerifyScope:true}

# Skip scope verification if the endpoint specification is missing.
skipVerifyScopeWithoutSpec: ${security.skipVerifyScopeWithoutSpec:false}

# If set true, the JWT verifier handler will pass if the JWT token is expired already.
ignoreJwtExpiry: ${security.ignoreJwtExpiry:false}

# User for test only. should be always be false on official environment.
enableMockJwt: ${security.enableMockJwt:false}

# Enables relaxed verification for jwt. e.g. Disables key length requirements.
enableRelaxedKeyValidation: ${security.enableRelaxedKeyValidation:false}

# JWT signature public certificates. kid and certificate path mappings.
jwt:
  certificate: ${security.certificate:100=primary.crt&101=secondary.crt}
  clockSkewInSeconds: ${security.clockSkewInSeconds:60}
  # Key distribution server standard: JsonWebKeySet for other OAuth 2.0 provider| X509Certificate for light-oauth2
  keyResolver: ${security.keyResolver:JsonWebKeySet}

# Enable or disable JWT token logging for audit.
logJwtToken: ${security.logJwtToken:true}

# Enable or disable client_id, user_id and scope logging.
logClientUserScope: ${security.logClientUserScope:false}

# Enable JWT token cache to speed up verification.
enableJwtCache: ${security.enableJwtCache:true}

# Max size of the JWT cache before a warning is logged.
jwtCacheFullSize: ${security.jwtCacheFullSize:100}

# Retrieve public keys dynamically from the OAuth2 provider.
bootstrapFromKeyService: ${security.bootstrapFromKeyService:false}

# Provider ID for federated deployment.
providerId: ${security.providerId:}

# Define a list of path prefixes to skip security.
skipPathPrefixes: ${security.skipPathPrefixes:}

# Pass specific claims from the token to the backend API via HTTP headers.
passThroughClaims:
  clientId: client_id
  tokenType: token_type
```

## Key Features

### JWT Verification
Verification includes signature check, expiration check, and scope verification. The handler uses the `JwtVerifier` to perform these checks.

### SWT Verification
The handler also supports Shared Secret Token (SWT) verification. When enabled, it checks the token using client ID and secret, which can be passed in headers specified by `swtClientIdHeader` and `swtClientSecretHeader`.

### Scope Verification
If `enableVerifyScope` is true, the handler compares the scopes in the JWT against the scopes defined in the API specification (e.g., `openapi.yaml`). If the specification is missing and `skipVerifyScopeWithoutSpec` is true, scope verification is skipped.

### Token Caching
To improve performance, verified tokens can be cached. This avoids expensive signature verification for subsequent requests with the same token. The cache size can be monitored using `jwtCacheFullSize`.

### Dynamic Key Loading
By setting `bootstrapFromKeyService` to true, the handler can pull public keys (JWK) from the OAuth2 provider's key service based on the `kid` (Key ID) in the JWT header.

### Path Skipping
You can bypass security for specific endpoints by adding their prefixes to `skipPathPrefixes`.

### Claim Pass-through
Claims from the JWT or SWT can be mapped to HTTP headers and passed to the backend service using `passThroughClaims`. This is useful for passing user information or client details without the backend having to parse the token again.

## Security Attacks Mitigation

### alg header attacks
The light-4j implementation is opinionated and primarily uses `RS256`. It prevents "alg: none" attacks and HMAC-RSA confusion attacks by explicitly specifying the expected algorithm.

### kid and Key Rotation
The use of `kid` allows the framework to support multiple keys simultaneously, which is essential for seamless key rotation.

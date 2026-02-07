# Token Exchange Handler

The Token Exchange Handler implements [RFC 8693 OAuth 2.0 Token Exchange](https://datatracker.ietf.org/doc/html/rfc8693). It allows a gateway or sidecar to exchange an incoming token (e.g., an external Client Credentials token) for an internal token (e.g., an Authorization Code token with a user identity) by calling an OAuth 2.0 provider.

## Introduction

In a microservices architecture, especially when integrating with external partners, the incoming request often carries a token issued by an external Identity Provider (IdP). This token might lack the necessary context (like internal user ID, roles, or custom claims) required by internal services.

The Token Exchange Handler intercepts the request, validates the incoming token (optional), exchanges it for an internal token via the `urn:ietf:params:oauth:grant-type:token-exchange` grant type, and replaces the `Authorization` header with the new token.

## Configuration

The configuration is managed via `token-exchange.yml`.

| Field | Description | Default |
|-------|-------------|---------|
| enabled | Enable or disable the handler | true |
| tokenExUri | The token endpoint of the OAuth 2.0 provider | null |
| tokenExClientId | The client ID used to authenticate the exchange request | null |
| tokenExClientSecret | The client secret used to authenticate the exchange request | null |
| tokenExScope | Optional list of scopes to request for the new token | null |
| subjectTokenType | The type of the input token (RFC 8693 URN) | urn:ietf:params:oauth:token-type:jwt |
| requestedTokenType | The desired type of the output token (RFC 8693 URN) | urn:ietf:params:oauth:token-type:jwt |
| mappingStrategy | Strategy for mapping client IDs (config/database) | database |

### Example token-exchange.yml

```yaml
# Enable or disable the handler
enabled: true

# The path to the token exchange endpoint on OAuth 2.0 provider
tokenExUri: https://localhost:6882/oauth2/token

# The client ID for the token exchange authentication
tokenExClientId: portal-client

# The client secret for the token exchange authentication
tokenExClientSecret: portal-secret

# The scope of the returned token
tokenExScope:
  - portal.w
  - portal.r

# The subject token type. Default is urn:ietf:params:oauth:token-type:jwt
subjectTokenType: urn:ietf:params:oauth:token-type:jwt

# The requested token type. Default is urn:ietf:params:oauth:token-type:jwt
requestedTokenType: urn:ietf:params:oauth:token-type:jwt
```

## Usage

### 1. Add Dependency

Add the `token-exchange` module to your `pom.xml`.

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>token-exchange</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

### 2. Register Handler

In your `handler.yml`, add `com.networknt.token.exchange.TokenExchangeHandler` to the desired chain. It should typically be placed *after* the `TraceabilityHandler` or `CorrelationHandler` and *before* the `security` handler if you want to validate the *new* token, or *before* the `security` handler if you want to validate the *original* token. 

**Note**: If you place it before the `AuditHandler`, the audit log will reflect the exchanged token.

```yaml
handlers:
  - com.networknt.token.exchange.TokenExchangeHandler@token
  # ... other handlers ...

chains:
  default:
    - exception
    - metrics
    - traceability
    - correlation
    - token 
    - specification
    - security
    - body
    - validator
    # ...
```

## Mapping Strategies

To map an external Client ID (from the `subject_token`) to an internal User ID:

1.  **Database (Recommended)**: Use a "Shadow Client" in `light-oauth2`. Create a client with the same ID as the external client and populate `custom_claims` with the internal `userId` or `functionId`. `light-oauth2` will automatically inject these claims into the minted token.
2.  **Config**: (Future support) Map IDs directly in `token-exchange.yml`.

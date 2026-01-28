# GraphQL Security

The `graphql-security` module provides JWT verification for GraphQL endpoints in `light-graphql-4j`. Since GraphQL relies on business-logic authorization rather than fine-grained URL-based access control, this module focuses on verifying the identity and validity of the incoming request's token.

## Core Components

### JwtVerifyHandler
The `JwtVerifyHandler` is a middleware that intercepts requests to verify the Bearer token in the `Authorization` header.

**Key responsibilities:**
1.  **Token Verification**: Validates the JWT signature, potential expiry (if configured), and audience.
2.  **Audit Info Population**: Extracts the `clientId`, `userId`, `scopes`, and other claims from the token and injects them into the `auditInfo` attachment of the exchange. This data is then available for the GraphQL execution engine and resolvers to make authorization decisions.
3.  **Scope Verification**: While GraphQL schema defines authorization, the handler can perform an initial scope check against the token if `enableVerifyScope` is true in `security.yml`.
4.  **Skip Auth**: Respects the `skipPathPrefixes` configuration in `security.yml` to allow public access to specific endpoints (though typically GraphQL is a single endpoint).

## Configuration

This module primarily uses the standard `security.yml` configuration from the `light-4j` framework.

**security.yml**
```yaml
enableVerifyJwt: true
enableVerifyScope: true
# ... standard security config ...
```

## Usage

Register the `JwtVerifyHandler` in your `handler.yml` chain, typically after the `ValidatorHandler` but before the `GraphqlHandler`.

```yaml
handlers:
  - com.networknt.graphql.validator.ValidatorHandler@validator
  - com.networknt.graphql.security.JwtVerifyHandler@security
  - com.networknt.graphql.router.GraphqlRouter@graphql

paths:
  - path: '/graphql'
    method: 'post'
    handler:
      - validator
      - security
      - graphql
```

## Hot Reload
The `JwtVerifyHandler` supports dynamic reloading. Updates to `security.yml` (e.g., rotating keys or changing validation flags) will be picked up by the handler without restarting the service.

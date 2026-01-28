---
description: OpenAPI Security Module
---

# OpenAPI Security

The `openapi-security` module is a fundamental component of the `light-rest-4j` framework, specifically designed to protect RESTful APIs defined with OpenAPI Specification 3.0. It provides several middleware handlers that integrate with the framework's security infrastructure to ensure that only authorized requests reach your business logic.

## Overview

Modern web services often require robust security mechanisms such as OAuth 2.0. The `openapi-security` module simplifies the implementation of these standards by providing out-of-the-box handlers for token verification and authorization based on the OpenAPI specification.

## Core Handlers

### JwtVerifyHandler

The `JwtVerifyHandler` is the most commonly used handler in this module. It performs the following tasks:
- **Token Verification**: Validates the signature and expiration of the OAuth 2.0 JWT access token.
- **Scope Authorization**: Automatically checks the `scope` or `scp` claim in the JWT against the required scopes defined for each operation in the OpenAPI specification.
- **Integration**: Works seamlessly with `openapi-meta` to identify the current operation and its security requirements.

### SimpleJwtVerifyHandler

The `SimpleJwtVerifyHandler` is a lightweight alternative to the standard `JwtVerifyHandler`. It is used when scope validation is not required. It still verifies the JWT signature and expiration but skips the complex authorization checks against the OpenAPI spec.

### SwtVerifyHandler

The `SwtVerifyHandler` is designed for Simple Web Tokens (SWT). Unlike JWTs, which are self-contained, SWTs often require token introspection against an OAuth 2.0 provider. This handler manages the introspection process and validates the token's validity and associated scopes.

## Configuration

The security handlers are primarily configured via `security.yml`. Key configuration options include:

- **enableVerifyJwt**: Toggle for JWT verification.
- **enableVerifyScope**: Toggle for scope-based authorization.
- **jwt**: Configuration for JWT verification (JWK URLs, certificates, etc.).
- **swt**: Configuration for SWT introspection.
- **skipPathPrefixes**: A list of path prefixes that should bypass security checks (e.g., `/health`, `/info`).
- **passThroughClaims**: Enables passing specific claims from the token into request headers for downstream use.

## Unified Security

For complex scenarios where multiple security methods (Bearer, Basic, ApiKey) need to be supported simultaneously within a single gateway or service, the `UnifiedSecurityHandler` (from the `unified-security` module) can be used to coordinate these different handlers based on the request path and headers.

## Hot Reload Support

All handlers in the `openapi-security` module support hot-reloading of their configurations. If the `security.yml` or related configuration files are updated on the file system, the handlers will automatically detect the changes and re-initialize their verifiers without requiring a server restart.

## Integration with OpenAPI Meta

The `openapi-security` handlers depend on the `OpenApiHandler` (from the `openapi-meta` module) being placed earlier in the request chain. The `OpenApiHandler` identifies the matching operation in the specification and attaches it to the request, which the security handlers then use for validation.

## Best Practices

1. **Enable Scope Verification**: Always define required scopes in your OpenAPI specification and ensure `enableVerifyScope` is set to `true`.
2. **Use Skip Paths Sparingly**: Only skip security for public-facing informational endpoints.
3. **Secure Configuration**: Use the encrypted configuration feature of `light-4j` to protect sensitive information like client secrets or certificate passwords.

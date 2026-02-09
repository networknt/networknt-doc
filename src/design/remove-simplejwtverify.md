# Consolidating AbstractJwtVerifyHandler and AbstractSimpleJwtVerifyHandler

## Introduction

The `light-4j` framework previously had two abstract base classes for JWT verification: `AbstractJwtVerifyHandler` and `AbstractSimpleJwtVerifyHandler`.

- `AbstractJwtVerifyHandler`: Used by `JwtVerifyHandler` (in `light-rest-4j`) and `HybridJwtVerifyHandler` (in `light-hybrid-4j`). It included logic for verifying OAuth 2.0 scopes against an OpenAPI specification.
- `AbstractSimpleJwtVerifyHandler`: Used by `SimpleJwtVerifyHandler` (in `light-rest-4j`). It provided basic JWT verification (signature, expiration) but skipped all scope verification logic.

This design document outlines the consolidation of these two classes into a single `AbstractJwtVerifyHandler` to reduce code duplication and simplify maintenance.

## Motivation

The two abstract classes shared approximately 85% of their code, including token extraction, JWT signature verification using `JwtVerifier`, and audit info population. The primary difference was the presence of scope verification logic in `AbstractJwtVerifyHandler`.

Maintaining two separate hierarchies for largely identical logic increased the risk of inconsistencies (e.g., bug fixes applied to one but missed in the other) and added unnecessary complexity.

## Changes

### 1. AbstractJwtVerifyHandler Updates

The `AbstractJwtVerifyHandler` in `light-4j/unified-security` has been updated to support scopeless verification, effectively merging the behavior of `AbstractSimpleJwtVerifyHandler`.

- **`getSpecScopes()` is no longer abstract**: It now has a default implementation that returns `null`.
  ```java
  public List<String> getSpecScopes(HttpServerExchange exchange, Map<String, Object> auditInfo) throws Exception {
      return null;  // Default: no scope verification (simple JWT behavior)
  }
  ```
- **Conditional Scope Verification**: The `handleJwt` method now checks if `getSpecScopes()` returns `null`. If it does, the scope verification logic (checking secondary scopes and matching against spec scopes) is skipped.
- **Enriched Audit Info**: `AbstractJwtVerifyHandler` extracts additional claims like `email`, `host`, and `role` into the audit info map. By using this class for simple JWT verification, these claims are now populated for `SimpleJwtVerifyHandler` as well, improving audit capabilities.

### 2. Removal of AbstractSimpleJwtVerifyHandler

The `AbstractSimpleJwtVerifyHandler` class in `light-4j/unified-security` has been deleted.

### 3. SimpleJwtVerifyHandler Update

The `SimpleJwtVerifyHandler` in `light-rest-4j/openapi-security` has been updated to extend `AbstractJwtVerifyHandler` directly.

- It inherits the default `getSpecScopes()` implementation (returning `null`), which triggers the scopeless verification path in the base class.
- It implements the required `isSkipAuth()` method, same as before.

### 4. UnifiedSecurityHandler Update

The `UnifiedSecurityHandler` in `light-4j/unified-security` has been updated to cast the "sjwt" (Simple JWT) handler to `AbstractJwtVerifyHandler` instead of the removed `AbstractSimpleJwtVerifyHandler`.

## Impact

### Simplified Hierarchy
There is now a single abstract base class (`AbstractJwtVerifyHandler`) for all JWT verification handlers in the framework.

### Backward Compatibility
- **Functionality**: Existing handlers (`JwtVerifyHandler`, `SimpleJwtVerifyHandler`, `HybridJwtVerifyHandler`) continue to work as before.
- **Configuration**: No changes to `security.yml` or `openapi-security.yml` are required.
- **Behavior**: `SimpleJwtVerifyHandler` now extracts more comprehensive audit information (e.g., user email, host, role) if present in the token, which is a beneficial side effect.

## Conclusion

This refactoring simplifies the codebase, removes duplication, and ensures consistent JWT handling across different modules while maintaining full backward compatibility.

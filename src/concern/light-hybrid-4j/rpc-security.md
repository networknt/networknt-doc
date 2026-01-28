# RPC Security

The `rpc-security` module in `light-hybrid-4j` provides security handlers specifically designed for the hybrid framework's routing mechanism. It extends the core security capabilities of `light-4j` to work seamlessly with the `rpc-router`'s service dispatching model.

## Overview

In `light-hybrid-4j`, requests are routed based on a service ID in the payload (e.g., `lightapi.net/petstore/getPetById/1.0.0`) rather than URL paths. This difference requires specialized security handlers that can:
1.  Verify JWT tokens protecting the RPC endpoint.
2.  Extract required scopes from the target service's schema (specifically the `spec.yaml` loaded by the `rpc-router`).
3.  Authorize the request by comparing the token's scopes against the service's required scopes.

## Core Components

### 1. `HybridJwtVerifyHandler`
This handler extends `AbstractJwtVerifyHandler` to provide JWT validation for hybrid services.

*   **Audit Info Integration**: It expects the `SchemaHandler` (from `rpc-router`) to have already parsed the request and populated the `auditInfo` attachment map with the `HYBRID_SERVICE_MAP`.
*   **Dynamic Scope Resolution**: Instead of hardcoding scopes or using Swagger endpoints, it retrieves the required scopes directly from the `scope` property defined in the service's `spec.yaml`.
*   **Skip Auth Support**: It respects the `skipAuth` flag in the service specification, allowing individual handlers to be public while others remain protected.

### 2. `AccessControlHandler`
Provides fine-grained access control based on rule definitions.
*   **Rule-Based Access**: Can enforce complex authorization rules (e.g., "deny if IP is X" or "allow if time is Y") beyond simple RBAC.
*   **Runtime Configuration**: Supports hot-reloading of access control rules and settings via `access-control.yml`.

## Configuration

This module relies on `security.yml` (shared with the core framework) and specific service definitions in `spec.yaml`.

### `security.yml`
Standard configuration for JWT verification:
```yaml
enableVerifyJwt: true
enableVerifyScope: true
enableJwtCache: true
```

### `access-control.yml`
Configuration for the `AccessControlHandler`:
```yaml
enabled: true
accessRuleLogic: 'any' # or 'all'
defaultDeny: true
```

## Usage Example

### 1. Register Handlers
In your `handler.yml`, add the security handlers to your chain *after* the `SchemaHandler` but *before* the `JsonHandler`. The `SchemaHandler` is required first to resolve the service definition.

```yaml
handlers:
  - com.networknt.rpc.router.SchemaHandler@schema
  - com.networknt.rpc.security.HybridJwtVerifyHandler@jwt
  - com.networknt.rpc.router.JsonHandler@json

paths:
  - path: '/api/json'
    method: 'post'
    handler:
      - schema
      - jwt
      - json
```

### 2. Define Security in `spec.yaml`
In your hybrid service's `spec.yaml` file, define the `scope` required to access the action, or set `skipAuth` to true for public endpoints.

**Example: Protected Endpoint**
```yaml
service: petstore
action:
  - name: getPetById
    version: 1.0.0
    handler: getPetById
    scope: "petstore.r" 
    request: ...
```
*   The `HybridJwtVerifyHandler` will extract `petstore.r` and ensure the caller's JWT has this scope.

**Example: Public Endpoint**
```yaml
service: petstore
action:
  - name: login
    version: 1.0.0
    handler: loginHandler
    skipAuth: true
    request: ...
```
*   The handler will skip JWT verification for this action.

## Hot Reload
The handlers in this module support hot-reloading. If `security.yml` or `access-control.yml` are updated via the config server, the changes will be applied dynamically without restarting the server.

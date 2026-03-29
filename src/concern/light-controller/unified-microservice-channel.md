# Unified Microservice Channel

## 1. Overview

This document defines the target contract between `light-4j/portal-registry` and the controller backend.

The main goals are:

- use a single registration payload for both `light-controller` and `controller-rs`
- allow a service to use one WebSocket channel for both registration and discovery
- use the client-supplied service address instead of deriving the service address from the remote socket IP
- enforce tenant isolation with a controller-configured `hostId` and the JWT `host` claim
- align runtime instance persistence with existing `RuntimeInstanceCreatedEvent`, `RuntimeInstanceUpdatedEvent`, and `RuntimeInstanceDeletedEvent`

## 2. Core Decisions

### 2.1 Single Service Channel

For microservices, the controller should expose one WebSocket channel:

- `/ws/microservice`

This channel is used for:

- `service/register`
- `service/update_metadata`
- `discovery/lookup`
- `discovery/subscribe`
- `discovery/unsubscribe`
- controller-to-service commands and service responses

For client applications that do not register as services, the controller should expose:

- `/ws/discovery`

This channel is discovery-only.

### 2.2 Client-Supplied Address

The service address advertised to discovery must come from the registration payload.

The controller must not use the remote socket IP as the canonical service address because real deployments can involve:

- NAT
- reverse proxies
- ingress controllers
- sidecars
- container overlays

The remote socket IP can still be recorded for logging and audit, but it must not be used for routing or discovery.

### 2.3 One Controller per Tenant

Each controller instance serves exactly one tenant.

The controller configuration contains the authoritative internal tenant identifier, `hostId`. Services do not send `hostId` in the registration payload.

The service JWT carries the tenant in the `host` claim. The controller validates `jwt.host == configured hostId`.

This avoids cross-tenant routing mistakes and keeps the controller state, discovery subscriptions, and event streams tenant-scoped by construction.

## 3. Unified Registration Contract

The `service/register` request should have the same payload for both controller implementations.

Example:

```json
{
  "jsonrpc": "2.0",
  "id": "register-1",
  "method": "service/register",
  "params": {
    "jwt": "<service-jwt>",
    "serviceId": "com.networknt.user-1.0.0",
    "envTag": "prod",
    "version": "1.0.0",
    "protocol": "https",
    "address": "10.0.5.12",
    "port": 8443,
    "tags": {}
  }
}
```

Required behavior:

- `address` is the canonical advertised service address
- `port` is the advertised service port
- `jwt` authenticates the service session
- `serviceId` must match the service identity in the JWT
- `envTag` is the only environment field in the unified registration contract
- `envTag` should match the JWT `env` claim when that claim is present

`portal-registry` should build one registration payload for all controller backends. There should be no separate `toControllerRsRegisterParams()` path.

## 4. Authentication and Tenant Isolation

### 4.1 `/ws/microservice`

Authentication for `/ws/microservice` is performed with the JWT inside `service/register`.

The service JWT must be validated for:

- signature
- issuer and audience when configured
- service identity
- tenant identity

The controller configuration contains a single internal tenant identifier, `hostId`. The service JWT must contain the same tenant identifier in the `host` claim.

Recommended claim handling:

- read `host`

If the JWT `host` claim does not match the controller-configured `hostId`, the controller must reject the registration. In other words, the JWT tenant claim `host` is validated against the controller's internal tenant key `hostId`. This prevents a service from connecting to the wrong tenant controller instance.

### 4.2 `/ws/discovery`

Client applications that use `/ws/discovery` must authenticate on the WebSocket upgrade request with:

```text
Authorization: Bearer <jwt>
```

The discovery JWT should also be checked using the `host` claim against the controller-configured `hostId` so discovery clients cannot attach to the wrong tenant controller.

### 4.3 No Extra Discovery Token for Services

Once a service is connected on `/ws/microservice`, no separate discovery token is needed for service-owned discovery requests on that same socket.

The separate discovery bearer token remains relevant only for discovery-only clients using `/ws/discovery`.

## 5. Runtime Instance Identity

### 5.1 Business Key

Before emitting `RuntimeInstanceCreatedEvent`, the controller must query the database using this business key:

- `hostId` from controller configuration, validated from the JWT `host` claim
- `serviceId`
- `envTag`
- `address`
- `port`

If a matching runtime instance already exists, the controller must reuse its `runtimeInstanceId`.

If no matching runtime instance exists, the controller must create a new UUID for `runtimeInstanceId`.

### 5.2 Aggregate Identity

The event aggregate identity is:

- `hostId|runtimeInstanceId`

This matches the existing Light Portal aggregate-id derivation for runtime instance events. Event payloads and persistence continue to use `hostId`, even though JWT validation reads the tenant from the `host` claim.

## 6. Event Persistence

### 6.1 Event Types

The controller should align to the existing event types:

- `RuntimeInstanceCreatedEvent`
- `RuntimeInstanceUpdatedEvent`
- `RuntimeInstanceDeletedEvent`

At present, the primary lifecycle use cases are:

- create on successful registration
- delete on disconnect or explicit removal

`RuntimeInstanceUpdatedEvent` should remain supported by the contract, but it does not need to be emitted until there is a concrete mutable metadata use case.

### 6.2 Create Flow

After a successful `service/register`:

1. resolve `hostId` from controller configuration after validating the JWT `host` claim
2. query by `hostId + serviceId + envTag + address + port`
3. reuse existing `runtimeInstanceId` if found, otherwise create a new UUID
4. determine the next aggregate version for `hostId|runtimeInstanceId`
5. persist `RuntimeInstanceCreatedEvent` to `event_store_t`
6. persist the matching integration message to `outbox_message_t`

The downstream processing of `RuntimeInstanceCreatedEvent` should perform an upsert into `runtime_instance_t`.

### 6.3 Delete Flow

When the `/ws/microservice` socket closes:

1. locate the current runtime instance within the controller's configured `hostId`
2. determine the next aggregate version for `hostId|runtimeInstanceId`
3. persist `RuntimeInstanceDeletedEvent` to `event_store_t`
4. persist the matching integration message to `outbox_message_t`

The downstream projection should mark the runtime instance as disconnected or inactive in `runtime_instance_t`.

### 6.4 Update Flow

`RuntimeInstanceUpdatedEvent` is reserved for future use.

It should only be emitted if there is a real business need to update mutable runtime metadata after registration.

If the service `address` or `port` changes, that should normally be treated as a different runtime identity rather than an in-place update.

## 7. Discovery Semantics

Discovery requests on `/ws/microservice` and `/ws/discovery` use the same runtime registry state.

Discovery operations:

- do not create runtime instance lifecycle events
- do not affect runtime identity
- are session-level behavior, not aggregate lifecycle changes

On disconnect:

- a `/ws/microservice` disconnect removes the service session and emits `RuntimeInstanceDeletedEvent`
- a `/ws/discovery` disconnect only clears that discovery client's subscriptions

## 8. Migration Impact

### 8.1 `controller-rs`

`controller-rs` should be updated to:

- accept `address` in `service/register`
- use request `address` as the canonical advertised address
- keep remote socket IP for logging and audit only
- validate JWT `host` claim against controller configuration
- allow discovery requests on `/ws/microservice`
- keep `/ws/discovery` for discovery-only clients
- move runtime instance persistence toward the Light Portal aggregate and tenant model

### 8.2 `light-controller`

`light-controller` already accepts request `address` with remote-address fallback. It should move to the same stricter tenant model:

- controller-configured internal tenant key `hostId`
- JWT `host` claim match required
- one tenant per controller instance

### 8.3 `portal-registry`

`portal-registry` should be simplified to:

- send one `service/register` payload shape to both backends
- use `/ws/microservice` as the only channel for service registration and service-owned discovery
- stop relying on a separate service-side discovery token for normal service use

## 9. Summary

The target model is:

- one controller instance per tenant
- one service socket on `/ws/microservice`
- one discovery-only socket on `/ws/discovery` for non-service clients
- one shared registration payload for all controller backends
- client-supplied `address` as the canonical service address
- controller-configured `hostId` enforced by validating it against the JWT `host` claim
- runtime instance lifecycle persisted with existing runtime instance events and aggregate versioning

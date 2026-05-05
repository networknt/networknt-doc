# Portal Registry Transition

Status: Draft

Last updated: 2026-05-04

## Introduction

Light-4j services are moving from `direct-registry` to `portal-registry` backed
by the controller implementation. In the final state, service registration and
consumer-side discovery should flow through the controller. During the
transition, however, some gateways and tools still depend on static
`direct-registry.directUrls` mappings from `serviceId` to target host.

This design adds a compatibility lookup in `portal-registry`: when a
`direct-registry.directUrls` entry exists for the requested `serviceId`, the
portal registry uses that entry before it opens controller discovery through
`/ws/discovery`.

Example transition configuration:

```yaml
direct-registry.directUrls:
  com.networknt.llmchat-1.0.0: http://localhost:8080
  com.networknt.petstore-1.0.0: https://localhost:9443
  com.networknt.local.mcp-1.0.0: http://localhost:5000
  com.networknt.mcp.github.nnts-1.0.0: https://gitmcp.io/networknt/networknt-site
```

## Goals

- Allow applications to switch the registry implementation from
  `DirectRegistry` to `PortalRegistry` without losing existing static
  `directUrls` mappings.
- Preserve the existing `direct-registry.directUrls` key format:
  `serviceId` and `serviceId|envTag`.
- Keep controller discovery authoritative for services that do not have a local
  direct mapping.
- Avoid changes to `controller-rs` for this transitional behavior.
- Keep the behavior deterministic when both a direct mapping and controller
  registration exist.

## Non-Goals

- Do not reintroduce the legacy light-portal REST registry path.
- Do not push `directUrls` entries into the controller or make them visible as
  controller live instances.
- Do not create synthetic `discovery/changed` events for static direct URLs.
- Do not replace the existing controller WebSocket registration contract.
- Do not change router or MCP tool path handling as part of this registry
  fallback.

## Current State

### Direct Registry

`DirectRegistry` resolves a subscribe URL by loading
`direct-registry.directUrls` through `DirectRegistryConfig`. The config already
supports these forms:

- YAML map
- JSON string map
- string map using `key=value&key2=value2`

The direct registry key is built with the same rule used across the registry
interfaces:

```text
serviceId
serviceId|envTag
```

The value is parsed into one or more `URL` objects, separated by commas.

### Portal Registry

`PortalRegistry` currently uses the controller WebSocket contract:

- registration through `/ws/microservice`
- discovery lookup and subscription through `/ws/discovery`
- cache updates through `discovery/changed`

For consumer-side discovery, `PortalRegistry.doDiscover()` currently checks the
local service cache and then falls back to controller discovery. `doSubscribe()`
opens the discovery WebSocket and sends `discovery/subscribe`.

## Proposed Behavior

`PortalRegistry` should check `direct-registry.directUrls` before controller
discovery. The resolution order becomes:

1. Compute the direct key from `serviceId` and optional `envTag`.
2. Load `DirectRegistryConfig`.
3. If `directUrls` contains that key, return the configured URLs after protocol
   filtering.
4. If no direct mapping exists, keep the current portal cache and controller
   discovery behavior.

Direct mappings are local and authoritative for their key. If a direct mapping
exists but none of its URLs match the requested protocol, return an empty list
and log the mismatch instead of falling through to controller discovery. This
prevents a bad transition config from being hidden by a live controller entry.

## Discovery Algorithm

The `PortalRegistry` direct lookup helper should distinguish these cases:

- no `directUrls` config or no matching key: no direct override
- matching key with one or more URLs: direct override
- matching key with zero matching URLs after protocol filtering: direct
  override with empty result

Pseudo-code:

```java
private List<URL> lookupDirectUrls(String serviceId, String envTag, String protocol) {
    DirectRegistryConfig config = DirectRegistryConfig.load();
    Map<String, List<URL>> directUrls = config.getDirectUrls();
    if (directUrls == null || directUrls.isEmpty()) {
        return null;
    }

    String key = serviceKey(serviceId, envTag);
    if (!directUrls.containsKey(key)) {
        return null;
    }

    List<URL> configured = directUrls.get(key);
    if (protocol == null) {
        return configured;
    }

    return configured.stream()
            .filter(url -> protocol.equals(url.getProtocol()))
            .toList();
}
```

The actual implementation may avoid `Stream.toList()` if the target Java
version requires broader compatibility.

## Subscribe Behavior

`doSubscribe()` should also check direct URLs first:

1. If a direct mapping exists, notify the listener with the direct URL list and
   do not call `client.ensureWebSocketConnected()`.
2. Do not add the URL to the controller subscription set for direct mappings.
3. If no direct mapping exists, keep the current controller subscribe path.

This keeps static services independent from controller availability.

## Unsubscribe Behavior

The controller subscription set should represent only real controller
subscriptions. `doUnsubscribe()` should remove the URL from that set first and
only send `discovery/unsubscribe` when the URL was actually subscribed through
the controller.

This matters if configuration changes while the process is running. A direct
mapping added later should not prevent cleanup of an older controller
subscription.

## Cache Behavior

Direct URL results should not be written into the portal registry controller
cache. The direct lookup should run before the controller cache lookup every
time.

This gives the transition behavior two useful properties:

- adding a direct mapping takes effect on the next lookup
- removing a direct mapping naturally exposes the existing controller cache or
  triggers controller discovery

Controller `discovery/changed` notifications should continue to update only
controller-discovered services.

## Protocol and Path Semantics

Protocol filtering should be strict. If the caller asks for `https`, a direct
entry with only `http` URLs should not be returned.

Existing `LightCluster.serviceToUrl()` returns a base target in the form:

```text
protocol://host:port
```

Because of that contract, paths embedded in `directUrls` values should not be
relied on by consumers that resolve through `LightCluster.serviceToUrl()`. Route
paths and MCP tool paths should remain in the router or tool configuration.
The direct URL is best treated as the target host, scheme, and port.

## Configuration

No new `portal-registry.yml` property is required. The transition uses the
existing `direct-registry.directUrls` config so deployments can switch the
registry implementation without duplicating mappings.

Example service singleton change:

```yaml
service.singletons:
  - com.networknt.registry.Registry:
    - com.networknt.portal.registry.PortalRegistry
```

The existing direct URL config remains:

```yaml
direct-registry.directUrls:
  com.networknt.llmchat-1.0.0: http://localhost:8080
  com.networknt.petstore-1.0.0: https://localhost:9443
```

## Implementation Plan

1. Import and use `DirectRegistryConfig` from the `registry` module in
   `portal-registry`.
2. Add a direct lookup helper in `PortalRegistry`.
3. Update `doDiscover()` to return direct URLs before checking the portal cache
   or calling controller discovery.
4. Update `doSubscribe()` to notify the listener from direct URLs and skip the
   controller discovery WebSocket for direct mappings.
5. Update `doUnsubscribe()` so it only sends controller unsubscribe requests for
   URLs that were controller-subscribed.
6. Harden `DirectRegistryConfig` so blank or missing `directUrls` produces an
   empty map instead of a malformed empty URL entry.
7. Add focused unit tests in `portal-registry`.

## Test Plan

Add tests for these cases:

- direct mapping resolves before controller discovery
- direct mapping does not call `subscribeService`
- missing direct mapping falls back to controller discovery
- `serviceId|envTag` direct mapping is honored
- protocol filtering keeps `http` and `https` isolated
- protocol mismatch with an explicit direct key returns empty result
- blank or missing `direct-registry.directUrls` falls back to controller without
  error
- unsubscribe skips the controller only for direct-only subscriptions

Run the focused module test:

```bash
mvn -q -pl portal-registry test
```

## Migration Flow

1. Keep existing `direct-registry.directUrls` entries for services that are not
   registered with the controller yet.
2. Switch the application registry singleton to `PortalRegistry`.
3. Register services with the controller as they become ready.
4. Remove each direct mapping after its service is reliably registered and
   discoverable through the controller.
5. Once all direct mappings are removed, the application uses controller
   discovery only.

## Risks and Mitigations

### Hidden Protocol Mismatch

A direct key could exist with an `http` target while the caller asks for
`https`.

Mitigation: treat the direct key as authoritative and return an empty list after
protocol filtering, with a clear log message.

### Stale Local Config

Operators may update direct mappings and expect them to take effect.

Mitigation: load `DirectRegistryConfig` through the existing config loader on
each direct lookup, matching the direct registry reload pattern.

### Controller Visibility

Static direct targets will not appear as live controller instances.

Mitigation: document that this is a transition-only local override. Controller
visibility starts only after the target service registers with the controller.

### Path Confusion

Some direct URL values include a path.

Mitigation: preserve the parsed `URL` object in registry discovery, but document
that the common `LightCluster.serviceToUrl()` path returns only
`protocol://host:port`. Consumers that need path routing must keep path metadata
in their own router or tool config.

## Decision

The transition should be implemented as a local `PortalRegistry` direct URL
override. It keeps the controller contract clean, allows incremental migration,
and lets operators remove static mappings one service at a time.

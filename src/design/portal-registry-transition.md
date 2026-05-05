# Portal Registry Transition

Status: Draft

Last updated: 2026-05-05

## Introduction

Light-4j services are moving from local `direct-registry` configuration to
`portal-registry` backed by the controller. In the final state, runtime service
registration, discovery, and subscription should flow through the controller.

The current MicroK8s milestone already has the AI gateway loading runtime
configuration from the config server and registering with the controller. The
next milestone is to deploy `openapi-petstore` with `http-sidecar` in the same
pod under the `openapi-petstore` namespace. This deployment is both a working
example and the first reusable pattern for API pods that want Light platform
cross-cutting concerns at the pod boundary.

This design covers two connected transition concerns:

- how Kubernetes folders and templates should be owned across `http-sidecar`
  and API repositories
- how registry discovery should behave while some services still need static
  `direct-registry.directUrls` mappings and others are registered through the
  controller

## Goals

- Deploy `openapi-petstore` and `http-sidecar` as two containers in one
  Kubernetes pod.
- Expose the sidecar as the pod's service entry point and keep the backend API
  private to the pod.
- Keep `http-sidecar` reusable for other APIs without copying API-specific
  deployment templates into the sidecar repository.
- Let applications switch from `DirectRegistry` to `PortalRegistry` without
  losing existing static `direct-registry.directUrls` mappings.
- Make controller discovery authoritative when no local direct mapping exists.
- Avoid controller changes for the direct URL compatibility behavior.
- Keep the MicroK8s deployment compatible with `light-deployer` template
  rendering and the config server bootstrap flow.

## Non-Goals

- Do not introduce Helm, Kustomize, or an operator in the first iteration.
- Do not create a global Kubernetes template generator before the pattern is
  proven with at least one concrete API.
- Do not make `http-sidecar` own full deployments for every API that uses it.
- Do not push `directUrls` entries into the controller or show them as live
  controller instances.
- Do not create synthetic `discovery/changed` events for static direct URLs.
- Do not change the controller WebSocket registration contract.

## Current State

### AI Gateway

The AI gateway is already deployed to MicroK8s. It loads startup configuration
from the config server and registers with the controller. This proves the
cluster can run a Light service that uses the config-server and controller
bootstrap flow.

### OpenAPI Petstore

`openapi-petstore` has a `k8s` folder with a single-container Deployment and
Service template. The current service points directly at the backend API
container. This is suitable for a standalone service but not for the sidecar
pattern, because callers can bypass the sidecar.

### HTTP Sidecar

`http-sidecar` is a reusable Light service that can handle ingress and egress
traffic for a backend container in the same pod. Its existing configuration
supports proxying to a local backend such as:

```yaml
proxy.hosts: http://localhost:8080
```

The sidecar repository also contains sidecar-specific config examples and
environment overlays. It should become the source of truth for the sidecar
contract, not for API-specific pod composition.

### Registry

`DirectRegistry` resolves service targets from `direct-registry.directUrls`.
`PortalRegistry` resolves services through the controller WebSocket discovery
contract:

- registration through `/ws/microservice`
- discovery lookup and subscription through `/ws/discovery`
- cache updates through `discovery/changed`

During the transition, some deployments will still need direct URL mappings
while other deployments register with the controller.

## Kubernetes Template Ownership

Both repositories should have `k8s` folders, but they should not own the same
thing.

### `http-sidecar/k8s`

The sidecar repository owns reusable sidecar material:

- sidecar image defaults
- sidecar ports
- required environment variables
- config-server bootstrap expectations
- health probe conventions
- sidecar-specific config examples
- documentation for `proxy.hosts`, handler chains, and registry settings

It should not contain a full `openapi-petstore` Deployment. If it contains an
example, the example must be clearly labeled as illustrative and not as the
deployment source of truth for the API.

### `openapi-petstore/k8s`

The API repository owns the deployable pod composition:

- namespace-aware Deployment template
- backend container definition
- `http-sidecar` container definition
- Service exposing the sidecar
- API-specific resource requests and limits
- API-specific config-server startup values
- deployer render request examples

This keeps the concrete deployment close to the API while still making the
sidecar reusable.

### Future APIs

Future APIs should start with an API-owned `k8s` template that composes the
backend with `http-sidecar`. Once two or three APIs repeat the same structure,
the repeated sidecar block can be promoted into a shared documented template or
generation script. Until then, a simple API-owned template is easier to review
and debug.

## Target Kubernetes State

Create the namespace separately from the deployer template:

```bash
microk8s kubectl create namespace openapi-petstore --dry-run=client -o yaml \
  | microk8s kubectl apply -f -
```

The deployer template should then create namespaced application resources.
Keeping Namespace creation outside the template avoids cluster-policy problems
and matches the existing deployer flow used for local MicroK8s work.

### Resource Model

Recommended resources:

- Namespace: `openapi-petstore`
- Deployment: `openapi-petstore`
- Service: `openapi-petstore`
- ConfigMap: startup/config bootstrap for sidecar and, if needed, backend
- Secret: config-server truststore, config password, and portal authorization

The Service should expose the sidecar, not the backend:

```text
client or gateway -> Service/openapi-petstore -> http-sidecar -> localhost backend
```

### Pod Containers

The pod should contain two containers:

- `openapi-petstore`: business API container
- `http-sidecar`: ingress and egress proxy container

The backend container should not need a Kubernetes Service of its own for
normal traffic. The sidecar reaches it over the pod-local loopback address.

### Port Model

Recommended first-class port model:

| Container | Port | Purpose |
| --- | --- | --- |
| `http-sidecar` | `9445` | external HTTPS ingress exposed by Service |
| `http-sidecar` | `9080` | HTTP sidecar port exposed by Service |
| `openapi-petstore` | `8080` | pod-local backend HTTP |

The `openapi-petstore` backend should run HTTP-only on `8080` inside the pod.
The sidecar owns TLS, JWT verification, validation, audit, rate limiting, and
proxy behavior at the pod boundary:

```yaml
proxy.hosts: http://127.0.0.1:8080
```

## Service Identity

Only one service identity should represent the externally routable API.
Otherwise the controller can see duplicate or misleading registrations.

Recommended identity model:

- `http-sidecar` registers with the controller using the external API service
  id `com.networknt.petstore-lpsc-3.1.0` in `dev`.
- `openapi-petstore` uses backend service id `com.networknt.petstore-3.1.0` in
  `dev`, but disables controller registration in the sidecar pod.

The controller-facing identity belongs to the sidecar because all external
traffic should enter through the sidecar. The backend remains an implementation
detail of the pod.

Config-server authorization tokens should match the runtime identity they
bootstrap. The `openapi-petstore` token is for backend service id
`com.networknt.petstore-3.1.0` in `dev`. The sidecar should use a token for
`com.networknt.petstore-lpsc-3.1.0` in `dev`.

## Runtime Configuration

Both containers may load configuration from the config server, but they should
not compete for the same runtime role.

### Sidecar Configuration

The sidecar runtime config should define:

- `server.serviceId` for the externally routable API identity
- `server.httpsPort` for sidecar ingress
- `server.httpPort` if backend egress through the sidecar is needed
- `proxy.hosts` pointing to the pod-local backend
- handler chain for security, validation, audit, router, token, and proxy
- registry singleton set to `PortalRegistry` when using controller discovery

Example target values:

```yaml
server.serviceId: com.networknt.petstore-lpsc-3.1.0
server.environment: dev
server.httpPort: 9080
server.httpsPort: 9445

proxy.hosts: http://127.0.0.1:8080

service.singletons:
  - com.networknt.registry.Registry:
    - com.networknt.portal.registry.PortalRegistry
```

### Backend Configuration

The backend should focus on business handlers. In the sidecar pod it can reduce
or disable concerns already handled by the sidecar.

Example target values:

```yaml
server.serviceId: com.networknt.petstore-3.1.0
server.environment: dev
server.httpPort: 8080
server.enableHttp: true
server.enableHttps: false
server.enableRegistry: false
security.enableVerifyJwt: false
```

## Portal Registry Compatibility

During the transition, `PortalRegistry` should honor existing
`direct-registry.directUrls` mappings before opening controller discovery.
This lets services move to the portal registry implementation before every
target service is registered with the controller.

Example transition configuration:

```yaml
direct-registry.directUrls:
  com.networknt.llmchat-1.0.0: http://localhost:8080
  com.networknt.petstore-lpsc-3.1.0|dev: https://openapi-petstore.openapi-petstore.svc.cluster.local:9445
  com.networknt.local.mcp-1.0.0: http://localhost:5000
  com.networknt.mcp.github.nnts-1.0.0: https://gitmcp.io/networknt/networknt-site
```

The direct URL for `com.networknt.petstore-lpsc-3.1.0|dev` should point to the
sidecar Service, not to the backend container.

## Direct URL Lookup Behavior

`PortalRegistry` should check `direct-registry.directUrls` before controller
discovery. The resolution order is:

1. Compute the direct key from `serviceId` and optional `envTag`.
2. Load `DirectRegistryConfig`.
3. If `directUrls` contains that key, return the configured URLs after
   protocol filtering.
4. If no direct mapping exists, keep the current portal cache and controller
   discovery behavior.

Direct mappings are local and authoritative for their key. If a direct mapping
exists but none of its URLs match the requested protocol, return an empty list
and log the mismatch instead of falling through to controller discovery. This
prevents a bad transition config from being hidden by a live controller entry.

The helper should distinguish these cases:

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

    List<URL> filtered = new ArrayList<>();
    for (URL url : configured) {
        if (protocol.equals(url.getProtocol())) {
            filtered.add(url);
        }
    }
    return filtered;
}
```

## Subscribe and Cache Behavior

`doSubscribe()` should also check direct URLs first:

1. If a direct mapping exists, notify the listener with the direct URL list and
   do not call `client.ensureWebSocketConnected()`.
2. Do not add the URL to the controller subscription set for direct mappings.
3. If no direct mapping exists, keep the current controller subscribe path.

Direct URL results should not be written into the portal registry controller
cache. The direct lookup should run before the controller cache lookup every
time. This gives the transition behavior two useful properties:

- adding a direct mapping takes effect on the next lookup
- removing a direct mapping exposes the existing controller cache or triggers
  controller discovery

`doUnsubscribe()` should only send controller unsubscribe requests for URLs that
were actually subscribed through the controller.

## Migration Flow

1. Create the `openapi-petstore` namespace.
2. Add a two-container sidecar Deployment template in `openapi-petstore/k8s`.
3. Expose the sidecar port through the `openapi-petstore` Service.
4. Configure the sidecar to register as the external petstore service id.
5. Disable backend controller registration inside the sidecar pod.
6. Add or keep a `direct-registry.directUrls` entry pointing to the sidecar
   Service for consumers that are not fully controller-discovered yet.
7. Deploy through `light-deployer` and verify rollout status.
8. Verify the sidecar registers with the controller.
9. Route traffic from the AI gateway to the sidecar service.
10. Remove the direct URL entry after controller discovery is reliable for all
    consumers.

## Implementation Plan

### Phase 1: Documentation and Template Boundaries

- Document the Kubernetes ownership split between `http-sidecar` and API repos.
- Keep sidecar reusable material in `http-sidecar/k8s`.
- Keep the concrete petstore pod composition in `openapi-petstore/k8s`.

### Phase 2: OpenAPI Petstore Pod Template

- Update `openapi-petstore/k8s/deployment.yaml` to include both containers.
- Add namespaced metadata placeholders.
- Add config bootstrap mounts for sidecar and backend as needed.
- Update `openapi-petstore/k8s/service.yaml` so it targets the sidecar port.
- Add a render request example for local MicroK8s.

### Phase 3: Runtime Config

- Add config-server values for the sidecar service identity.
- Point sidecar `proxy.hosts` to the pod-local backend.
- Disable backend controller registration or assign it a private internal id.
- Keep direct URL fallback entries for consumers that still need them.

### Phase 4: Portal Registry Compatibility

- Import and use `DirectRegistryConfig` from the `registry` module in
  `portal-registry`.
- Add a direct lookup helper in `PortalRegistry`.
- Update `doDiscover()` to return direct URLs before checking the portal cache
  or calling controller discovery.
- Update `doSubscribe()` to notify the listener from direct URLs and skip the
  controller discovery WebSocket for direct mappings.
- Update `doUnsubscribe()` so it only sends controller unsubscribe requests for
  URLs that were controller-subscribed.
- Harden `DirectRegistryConfig` so blank or missing `directUrls` produces an
  empty map instead of a malformed empty URL entry.
- Add focused unit tests in `portal-registry`.

## Validation Plan

### Template Validation

Render the deployer template and verify the generated resources:

- Deployment has namespace `openapi-petstore`.
- Deployment has both `openapi-petstore` and `http-sidecar` containers.
- Service selector matches the pod labels.
- Service target port points to the sidecar.
- Backend container port is not exposed as the public Service target.

### Kubernetes Validation

Run the narrow MicroK8s checks:

```bash
microk8s kubectl get namespace openapi-petstore
microk8s kubectl -n openapi-petstore get deploy,svc,pod
microk8s kubectl -n openapi-petstore rollout status deploy/openapi-petstore
microk8s kubectl -n openapi-petstore logs deploy/openapi-petstore -c http-sidecar
microk8s kubectl -n openapi-petstore logs deploy/openapi-petstore -c openapi-petstore
```

Port-forward the sidecar Service and test API traffic:

```bash
microk8s kubectl -n openapi-petstore port-forward svc/openapi-petstore 9445:9445
curl -k https://127.0.0.1:9445/health
curl -k https://127.0.0.1:9445/v1/pets
```

### Controller Validation

Verify that the controller sees the sidecar-facing service identity, not a
duplicate backend identity. Then verify AI gateway routing reaches the
petstore sidecar.

### Portal Registry Tests

Add tests for these cases:

- direct mapping resolves before controller discovery
- direct mapping does not call controller subscribe
- missing direct mapping falls back to controller discovery
- `serviceId|envTag` direct mapping is honored
- protocol filtering keeps `http` and `https` isolated
- protocol mismatch with an explicit direct key returns empty result
- blank or missing `direct-registry.directUrls` falls back to controller
  without error
- unsubscribe skips the controller only for direct-only subscriptions

Run the focused module test:

```bash
mvn -q -pl portal-registry test
```

## Risks and Mitigations

### Duplicate Registration

Both sidecar and backend could register the same service id.

Mitigation: the sidecar owns the external API service id. Disable backend
registration inside the sidecar pod.

### Bypassing the Sidecar

A Service could accidentally target the backend port.

Mitigation: the API-owned Service must target the sidecar port only. Backend
access stays pod-local.

### Hidden Protocol Mismatch

A direct key could exist with an `http` target while the caller asks for
`https`.

Mitigation: treat the direct key as authoritative and return an empty list
after protocol filtering, with a clear log message.

### Stale Local Config

Operators may update direct mappings and expect them to take effect.

Mitigation: load `DirectRegistryConfig` through the existing config loader on
each direct lookup, matching the direct registry reload pattern.

### Controller Visibility

Static direct targets will not appear as live controller instances.

Mitigation: document direct URLs as transition-only local overrides. Controller
visibility starts only after the target service registers with the controller.

### Template Drift Across APIs

Each API repository will initially own its composed sidecar Deployment.

Mitigation: keep the first template small and documented. Promote a shared
template only after repeated use proves the common surface.

## Decision

Use API-owned Kubernetes templates for concrete sidecar pod composition.
`openapi-petstore/k8s` owns the Deployment and Service that run petstore with
`http-sidecar`. `http-sidecar/k8s` owns only reusable sidecar contract material.
The petstore backend runs HTTP-only on `8080`, remains pod-local, and is
exposed only through the sidecar. The Kubernetes Service exposes the sidecar
ports `9445` and `9080`. The first deployment uses
`networknt/openapi-petstore:2.3.4` and `networknt/http-sidecar:2.2.1`.
The sidecar service identity is `com.networknt.petstore-lpsc-3.1.0` in `dev`;
the backend service identity is `com.networknt.petstore-3.1.0` in `dev`.

During the registry migration, `PortalRegistry` should support local
`direct-registry.directUrls` overrides before controller discovery. This keeps
the controller contract clean, allows incremental migration, and lets
operators remove static mappings one service at a time.

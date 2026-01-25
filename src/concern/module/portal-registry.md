# Portal Registry

The `portal-registry` module is a decentralized service registry and discovery implementation designed to replace external tools like Consul or Etcd. It connects directly to the `light-controller`, which acts as the centralized control panel for the light-portal ecosystem.

### Why Portal Registry?

While Consul is a popular choice for service discovery, `portal-registry` was developed to address several pain points:
1.  **Resource Efficiency**: Eliminates the need for local Consul agents on every node, reducing infrastructure overhead.
2.  **WebSocket-Based Updates**: Unlike Consul's long-polling (blocking queries) for discovery updates, `portal-registry` uses WebSockets. This significantly reduces thread utilization in cloud environments.
3.  **Simpler Procurement**: Provides an out-of-the-box solution within the `light-platform`, avoiding lengthy enterprise software approval cycles often required for third-party tools.

### Implementation Guide

To use the portal registry in your `light-4j` service, follow these steps:

#### 1. Dependency Management
Add the following dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>portal-registry</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

#### 2. Enable Registry in server.yml
Ensure that registry support is enabled in your `server.yml` or via `values.yml`:

```yaml
enableRegistry: ${server.enableRegistry:true}
```

#### 3. Service Configuration (service.yml)
Update your `service.yml` to use `PortalRegistry` as the registry implementation. Note the use of the `light` protocol in the URL configuration.

```yaml
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: light
      host: localhost
      port: 8080
      path: portal
      parameters:
        registryRetryPeriod: '30000'
- com.networknt.portal.registry.client.PortalRegistryClient:
  - com.networknt.portal.registry.client.PortalRegistryClientImpl
- com.networknt.registry.Registry:
  - com.networknt.portal.registry.PortalRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster
```

#### 4. Module Configuration (portal-registry.yml)
Create or update the `portal-registry.yml` file to configure connectivity to the controller.

```yaml
---
# Portal URL for accessing controller API. 
portalUrl: ${portalRegistry.portalUrl:https://localhost:8443}

# JWT token for authorized access to the light-controller.
portalToken: ${portalRegistry.portalToken:}

# Max requests before resetting connection (HTTP/2 optimization).
maxReqPerConn: ${portalRegistry.maxReqPerConn:1000000}

# Timing for critical health state and de-registration.
deregisterAfter: ${portalRegistry.deregisterAfter:120000}
checkInterval: ${portalRegistry.checkInterval:10000}

# Health Check Mechanisms
httpCheck: ${portalRegistry.httpCheck:false}
ttlCheck: ${portalRegistry.ttlCheck:true}
healthPath: ${portalRegistry.healthPath:/health/}
```

### Operational Workflow

#### Registry (Service-Side)
When a service starts, it registers itself with the `light-controller`. During heartbeats (TTL) or polling (HTTP), the controller maintains the service status. When the service shuts down gracefully, it unregisters itself to remove stale nodes immediately.

#### Discovery (Consumer-Side)
1.  **Initial Query**: On the first request to a downstream service, the consumer queries the `light-controller` for available nodes based on `serviceId` and environment tags.
2.  **Streaming Updates**: After the initial lookup, the `portal-registry` opens a WebSocket connection to the controller. Any changes to the downstream service instances (new nodes, failures, or de-registrations) are pushed to the consumer in real-time to update its local cache.

### Auto-Registration & Security

The module automatically registers itself with the `ModuleRegistry` during configuration loading. This allows you to inspect the registry status and configuration via the [Server Info](/concern/admin/server-info.md) endpoint. Sensitive tokens like `portalToken` are automatically masked in these logs/endpoints.

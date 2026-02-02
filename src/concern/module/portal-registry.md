# Portal Registry

The `portal-registry` module enables light-4j services to self-register with a centralized Light Portal controller (light-controller) instead of using local agents like Consul. It addresses the issues of resource utilization, long polling inefficiencies, and complex setup associated with Consul.

## Key Features

1.  **Centralized Registration**: Services register directly with the controller cluster.
2.  **WebSocket Discovery**: Instead of HTTP blocking queries/long polling, the client opens a WebSocket connection to the controller to receive real-time updates about service instances. This significantly reduces thread usage and network overhead.
3.  **No Local Agent**: Eliminates the need for a sidecar or node-level agent.

## Configuration

To switch from Consul to Portal Registry, you need to update `pom.xml`, `service.yml`, and add `portal-registry.yml`.

### 1. `pom.xml`

Replace `consul` dependency with `portal-registry`.

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>portal-registry</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

### 2. `service.yml`

Configure the `Registry` singleton to use `PortalRegistry`.

```yaml
singletons:
  # Define the URL parameters for the registry connection
  - com.networknt.registry.URL:
      - com.networknt.registry.URLImpl:
          protocol: light
          host: localhost
          port: 8080
          path: portal
          parameters:
            registryRetryPeriod: '30000'
  # The client implementation (connects to Portal)
  - com.networknt.portal.registry.client.PortalRegistryClient:
      - com.networknt.portal.registry.client.PortalRegistryClientImpl
  # The Registry interface implementation
  - com.networknt.registry.Registry:
      - com.networknt.portal.registry.PortalRegistry
```

### 3. `portal-registry.yml`

This module has its own configuration file.

| Property | Description | Default |
| :--- | :--- | :--- |
| `portalUrl` | URL of the light-controller (e.g., `https://lightapi.net` or `https://light-controller`). | `https://lightapi.net` |
| `portalToken` | Bootstrap token for accessing the controller. | - |
| `checkInterval` | Interval for health checks (e.g., `10s`). | `10000` (ms) |
| `deregisterAfter` | Time to wait after failure before deregistering. | `120000` (ms) |
| `httpCheck` | Enable HTTP health check (Controller polls Service). | `false` |
| `ttlCheck` | Enable TTL heartbeat (Service calls Controller). | `true` |
| `healthPath` | Path for HTTP health check (e.g., `/health/`). | `/health/` |

**Example `portal-registry.yml`:**

```yaml
portalUrl: https://light-controller-test.networknt.com
portalToken: ${portalRegistry.portalToken}
ttlCheck: true
```

### 4. `server.yml`

Ensure registry is enabled.

```yaml
enableRegistry: true
serviceId: com.networknt.petstore-1.0.0
```

## How It Works

### Registration (Service Startup)

When a service starts, `PortalRegistry` sends a registration request to the `portalUrl`. It includes the simplified service metadata (host, port, serviceId, environment).
Depending on configuration:
-   **TTL Check**: The service starts a background thread (`PortalRegistryHeartbeatManager`) to send periodic heartbeats to the controller.
-   **HTTP Check**: The controller is expected to poll the service's `healthPath`.

### Discovery (Client Side)

When a client (like `light-router`) needs to find a service:
1.  **Initial Lookup**: It queries the controller for the current list of healthy instances for a given `serviceId` and `tag`.
2.  **Subscription**: It establishes a WebSocket connection to `wss://{portalUrl}/ws`.
3.  **Real-time Updates**: When service instances change (up/down/scaling), the controller pushes the new list to the client via the WebSocket. The client updates its local cache immediately.

This WebSocket approach is much more efficient than Consul's blocking query, especially when subscribing to many services, as it multiplexes updates over a single connection.

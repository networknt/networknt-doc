# Consul Registry

The `consul` module integrates with HashiCorp Consul to provide service registry and discovery for the light-4j framework. It allows services to automatically register themselves upon startup and discovering other services using client-side load balancing.

## Overview

-   **Registry**: When a service starts, it registers itself with the Consul agent. It deregisters on graceful shutdown.
-   **Discovery**: Clients using `Cluster` and `LoadBalance` modules can discover available service instances. The `ConsulRegistry` implementation uses blocking queries (long polling) to receive real-time updates from Consul about service health and availability.
-   **Health Checks**: Supports TCP, HTTP, and TTL health checks to ensure only healthy instances are returned to clients.

## Configuration

The module requires configuration in multiple files.

### 1. `service.yml`

You must configure the `Registry` singleton to use `ConsulRegistry`.

```yaml
singletons:
  - com.networknt.registry.URL:
      - com.networknt.registry.URLImpl:
          protocol: light
          host: localhost
          port: 8080
          path: consul
          parameters:
            registryRetryPeriod: '30000'
  - com.networknt.consul.client.ConsulClient:
      - com.networknt.consul.client.ConsulClientImpl
  - com.networknt.registry.Registry:
      - com.networknt.consul.ConsulRegistry
```

### 2. `consul.yml`

This file controls the connection to Consul and the health check behavior.

| Property | Description | Default |
| :--- | :--- | :--- |
| `consulUrl` | URL of the Consul agent. | `http://localhost:8500` |
| `consulToken` | Access token for Consul ACL. | `the_one_ring` |
| `checkInterval` | Interval for health checks (e.g., `10s`). | `10s` |
| `deregisterAfter` | Time to wait after a failure before deregistering. | `2m` |
| `tcpCheck` | Enable TCP ping health check. | `false` |
| `httpCheck` | Enable HTTP endpoint health check. | `false` |
| `ttlCheck` | Enable active TTL heartbeat. | `true` |
| `wait` | Long polling wait time (max 10m). | `600s` |

**Health Check Options:**

-   **TCP**: Consul pings the service IP/Port. Simple, but less reliable in containerized environments.
-   **HTTP**: Consul calls `/health/{serviceId}`. Recommended for most services as it proves the application is responsive.
-   **TTL**: The service actively sends heartbeats to Consul. Useful if the service is behind a NAT or cannot be reached by Consul.

### 3. `server.yml`

The service must be configured to enable the registry and usually dynamic ports (especially for Kubernetes).

```yaml
serviceId: com.networknt.petstore-1.0.0
enableRegistry: true
dynamicPort: true
minPort: 2400
maxPort: 2500
```

### 4. `secret.yml`

For security, the Consul token should be managed in `secret.yml`.

```yaml
consulToken: your-secure-jwt-or-uuid-token
```

## Deployment

When deploying to Kubernetes:

1.  **Host Network**: Services should use `hostNetwork: true` so they register with the Node's IP address.
2.  **Downward API**: Pass the host IP to the container so the service knows what IP to register.

```yaml
    spec:
      hostNetwork: true
      containers:
      - env:
        - name: STATUS_HOST_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
```

## Blocking Queries

`ConsulRegistry` uses Consul's blocking query feature. Instead of polling every X seconds, it opens a connection that Consul holds open for up to `wait` time (default 10 minutes). If the service catalog changes (new instance, health failure), Consul returns the result immediately. This provides near-real-time updates with minimal overhead.

## Usage

You rarely interact with `ConsulRegistry` directly. Instead, you use the `Cluster` or `Http2Client` which uses the registry behind the scenes.

```java
// The cluster lookups will automatically use ConsulRegistry to find instances
Cluster cluster = SingletonServiceFactory.getBean(Cluster.class);
String url = cluster.serviceToUrl("https", "com.networknt.petstore-1.0.0", "dev", null);
```

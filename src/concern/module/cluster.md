# Cluster

The `Cluster` module acts as the orchestrator for client-side service discovery. It integrates the [Service Registry][] (to find service instances) and [Load Balance][] (to select the best instance) modules, providing a simple interface for clients to resolve a service ID to a concrete URL for invocation.

It maintains a local cache of discovered services to minimize calls to the underlying registry (Consul, ZooKeeper, or Direct) and handles notifications when service instances change.

## Cluster Interface

The module's core is the `Cluster` interface, which provides methods to resolve service addresses.

```java
public interface Cluster {
    // Resolve a serviceId to a single URL string (e.g., "https://192.168.1.10:8443")
    String serviceToUrl(String protocol, String serviceId, String tag, String requestKey);

    // Get a list of all available service instances as URIs
    List<URI> services(String protocol, String serviceId, String tag);
}
```

### Parameters

-   **protocol**: `http` or `https`.
-   **serviceId**: The unique identifier of the service (e.g., `com.networknt.petstore-1.0.0`).
-   **tag**: An optional environment tag (e.g., `dev`, `prod`). If used, discovery will be filtered by this tag.
-   **requestKey**: An optional key used for load balancing (e.g., for Consistent Hash). Can be null for policies like Round Robin.

## Default Implementation: LightCluster

`LightCluster` is the default implementation provided by `light-4j`.

### Workflow

1.  **Check Cache**: It first checks its internal in-memory cache (`serviceMap`) for the list of URLs associated with the `serviceId` and `tag`.
2.  **Discovery (if cache miss)**:
    -   It creates a "subscribe URL" representing the service.
    -   It subscribes to the `Registry` (Direct, Consul, etc.) to receive updates (notifications) about this service.
    -   It performs an initial discovery lookup to get the current list of instances.
    -   It populates the cache with the results.
3.  **Load Balance**:
    -   Once the list of URLs is retrieved (from cache or discovery), `LightCluster` delegates to the `LoadBalance` module.
    -   The load balancer selects a single `URL` based on the configured strategy (Round Robin, Local First, etc.) and the `requestKey`.
4.  **Return**: The selected URL is returned as a string.

### Caching and Updates

`LightCluster` registers a `ClusterNotifyListener` with the registry. When service instances change (e.g., a node goes down or a new one comes up), the registry notifies this listener, ensuring the `serviceMap` cache is updated in near real-time.

## Configuration

The cluster module does not have its own configuration file (e.g., `cluster.yml`). Instead, it relies on dependency injection via `service.yml` to wire up the `Cluster`, `Registry`, and `LoadBalance` implementations.

### configuring `service.yml`

To use `LightCluster`, ensure your `service.yml` typically looks like this (it may vary depending on your specific registry choice):

```yaml
singletons:
  # Registry configuration (e.g., DirectRegistry for local testing)
  - com.networknt.registry.URL:
      - com.networknt.registry.URLImpl:
          protocol: https
          host: localhost
          port: 8080
          path: direct
          parameters:
            com.networknt.petstore-1.0.0: https://localhost:8443,https://localhost:8444
  - com.networknt.registry.Registry:
      - com.networknt.registry.support.DirectRegistry

  # Load Balancer configuration
  - com.networknt.balance.LoadBalance:
      - com.networknt.balance.RoundRobinLoadBalance

  # Cluster configuration
  - com.networknt.cluster.Cluster:
      - com.networknt.cluster.LightCluster
```

In a production environment using Consul, the `service.yml` would specify `ConsulRegistry` instead.

## Usage

You primarily use the `Cluster` instance via the `Http2Client` or `Client` methods that abstract this lookup, but you can also use it directly if needed.

```java
Cluster cluster = SingletonServiceFactory.getBean(Cluster.class);

// Find a healthy instance for Petstore service
String url = cluster.serviceToUrl("https", "com.networknt.petstore-1.0.0", "dev", null);

if (url != null) {
    // Make call to url
}
```

[Service Registry]: /concern/module/registry/
[Load Balance]: /concern/module/load-balance/

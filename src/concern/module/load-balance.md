# Load Balance

The light-4j platform encourages client-side discovery to avoid proxies in front of multiple service instances. This architecture reduces network hops and latency. The client-side load balancer is responsible for selecting a single available service instance from a list of discovered URLs for a downstream request.

## LoadBalance Interface

All load balancing strategies enforce the `LoadBalance` interface.

```java
public interface LoadBalance {
    /**
     * Select one url from a list of url with requestKey as optional.
     *
     * @param urls List of available URLs
     * @param serviceId Service ID
     * @param tag Service tag (optional)
     * @param requestKey Optional key for hashing (e.g., userId, clientId)
     * @return Selected URL
     */
    URL select(List<URL> urls, String serviceId, String tag, String requestKey);

    default int getPositive(int originValue){
        return 0x7fffffff & originValue;
    }
}
```

## Strategies

### Round Robin

The `RoundRobinLoadBalance` strategy picks a URL from the list sequentially. It distributes the load equally across all available instances.

-   **Assumption**: All service instances have similar resource configurations and capabilities.
-   **Mechanism**: Uses an internal `AtomicInteger` index per service to cycle through the list.
-   **Parameters**: The `requestKey` is ignored.

### Local First

The `LocalFirstLoadBalance` strategy prioritizes service instances running on the same host (same IP address) to minimize network latency.

-   **Mechanism**:
    1.  It identifies the local IP address.
    2.  It filters the list of available URLs for those matching the local IP.
    3.  If local instances are found, it performs **Round Robin** among them.
    4.  If no local instances are found, it falls back to **Round Robin** across all available URLs.
-   **Use Case**: Ideal for deployments where multiple services run on the same physical or virtual machine (e.g., standalone Java processes, `light-hybrid-4j` services).

### Consistent Hash

The `ConsistentHashLoadBalance` strategy ensures that requests with the same `requestKey` (e.g., `client_id`, `user_id`) are routed to the same service instance. This is useful for caching or stateful services (Data Sharding / Z-Axis Scaling).

-   **Mechanism**: Uses the hash code of the `requestKey` to select a specific instance.
-   **Current Status**: Basic implementation using modulo hashing. A more advanced consistent hashing ring implementation is planned.

## Configuration

The load balancer implementation is typically configured as a singleton in `service.yml`. Only one implementation should be active per client instance during runtime.

### Example: Round Robin Configuration

In `service.yml`:

```yaml
singletons:
  - com.networknt.balance.LoadBalance:
    - com.networknt.balance.RoundRobinLoadBalance
```

### Example: Local First Configuration

In `service.yml`:

```yaml
singletons:
  - com.networknt.balance.LoadBalance:
    - com.networknt.balance.LocalFirstLoadBalance
```

## Usage

The load balancer is used internally by `Cluster` implementations or directly by clients (like `light-consumer-4j`) after service discovery returns a list of healthy nodes.

```java
// Example usage pattern
List<URL> urls = registry.discover(serviceId, tag);
URL selectedUrl = loadBalance.select(urls, serviceId, tag, requestKey);
```

# Cache Manager

The `CacheManager` module provides a unified interface for managing in-memory caches within the light-4j application. It abstracts the underlying caching implementation, allowing developers to configure and use caches without tying their code to a specific library.

The default implementation provided by `light-4j` is based on [Caffeine](https://github.com/ben-manes/caffeine), a high-performance Java caching library.

## CacheManager Interface

The `CacheManager` interface defines standard operations for interacting with caches:

```java
public interface CacheManager {
    void addCache(String cacheName, long maxSize, long expiryInMinutes);
    Map<Object, Object> getCache(String cacheName);
    void put(String cacheName, String key, Object value);
    Object get(String cacheName, String key);
    void delete(String cacheName, String key);
    void removeCache(String cacheName);
    int getSize(String cacheName);
}
```

## Configuration

The configuration is managed by `CacheConfig` and corresponds to `cache.yml` (or `cache.json`/`cache.properties`).
The configuration allows defining multiple named caches, each with its own size limit and expiration policy.

### Configuration Properties

| Property | Type | Description |
| :--- | :--- | :--- |
| `caches` | list | A list of cache definitions. |

### Cache Item Properties

Each item in the `caches` list contains:

| Property | Type | Description |
| :--- | :--- | :--- |
| `cacheName` | string | Unique name for the cache. |
| `expiryInMinutes` | int | Expiration time in minutes (TTL). |
| `maxSize` | int | Maximum number of entries in the cache. |

### Configuration Example (cache.yml)

```yaml
# Cache Configuration
caches:
  - cacheName: jwt
    expiryInMinutes: 10
    maxSize: 10000
  - cacheName: jwk
    expiryInMinutes: 1440
    maxSize: 100
```

### Values Example (values.yml)

You can override the cache configuration in `values.yml`. Note that since `caches` is a list, you typically override the entire list or use stringified JSON if supported by the config loader, but YAML list format is standard.

```yaml
cache.caches:
  - cacheName: userCache
    expiryInMinutes: 60
    maxSize: 500
  - cacheName: tokenCache
    expiryInMinutes: 15
    maxSize: 2000
```

## Usage

To use the cache manager, you can retrieve the singleton instance and interact with your named caches.

### Retrieving the Instance

The `CacheManager` follows a singleton pattern (or can be injected via dependency injection if using a DI framework).

```java
CacheManager cacheManager = CacheManager.getInstance();
```

### Basic Operations

```java
// Put a value into the cache
cacheManager.put("userCache", "userId123", userObject);

// Get a value from the cache
Object value = cacheManager.get("userCache", "userId123");
if (value != null) {
    // Cache hit
}

// Delete a specific key
cacheManager.delete("userCache", "userId123");

// Remove an entire cache
cacheManager.removeCache("userCache");
```

## Implementation

The default implementation `CaffeineCacheManager` uses Caffeine's `Caffeine.newBuilder()` to create caches with:
-   `maximumSize`: Limits the cache size.
-   `expireAfterWrite`: Expires entries after the specified duration.

This implementation is registered automatically via `SingletonServiceFactory` or module registration when `CacheConfig` loads.

# Configuration Hot Reload Design

## Introduction

In the light-4j framework, minimizing downtime is crucial for microservices. The Configuration Hot Reload feature allows services to update their configuration at runtime without restarting the server. This design document outlines the centralized caching architecture used to achieve consistent and efficient hot reloads.

## Architecture Evolution

### Previous Approach (Decentralized)
Initially, configuration reload was handled in a decentralized manner:
- Each handler maintained its own static configuration object.
- A `reload()` method was required on every handler to manually refresh this object.
- The `ConfigReloadHandler` used reflection to search for and invoke these `reload()` methods.

**Drawbacks:**
- **Inconsistency:** Different parts of the application could hold different versions of the configuration.
- **Complexity:** Every handler needed boilerplate code for reloading.
- **State Management:** Singleton classes (like `ClientConfig`) often held stale references that were difficult to update.

### Current Approach (Centralized Cache)
The new architecture centralizes the "source of truth" within the `Config` class itself.

- **Centralized Cache**: The `Config` class maintains a `ConcurrentHashMap` of all loaded configurations.
- **Cache Invalidation**: Instead of notifying components to reload, we simply invalidate the specific entry in the central cache.
- **Lazy Loading**: Consumers (Handlers, Managers) fetch the configuration from the `Config` class at the moment of use. If the cache is empty (cleared), `Config` reloads it from the source files.

## Detailed Design

### 1. The Config Core (`Config.java`)

The `Config` class is enhanced to support targeted cache invalidation.

```java
public abstract class Config {
    // ... existing methods
    
    // New method to remove a specific config from memory
    public abstract void clearConfigCache(String configName);
}
```

When `clearConfigCache("my-config")` is called:
1. The entry for "my-config" is removed from the internal `configCache`.
2. The next time `getJsonMapConfig("my-config")` or `getJsonObjectConfig(...)` is called, the `Config` class detects the miss and reloads the file from the filesystem or external config server.

### 2. The Admin Endpoint (`ConfigReloadHandler`)

The `ConfigReloadHandler` exposes the `/adm/config-reload` endpoint. Its responsibility has been simplified:

1. **Receive Request**: Accepts a list of modules/plugins to reload.
2. **Resolve Config Names**: Looks up the configuration file names associated with the requested classes using the `ModuleRegistry`.
3. **Invalidate Cache**: Calls `Config.getInstance().clearConfigCache(configName)` for each identified module.

It no longer relies on reflection to call methods on the handlers.

### 3. Configuration Consumers

Handlers and other components must follow a stateless pattern regarding configuration.

**Anti-Pattern (Old Way):**
```java
public class MyHandler {
    static MyConfig config = MyConfig.load(); // Static load at startup
    
    public void handleRequest(...) {
        // Use static config - will remain stale even if file changes
        if (config.isEnabled()) ... 
    }
}
```

**Recommended Pattern (New Way):**
```java
public class MyHandler {
    // No static config field
    
    public void handleRequest(...) {
        // Fetch fresh config from central cache every time
        // This is fast due to HashMap lookup
        MyConfig config = MyConfig.load(); 
        
        if (config.isEnabled()) ...
    }
}
```

**Implementation in Config Classes:**
The `load()` method in configuration classes (e.g., `CorrelationConfig`) simply delegates to the cached `Config` methods:

```java
private CorrelationConfig(String configName) {
    // Always ask Config class for the Map. 
    // If cache was cleared, this call triggers a file reload.
    mappedConfig = Config.getInstance().getJsonMapConfig(configName); 
}
```

### 4. Handling Singletons (e.g., ClientConfig)

For Singleton classes that parse configuration into complex objects, they must check if the underlying configuration has changed.

```java
public static ClientConfig get() {
    // Check if the Map instance in Config.java is different from what we typically hold
    Map<String, Object> currentMap = Config.getInstance().getJsonMapConfig(CONFIG_NAME);
    if (instance == null || instance.mappedConfig != currentMap) {
        synchronized (ClientConfig.class) {
            if (instance == null || instance.mappedConfig != currentMap) {
                instance = new ClientConfig(); // Re-parse and create new instance
            }
        }
    }
    return instance;
}
```

    }
    return instance;
}


### 5. Lazy Rebuild on Config Change

Some handlers (like `LightProxyHandler`) maintain expensive internal objects that depend on the configuration (e.g., `LoadBalancingProxyClient`, `ProxyHandler`). Recreating these on every request is not feasible due to performance. However, they must still react to configuration changes.

For these cases, we use a **Lazy Rebuild** pattern:

1.  **Volatile Config Reference**: The handler maintains a `volatile` reference to its configuration object.
2.  **Check on Request**: At the start of `handleRequest`, it checks if the cached config object is the same as the one returned by `Config.load()`.
3.  **Rebuild if Changed**: If the reference has changed (identity check), it synchronizes and rebuilds the internal components.

**Example Implementation (`LightProxyHandler`):**

```java
public class LightProxyHandler implements HttpHandler {
    private volatile ProxyConfig config;
    private volatile ProxyHandler proxyHandler;

    public LightProxyHandler() {
        this.config = ProxyConfig.load();
        buildProxy(); // Initial build
    }

    private void buildProxy() {
        // Expensive object creation based on config
        this.proxyHandler = ProxyHandler.builder()
                .setProxyClient(new LoadBalancingProxyClient()...)
                .build();
    }

    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        ProxyConfig newConfig = ProxyConfig.load();
        // Identity check: ultra-fast
        if (newConfig != config) {
            synchronized (this) {
                newConfig = ProxyConfig.load(); // Double-check
                if (newConfig != config) {
                    config = newConfig;
                    buildProxy(); // Rebuild internal components
                }
            }
        }
        // Use the (potentially new) proxyHandler
        proxyHandler.handleRequest(exchange);
    }
}
```

This pattern ensures safe updates without the overhead of rebuilding on every request, and without requiring a manual `reload()` method.

### 6. Config Class Implementation Pattern (Singleton with Caching)


For configuration classes that are frequently accessed (per request), instantiating a new object each time can be expensive. We recommend implementing a Singleton pattern that caches the configuration object and only invalidates it when the underlying configuration map changes.

**Example Implementation (`ApiKeyConfig`):**

```java
public class ApiKeyConfig {
    private static final String CONFIG_NAME = "apikey";
    // Cache the instance
    private static ApiKeyConfig instance;
    private final Map<String, Object> mappedConfig;
    
    // Private constructor to force use of load()
    private ApiKeyConfig(String configName) {
        mappedConfig = Config.getInstance().getJsonMapConfig(configName);
        setConfigData();
    }

    public static ApiKeyConfig load() {
        return load(CONFIG_NAME);
    }

    public static ApiKeyConfig load(String configName) {
        // optimistically check if we have a valid cached instance
        Map<String, Object> mappedConfig = Config.getInstance().getJsonMapConfig(configName);
        if (instance != null && instance.getMappedConfig() == mappedConfig) {
            return instance;
        }
        
        // Double-checked locking for thread safety
        synchronized (ApiKeyConfig.class) {
            mappedConfig = Config.getInstance().getJsonMapConfig(configName);
            if (instance != null && instance.getMappedConfig() == mappedConfig) {
                return instance;
            }
            instance = new ApiKeyConfig(configName);
            // Register the module with the configuration. masking the apiKey property.
            // As apiKeys are in the config file, we need to mask them.
            List<String> masks = new ArrayList<>();
            // if hashEnabled, there is no need to mask in the first place.
            if(!instance.hashEnabled) {
                masks.add("apiKey");
            }
            ModuleRegistry.registerModule(configName, ApiKeyConfig.class.getName(), Config.getNoneDecryptedInstance().getJsonMapConfigNoCache(configName), masks);

            return instance;
        }
    }
    
    public Map<String, Object> getMappedConfig() {
        return mappedConfig;
    }
}
```

This pattern ensures that:
1. **Performance**: Applications use the cached `instance` for the majority of requests (fast reference check).
2. **Freshness**: If `Config.getInstance().getJsonMapConfig(name)` returns a new Map object (due to a reload), the equality check fails, and a new `ApiKeyConfig` is created.
3. **Consistency**: The `handleRequest` method still calls `ApiKeyConfig.load()`, but receives the singleton instance transparently.

### 6. Thread Safety

The `Config` class handles concurrent access using the **Double-Checked Locking** pattern to ensure that the configuration file is loaded **exactly once**, even if multiple threads request it simultaneously immediately after the cache is cleared.

**Scenario:**
1. **Thread A** and **Thread B** both handle a request for `MyHandler`.
2. Both call `MyConfig.load()`, which calls `Config.getJsonMapConfig("my-config")`.
3. Both see that the cache is empty (returning `null`) because it was just cleared by the reload handler.
4. **Thread A** acquires the lock (`synchronized`). **Thread B** waits.
5. **Thread A** checks the cache again (still null), loads the file from disk, puts it in the `configCache`, and releases the lock.
6. **Thread B** acquires the lock.
7. **Thread B** checks the cache again. This time it finds the config loaded by **Thread A**.
8. **Thread B** uses the existing config without loading from disk.

This ensures no race conditions or redundant file I/O operations occur in high-concurrency environments.

## Workflow Summary

1. **Update**: User updates `values.yml` or a config file on the server/filesystem.
2. **Trigger**: User calls `POST https://host:port/adm/config-reload` with the module name.
3. **Clear**: `ConfigReloadHandler` tells `Config.java` to `clearConfigCache` for that module.
4. **Processing**:
    - **Step 4a**: Request A arrives at `MyHandler`.
    - **Step 4b**: `MyHandler` calls `MyConfig.load()`.
    - **Step 4c**: `MyConfig` calls `Config.getJsonMapConfig()`.
    - **Step 4d**: `Config` sees cache miss, reads file from disk, parses it, puts it in cache, and returns it.
    - **Step 4e**: `MyHandler` processes request with NEW configuration.
5. **Subsequent Requests**: Step 4d is skipped; data is served instantly from memory.

## Configuration Consistency During Request Processing

### Can Config Objects Change During a Request?

A common question arises: **Can the config object created in `handleRequest` be changed during the request/response exchange?**

**Short Answer**: Theoretically possible but extremely unlikely, and the design handles this correctly.

### Understanding the Behavior

When a handler processes a request using the recommended pattern:

```java
public void handleRequest(HttpServerExchange exchange) {
    MyConfig config = MyConfig.load(); // Creates local reference
    
    // Use config throughout request processing
    if (config.isEnabled()) {
        // ... process request
    }
}
```

The `config` variable is a **local reference** to a configuration object. Here's what happens:

1. **Cache Hit**: `MyConfig.load()` calls `Config.getJsonMapConfig(configName)` which returns a reference to the cached `Map<String, Object>`.
2. **Object Construction**: A new `MyConfig` object is created, wrapping this Map reference.
3. **Local Scope**: The `config` variable holds this reference for the duration of the request.

### Scenario: Reload During Request Processing

Consider this timeline:

1. **T1**: Request A starts, calls `MyConfig.load()`, gets reference to Config Object v1
2. **T2**: Admin calls `/adm/config-reload`, cache is cleared
3. **T3**: Request B starts, calls `MyConfig.load()`, triggers reload, gets reference to Config Object v2
4. **T4**: Request A continues processing with Config Object v1
5. **T5**: Request A completes successfully with Config Object v1

**Key Points**:

- **Request A** maintains its reference to the original config object throughout its lifecycle
- **Request B** gets a new config object with reloaded values
- Both requests process correctly with **consistent** configuration for their entire duration
- No race conditions or inconsistent state within a single request

### Why This Design is Safe

#### 1. **Immutable Config Objects**
Configuration objects are effectively immutable once constructed:
```java
private MyConfig(String configName) {
    mappedConfig = Config.getInstance().getJsonMapConfig(configName);
    setConfigData(); // Parses and sets final fields
}
```
Fields are set during construction and never modified afterward.

#### 2. **Local Variable Isolation**
Each request has its own local `config` variable:
- The reference is stored on the thread's stack
- Even if the cache is cleared, the reference remains valid
- The underlying Map object continues to exist until no references remain (garbage collection)

#### 3. **Per-Request Consistency**
This design ensures that each request has a **consistent view** of configuration from start to finish:
- No mid-request configuration changes
- Predictable behavior throughout request processing
- Easier debugging and reasoning about request flow

#### 4. **Graceful Transition**
The architecture enables zero-downtime config updates:
- **In-flight requests**: Complete with the config they started with
- **New requests**: Use the updated configuration
- **No interruption**: No requests fail due to config reload

### Edge Cases and Considerations

#### Long-Running Requests
For requests that take significant time to process (e.g., minutes):
- The request will complete with the configuration it started with
- If config is reloaded during processing, the request continues with "old" config
- **This is correct behavior** - we want consistent config per request

#### High-Concurrency Scenarios
During a config reload under heavy load:
- Multiple threads may simultaneously detect cache miss
- Double-checked locking ensures only one thread loads from disk
- All threads eventually get the same new config instance
- No duplicate file I/O or parsing overhead

### Memory Implications

**Question**: If old config objects are still referenced by in-flight requests, do we have memory leaks?

**Answer**: No, this is handled by Java's garbage collection:

1. Request A holds reference to Config Object v1
2. Cache is cleared and reloaded with Config Object v2
3. Request A completes and goes out of scope
4. Config Object v1 has no more references
5. Garbage collector reclaims Config Object v1

The memory overhead is minimal and temporary, lasting only as long as the longest in-flight request.

### Best Practices

To ensure optimal behavior with hot reload:

1. **Always Load Fresh**: Call `MyConfig.load()` at the start of `handleRequest`, not in constructor
   ```java
   // ✅ GOOD
   public void handleRequest(...) {
       MyConfig config = MyConfig.load();
   }
   
   // ❌ BAD
   private static MyConfig config = MyConfig.load();
   ```

2. **Use Local Variables**: Store config in local variables, not instance fields
   ```java
   // ✅ GOOD
   MyConfig config = MyConfig.load();
   
   // ❌ BAD
   this.config = MyConfig.load();
   ```

3. **Don't Cache in Handlers**: Let the `Config` class handle caching
   ```java
   // ✅ GOOD - Load on each request
   public void handleRequest(...) {
       MyConfig config = MyConfig.load();
   }
   
   // ❌ BAD - Caching in handler
   private MyConfig cachedConfig;
   public void handleRequest(...) {
       if (cachedConfig == null) cachedConfig = MyConfig.load();
   }
   ```

### Summary

The centralized cache design ensures:
- **Thread Safety**: Multiple threads can safely reload and access config
- **Request Consistency**: Each request has a stable config view from start to finish  
- **Zero Downtime**: Config updates don't interrupt in-flight requests
- **Performance**: HashMap lookups are extremely fast (O(1))
- **Simplicity**: No complex synchronization needed in handlers

## Benefits

1. **Performance**: Only one disk read per reload cycle. Subsequent accesses are Hash Map lookups.
2. **Reliability**: Config state is consistent. No chance of "half-reloaded" application state.
3. **Simplicity**: drastic reduction in boilerplate code across the framework.

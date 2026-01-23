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

### 5. Thread Safety

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

## Benefits

1. **Performance**: Only one disk read per reload cycle. Subsequent accesses are Hash Map lookups.
2. **Reliability**: Config state is consistent. No chance of "half-reloaded" application state.
3. **Simplicity**: drastic reduction in boilerplate code across the framework.

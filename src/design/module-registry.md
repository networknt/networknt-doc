# Module Registry Design

## Introduction

The Module Registry is a core component in the light-4j framework that tracks all active modules (middleware handlers, plugins, utilities) and their current configurations. This info is primarily exposed via the `/server/info` admin endpoint, allowing operators to verify the runtime state of the service.

With the introduction of Centralized Configuration Management and Hot Reload, the Module Registry design has been updated to ensure consistency updates without manual intervention from handlers.

## Architecture

### Centralized Registration

Previously, each `Handler` was responsible for registering its configuration in its constructor or `reload()` method. This led to decentralized logic and potential inconsistencies during hot reloads.

In the new design, **Configuration Classes (`*Config.java`)** are responsible for registering themselves with the `ModuleRegistry` immediately upon instantiation.

**Workflow:**
1. `Handler` requests config: `MyConfig.load()`.
2. `Config` class loads data from the central cache.
3. `MyConfig` constructor initializes fields.
4. `MyConfig` constructor calls `ModuleRegistry.registerModule(...)`.

### Caching and Optimization

Since `MyConfig` objects are instantiated per-request (to ensure fresh config is used), the registration call happens frequently. To prevent performance degradation, `ModuleRegistry` implements an **Identity Cache**.

```java
// Key: configName + ":" + moduleClass
private static final Map<String, Object> registryCache = new HashMap<>();

public static void registerModule(String configName, String moduleClass, Map<String, Object> config, List<String> masks) {
    String key = configName + ":" + moduleClass;
    
    // Optimization: Identity Check
    if (config != null && registryCache.get(key) == config) {
        return; // Exact same object already registered, skip overhead
    }
    // ... proceed with registration
}
```

This ensures that while the registration *intent* is declared on every request, the heavy lifting (deep copying, masking) only happens when the configuration object *actually changes* (i.e., after a reload).

### Security and Masking

Configurations often contain sensitive secrets (passwords, API keys). The `ModuleRegistry` must never store or expose these in plain text.

#### 1. Non-Decrypted Config
The `Config` framework supports auto-decryption of `CRYPT:...` values. However, `server/info` should show the original encrypted value (or a mask), not the decrypted secret.

Config classes register the **Non-Decrypted** version of the config map:

```java
// Inside MyConfig constructor
ModuleRegistry.registerModule(
    CONFIG_NAME, 
    MyConfig.class.getName(), 
    Config.getNoneDecryptedInstance().getJsonMapConfigNoCache(CONFIG_NAME), 
    List.of("secretKey", "password")
);
```

#### 2. Deep Copy & Masking
To prevent the `ModuleRegistry` from accidentally modifying the active configuration object (or vice versa), and to safely apply masks without affecting the runtime application:

1. `ModuleRegistry` creates a **Deep Copy** of the configuration map.
2. Masks (e.g., replacing values with `*`) are applied to the **copy**.
3. The masked copy is stored in the registry for display.

## Best Practices for Module Developers

When creating a new module with a configuration file:

1. **Self-Register in Config Constructor**: Call `ModuleRegistry.registerModule` inside your `Config` class constructor.
2. **Use Non-Decrypted Instance**: Always fetch the config for registration using `Config.getNoneDecryptedInstance()`.
3. **Define Masks**: specific attributes that should be masked (e.g., passwords, tokens) in the registration call.
4. **Remove Handler Registration**: Do not call register in your `Handler`, static blocks, or `reload()` methods.

## Example

```java
public class MyConfig {
    private MyConfig(String configName) {
        // 1. Load runtime config
        config = Config.getInstance();
        mappedConfig = config.getJsonMapConfig(configName);
        setConfigData();
        
        // 2. Register with ModuleRegistry
        ModuleRegistry.registerModule(
            configName, 
            MyConfig.class.getName(), 
            Config.getNoneDecryptedInstance().getJsonMapConfigNoCache(configName), 
            List.of("clientSecret") // Mask sensitive fields
        );
    }
}
```

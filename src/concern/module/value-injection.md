# Value Injection

The light-4j framework provides a powerful mechanism to inject values into configuration files from external sources, such as environment variables or a centralized `values.yml` file. This allows for dynamic and flexible configuration management without modifying the application code.

## Basic Usage

Property injection uses the `${key:defaultValue}` pattern. When a configuration file is loaded, any property with this pattern will be replaced by:

1.  **Environment Variable**: If an environment variable matching the `key` (often converted to uppercase with underscores) exists, its value is used.
2.  **values.yml**: If the key is defined in the `values.yml` file, its value is used.
3.  **Default Value**: If neither is found, the `defaultValue` is used.

Example:
```yaml
server.port: ${PORT:8080}
```

## Centralized values.yml

The `values.yml` file (also supports `.yaml` or `.json`) acts as a central repository for all injected values. It is typically located in the same config directory as other configuration files.

When using `values.yml`, you should prefix keys with the name of the configuration file they correspond to, although this is not strictly required if you use generic keys.

Example `values.yml`:
```yaml
server.serviceId: com.networknt.gateway-1.0.0
router.maxRequestTime: 5000
db.user: root
db.password: ${DB_PASS:password}
```

## Self-Injection in values.yml

A recent enhancement allows `values.yml` to support property injection from its own entries. This enables you to reuse values and maintain consistency within the centralized configuration file.

### Implementation Details

To support self-injection, the `ConfigInjection` class follows a **two-pass initialization strategy**:

1.  **Exclusion**: `values.yml` is explicitly excluded from the standard injection process during its initial load. This ensures it is loaded as a raw map, preventing any "cannot be expanded" exceptions during startup.
2.  **Pass 1**: The raw values are loaded into the internal `decryptedValueMap` and `undecryptedValueMap`.
3.  **Pass 2**: Once the maps are assigned to the static fields, the system performs a second pass using `CentralizedManagement.mergeMap`. During this pass, ${key} patterns are resolved against the internal maps, allowing for recursive resolution of self-references and environment variables.

### Example of Self-Injection

```yaml
# values.yml
db.host: localhost
db.user: root
db.url: jdbc:mysql://${db.host}:3306/mydb?user=${db.user}
```

In this example, `db.url` correctly resolves to `jdbc:mysql://localhost:3306/mydb?user=root` during application startup.

## Exclusion List

If you want to prevent certain configuration files from being processed for injection, you can add them to the `exclusionConfigFileList` in `config.yml`.

```yaml
# config.yml
exclusionConfigFileList:
  - openapi
  - values
```

> [!NOTE]
> `values.yml` and `config.yml` are automatically excluded from the initial injection pass by the framework to maintain system stability.

## Decryption

Value injection also supports automatic decryption of values. If a value is encrypted (usually prefixed with `CRYPT:`), it will be decrypted using the configured `Decryptor` before being injected.

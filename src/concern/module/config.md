# Config

The `Config` module is the backbone of the `light-4j` framework, handling the loading and management of configuration files for all modules and the application itself. It follows a "configuration over code" philosophy and supports externalized configuration for containerized deployments.

## Singleton and Instance

The `Config` class is a singleton. It caches loaded configurations to ensure high performance.

```java
// Get the singleton instance
Config config = Config.getInstance();
```

## Formats and Priority

Supported formats are `yml`, `yaml`, and `json`.
The loading priority for a given configuration name (e.g., `server`) is:
1.  `server.yml`
2.  `server.yaml`
3.  `server.json`

YAML is highly recommended due to its readability and support for comments.

## Loading Sequence

The `Config` module loads configuration files in a specific order, allowing for hierarchical overrides. This is critical for managing configurations across different environments (Dev, Test, Prod) without rebuilding the application.

1.  **Externalized Directory**: Specified by the system property `-Dlight-4j-config-dir=/config`. This is the highest priority and is typically used in Docker/Kubernetes to map a volume containing environment-specific configs.
2.  **Application Resources**: Files found in `src/main/resources/config` of your application.
3.  **Module Resources**: Default configuration files packaged within the jar of each module (e.g., `light-4j` core modules).

## Usage

### Loading as Map

Useful for dynamic configurations.

```java
Map<String, Object> serverConfig = Config.getInstance().getJsonMapConfig("server");
int port = (int) serverConfig.get("httpsPort");
```

### Loading as Object (POJO)

Encouraged for type safety. The POJO must map to the JSON/YAML structure.

```java
ServerConfig serverConfig = (ServerConfig) Config.getInstance().getJsonObjectConfig("server", ServerConfig.class);
```

### Loading Strings or Streams

For loading raw files like certificates or text files.

```java
String certContent = Config.getInstance().getStringFromFile("primary.crt");
InputStream is = Config.getInstance().getInputStreamFromFile("primary.crt");
```

## Configuration Injection

Starting from 1.5.26, you can inject values into your `yml` files using the syntax `${VAR:default}`. The values are resolved from:
1.  **values.yml**: A global values file in the config path.
2.  **Environment Variables**: System environment variables.

### Syntax

-   `${VAR}`: value of VAR. Throws exception if missing.
-   `${VAR:default}`: value of VAR. Uses `default` if missing.
-   `${VAR:?error}`: value of VAR. Throws custom error message if missing.
-   `${VAR:$}`: Escapes the injection (keeps `${VAR}`).

### Example

`server.yml`:
```yaml
serviceId: ${SERVER_SERVICEID:com.networknt.petstore-1.0.0}
```

`values.yml`:
```yaml
SERVER_SERVICEID: com.networknt.petstore-2.0.0
```

### Injecting Environment Variables into values.yml

You can also inject environment variables directly into `values.yml`. This allows for a two-stage injection where environment variables populate `values.yml`, and `values.yml` populates other configuration files. This is particularly useful when using secret management tools (like HashiCorp Vault) that populate environment variables, allowing you to manage sensitive data without committing it to the code repository.

`values.yml`:
```yaml
router.hostWhitelist:
  - 192.168.0.*
  - localhost
  - ${DOCKER_HOST_IP}
```

If the environment variable `DOCKER_HOST_IP` is set to `192.168.5.10`, the final loaded configuration for `router.hostWhitelist` will include `192.168.5.10`.

**Benefits:**
*   **Security**: Sensitive information (IPs, credentials) remains in environment variables/secrets engine.
*   **Flexibility**: You can use the same `values.yml` structure across environments, populated by environment-specific variables.


## Handling Secrets

Release 1.6.0+ supports transparent decryption of sensitive values.

### Setup

In your `config.yml` (or `service.yml` in newer versions):

```yaml
decryptorClass: com.networknt.decrypt.AESDecryptor
```

### Decryptors

-   **AESDecryptor**: Uses default password `light`.
-   **AutoAESDecryptor**: Reads password from environment variable `light-4j-config-password`.
-   **ManualAESDecryptor**: Prompts for password in the console/terminal at startup.

If a value in your config file starts with `CRYPT:` (e.g., `CRYPT:adfasdfadf...`), the `Config` module will automatically attempts to decrypt it when loading.

## Best Practices

1.  **Externalize Environments**: Use `-Dlight-4j-config-dir` or mapped volumes in Kubernetes for strictly environment-specific configs (like DB connections, secrets).
2.  **App Level Configs**: Put common configurations that differ from defaults in `src/main/resources/config`.
3.  **Values.yml**: Use `values.yml` to centrally manage variables that are injected into multiple other config files.
4.  **Secrets**: Never commit plain text passwords. Use the encryption feature or Kubernetes Secrets mapped to files in the config directory.

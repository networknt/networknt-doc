# Server Info

### Introduction

The `ServerInfoGetHandler` is a middleware component in the light-4j framework that provides a comprehensive overview of the server's runtime state. This includes:

*   **Environment Info:** Host IP, hostname, DNS, and runtime metrics (processors, memory).
*   **System Properties:** Java version, OS details, timezone.
*   **Component Configuration:** Configuration details for all registered modules.
*   **Specification:** The API specification (OpenAPI, etc.) if applicable.

This handler is crucial for monitoring, debugging, and integration with the light-controller and light-portal for runtime dashboards.

### Configuration

The handler is configured via `info.yml`.

```yaml
# Server info endpoint that can output environment and component along with configuration

# Indicate if the server info is enabled or not.
enableServerInfo: ${info.enableServerInfo:true}

# String list keys that should not be sorted in the normalized info output.
keysToNotSort: ${info.keysToNotSort:["admin", "default", "defaultHandlers", "request", "response"]}

# Downstream configuration (for gateways/sidecars)
# Indicate if the server info needs to invoke downstream APIs.
downstreamEnabled: ${info.downstreamEnabled:false}
# Downstream API host.
downstreamHost: ${info.downstreamHost:http://localhost:8081}
# Downstream API server info path.
downstreamPath: ${info.downstreamPath:/adm/server/info}
```

If `enableServerInfo` is false, the endpoint will return an error `ERR10013 - SERVER_INFO_DISABLED`.

### Handler Registration

Unlike most middleware handlers, `ServerInfoGetHandler` is typically configured as a normal handler in `handler.yml` but mapped to a specific path like `/server/info`.

```yaml
handlers:
  - com.networknt.info.ServerInfoGetHandler@info

paths:
  - path: '/server/info'
    method: 'get'
    exec:
      - security
      - info
```

**Security Note:** The `/server/info` endpoint exposes sensitive configuration and system details. It **must** be protected by security handlers. In a typical light-4j deployment, this endpoint is secured requiring a special bootstrap token (e.g., from light-controller).

### Module Registration

Modules in light-4j can register themselves to be included in the server info output. This allows the info endpoint to display configuration for custom or contributed modules.

To register a module:

```java
import com.networknt.utility.ModuleRegistry;
import com.networknt.config.Config;

// Registering a module
ModuleRegistry.registerModule(
    MyHandler.class.getName(), 
    Config.getInstance().getJsonMapConfigNoCache(MyHandler.CONFIG_NAME), 
    null
);
```

### Masking Sensitive Data

Configuration files often contain sensitive data (passwords, secrets). When registering a module, you can provide a list of keys to mask in the output.

```java
List<String> masks = new ArrayList<>();
masks.add("trustPass");
masks.add("keyPass");
masks.add("clientSecret");

ModuleRegistry.registerModule(
    Client.class.getName(), 
    Config.getInstance().getJsonMapConfigNoCache(clientConfigName), 
    masks
);
```

### Downstream Info Aggregation

For components like `light-gateway`, `http-sidecar`, or `light-proxy`, the server info endpoint can be configured to aggregate information from a downstream service.

*   **`downstreamEnabled`**: When set to true, the handler will attempt to fetch server info from the configured downstream service.
*   **`downstreamHost`**: The base URL of the downstream service.
*   **`downstreamPath`**: The path to the server info endpoint on the downstream service.

If the downstream service does not implement the info endpoint, the handler usually fails gracefully or returns its own info depending on the implementation specifics (e.g., in light-proxy scenarios).

### Sample Output

The output is a JSON object structured as follows:

```json
{
  "environment": {
    "host": {
      "ip": "10.0.0.5",
      "hostname": "service-pod-1",
      "dns": "localhost"
    },
    "runtime": {
      "availableProcessors": 4,
      "freeMemory": 12345678,
      "totalMemory": 23456789,
      "maxMemory": 1234567890
    },
    "system": {
      "javaVendor": "Oracle Corporation",
      "javaVersion": "17.0.1",
      "osName": "Linux",
      "osVersion": "5.4.0",
      "userTimezone": "UTC"
    }
  },
  "specification": { ... }, 
  "component": {
      "com.networknt.info.ServerInfoConfig": {
          "enableServerInfo": true,
          "downstreamEnabled": false,
          ...
      },
      ... other modules ...
  }
}
```

# Header Handler

The `HeaderHandler` is a middleware component in the light-4j framework designed to modify request and response headers as they pass through the handler chain. This is particularly useful for cross-cutting concerns such as:

*   **Security:** Updating or removing authorization headers (e.g., converting a Bearer token to a Basic Authorization header in a proxy scenario).
*   **Privacy:** Removing sensitive internal headers before sending a response to the client.
*   **Context Propagation:** Injecting correlation IDs or other context information into headers.
*   **API Management:** Standardizing headers across different microservices.

The handler is highly configurable, allowing for global header manipulation as well as path-specific configurations.

## Configuration

The `HeaderHandler` uses a configuration file named `header.yml` to define its behavior.

### Configuration Options

The configuration supports the following sections:

1.  **`enabled`**: A boolean flag to enable or disable the handler globally. Default is `false`.
2.  **`request`**: Defines header manipulations for incoming requests.
    *   **`remove`**: A list of header names to remove from the request.
    *   **`update`**: A map of key/value pairs to add or update in the request headers. If a key exists, its value is replaced.
3.  **`response`**: Defines header manipulations for outgoing responses.
    *   **`remove`**: A list of header names to remove from the response.
    *   **`update`**: A map of key/value pairs to add or update in the response headers.
4.  **`pathPrefixHeader`**: A map where keys are URL path prefixes and values are `request` and `response` configurations specific to that path. This allows granular control over header manipulation based on the endpoint being accessed.

### Example `header.yml` (Fully Expanded)

```yaml
# Enable header handler or not, default to false
enabled: false

# Global Request header manipulation
request:
  # Remove all the headers listed here
  remove:
  - header1
  - header2
  # Add or update the header with key/value pairs
  update:
    key1: value1
    key2: value2

# Global Response header manipulation
response:
  # Remove all the headers listed here
  remove:
  - header1
  - header2
  # Add or update the header with key/value pairs
  update:
    key1: value1
    key2: value2

# Per-path header manipulation
pathPrefixHeader:
  /petstore:
    request:
      remove:
        - headerA
        - headerB
      update:
        keyA: valueA
        keyB: valueB
    response:
      remove:
        - headerC
        - headerD
      update:
        keyC: valueC
        keyD: valueD
  /market:
    request:
      remove:
        - headerE
        - headerF
      update:
        keyE: valueE
        keyF: valueF
    response:
      remove:
        - headerG
        - headerH
      update:
        keyG: valueG
        keyH: valueH
```

### Reference `header.yml` (Template)

This is the default configuration file packaged with the module (`src/main/resources/config/header.yml`). It uses placeholders that can be overridden by `values.yml` or handling external configurations.

```yaml
# Enable header handler or not. The default to false and it can be enabled in the externalized
# values.yml file. It is mostly used in the http-sidecar, light-proxy or light-router.
enabled: ${header.enabled:false}
# Request header manipulation
request:
  # Remove all the request headers listed here. The value is a list of keys.
  remove: ${header.request.remove:}
  # Add or update the header with key/value pairs. The value is a map of key and value pairs.
  # Although HTTP header supports multiple values per key, it is not supported here.
  update: ${header.request.update:}
# Response header manipulation
response:
  # Remove all the response headers listed here. The value is a list of keys.
  remove: ${header.response.remove:}
  # Add or update the header with key/value pairs. The value is a map of key and value pairs.
  # Although HTTP header supports multiple values per key, it is not supported here.
  update: ${header.response.update:}
# requestPath specific header configuration. The entire object is a map with path prefix as the
# key and request/response like above as the value. For config format, please refer to test folder.
pathPrefixHeader: ${header.pathPrefixHeader:}
```

### Configuring via `values.yml`

You can use `values.yml` to override specific properties. The properties can be defined in various formats (YAML, JSON string, comma-separated string) to suit different configuration sources (file system, config server).

```yaml
# header.yml overrides
header.enabled: true

# List of strings (JSON format)
header.request.remove: ["header1", "header2"]

# Map (JSON format)
header.request.update: {"key1": "value1", "key2": "value2"}

# List (Comma separated string)
header.response.remove: header1,header2

# Map (Comma and colon separated string)
header.response.update: key1:value1,key2:value2

# Map (YAML format)
# Note: YAML format is suitable for file-based configuration. 
# For config server usage, use a JSON string representation.
header.pathPrefixHeader:
  /petstore:
    request:
      remove:
        - headerA
        - headerB
      update:
        keyA: valueA
        keyB: valueB
    response:
      remove:
        - headerC
        - headerD
      update:
        keyC: valueC
        keyD: valueD
  /market:
    request:
      remove:
        - headerE
        - headerF
      update:
        keyE: valueE
        keyF: valueF
    response:
      remove:
        - headerG
        - headerH
      update:
        keyG: valueG
        keyH: valueH
```

## Usage

To use the `HeaderHandler` in your application:

1.  **Add the Dependency:** Ensure the `header` module is included in your project's `pom.xml`.
2.  **Register the Handler:** Add `com.networknt.header.HeaderHandler` to your middleware chain in `handler.yml`.
3.  **Configure:** Provide a `header.yml` or configured `values.yml` in your `src/main/resources/config` folder (or external config directory) to define the desired header manipulations.

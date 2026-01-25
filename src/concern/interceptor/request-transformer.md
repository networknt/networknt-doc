# Request Transformer Interceptor

The `request-transformer` is a powerful middleware interceptor that allows for dynamic modification of incoming requests using the `light-4j` rule engine. It provides a highly flexible way to manipulate request metadata (headers, query parameters, path) and the request body based on custom business logic defined in rules.

### Features

- **Dynamic Transformation**: Modify request elements at runtime without changing application code.
- **Rule-Based**: Leverage the `light-4j` rule engine to define complex transformation logic.
- **Path-Based Activation**: Target specific request paths using the `appliedPathPrefixes` configuration.
- **Metadata Manipulation**: Add, update, or remove request headers, query parameters, path, and URI.
- **Body Transformation**: Overwrite the request body (supports JSON, XML, and other text-based formats).
- **Short-Circuiting**: Generate an immediate response body or validation error to return to the caller, stopping the handler chain.
- **Encoding Support**: Configurable body encoding per path prefix for legacy API compatibility.
- **Auto-Registration**: Automatically registers with the `ModuleRegistry` for administrative visibility.

### Configuration (request-transformer.yml)

The interceptor is configured via `request-transformer.yml`.

```yaml
# Request Transformer Configuration
---
# Indicate if this interceptor is enabled or not. Default is true.
enabled: ${request-transformer.enabled:true}

# Indicate if the transform interceptor needs to change the request body. Default is true.
requiredContent: ${request-transformer.requiredContent:true}

# Default body encoding for the request body. Default is UTF-8.
defaultBodyEncoding: ${request-transformer.defaultBodyEncoding:UTF-8}

# A list of applied request path prefixes. Only requests matching these prefixes will be processed.
# This can be a single string or a list of strings.
appliedPathPrefixes: ${request-transformer.appliedPathPrefixes:}

# Customized encoding for specific path prefixes.
# This is useful for legacy APIs that require non-UTF-8 encoding (e.g., ISO-8859-1).
# pathPrefixEncoding:
#   /v1/pets: ISO-8859-1
#   /v1/party/info: ISO-8859-1
```

### Setup

#### 1. Add Dependency
Add the following dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>request-transformer</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

#### 2. Register Interceptor
In your `handler.yml`, add the `RequestTransformerInterceptor` to the interceptors list and include it in the appropriate handler chains.

```yaml
handlers:
  - com.networknt.reqtrans.RequestTransformerInterceptor@requestTransformer

chains:
  default:
    - ...
    - requestTransformer
    - ...
```

### Transformation Rules

The interceptor passes a context map (`objMap`) to the rule engine. Your rules can inspect these values and return a result map to trigger specific modifications.

#### Rule Input Map (`objMap`)
- `auditInfo`: Map of audit information (e.g., clientId, endpoint).
- `requestHeaders`: Map of current request headers.
- `queryParameters`: Map of current query parameters.
- `pathParameters`: Map of current path parameters.
- `method`: HTTP method (e.g., "POST", "GET").
- `requestURL`: The full request URL.
- `requestURI`: The request URI.
- `requestPath`: The request path.
- `requestBody`: The request body string (only if `requiredContent` is true).

#### Rule Output Result Map
To perform a transformation, your rule must return a map containing one or more of the following keys:

- **`requestPath`**: `String` - Overwrites the exchange request path.
- **`requestURI`**: `String` - Overwrites the exchange request URI.
- **`queryString`**: `String` - Overwrites the exchange query string.
- **`requestHeaders`**: `Map` - Contains two sub-keys:
    - `remove`: `List<String>` - List of header names to remove.
    - `update`: `Map<String, String>` - Map of header names and values to add or update.
- **`requestBody`**: `String` - Overwrites the request body buffer with the provided string.
- **`responseBody`**: `String` - Immediately returns this string as the response body, bypassing the rest of the chain.
    - Also supports `statusCode` (Integer) and `contentType` (String) in the result map.
- **`validationError`**: `Map` - Short-circuits the request with a validation error.
    - Requires `errorMessage` (String), `contentType` (String), and `statusCode` (Integer).

### Example Transformation Logic

If a rule decides to update a header and the request path, the result map returned from the rule engine might look like this:

```json
{
  "result": true,
  "requestPath": "/v2/new-endpoint",
  "requestHeaders": {
    "update": {
      "X-Transformation-Status": "transformed"
    },
    "remove": ["X-Old-Header"]
  }
}
```

### Operational Visibility

The `request-transformer` module automatically registers itself with the `ModuleRegistry` during configuration loading. You can inspect its current status and configuration parameters at runtime via the [Server Info](/concern/admin/server-info.md) endpoint.

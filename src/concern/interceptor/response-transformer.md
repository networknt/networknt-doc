# Response Transformer Interceptor

The `response-transformer` is a powerful middleware interceptor that allows for dynamic modification of outgoing responses using the `light-4j` rule engine. It provides a highly flexible way to manipulate response headers and the response body based on custom business logic defined in rules.

### Features

- **Dynamic Transformation**: Modify response elements at runtime without changing application code.
- **Rule-Based**: Leverage the `light-4j` rule engine to define complex transformation logic.
- **Path-Based Activation**: Target specific request paths using the `appliedPathPrefixes` configuration.
- **Header Manipulation**: Add, update, or remove response headers.
- **Body Transformation**: Overwrite the response body (supports JSON, XML, and other text-based formats).
- **Encoding Support**: Configurable body encoding per path prefix for legacy API compatibility.
- **Auto-Registration**: Automatically registers with the `ModuleRegistry` during configuration loading for administrative visibility.

### Configuration (response-transformer.yml)

The interceptor is configured via `response-transformer.yml`.

```yaml
# Response Transformer Configuration
---
# Indicate if this interceptor is enabled or not. Default is true.
enabled: ${response-transformer.enabled:true}

# Indicate if the transform interceptor needs to change the response body. Default is true.
requiredContent: ${response-transformer.requiredContent:true}

# Default body encoding for the response body. Default is UTF-8.
defaultBodyEncoding: ${response-transformer.defaultBodyEncoding:UTF-8}

# A list of applied request path prefixes. Only requests matching these prefixes will be processed.
appliedPathPrefixes: ${response-transformer.appliedPathPrefixes:}

# For certain path prefixes that are not using the defaultBodyEncoding UTF-8, you can define the customized
# encoding like ISO-8859-1 for the path prefixes here.
# pathPrefixEncoding:
#   /v1/pets: ISO-8859-1
#   /v1/party/info: ISO-8859-1
```

### Setup

#### 1. Add Dependency
Include the following dependency in your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>response-transformer</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

#### 2. Register Interceptor
In your `handler.yml`, add the `ResponseTransformerInterceptor` to the interceptors list and include it in the appropriate handler chains.

```yaml
handlers:
  - com.networknt.restrans.ResponseTransformerInterceptor@responseTransformer

chains:
  default:
    - ...
    - responseTransformer
    - ...
```

### Transformation Rules

The interceptor passes a context map (`objMap`) to the rule engine. Your rules can inspect these values and return a result map to trigger specific modifications.

#### Rule Input Map (`objMap`)
- `auditInfo`: Map of audit information (e.g., clientId, endpoint).
- `requestHeaders`: Map of current request headers.
- `responseHeaders`: Map of current response headers.
- `queryParameters`: Map of current query parameters.
- `pathParameters`: Map of current path parameters.
- `method`: HTTP method (e.g., "POST", "GET").
- `requestURL`: The full request URL.
- `requestURI`: The request URI.
- `requestPath`: The request path.
- `requestBody`: The request body (if available in the exchange attachment).
- `responseBody`: The original response body string (only if `requiredContent` is true).
- `statusCode`: The HTTP status code of the response.

#### Rule Output Result Map
To perform a transformation, your rule must return a map containing one or more of the following keys:

- **`responseHeaders`**: `Map` - Contains two sub-keys:
    - `remove`: `List<String>` - List of header names to remove.
    - `update`: `Map<String, String>` - Map of header names and values to add or update.
- **`responseBody`**: `String` - Overwrites the response body buffer with the provided string.

### Example Transformation Logic

If a rule decides to update a header and the response body, the result map returned from the rule engine might look like this:

```json
{
  "result": true,
  "responseHeaders": {
    "update": {
      "X-Transformation-Version": "2.0"
    },
    "remove": ["Server"]
  },
  "responseBody": "{\"status\":\"success\", \"data\": \"transformed content\"}"
}
```

### Operational Visibility

The `response-transformer` module automatically registers itself with the `ModuleRegistry`. This allows administrators to inspect the active configuration and status of the interceptor at runtime via the [Server Info](/concern/admin/server-info.md) endpoint.

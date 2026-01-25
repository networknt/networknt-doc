# Data Mask

In a production environment, various logging statements are written to log files or persistent storage to assist in identifying and resolving issues. Since a broad group of people might have access to these logs, sensitive information such as credit card numbers, SIN numbers, or passwords must be masked before logging for confidentiality and compliance.

The `mask` module provides a utility to handle these masking requirements across different data formats.

## StartupHookProvider

The mask module depends on [JsonPath](https://github.com/json-path/JsonPath), which allows for easy access to nested elements in JSON strings. To ensure consistency and performance, you should configure JsonPath to use the Jackson parser (which is standard in light-4j) instead of the default `json-smart` parser.

The `light-4j` framework provides `JsonPathStartupHookProvider` for this purpose. You can enable it by updating your `service.yml`:

```yaml
singletons:
- com.networknt.server.StartupHookProvider:
    - com.networknt.server.JsonPathStartupHookProvider
```

When using `light-codegen`, this provider is often included in the generated `service.yml` but commented out by default.

## Configuration (mask.yml)

The masking behavior is fully configurable via `mask.yml`. It supports four sections: `string`, `regex`, `json`, and `map`.

### Example mask.yml

```yaml
---
# Replacement rules for specific keys. 
# Key maps to a map of "regex-to-find": "replacement-string"
string:
  uri:
    "password=[^&]*": "password=******"

# Masking based on regex groups. 
# Matching groups in the value will be replaced by stars (*) of the same length.
regex:
  header:
    Authorization: "^Bearer\\s+(.*)$"

# Masking based on JSON Path. 
# Key maps to a map of "json-path": "mask-expression"
json:
  user:
    "$.password": ""
    "$.creditCard.number": "^(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})$"
```

## Usage

The `com.networknt.mask.Mask` utility class provides static methods to apply masking based on the configuration above.

### 1. Mask with String
Used for simple replacements, typically in URIs or query parameters.

```java
// Uses the 'uri' key from the 'string' section in mask.yml
String maskedUri = Mask.maskString("https://localhost?user=admin&password=123", "uri");
// Result: https://localhost?user=admin&password=******
```

### 2. Mask with Regex
Replaces the content of matching groups with the `*` character. This is useful for headers or cookies where you want to preserve the surrounding structure.

```java
// Uses the 'header' key and 'Authorization' name from the 'regex' section
String input = "Bearer eyJhbGciOiJIUzI1...";
String masked = Mask.maskRegex(input, "header", "Authorization");
// Result: Bearer ****************...
```

### 3. Mask with JsonPath
The most powerful way to mask JSON data. It supports single values, nested objects, and arrays.

```java
// Uses the 'user' key from the 'json' section
String json = "{\"username\":\"admin\", \"password\":\"secret123\"}";
String maskedJson = Mask.maskJson(json, "user");
// Result: {"username":"admin", "password":"*********"}
```

#### Masking Lists/Arrays
The JSON masking logic automatically handles arrays if the JSON Path targets an array element (e.g., `$.items[*].id`). It ensures that only the values are masked while preserving the array structure.

## Dependency

Add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>mask</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

## Auto-Registration

The `mask` module registers itself with the `ModuleRegistry` automatically when the configuration is loaded. You can verify the loaded masking rules through the [Server Info](/concern/admin/server-info.md) endpoint.

# OpenAPI Meta

The `OpenApiHandler` is a core middleware handler in the `light-rest-4j` framework. It is responsible for parsing the OpenAPI specification, matching the incoming request to a specific operation defined in the spec, and attaching that metadata to the request context.

## Overview

The `OpenApiHandler` acts as the "brain" for RESTful services. By identifying the exact endpoint and operation (e.g., `GET /pets/{petId}`) at the start of the request chain, it enables subsequent handlers—such as Security, Validator, and Metrics—to perform their tasks efficiently without re-parsing the request or the specification.

### Key Features
- **Operation Identification**: Matches URI and HTTP method to OpenAPI operations.
- **Support for Multiple Specs**: Can host multiple OpenAPI specifications in a single instance using path-based routing.
- **Parameter Deserialization**: Correctly parses query, path, header, and cookie parameters according to the OpenAPI 3.0 styles.
- **Specification Injection**: Supports merging a common "inject" specification (e.g., for administrative endpoints) into the main spec.
- **Hot Reload**: Automatically detects and applies changes to its configuration without downtime.

## Configuration (openapi-handler.yml)

The handler's behavior is governed by the `openapi-handler.yml` file, which maps to `OpenApiHandlerConfig`.

```yaml
# OpenAPI Handler Configuration
---
# An indicator to allow multiple openapi specifications.
# Default to false which only allow one spec named openapi.yml/yaml/json.
multipleSpec: ${openapi-handler.multipleSpec:false}

# To allow the call to pass through the handler even if the path is not found in the spec.
# Useful for gateways where some APIs might not have specs deployed.
ignoreInvalidPath: ${openapi-handler.ignoreInvalidPath:false}

# Path to spec mapping. Required if multipleSpec is true.
# The key is the base path and the value is the specification name.
pathSpecMapping:
  v1: openapi-v1
  v2: openapi-v2
```

### Configuration Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `multipleSpec` | `false` | When `true`, the handler uses `pathSpecMapping` to support multiple APIs. |
| `ignoreInvalidPath` | `false` | If `true`, requests with no matching path in the spec proceed to the next handler instead of returning an error. |
| `pathSpecMapping` | `{}` | A map where keys are URI base paths and values are the names of the corresponding spec files. |

## How it Works

### 1. Specification Loading
At startup, the handler loads the specification(s). If `multipleSpec` is disabled, it looks for `openapi.yml`. If enabled, it loads all specs defined in `pathSpecMapping`. It also checks for `openapi-inject.yml` to merge common definitions.

### 2. Request Matching
For every incoming request, the handler:
1. Normalizes the request path.
2. If `multipleSpec` is on, it determines which specification to use based on the path prefix.
3. Finds the matching template in the OpenAPI paths (e.g., matching `/pets/123` to `/pets/{petId}`).
4. Resolves the HTTP method to the specific `Operation` object.

### 3. Parameter Deserialization
The handler uses the `ParameterDeserializer` to extract values from the request according to the styles defined in the OpenAPI spec (e.g., `matrix`, `label`, `form`). These deserialized values are attached to the exchange using specific attachment keys.

### 4. Audit Info Integration
The handler attaches the following to the `auditInfo` object:
- `endpoint`: The resolved endpoint string (e.g., `/pets/{petId}@get`).
- `openapi_operation`: The full `OpenApiOperation` object, containing the path, method, and operation metadata.

## Hot Reload

The `OpenApiHandler` supports hot-reloading through a thread-safe check in its `handleRequest` method.
- It leverages `OpenApiHandlerConfig.load()` which returns a cached singleton.
- If the configuration is updated (e.g., via a config server or manual trigger), the handler detects the change, re-initializes its internal `OpenApiHelper` state, and builds the new mapping logic immediately.

## Performance Optimization
For the most common use case (99%) where a service has only one specification, the handler bypasses the mapping logic and uses a high-performance direct reference to the `OpenApiHelper`.
